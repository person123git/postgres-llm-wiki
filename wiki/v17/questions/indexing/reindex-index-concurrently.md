---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: not yet
---

# How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [How it differs from CREATE INDEX CONCURRENTLY](#how-it-differs-from-create-index-concurrently)
  - [Dispatch, preconditions, and restrictions](#dispatch-preconditions-and-restrictions)
  - [The six phases](#the-six-phases)
  - [GUCs that affect REINDEX INDEX CONCURRENTLY performance](#gucs-that-affect-reindex-index-concurrently-performance)
  - [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table)
  - [State flags for the old and new index](#state-flags-for-the-old-and-new-index)
  - [Failure scenarios and the outcome on the table](#failure-scenarios-and-the-outcome-on-the-table)
  - [Can a failure leave an invalid index with the original index name?](#can-a-failure-leave-an-invalid-index-with-the-original-index-name)
  - [Watching the phases](#watching-the-phases)
  - [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)
  - [Test coverage](#test-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

give a comprehensive explanation of how REINDEX INDEX CONCURRENTLY is
implemented in PostgreSQL 17, including the steps, locks, and failure scenarios,
what changed since PostgreSQL 12

(The prompt wording was corrected for grammar and capitalization at the user's
request — "what change since postgresql 12" → "what changed since PostgreSQL 12";
meaning preserved.)

Follow-up (2026-06-12): Can a failure leave an invalid index with the original
index name? Confirm whether `REINDEX INDEX CONCURRENTLY index_name;` can, on
failure, leave the table with an index called `index_name` that is marked invalid
— so query plans can no longer use it. Before, the index was only bloated; after
such a failure the table would effectively have no usable index.

(The follow-up wording was corrected from the original prompt for grammar at the
user's request; meaning preserved.)

Follow-up (2026-07-17): What GUCs have a performance impact on it?

(The follow-up wording was corrected from "what GUCs have performance impact on
it." for grammar and capitalization at the user's request; meaning preserved.)

## Answer

`REINDEX INDEX CONCURRENTLY` (RIC) rebuilds an index without ever blocking
`INSERT`/`UPDATE`/`DELETE`, and without leaving the index unavailable to queries
in between. It builds a **brand-new copy** of the index next to the original,
catches that copy up to concurrent writes, atomically **swaps** the copy in for
the original, and only then marks the original dead and drops it. The build and
catch-up reuse the exact `CREATE INDEX CONCURRENTLY` (CIC) machinery; the
swap-and-drop tail is what makes a reindex.

All of it runs in `ReindexRelationConcurrently()`, which lays out **six phases**
across many short transactions
([indexcmds.c#ReindexRelationConcurrently](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3437-L4295)):

1. create the new index copies in the catalog (not built),
2. build each new copy,
3. let each new copy catch up and validate it,
4. swap each new copy in for its old index (new becomes valid, old invalid),
5. mark the old indexes dead, and
6. drop the old indexes.

The table lock used throughout is **`ShareUpdateExclusiveLock`** — the same level
CIC, `VACUUM` (non-FULL), and `ANALYZE` use, weak enough to let DML proceed but
strong enough to keep out a second concurrent build, `VACUUM`, `ANALYZE`, and
schema changes
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)).

The algorithm is the same six-phase shape PostgreSQL 12 used, but several things
changed since v12 — most visibly, RIC now works on **partitioned** indexes and
tables, can move indexes to a new **tablespace**, advertises `PROC_IN_SAFE_IC`
so other concurrent builds can ignore it, makes its `pg_index` flag writes
**transactional**, and fixes a progress-view phase bug. See
[What changed from PostgreSQL 12](#what-changed-from-postgresql-12).

### How it differs from CREATE INDEX CONCURRENTLY

RIC is CIC plus a swap and a drop. The build (`index_concurrently_build`) and the
validation (`validate_index`) are literally the same functions CIC uses; see the
[v17 CIC page](create-index-concurrently.md) for that shared core. The
differences:

| Aspect | CREATE INDEX CONCURRENTLY | REINDEX INDEX CONCURRENTLY |
|---|---|---|
| Target | a new index that did not exist | a fresh copy `<name>_ccnew` of an existing index |
| Phases / waits | build, validate, mark valid (3 waits) | create copy, build, validate, swap, set-dead, drop (**5 waits**) |
| Extra waits | none beyond the snapshot wait | two `AccessExclusiveLock` waits that **wait out readers** of the old index before set-dead and drop ([indexcmds.c:4180-4216](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4216)) |
| Names | the index keeps its name | new copy takes the original's name; the old index is renamed `<name>_ccold` ([index.c#swap-names](../../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1593), [indexcmds.c#ccold](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4124-L4129)) |
| On failure | leaves the **target** index invalid | leaves an invalid `_ccnew` (before swap) or `_ccold` (after swap); a healthy original is preserved until the swap |

The "wait out readers" difference is the operationally important one. CIC's two
lock-based waits use `ShareLock` conflict checks and do not wait for
`AccessShareLock` readers (its old-snapshot wait can still wait for a plain
`SELECT` holding an old snapshot). RIC adds two later `AccessExclusiveLock`
conflict checks on the heap lock tag, so current table lock holders — including
`AccessShareLock` readers — can delay marking the old index dead and dropping it
([lock.c#AccessExclusive-conflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102),
[reindex.sgml#concurrent-steps](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L382-L443)).

### Dispatch, preconditions, and restrictions

`REINDEX` is dispatched from `standard_ProcessUtility` into `ExecReindex`, which
parses the `CONCURRENTLY`, `VERBOSE`, and `TABLESPACE` options into a
`ReindexParams` bitmask and then routes by object kind
([utility.c:1566-1567](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1566-L1567),
[indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781)).
Because the concurrent path commits many times, it cannot run inside a
transaction block:

```c
if (concurrently)
    PreventInTransactionBlock(isTopLevel, "REINDEX CONCURRENTLY");
```

([indexcmds.c:2720-2722](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722)).
`REINDEX` is also forbidden during recovery, but in v17 that is enforced
generically: `ClassifyUtilityCommandAsReadOnly` returns only
`COMMAND_OK_IN_READ_ONLY_TXN` (not `COMMAND_OK_IN_RECOVERY`) for a `ReindexStmt`,
so `standard_ProcessUtility` calls `PreventCommandDuringRecovery`
([utility.c#classify-reindex](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296),
[utility.c:581-582](../../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582)).

`REINDEX INDEX` routes through `ReindexIndex`, which locks the index in
`ShareUpdateExclusiveLock` for the concurrent case (vs `AccessExclusiveLock` for
a plain reindex). It then dispatches in three ways
([indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2788-L2834)):

- a **partitioned** index goes to `ReindexPartitions` (new since v12);
- a concurrent, non-temporary index goes to `ReindexRelationConcurrently`;
- otherwise (including a temporary index, even if `CONCURRENTLY` was requested) it
  falls back to the non-concurrent `reindex_index`.

`REINDEX TABLE` behaves the same way via `ReindexTable`, emitting
`table "..." has no indexes that can be reindexed concurrently` when nothing
qualifies
([indexcmds.c#ReindexTable](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2918-L2966)).

The full restriction set, with where each is enforced:

| Restriction | Behavior | Where enforced |
|---|---|---|
| Inside a transaction block | error: `REINDEX CONCURRENTLY cannot run inside a transaction block` | [indexcmds.c:2720-2722](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722) |
| During recovery (standby) | error: `cannot execute REINDEX during recovery` | [utility.c:280-296](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296), [utility.c:581-582](../../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582) |
| Temporary table/index | falls back to a **non-concurrent** reindex | [indexcmds.c:2824-2833](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2824-L2833), [indexcmds.c:2940-2958](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2940-L2958) |
| System catalog target, or `REINDEX SYSTEM CONCURRENTLY` | error: `cannot reindex system catalogs concurrently` | [indexcmds.c:3516-3519](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3516-L3519), [indexcmds.c:3646-3649](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3646-L3649) |
| Partitioned table or index | **supported** — recurses into leaf partitions | [indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301) |
| Exclusion-constraint index named directly | error: `concurrent index creation for exclusion constraints is not supported` | [index.c:1328-1331](../../../../raw/postgres-17/src/backend/catalog/index.c#L1328-L1331) |
| Exclusion-constraint index reached via a table | warning, skipped | [indexcmds.c:3555-3560](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3555-L3560) |
| Invalid index reached via a table | warning, skipped | [indexcmds.c:3548-3554](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3548-L3554) |
| Invalid index named directly | **allowed** (this is how you repair one) | [indexcmds.c:3695-3701](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3695-L3701) |
| Invalid index on a TOAST table named directly | error: `cannot reindex invalid index on TOAST table` | [indexcmds.c:3656-3660](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3656-L3660) |

When the target is a table, matview, or toast relation, RIC gathers every valid,
non-exclusion index plus the relation's **toast** indexes and processes them all
together; when the target is an index, it processes just that one
([indexcmds.c#gather-indexes](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3496-L3716)).
A directly named partitioned table/index never reaches the concurrent core —
`ReindexRelationConcurrently` errors with `cannot reindex this type of relation
concurrently` if one ever did
([indexcmds.c:3708-3714](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3708-L3714)).

For a **partitioned** target, `ReindexPartitions` re-checks the transaction-block
rule, collects the physical leaf partitions with `find_all_inheritors` under
`ShareLock`, and hands them to `ReindexMultipleInternal`, which reindexes each
leaf **in its own transaction**; a concurrent run calls
`ReindexRelationConcurrently` per leaf with `REINDEXOPT_MISSING_OK`
([indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301),
[indexcmds.c#ReindexMultipleInternal](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3311-L3410)).

### The six phases

The phase comment in the source is the canonical map
([indexcmds.c#phase-overview](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3737-L3751)).
Each phase loops over **all** indexes before the next phase begins.

#### Phase 1: create the catalog copies and take session locks

For each index, RIC switches to the table owner's userid and calls
`RestrictSearchPath()` (so index/predicate functions run safely), determines
whether the index is `safe` (no expressions, no partial predicate), chooses the
temporary name `<orig>_ccnew`, picks the target tablespace, and calls
`index_concurrently_create_copy`
([indexcmds.c:3786-3836](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3786-L3836)).
That helper rebuilds the new index's `IndexInfo` from the old index's catalog
rows and creates the catalog entry with
`INDEX_CREATE_SKIP_BUILD | INDEX_CREATE_CONCURRENT` — so it has identity but no
data and is **not ready, not valid**; exclusion constraints are rejected here
([index.c#index_concurrently_create_copy](../../../../raw/postgres-17/src/backend/catalog/index.c#L1298-L1469),
[index.c:1442-1462](../../../../raw/postgres-17/src/backend/catalog/index.c#L1442-L1462)).

RIC then takes a **session-level** `ShareUpdateExclusiveLock` on the old index,
the new index, and the heap table(s) (including toast), saving a `LOCKTAG` per
heap for the later waits, and commits
([indexcmds.c#index-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3863-L3868),
[indexcmds.c#heap-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3902-L3934)).
These session locks survive the upcoming commits so nobody can drop the relations
mid-rebuild. No `PROC_IN_SAFE_IC` flag is set here because this transaction takes
no snapshot
([indexcmds.c:3940-3943](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3940-L3943)).

#### Phase 2: wait, then build each new index

**Wait 1** is `WaitForLockersMultiple(lockTags, ShareLock, true)`: it waits out
the current transactions holding locks that conflict with `ShareLock`, so no
running writer can still have the table open without the new index in its index
list
([indexcmds.c:3955-3957](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3955-L3957)).
Then, in a **separate transaction per index**, RIC advertises `PROC_IN_SAFE_IC`
if the index is safe, takes a fresh snapshot, and calls
`index_concurrently_build`, which scans the heap, builds the new copy, and sets
`indisready = true`
([indexcmds.c:3960-3997](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3960-L3997),
[index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1540)).
This is the same build CIC uses.

#### Phase 3: wait, validate, then wait for old snapshots

**Wait 2** is another `WaitForLockersMultiple(lockTags, ShareLock, true)`, so
every current `ShareLock`-conflicting transaction finishes before validation
assumes new writers see the new copy as ready
([indexcmds.c:4014-4016](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4014-L4016)).
Then, per new index, RIC registers a **reference snapshot**, runs `validate_index`
to backfill any tuples the build scan missed, saves the snapshot's `xmin` as
`limitXmin`, commits, and in a fresh transaction does **Wait 3** =
`WaitForOlderSnapshots(limitXmin, true)` to wait out transactions whose snapshot
predates the reference snapshot
([indexcmds.c:4019-4088](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4019-L4088)).
At the end of phase 3 the new copy contains every interesting tuple but is
**still invalid** — its `indisvalid` is not flipped here.

#### Phase 4: swap each old index with its new copy

In a single transaction (which sets `PROC_IN_SAFE_IC` unconditionally because it
only manipulates the catalog), RIC chooses a name `<orig>_ccold` for the old
index, calls `index_concurrently_swap`, invalidates the table's relcache, and
issues a `CommandCounterIncrement` so the next index in the loop sees the new
names
([indexcmds.c:4102-4163](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4163)).
`index_concurrently_swap` does the heavy lifting in one transaction
([index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1549-L1809)):

- swaps the `relname`s — the new copy takes the original's name, the old index
  becomes `<orig>_ccold` — and swaps `relispartition`
  ([index.c:1591-1601](../../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1601));
- copies the constraint flags `indisprimary`, `indisexclusion`, `indimmediate`
  from old to new, and preserves `indisreplident` / `indisclustered` on the new
  ([index.c:1625-1636](../../../../raw/postgres-17/src/backend/catalog/index.c#L1625-L1636));
- marks the **new index valid and the old index invalid** at the same time via the
  **transactional** `CatalogTupleUpdate`, clearing `indisclustered` /
  `indisreplident` on the old
  ([index.c:1642-1648](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1648));
- moves constraints and their triggers, the comment, the partition inheritance
  link, and all dependencies from the old index to the new
  ([index.c:1653-1792](../../../../raw/postgres-17/src/backend/catalog/index.c#L1653-L1792));
- copies the cumulative statistics and the planner column statistics from old to
  new via `pgstat_copy_relation_stats` and `CopyStatistics`
  ([index.c:1794-1798](../../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798)).

The relcache invalidation makes every session re-plan against the rebuilt index
after this commit.

#### Phase 5: wait for readers, then mark old indexes dead

**Wait 4** is `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c:4180-4182](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4182)).
Because `AccessExclusiveLock` conflicts with every lock mode including
`AccessShareLock`, this waits out current table lock holders, including
**readers**. The wait is conservative: a reader need not actually use the old
index to be in the wait set, because the wait is on the heap relation lock tag
([lock.c#AccessExclusive-conflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)).
Then `index_concurrently_set_dead` transfers predicate locks to the heap and sets
`indisready = false, indislive = false` (dead) on each old index
([indexcmds.c:4184-4196](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4184-L4196),
[index.c#index_concurrently_set_dead](../../../../raw/postgres-17/src/backend/catalog/index.c#L1820-L1854)).

#### Phase 6: wait for readers, then drop old indexes

After another `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c:4214-4216](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4216)),
RIC drops the old indexes with
`performMultipleDeletions(objects, DROP_RESTRICT, PERFORM_DELETION_CONCURRENT_LOCK | PERFORM_DELETION_INTERNAL)`
([indexcmds.c:4218-4244](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4218-L4244)).
`PERFORM_DELETION_CONCURRENT_LOCK` becomes `concurrent_lock_mode` in the
dependency deletion path, so `index_drop` takes `ShareUpdateExclusiveLock` (not
`AccessExclusiveLock`) on the heap and index while **skipping** its own
`if (concurrent)` set-dead/two-wait branch — RIC already did the dead-marking and
reader waits itself
([dependency.c#doDeletion-index](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1356-L1368),
[index.c#index_drop-lockmode](../../../../raw/postgres-17/src/backend/catalog/index.c#L2154-L2196)).
Finally RIC releases all the session locks
([indexcmds.c:4249-4254](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4249-L4254)).

### GUCs that affect REINDEX INDEX CONCURRENTLY performance

For a typical B-tree RIC, examine `maintenance_work_mem`,
`max_parallel_maintenance_workers`, `min_parallel_table_scan_size`,
`max_parallel_workers`, and `max_worker_processes` first. The first setting
controls build and validation memory. The other four determine whether the first
heap scan requests parallel workers and whether those workers can launch. Only
B-tree and BRIN support parallel builds in v17
([index.c#parallel-AM-boundary](../../../../raw/postgres-17/src/backend/catalog/index.c#L2966-L2976),
[planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7018)).

RIC reuses the CIC build and validation functions, but it has more commit and
wait exposure. Within one `ReindexRelationConcurrently` invocation over `N`
indexes, the source has six fixed internal commits plus three per index: one
after each build, one after each validation scan, and one after each old-snapshot
wait. A one-index RIC therefore executes nine internal commits, then starts the
transaction that the utility-command caller finishes. It also has five blocker
waits, including two reader waits that CIC lacks
([indexcmds.c#build-and-validation-commits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3936-L4088),
[indexcmds.c#swap-set-dead-drop-commits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4257),
[postgres.c#command-end-commit](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2807)).

This section uses a bounded meaning of “performance impact”: the setting must
change a core RIC phase, a shipped access method, worker availability, heap or
temporary I/O, WAL/commit latency, or a command wait. Index expressions and
third-party access methods can execute arbitrary code and read additional GUCs;
the core dispatch invokes the access method's `ambuild` callback, so no
core-source list can cover those additions
([amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L98-L108),
[index.c#ambuild-dispatch](../../../../raw/postgres-17/src/backend/catalog/index.c#L3020-L3025)).

**Application scope used below.** “Session” means a `PGC_USERSET` setting, or a
`PGC_SUSET` setting changed by an authorized role, with no reload or restart.
Although those contexts can also be transaction-local, a session value is the
usable per-run scope: RIC cannot run inside an explicit transaction, and its own
commits end transaction-local values. Parallel workers restore the leader's
serialized GUC state
([indexcmds.c#transaction-block-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722),
[ref/set.sgml#SET-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L35-L60),
[ref/set.sgml#LOCAL](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L100-L117),
[parallel.c#GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L376-L385),
[parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1449-L1458)).

**Build, scan, and storage settings.**

| GUC | Exact RIC effect and boundary | Default and application |
|---|---|---|
| `maintenance_work_mem` | The main direct control. Phase 2 passes through the AM's build algorithm; phase 3 uses this setting for the serial encoded-TID validation sort. The automatic B-tree/BRIN worker request also reserves at least 32 MB per participant, including the leader ([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1516-L1522), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3361-L3405), [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)). | 64 MB; session; no reload or restart ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2459-L2473)). |
| `max_parallel_maintenance_workers` | Caps the workers requested for a B-tree or BRIN first build. Zero disables parallel utility workers. It is a request cap, not a guarantee that workers launch ([planner.c#worker-request-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L7018)). | 2; session; no reload or restart ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3408-L3416)). |
| `min_parallel_table_scan_size` | The automatic model rejects a heap below this threshold, then adds workers as heap size crosses successive threefold thresholds. A table `parallel_workers` reloption bypasses both this size model and the 32 MB request test, but remains capped by `max_parallel_maintenance_workers` ([allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4278), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L7012)). | 8 MB; session; no reload or restart ([guc_tables.c#parallel-scan-thresholds](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3519-L3538)). |
| `max_parallel_workers`; `max_worker_processes` | The first caps live parallel workers when RIC registers them. The second sizes the shared background-worker slot array. Pool pressure can launch fewer workers than requested; B-tree and BRIN then continue with the participants that launched or fall back to serial execution ([bgworker.c#worker-slot-pool](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L146-L175), [bgworker.c#parallel-and-slot-caps](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L969-L1043), [nbtsort.c#parallel-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594)). | Both default to 8. `max_parallel_workers` is session-scoped; `max_worker_processes` is `postmaster` context and needs restart ([guc_tables.c#max_worker_processes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3162-L3171), [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3429-L3437)). |
| `effective_cache_size` | Only GiST's buffered build reads it. Together with `maintenance_work_mem`, it limits the buffered subtree depth; it does not reserve cache or control the other shipped AMs ([gistbuild.c#gistInitBuffering](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L757)). | 524,288 blocks, normally 4 GB at 8 kB per block; session; no reload or restart ([cost.h#DEFAULT_EFFECTIVE_CACHE_SIZE](../../../../raw/postgres-17/src/include/optimizer/cost.h#L31-L35), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3507-L3517)). |
| `shared_buffers` | Its `NBuffers` value changes two decisions: each heap scan gets the bulk-read path only when the heap exceeds `NBuffers / 4`, and a permanent hash-index build caps its sort-selection threshold at `NBuffers`. More shared buffers can therefore move either threshold; it is not a monotonic RIC speed control ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L366-L429), [hash.c#hashbuild-sort-choice](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)). | 16,384 blocks, normally 128 MB; `postmaster` context; restart ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2256-L2269)). |
| `io_combine_limit` | Both heap scans use v17's sequential read stream. This setting caps adjacent blocks combined into one read, and each scan captures it when its stream starts ([heapam.c#sequential-read-stream](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1166-L1192), [read_stream.c#combined-read-limit](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L360-L376), [read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L507-L522)). | Normally 128 kB on an 8 kB build, subject to the platform maximum; session; no reload or restart ([guc_tables.c#io_combine_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3137-L3149), [bufmgr.h#DEFAULT_IO_COMBINE_LIMIT](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L170)). |
| `effective_io_concurrency` | The heap stream selects the heap tablespace's effective-I/O value because the heap AM passes `READ_STREAM_SEQUENTIAL`, not `READ_STREAM_MAINTENANCE`. Sequential mode disables explicit prefetch advice and drives look-ahead toward `io_combine_limit`, so this is a secondary source-visible boundary, not a primary RIC lever ([read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L407-L463), [read_stream.c#sequential-advice-boundary](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L493-L521), [read_stream.c#sequential-distance](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L689-L713)). | 1 when prefetch support is compiled in, otherwise 0; session; no reload or restart. A tablespace reloption overrides the GUC ([bufmgr.h#I/O-concurrency-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L166), [guc_tables.c#effective_io_concurrency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3108-L3120), [spccache.c#tablespace-I/O-override](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L207-L223)). |
| `synchronize_seqscans` | On a sufficiently large heap, the first scan can start at another synchronized scan's location and wrap around; it still reads every page. B-tree, hash, GiST, SP-GiST, Bloom, and parallel BRIN permit this. GIN and serial BRIN require physical order and disable it. Validation always disables it ([heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L400-L429), [gininsert.c#no-syncscan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L378-L384), [brin.c#serial-scan-order](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1217-L1224), [heapam_handler.c#validation-order](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1805)). | On; session; no reload or restart ([guc_tables.c#synchronize_seqscans](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1764-L1773)). |
| `temp_tablespaces`; `temp_file_limit` | `temp_tablespaces` places tuplesort spills, parallel worker tapes, and GiST buffered-build files. `temp_file_limit` does not throttle RIC: each process errors when its accounted temporary files cross the limit. Such an error occurs before the swap and can leave an invalid `_ccnew` while preserving the original index ([tuplesort.c#PrepareTempTablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1966), [fileset.c#parallel-temp-tablespaces](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L35-L70), [gistbuildbuffers.c#temporary-buffer-file](../../../../raw/postgres-17/src/backend/access/gist/gistbuildbuffers.c#L40-L58), [fd.c#temp-file-limit](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L2211-L2236), [indexcmds.c#pre-swap-phases](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3946-L4099)). | `temp_tablespaces` defaults to the database default tablespace and is session-scoped. `temp_file_limit` defaults to unlimited (`-1`), is `PGC_SUSET`, and is session-scoped for an authorized role; neither needs reload or restart ([guc_tables.c#temp_file_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2503-L2512), [guc_tables.c#temp_tablespaces](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4147-L4155)). |
| Writer-side `gin_pending_list_limit`; `work_mem` | Neither sizes the initial GIN build. After phase 2 makes the new index ready, concurrent writers can append to its pending list. Their `gin_pending_list_limit`, unless the copied index reloption overrides it, decides when regular cleanup starts; regular cleanup uses the writer's `work_mem`. Phase 3 forces cleanup with the RIC backend's `maintenance_work_mem`, so writer settings can change how much work validation inherits ([index.c#copy-reloptions](../../../../raw/postgres-17/src/backend/catalog/index.c#L1345-L1350), [index.c#apply-reloptions](../../../../raw/postgres-17/src/backend/catalog/index.c#L1442-L1458), [gin_private.h#pending-list-limit](../../../../raw/postgres-17/src/include/access/gin_private.h#L23-L45), [ginfast.c#regular-cleanup-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828), [ginvacuum.c#validation-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L584-L595)). | Both default to 4 MB and are session-scoped with no reload or restart; the relevant values are in concurrent writer sessions ([guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2446-L2457), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584)). |

The phase-2 `maintenance_work_mem` effect depends on the access method. B-tree
sorts index tuples; hash conditionally sorts by bucket; GiST may sort or use the
setting with `effective_cache_size` for buffered build; GIN bounds its build
accumulator; and parallel BRIN sorts range summaries. SP-GiST and contrib Bloom
do not read it during their first build. Phase 3 still creates the common TID
sort, although BRIN's `ambulkdelete` reports no per-tuple TIDs and therefore
leaves that sort empty
([nbtsort.c#primary-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L362-L431),
[hash.c#hashbuild-sort-choice](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L183),
[gistbuild.c#strategy-and-sort](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L210-L285),
[gininsert.c#build-threshold](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314),
[brin.c#serial-and-parallel-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1153-L1224),
[spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L147),
[blinsert.c#blbuild](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L117-L157),
[brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)).

`default_tablespace` does **not** place the rebuilt copy. Without an explicit
`REINDEX (TABLESPACE ...)`, RIC copies the old index's tablespace; with that
clause, it uses the requested tablespace except for TOAST indexes. This command
option can materially change output-write I/O, but it is not a GUC
([indexcmds.c#RIC-tablespace-selection](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3825-L3836),
[guc_tables.c#default_tablespace](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4136-L4145)).
RIC processes each index build and validation in its own transaction rather than
building all selected indexes in parallel, so one command does not multiply
`maintenance_work_mem` by its number of indexes at a single instant; a B-tree or
BRIN build can still divide that one operation's budget among its parallel
participants
([indexcmds.c#per-index-builds](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3946-L3997),
[indexcmds.c#per-index-validations](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4007-L4089),
[ref/create_index.sgml#parallel-memory](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L807-L844)).

**WAL, checkpoints, and commit latency.** Permanent RIC builds generate WAL.
B-tree and sorted GiST bulk writing log forced page images; GiST, GIN, and
SP-GiST can log the built page range after the first build
([bulk_write.c#bulk-WAL](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L310),
[gistbuild.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L328-L337),
[gininsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L408-L417),
[spginsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L132-L141)).

| GUC | Exact RIC effect and boundary | Default and application |
|---|---|---|
| `synchronous_commit`; `synchronous_standby_names` | WAL-writing internal transactions can flush locally and wait for a synchronous standby at commit. `synchronous_commit = off` permits eligible commits to be asynchronous and disables synchronous-replication waits, but phase 6's non-temporary relation deletion still forces a local WAL flush. Wait-only transactions that assign no XID or write no WAL have nothing to flush ([indexcmds.c#phase-6-drop](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4244), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.c#sync-rep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1537-L1546), [syncrep.h#SyncRepRequested](../../../../raw/postgres-17/src/include/replication/syncrep.h#L18-L25)). | `synchronous_commit` defaults to `on`; session; no reload or restart. `synchronous_standby_names` defaults empty; `sighup` context; reload ([guc_tables.c#synchronous_commit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4912-L4920), [guc_tables.c#synchronous_standby_names](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4570-L4579)). |
| `wal_compression` | Forced and ordinary full-page images can be compressed, trading CPU for fewer WAL bytes when pages compress well; the result is page- and codec-dependent ([xloginsert.c#full-page-compression](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L695), [xloginsert.c#forced-page-images](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1169-L1223)). | Off; `PGC_SUSET`; session for an authorized role; no reload or restart ([guc_tables.c#wal_compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4963-L4970)). |
| `wal_buffers`; `wal_sync_method` | These are generic WAL-path controls: the first sizes shared WAL buffering for RIC's WAL stream; the second selects the local method used to force WAL to disk. Neither changes worker count or scan shape ([guc_tables.c#wal_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2890-L2899), [guc_tables.c#wal_sync_method](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5013-L5020)). | `wal_buffers` defaults to `-1` (automatic), is `postmaster` context, and needs restart. `wal_sync_method` uses the platform default, is `sighup` context, and needs reload. |
| `max_wal_size`; `checkpoint_timeout`; `checkpoint_completion_target`; `checkpoint_flush_after` | The first two influence automatic checkpoint frequency; the latter two pace checkpoint writes and writeback. If a checkpoint starts during an AM bulk write, `smgr_bulk_finish()` immediately syncs the relation because the checkpoint missed earlier writes. These cluster-wide effects are not monotonic ([bulk_write.c#checkpoint-crossing-sync](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223), [guc_tables.c#checkpoint-size-time-flush](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2841-L2887), [guc_tables.c#checkpoint_completion_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3915-L3923)). | Normally 1 GB, 5 minutes, `0.9`, and a platform default respectively; all are `sighup` context and need reload, not restart. |
| `commit_delay`; `commit_siblings` | When enabled and enough other transactions are active, the WAL flush path deliberately sleeps before group commit. This can improve aggregate throughput while adding latency to affected RIC commits; defaults add no delay ([xlog.c#group-commit-delay](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2869-L2888)). | `commit_delay = 0`; `PGC_SUSET`, authorized session. `commit_siblings = 5`; session. Neither needs reload or restart ([guc_tables.c#commit-delay](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2979-L3000)). |

Disabling `fsync` or `full_page_writes` can reduce generic durability I/O, but it
is not a safe production RIC tuning method. Both are `sighup` settings and need
reload. In addition, `REGBUF_FORCE_IMAGE` makes bulk-build page images
independent of `full_page_writes`, so turning that setting off does not remove
those records
([guc_tables.c#fsync-and-full_page_writes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1095-L1106),
[guc_tables.c#full_page_writes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1155-L1167),
[xloginsert.c#force-image-wins](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L610)).

**Wait and cancellation settings.** These settings do not make either scan
faster. They bound or expose the initial relation lock and RIC's five waits:

| GUC | RIC behavior | Default and application |
|---|---|---|
| `statement_timeout` | Covers the client statement across all internal commits, scans, and five waits. RIC calls `StartTransactionCommand` directly, so it does not restart the statement timer; the normal command-end path disables it ([postgres.c#statement-timeout-lifetime](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2807), [postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5228-L5262)). | 0 (disabled); session; no reload or restart ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2610-L2619)). |
| `transaction_timeout` | Starts for every internal transaction and is disabled at each commit. It is a per-internal-transaction cap, not a whole-RIC cap; expiry terminates the connection with `FATAL` ([xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2168-L2178), [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2307-L2317), [postgres.c#transaction-timeout-error](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3463)). | 0 (disabled); session; no reload or restart ([guc_tables.c#transaction_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2643-L2652)). |
| `lock_timeout` | Applies to initial relation-lock acquisition and to each individual VXID lock used by all five waits. It restarts for each lock acquisition, so it is not an aggregate limit for a wait phase, blocker set, or whole command ([indexcmds.c#initial-index-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2807-L2811), [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L972), [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L492), [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4662), [proc.c#lock-timeout-timer](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1240-L1274)). | 0 (disabled); session; no reload or restart ([guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2621-L2630)). |
| `deadlock_timeout`; `log_lock_waits` | The first controls when a blocked lock acquisition runs deadlock detection; lowering it does not end a non-deadlocked wait. The second logs waits that survive that interval. They are detection and observability controls, not throughput controls ([proc.c#deadlock-and-lock-timeouts](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1240-L1292), [proc.c#long-wait-logging](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1458-L1492)). | 1 second and off; both are `PGC_SUSET`, so an authorized role can use session scope with no reload or restart ([guc_tables.c#deadlock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2145-L2156), [guc_tables.c#log_lock_waits](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1511-L1520)). |
| Blocker-side `idle_in_transaction_session_timeout` | It does not fire in the actively running RIC backend. In another session it can terminate an idle-in-transaction reader or writer that blocks RIC; that is blocker policy, not a scan-speed setting ([postgres.c#idle-in-transaction-timer](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4600-L4630)). | 0 (disabled); session in the blocker; no reload or restart ([guc_tables.c#idle_in_transaction_session_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2631-L2641)). |

A timeout or cancellation after phase 1 can leave the committed transient index
behind. Before the atomic swap it is the invalid `_ccnew`; after the swap the
rebuilt index has the original name and an invalid `_ccold` can remain. Timeouts
are therefore completion policy, not free acceleration
([indexcmds.c#phase-boundaries](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3936-L4244),
[index.c#atomic-swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1648)).

**Commonly confused settings and practical priority.**

- `min_parallel_index_scan_size` does not participate: RIC's worker planner
  passes `index_pages = -1`, so only `min_parallel_table_scan_size` supplies a
  size threshold. `max_parallel_workers_per_gather` and
  `parallel_leader_participation` govern query executor nodes, not the private
  maintenance build; B-tree and BRIN hard-code leader participation unless a
  compile-time test macro disables it. Planner cost GUCs likewise do not choose
  the worker count in this size-based function
  ([planner.c#heap-only-worker-model](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L7012),
  [allpaths.c#index-threshold-guard](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4269),
  [nbtsort.c#leader-participation](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1410-L1427),
  [brin.c#leader-participation](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2367-L2383)).
- `maintenance_io_concurrency` is not selected by the v17 RIC heap streams.
  They pass `READ_STREAM_SEQUENTIAL` without `READ_STREAM_MAINTENANCE`, so the
  executable path reaches the effective-I/O branch described above
  ([heapam.c#RIC-read-stream-flags](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1170-L1192),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L415-L429)).
- The RIC backend's `work_mem`, `autovacuum_work_mem`, and `vacuum_cost_*`
  settings do not size or throttle the common validation path. Validation uses
  `maintenance_work_mem`; its forced GIN cleanup chooses that value for a normal
  RIC backend. A unique B-tree allocates a secondary `work_mem` spool during the
  first build, but the concurrent MVCC scan supplies no dead tuples and discards
  it empty. `vacuum_delay_point()` returns without delaying when
  `VacuumCostActive` is false
  ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3361-L3405),
  [ginfast.c#forced-cleanup-memory](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828),
  [nbtsort.c#secondary-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L505),
  [heapam_handler.c#concurrent-tuples-are-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1621-L1629),
  [vacuum.c#vacuum_delay_point](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2383-L2419)).
- `wal_level = minimal` and `wal_skip_threshold` do not unlock the usual
  new-relation WAL skip for a permanent RIC copy. Phase 1 commits the catalog
  creation before phase 2 builds it, so the index is no longer new in the build
  transaction and `RelationNeedsWAL()` remains true
  ([indexcmds.c#catalog-then-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3825-L3996),
  [rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L621-L631)).
- `track_activities`, `trace_sort`, `log_temp_files`, and `track_counts` expose
  progress or spills rather than changing RIC's correctness algorithm.
  `track_io_timing` and `track_wal_io_timing` add measurements but can add
  significant clock-call overhead on some systems; both are `PGC_SUSET`, default
  off, and can use authorized session scope without reload or restart
  ([backend_progress.c#track_activities](../../../../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L20-L60),
  [guc_tables.c#trace_sort](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1658-L1668),
  [fd.c#temp-file-reporting](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1524-L1538),
  [pgstat_database.c#temp-file-accounting](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_database.c#L171-L185),
  [guc_tables.c#I/O-timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1419-L1435),
  [config.sgml#I/O-timing-overhead](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8374-L8422)).

In practice, tune and measure in this order: (1) `maintenance_work_mem`; (2) for
B-tree or BRIN, the full request-and-availability chain; (3) AM-specific GiST or
GIN behavior; (4) heap, temporary, and explicit output-tablespace I/O; and (5)
WAL, checkpoints, and commit confirmation. Diagnose the five blocker waits
separately, because memory and workers cannot shorten them
([ref/create_index.sgml#memory-and-parallelism](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L862),
[ref/reindex.sgml#concurrent-cost](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L366-L379),
[indexcmds.c#RIC-waits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3946-L4216)).
There is no source-derived universal best value; the direct RIC regression and
isolation tests check lifecycle and concurrency, not comparative GUC performance
([create_index.sql#RIC-tests](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165),
[reindex-concurrently.spec](../../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40)).
At the build boundary, both build systems list `guc_tables.c` directly; the
Meson manifest lists the generated `guc-file.c` scanner separately, so the cited
GUC-definition path does not depend on that generated source
([utils/misc/Makefile#guc_tables](../../../../raw/postgres-17/src/backend/utils/misc/Makefile#L17-L27),
[utils/misc/meson.build#guc_tables](../../../../raw/postgres-17/src/backend/utils/misc/meson.build#L3-L26)).

### All steps and locks required on the table

Two lock layers act on each **heap table**, exactly as in CIC: a
**transaction-level** `ShareUpdateExclusiveLock` re-taken inside each helper, and
a **session-level** `ShareUpdateExclusiveLock` that spans the commit gaps. RIC
additionally holds session `ShareUpdateExclusiveLock` on both the **old and new
index** relations
([indexcmds.c#session-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3928-L3934)).
DML never conflicts with any of these — `ShareUpdateExclusiveLock` does not
conflict with `RowExclusiveLock`
([lock.c#SUE-conflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80)).

The waits are **not** table locks. `WaitForLockersMultiple` reads the current set
of conflicting lock holders and sleeps on each holder's virtual transaction ID
until it ends; later transactions are not waited for
([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L903-L979)).
The conflict mode passed to the wait decides who is waited out:

| Phase | Relation locks held by RIC | Wait | Conflict mode | Who it waits out |
|---|---|---|---|---|
| 1: create copies | transaction, then session `ShareUpdateExclusiveLock` on heap and indexes | — | — | — |
| 2: before build | session `ShareUpdateExclusiveLock`; build later takes heap `ShareUpdateExclusiveLock` and new-index `RowExclusiveLock` | Wait 1 | `ShareLock` | current holders conflicting with `ShareLock` — open writers and stronger table operations ([lock.c:82-85](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L85)) |
| 3: before validation | session `ShareUpdateExclusiveLock` | Wait 2 | `ShareLock` | same `ShareLock` conflict set |
| 3: after validation | session `ShareUpdateExclusiveLock` | Wait 3 | `WaitForOlderSnapshots` | same-database **old-snapshot** holders, ignoring autovacuum, lazy `VACUUM`, and safe concurrent index builds ([indexcmds.c:433-442](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442)) |
| 5: before set-dead | session `ShareUpdateExclusiveLock` | Wait 4 | `AccessExclusiveLock` | current table lock holders, including **readers** ([lock.c:98-102](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)) |
| 6: before drop | session `ShareUpdateExclusiveLock` | Wait 5 | `AccessExclusiveLock` | current table lock holders, including **readers** |

So RIC has the same three CIC waits (`ShareLock`, `ShareLock`, old snapshots)
**plus** two reader-capable waits that CIC does not have. Those two reader waits
exist because the old index may still be usable by transactions that planned
before the swap, so RIC cannot remove it until current table lock holders that
might touch it have finished
([reindex.sgml#concurrent-steps](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L382-L443)).

Because `ShareUpdateExclusiveLock` is self-conflicting, only one concurrent build
(CIC or RIC), `VACUUM`, or `ANALYZE` can run on a given table at a time
([lock.c#self-conflict](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L78)).

### State flags for the old and new index

RIC drives the same three `pg_index` flags as CIC — `indislive`, `indisready`,
`indisvalid` — but on **two** indexes at once. They progress like this:

| Step | New copy (`_ccnew`) | Old index |
|---|---|---|
| after phase 1 | live, not ready, **not valid** | live, ready, valid |
| after phase 2 build | live, **ready**, not valid | unchanged |
| after phase 3 validate | live, ready, not valid (data complete) | unchanged |
| after phase 4 swap | live, ready, **valid** (takes original name) | **invalid** (renamed `_ccold`) |
| after phase 5 set-dead | unchanged | **dead** (`indisready=false`, `indislive=false`) |
| after phase 6 drop | the surviving index | gone |

The new copy's `indisready` is set by the shared `index_set_state_flags`
(`INDEX_CREATE_SET_READY`); its `indisvalid` is flipped by the swap, not by
`index_set_state_flags`
([index.c#swap-mark-valid](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1648)).
The old index is set dead by `index_set_state_flags(INDEX_DROP_SET_DEAD)`, whose
assert requires the index already be invalid — which the swap guarantees
([index.c#INDEX_DROP_SET_DEAD](../../../../raw/postgres-17/src/backend/catalog/index.c#L3500-L3514)).
In v17, **all** `index_set_state_flags` writes are made with the transactional
`CatalogTupleUpdate`, so they roll back with their transaction and become visible
via cache invalidation on commit
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521)).
This is a behavior change from v12 (see
[What changed from PostgreSQL 12](#what-changed-from-postgresql-12)).

### Failure scenarios and the outcome on the table

Like CIC, RIC commits many times, so the outcome depends on **which phase failed**.
The single most important property: a RIC failure on a **healthy** index never
leaves you without a working index, because the swap (phase 4) is the only step
that touches the original, and it runs only after the new copy is fully built and
validated.

| Failure point | Leftover on the table | Recovery |
|---|---|---|
| Phase 1 (gather / create copies / initial lock) | **none** — phase 1's transaction rolls back, so no `_ccnew` persists; original untouched | retry |
| Phase 2 (Wait 1 or build) | one or more **invalid `_ccnew`** copies (the failing one **not ready**, any earlier-built ones **ready**); original state unchanged | `DROP INDEX <name>_ccnew`, then retry |
| Phase 3 (Wait 2 / validate / Wait 3) | **invalid, ready `_ccnew`** copy; original state unchanged | `DROP INDEX <name>_ccnew`, then retry |
| Phase 4 (swap), before its commit | swap transaction rolls back: `_ccnew` still invalid, original state unchanged and normally named | `DROP INDEX <name>_ccnew`, then retry |
| Phase 5 or 6 (after swap committed) | the **new** index is already valid and carries the original name; the **old** `_ccold` index lingers (invalid, possibly dead) | `DROP INDEX <name>_ccold` |

**A failure never leaves `index_name` itself invalid for a healthy index.** The
swap in phase 4 is the only step that touches the original index's name or its
`indisvalid` flag, and it does both together in one transaction: it renames the
rebuilt copy to `index_name`, renames the original to `index_name_ccold`, and in
the same `CatalogTupleUpdate` flips `new.indisvalid = true`,
`old.indisvalid = false`
([index.c:1591-1648](../../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1648)).
The source comment states the intent — mark new valid and old invalid "at the
same time to make sure we only get constraint violations from the indexes with
the correct names"
([indexcmds.c:4096-4100](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4096-L4100)).
So a pre-swap abort discards the `_ccnew` copy and leaves the original valid (even
if bloated); a post-swap failure can only happen once the rebuilt `index_name` is
already valid. The invalid leftover is always a **differently named** index
(`_ccnew` or `_ccold`), never the bare `index_name`. The one exception is when
`index_name` was **already invalid before** RIC ran (the repair case) — then a new
pre-swap failure leaves it invalid only because it started that way.

Invalid leftovers are ignored for query planning: `get_relation_info` closes and
skips an index whose `indisvalid` is false
([plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)).
They can still matter for **writes**, because `RelationGetIndexList` omits only
indexes with `indislive = false`, and executor insertion skips only indexes whose
`ii_ReadyForInserts` is false. So a ready-but-invalid `_ccnew` or `_ccold` can
still impose write/HOT-safety overhead even though the planner will not choose it
([relcache.c#RelationGetIndexList](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4800-L4869),
[execIndexing.c#ExecInsertIndexTuples](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359)).
RIC's failure paths are ordinary ERROR/cancel/timeout aborts. Each phase loop
runs inside a transaction precisely so that an abort cleans up the session locks
rather than leaking them
([indexcmds.c:3967-3972](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3967-L3972)).

A unique `_ccnew` that reaches phase 3 enforces uniqueness while it is ready but
invalid, so a duplicate appearing during the second scan surfaces as a uniqueness
violation — which is how the documented "leaves an invalid index" failure arises
([reindex.sgml#failure-recovery](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L446-L478)).

### Can a failure leave an invalid index with the original index name?

**For a healthy index, no.** A `REINDEX INDEX CONCURRENTLY index_name` failure
never leaves the table with `index_name` marked invalid and therefore no usable
index. The name `index_name` always ends up on a **valid** index, so query plans
can still use it. `index_name` itself is left invalid only if it was **already
invalid before** the command ran — the repair case below.

Why the name is safe: the **swap in phase 4 is the only step that touches the
original index's name or its `indisvalid` flag**, and it does both together. In a
single transaction, `index_concurrently_swap` renames the rebuilt copy to
`index_name` and the old index to `index_name_ccold`, and in the same call flips
`new.indisvalid = true`, `old.indisvalid = false` through the transactional
`CatalogTupleUpdate`
([index.c#swap-names](../../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1593),
[index.c#swap-mark-valid](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1648)).
The whole phase-4 loop runs inside one transaction, so the rename and the
validity flip commit or roll back together
([indexcmds.c#phase4-txn](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4163)).
The source comment states the intent: mark the new valid and the old invalid "at
the same time to make sure we only get constraint violations from the indexes
with the correct names"
([indexcmds.c:4096-4100](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4096-L4100)).

So every committed outcome keeps a valid `index_name`:

| When the failure happens | What `index_name` is | Invalid leftover |
|---|---|---|
| Phases 1-3, or phase 4 **before** it commits | the **original** index — name and `indisvalid` untouched, still valid even if bloated | a separate `index_name_ccnew` build copy, invalid |
| Phases 5-6, **after** the swap commits | the **rebuilt** index — already marked valid under `index_name` | the renamed `index_name_ccold` old index, invalid |

A pre-swap abort discards the `_ccnew` copy's catalog work with its transaction
and never renamed or invalidated the original; a post-swap failure can only
happen once the rebuilt index is already valid under `index_name`. The invalid
leftover is therefore always a **differently named** index (`_ccnew` or
`_ccold`), never the bare `index_name`.

This directly answers the bloat worry: you do **not** go from "one bloated but
valid `index_name`" to "no usable index." Before the swap the planner still has
the original (bloated) `index_name`; after the swap it has the fresh rebuild. The
planner skips only indexes whose `indisvalid` is false
([plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)),
and in neither committed state is `index_name` the invalid one. A leftover
`_ccnew`/`_ccold` that is ready but invalid can still add write/HOT overhead until
you drop it, as the
[failure table](#failure-scenarios-and-the-outcome-on-the-table) notes, but it
does not remove your usable index.

Unlike v12, v17 makes **every** `pg_index` flag write transactional — the
set-ready (phase 2), the swap's validity flip (phase 4), and the set-dead (phase
5) all end in `CatalogTupleUpdate`, not the old in-place `heap_inplace_update`
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521);
see [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)). So each
phase's catalog effect commits or rolls back atomically with its transaction,
which is what keeps a pre-swap abort from ever leaving `index_name` half-flipped.
The exact recovered state across a hard crash is scoped under
[Open Questions](#open-questions).

**The one exception — repairing an already-invalid index.** If `index_name` was
invalid before you ran RIC (for example, left invalid by a failed
`CREATE INDEX CONCURRENTLY` or an earlier failed RIC), a new pre-swap failure
leaves it invalid — but only because it started that way, not because RIC broke a
healthy index. The v17 regression suite stages exactly this: a unique index made
invalid by a failed CIC over duplicate data, then
`REINDEX INDEX CONCURRENTLY concur_reindex_ind5` fails its build with
`could not create unique index "concur_reindex_ind5_ccnew"`, and `\d` shows
**both** `concur_reindex_ind5` and `concur_reindex_ind5_ccnew` as `INVALID`
([create_index.out#both-invalid](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2716-L2734)).
Dropping the `_ccnew` spare, deleting the duplicate, and re-running RIC makes
`concur_reindex_ind5` valid again
([create_index.out#repair](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2736-L2760)).

### Watching the phases

RIC reports through `pg_stat_progress_create_index` with `command` =
`REINDEX CONCURRENTLY`
([indexcmds.c#progress-command](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3810-L3816)).
The wait phases map to the view's `phase` text via the integer codes in
`progress.h`
([progress.h#phases](../../../../raw/postgres-17/src/include/commands/progress.h#L92-L114),
[system_views.sql#create-index-view](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277)):

| Phase | Code set | `phase` text shown |
|---|---|---|
| 2 (Wait 1) | `WAIT_1` (1) | `waiting for writers before build` |
| 3 (Wait 2) | `WAIT_2` (3) | `waiting for writers before validation` |
| 3 (Wait 3) | `WAIT_3` (7) | `waiting for old snapshots` |
| 5 (Wait 4) | `WAIT_4` (8) | `waiting for readers before marking dead` |
| 6 (Wait 5) | `WAIT_5` (9) | `waiting for readers before dropping` |

Unlike v12 — where phase 6 wrongly reused `WAIT_4`, so the
`waiting for readers before dropping` text was never emitted — v17 sets `WAIT_5`
in phase 6, so all five wait strings appear
([indexcmds.c:4214-4215](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4215)).

### What changed from PostgreSQL 12

The six-phase, five-wait shape, the `ShareUpdateExclusiveLock` footprint, and the
`indislive`/`indisready`/`indisvalid` progression are **unchanged** from v12. (For
the v12 trace, see
[How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../../v12/questions/indexing/reindex-index-concurrently.md).)
Several behavioral and structural changes landed between v12 and v17; each is
attributed to a commit present in this checkout's own history, with the major
release that first shipped it.

**1. `REINDEX` of partitioned tables and indexes is now supported (PostgreSQL
14).** In v12, naming a partitioned index errored with `REINDEX is not yet
implemented for partitioned indexes`. PostgreSQL 14 (commit `a6642b3ae06`, "Add
support for partitioned tables and indexes in REINDEX") made `ReindexIndex` /
`ReindexTable` route a partitioned relation to `ReindexPartitions`, which expands
the tree to its physical leaf partitions and reindexes each leaf in its own
transaction; with `CONCURRENTLY`, each leaf goes through
`ReindexRelationConcurrently`
([indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301),
[indexcmds.c#ReindexMultipleInternal](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3364-L3374),
[reindex.sgml#partitions](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334)).
Two earlier v13 fixes to the swap support this: `relispartition` is now swapped
(`d80be6f2f6c`) and partition dependencies are swapped (`68ac9cf2499`)
([index.c:1595-1601](../../../../raw/postgres-17/src/backend/catalog/index.c#L1595-L1601),
[index.c:1769-1792](../../../../raw/postgres-17/src/backend/catalog/index.c#L1769-L1792)).

**2. `REINDEX (TABLESPACE ...)` (PostgreSQL 14).** Commit `c5b286047cd` added a
`TABLESPACE` option; `ExecReindex` parses it into `params.tablespaceOid`, and the
phase-1 copy is created in the target tablespace (toast indexes are not moved).
For partitioned targets, only leaf-partition tablespaces are updated
([indexcmds.c:2732-2750](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2732-L2750),
[indexcmds.c:3825-3836](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3825-L3836),
[reindex.sgml#tablespace](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334)).

**3. `index_set_state_flags` is now transactional (PostgreSQL 14).** In v12 the
set-ready and set-dead flag writes used `heap_inplace_update` — a non-transactional
in-place overwrite that could not roll back. Commit `83158f74d3a` ("Make
index_set_state_flags() transactional") switched it to `CatalogTupleUpdate`, so in
v17 those flips obey MVCC, roll back with their transaction, and publish via cache
invalidation on commit
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3445-L3518)).
The phase-4 validity flip was already transactional in v12; this change makes the
phase-2 (`SET_READY`) and phase-5 (`SET_DEAD`) writes transactional as well.

**4. `PROC_IN_SAFE_IC` lets other concurrent builds ignore a running RIC
(PostgreSQL 14).** v12's RIC did not advertise itself, so two concurrent index
operations on different tables would wait on each other's snapshots. PostgreSQL 14
(commit `f9900df5`, "Avoid spurious wait in concurrent reindex") made RIC call
`set_indexsafe_procflags()` in phases 2 and 3 when the index is safe (no
expressions, no partial predicate) and unconditionally in phase 4
([indexcmds.c:3974-3976](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3974-L3976),
[indexcmds.c:4034-4036](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4034-L4036),
[indexcmds.c:4109](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4109)).
`WaitForOlderSnapshots` then excludes `PROC_IN_SAFE_IC` processes from the
snapshot wait, alongside autovacuum and lazy `VACUUM`
([indexcmds.c:433-442](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442),
[indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4487)).
This only shortens **Wait 3** (the snapshot wait); the lock-based Waits 1, 2, 4,
and 5 are unaffected.

**5. Progress-view phase fix (PostgreSQL 14).** Commit `e66bcfb4c66` ("Fix
progress reporting of REINDEX CONCURRENTLY") corrected phase 6 to set `WAIT_5`
instead of reusing `WAIT_4`, so `pg_stat_progress_create_index` now shows
`waiting for readers before dropping` during the final drop wait
([indexcmds.c:4214-4215](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4215),
[system_views.sql:1276-1277](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1276-L1277)).

**6. Statistics preservation reworked (PostgreSQL 15).** v12's swap hand-copied a
few cumulative counters into `newClassRel->pgstat_info->t_counts`. After the
PostgreSQL 15 shared-memory statistics rework (commit `8ea7963fc74`, "pgstat: add
pgstat_copy_relation_stats()"), the swap calls `pgstat_copy_relation_stats` to
move the cumulative counters and `CopyStatistics` to copy the planner's
`pg_statistic` column statistics (which matters for expression indexes)
([index.c:1794-1798](../../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798)).

**7. `MAINTAIN` privilege (PostgreSQL 17).** v12 required table ownership (or
superuser) to reindex. PostgreSQL 17 (commit `ecb0fd33720`, "Reintroduce MAINTAIN
privilege and pg_maintain predefined role") lets a non-owner reindex if granted
`MAINTAIN`; the index lookup callback checks `ACL_MAINTAIN`
([indexcmds.c#RangeVarCallbackForReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2887-L2896)).

**8. Security hardening and testability (post-v12).** v17 switches to the table
owner's userid and calls `RestrictSearchPath()` before running index/predicate
functions, both when creating the copy and inside `index_concurrently_build`
([indexcmds.c:3786-3790](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3786-L3790),
[index.c:1503-1507](../../../../raw/postgres-17/src/backend/catalog/index.c#L1503-L1507));
v12 did neither. v17 also adds `reindex-conc-index-safe` / `-not-safe` injection
points for deterministic testing
([indexcmds.c:3796-3801](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3796-L3801)).

**9. Dispatch refactor (PostgreSQL 14, mechanical).** v12 dispatched REINDEX
inline in `standard_ProcessUtility` and passed an `options` int. v17 routes
through `ExecReindex` with `ReindexStmt` / `ReindexParams` structs and a
`REINDEXOPT_*` bitmask, and prevents REINDEX during recovery generically via
`ClassifyUtilityCommandAsReadOnly` rather than an explicit
`PreventCommandDuringRecovery("REINDEX")`
([indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781),
[utility.c:280-296](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296)).

### Test coverage

- Functional coverage is in the `REINDEX CONCURRENTLY` block of
  `create_index.sql`: it reindexes tables, matviews, a single index, and toast
  indexes; checks dependency and comment preservation; rebuilds expression /
  predicate indexes; exercises clustered and replica-identity indexes; and tests
  the exclusion error-vs-skip
  ([create_index.sql#ric-block](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L976)).
- Partitioned coverage: the same file reindexes leaf partition tables and indexes
  concurrently, reindexes a **partitioned index** concurrently
  (`REINDEX INDEX CONCURRENTLY concur_reindex_part_index`), and asserts the
  TABLE-vs-INDEX object-type errors
  ([create_index.sql#partitions](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L978-L1116)).
- Restriction and repair coverage: system-catalog rejection, `REINDEX SYSTEM
  CONCURRENTLY`, and the invalid-index repair (an index left invalid by a failed
  CIC, then repaired by RIC)
  ([create_index.sql#system-and-repair](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L1131-L1165)).
- Isolation coverage: `reindex-concurrently.spec` runs one reading transaction,
  one writing transaction, and a `REINDEX TABLE CONCURRENTLY` session across six
  permutations
  ([reindex-concurrently.spec](../../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and the last ~20
  `wiki/log.md` entries (navigation only).
- The sibling page
  [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
  (shared build/validate core and `PROC_IN_SAFE_IC` discussion) and the v12
  counterpart
  [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../../v12/questions/indexing/reindex-index-concurrently.md)
  (the v12 baseline for the change comparison).
- Pinned checkout `raw/postgres-17/` at commit
  `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- `ExecReindex`, `ReindexIndex`, `ReindexTable`, `ReindexMultipleTables`,
  `ReindexPartitions`, `ReindexMultipleInternal`, `ReindexRelationConcurrently`,
  `WaitForOlderSnapshots`, and `set_indexsafe_procflags` in
  `src/backend/commands/indexcmds.c`.
- `index_concurrently_create_copy`, `index_concurrently_build`,
  `index_concurrently_swap`, `index_concurrently_set_dead`, `index_drop`, and
  `index_set_state_flags` in `src/backend/catalog/index.c`.
- `doDeletion` in `src/backend/catalog/dependency.c`; `WaitForLockersMultiple` in
  `src/backend/storage/lmgr/lmgr.c`; the `LockConflicts` table in
  `src/backend/storage/lmgr/lock.c`.
- `ClassifyUtilityCommandAsReadOnly` and the REINDEX dispatch in
  `src/backend/tcop/utility.c`.
- The CREATE INDEX progress phase codes in `src/include/commands/progress.h` and
  the `pg_stat_progress_create_index` view in
  `src/backend/catalog/system_views.sql`.
- `doc/src/sgml/ref/reindex.sgml`; `plancat.c`, `relcache.c`, `execIndexing.c`.
- GUC definitions and apply contexts in `src/backend/utils/misc/guc_tables.c`,
  with `src/backend/utils/misc/Makefile` and `meson.build` for the build boundary.
- Build and validation in `index.c` and `heapam_handler.c`; B-tree, hash, GiST,
  GIN, SP-GiST, BRIN, and contrib Bloom build and bulk-delete callbacks.
- Parallel worker planning and launch in `planner.c`, `allpaths.c`, `bgworker.c`,
  and `parallel.c`, including table reloptions and leader GUC restoration.
- Heap scan strategy and read streams, tablespace I/O overrides, tuplesort and
  temporary-file placement/limits, bulk writes, WAL insertion, checkpoints,
  commit synchronization, and timeout/lock paths.
- Same-checkout configuration, `CREATE INDEX`, and `REINDEX` documentation.
- Tests: `src/test/regress/sql/create_index.sql` and
  `src/test/isolation/specs/reindex-concurrently.spec`; neither directly sweeps
  the RIC GUC matrix.
- v12 vs v17 deltas established from the `raw/postgres-17/` checkout's own commit
  history (`a6642b3ae06`, `c5b286047cd`, `83158f74d3a`, `f9900df5`,
  `e66bcfb4c66`, `8ea7963fc74`, `ecb0fd33720`, `d80be6f2f6c`, `68ac9cf2499`),
  each verified present with `git show` and first-release-tagged with
  `git tag --contains` against `REL_NN_0`.

## Evidence Map

| Claim | Source |
|---|---|
| RIC runs as six phases (create copy, build, catch up, swap, set dead, drop), each phase looping all indexes | [indexcmds.c:3737-3751](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3737-L3751) |
| `CONCURRENTLY` cannot run in a transaction block; recovery blocked via read-only classification | [indexcmds.c:2720-2722](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722), [utility.c:280-296](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296), [utility.c:581-582](../../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582) |
| `ExecReindex` parses options into `ReindexParams` and dispatches by kind | [indexcmds.c:2693-2781](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781) |
| `ReindexIndex` locks SUE for concurrent; partitioned -> `ReindexPartitions`; temp -> non-concurrent | [indexcmds.c:2788-2834](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2788-L2834) |
| Partitioned target recurses into leaf partitions, each reindexed concurrently in its own transaction | [indexcmds.c:3217-3301](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301), [indexcmds.c:3364-3374](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3364-L3374) |
| Directly named partitioned relation never reaches the concurrent core | [indexcmds.c:3708-3714](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3708-L3714) |
| System catalog rejected; invalid/exclusion via table skipped; invalid named directly allowed | [indexcmds.c:3516-3519](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3516-L3519), [indexcmds.c:3548-3560](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3548-L3560), [indexcmds.c:3695-3701](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3695-L3701) |
| Exclusion index named directly errors in `index_concurrently_create_copy` | [index.c:1328-1331](../../../../raw/postgres-17/src/backend/catalog/index.c#L1328-L1331) |
| Phase 1 creates `_ccnew` not-ready/not-valid via the skip-build + concurrent create flags | [index.c:1395](../../../../raw/postgres-17/src/backend/catalog/index.c#L1395), [index.c:1442-1462](../../../../raw/postgres-17/src/backend/catalog/index.c#L1442-L1462) |
| Session SUE taken on old index, new index, and heap(s) | [indexcmds.c:3863-3934](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3863-L3934) |
| Phase 2: Wait 1 (`ShareLock`) then per-index build sets `indisready` | [indexcmds.c:3955-3997](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3955-L3997), [index.c:1482-1540](../../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1540) |
| Phase 3: Wait 2 (`ShareLock`), `validate_index`, then Wait 3 (`WaitForOlderSnapshots`) | [indexcmds.c:4014-4088](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4014-L4088) |
| One invocation has six fixed internal commits plus three per index; a one-index RIC has nine before the final caller-owned transaction | [indexcmds.c:3936-4088](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3936-L4088), [indexcmds.c:4102-4257](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4257), [postgres.c:2798-2807](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2807) |
| Direct build controls are AM-dependent `maintenance_work_mem` plus the B-tree/BRIN worker request and pool chain | [index.c:2966-2976](../../../../raw/postgres-17/src/backend/catalog/index.c#L2966-L2976), [index.c:3361-3405](../../../../raw/postgres-17/src/backend/catalog/index.c#L3361-L3405), [planner.c:6876-7018](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7018), [bgworker.c:969-1043](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L969-L1043) |
| Both heap scans use sequential read streams; `io_combine_limit` and the effective-I/O branch apply, not maintenance I/O advice | [heapam.c:1166-1192](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1166-L1192), [read_stream.c:407-521](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L407-L521) |
| RIC preserves the old index tablespace unless `REINDEX (TABLESPACE ...)` is explicit; `default_tablespace` does not place the copy | [indexcmds.c:3825-3836](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3825-L3836) |
| WAL/checkpoint/commit settings affect permanent builds and RIC's many eligible commits without changing its state machine | [bulk_write.c:124-223](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223), [xloginsert.c:604-695](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L695), [xact.c:1462-1546](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1546) |
| `statement_timeout` spans RIC, `transaction_timeout` restarts per internal transaction, and `lock_timeout` applies per VXID acquisition | [postgres.c:2767-2807](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2807), [xact.c:2168-2178](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2168-L2178), [lock.c:4550-4662](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4662) |
| Phase 4 swap: new valid + old invalid via transactional `CatalogTupleUpdate`, names/constraints/deps/stats moved | [indexcmds.c:4102-4163](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4163), [index.c:1549-1809](../../../../raw/postgres-17/src/backend/catalog/index.c#L1549-L1809) |
| Swap copies cumulative + planner stats via `pgstat_copy_relation_stats` + `CopyStatistics` | [index.c:1794-1798](../../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798) |
| Phase 5: Wait 4 (`AccessExclusiveLock`) then `index_concurrently_set_dead` | [indexcmds.c:4180-4196](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4196), [index.c:1820-1854](../../../../raw/postgres-17/src/backend/catalog/index.c#L1820-L1854) |
| Phase 6: Wait 5 (`AccessExclusiveLock`) then `performMultipleDeletions` drops old | [indexcmds.c:4214-4244](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4244) |
| Drop uses `PERFORM_DELETION_CONCURRENT_LOCK` -> `index_drop` takes SUE, skips its own concurrent branch | [dependency.c:1356-1368](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1356-L1368), [index.c:2154-2196](../../../../raw/postgres-17/src/backend/catalog/index.c#L2154-L2196) |
| `AccessExclusiveLock` conflicts with `AccessShareLock`, so phases 5/6 wait out readers | [lock.c:98-102](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102) |
| `ShareLock` waits (Waits 1/2) wait out `RowExclusiveLock` writers and stronger (not other `ShareLock` holders) | [lock.c:82-85](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L85) |
| `ShareUpdateExclusiveLock` self-conflicts; does not conflict with `RowExclusiveLock` | [lock.c:77-80](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80) |
| Wait 3 ignores autovacuum, lazy VACUUM, and safe concurrent builds (`PROC_IN_SAFE_IC`) | [indexcmds.c:433-442](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442) |
| RIC advertises `PROC_IN_SAFE_IC` in phases 2/3 (if safe) and 4 (unconditional) | [indexcmds.c:3974-3976](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3974-L3976), [indexcmds.c:4034-4036](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4034-L4036), [indexcmds.c:4109](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4109) |
| `index_set_state_flags` is transactional (`CatalogTupleUpdate`); `INDEX_DROP_SET_DEAD` asserts not valid | [index.c:3449-3521](../../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521) |
| Invalid indexes skipped by planner but can still affect writes while live/ready | [plancat.c:260-266](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266), [relcache.c:4800-4869](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4800-L4869), [execIndexing.c:356-359](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359) |
| A RIC failure never leaves a healthy `index_name` invalid: phase 4 renames the rebuilt copy to `index_name` and flips validity in one transaction; the invalid leftover is always `_ccnew`/`_ccold`. `index_name` is invalid only if it was already invalid (repair case) | [index.c:1591-1648](../../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1648), [indexcmds.c:4096-4163](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4096-L4163), [create_index.out:2716-2760](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2716-L2760) |
| Progress: command `REINDEX CONCURRENTLY`; phase 6 correctly sets `WAIT_5` (fixed vs v12) | [indexcmds.c:4214-4215](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4215), [progress.h:92-114](../../../../raw/postgres-17/src/include/commands/progress.h#L92-L114), [system_views.sql:1264-1277](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277) |
| v14 added partitioned REINDEX (`a6642b3ae06`), TABLESPACE (`c5b286047cd`), transactional flags (`83158f74d3a`), PROC_IN_SAFE_IC for RIC (`f9900df5`), progress fix (`e66bcfb4c66`) | checkout git history; [reindex.sgml:319-334](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334) |
| v15 stats rework (`8ea7963fc74`); v17 MAINTAIN privilege (`ecb0fd33720`) | [index.c:1794-1798](../../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798), [indexcmds.c:2887-2896](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2887-L2896) |
| Docs: two scans, wait for transactions, six steps, `_ccnew`/`_ccold` recovery, partitions | [reindex.sgml:366-478](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L366-L478) |
| Tests: functional, partitioned, restriction/repair, isolation | [create_index.sql:882-1165](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165), [reindex-concurrently.spec:1-40](../../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40) |

## Open Questions

- The fastest safe values for a particular heap, index definition, access method,
  concurrent workload, storage system, and worker pool cannot be derived from
  source. They require controlled measurements of build, validation, commit, and
  wait phases.
- `read_stream.h` names `CREATE INDEX` as an example user of
  `READ_STREAM_MAINTENANCE`, but the v17 heap scan passes only
  `READ_STREAM_SEQUENTIAL`. The executable path therefore selects
  `effective_io_concurrency`; the pinned source does not establish whether the
  header example is stale or the heap path omitted the maintenance flag
  ([read_stream.h#READ_STREAM_MAINTENANCE](../../../../raw/postgres-17/src/include/storage/read_stream.h#L19-L35),
  [heapam.c#read-stream-flags](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1170-L1192),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L415-L429)).
- The exact crash / immediate-shutdown outcome at each instruction boundary was
  not traced through crash recovery. In v17 the swap validity flip and the
  set-ready / set-dead flips all go through transactional `CatalogTupleUpdate`
  ([index.c:3518](../../../../raw/postgres-17/src/backend/catalog/index.c#L3518)), so
  the v12 "non-transactional in-place" recovery hazard no longer applies to these
  flag writes; the precise recovered catalog state mid-rebuild was still not
  isolated here.
- Whether two indexes sharing a long base name can collide on the `_ccnew` /
  `_ccold` names is mitigated by `ChooseRelationName` plus the per-iteration
  `CommandCounterIncrement`
  ([indexcmds.c:4152-4159](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4152-L4159)),
  but the exact truncation/uniqueness behavior of `ChooseRelationName` was not
  traced.
- The first build scan's exact tuple-visibility rule lives inside `index_build` ->
  `table_index_build_scan` (via `ii_Concurrent`), summarized from the shared CIC
  core rather than traced line-by-line into the table AM
  ([index.c:1518-1522](../../../../raw/postgres-17/src/backend/catalog/index.c#L1518-L1522)).

## Source References

- [indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781)
- [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2788-L2834)
- [indexcmds.c#ReindexTable](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2918-L2966)
- [indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301)
- [indexcmds.c#ReindexMultipleInternal](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3311-L3410)
- [indexcmds.c#ReindexRelationConcurrently](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3437-L4295)
- [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442)
- [indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4487)
- [index.c#index_concurrently_create_copy](../../../../raw/postgres-17/src/backend/catalog/index.c#L1298-L1469)
- [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1540)
- [index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1549-L1809)
- [index.c#index_concurrently_set_dead](../../../../raw/postgres-17/src/backend/catalog/index.c#L1820-L1854)
- [index.c#index_drop](../../../../raw/postgres-17/src/backend/catalog/index.c#L2114-L2196)
- [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521)
- [index.c#index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L2930-L3040)
- [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3233-L3423)
- [heapam_handler.c#heapam_index_build_range_scan](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1171-L1744)
- [heapam_handler.c#heapam_index_validate_scan](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1747-L1986)
- [nbtsort.c#_bt_spools_heapscan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L351-L509)
- [hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L111-L194)
- [gistbuild.c#gistbuild](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L175-L355)
- [gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L246-L427)
- [spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L147)
- [brin.c#brinbuild](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1091-L1261)
- [blinsert.c#blbuild](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L117-L157)
- [planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019)
- [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4278)
- [bgworker.c#RegisterDynamicBackgroundWorker](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L969-L1049)
- [heapam.c#scan-strategy-and-read-stream](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L350-L429)
- [heapam.c#heap_beginscan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1084-L1195)
- [read_stream.c#read_stream_begin_relation](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L379-L555)
- [tuplesort.c#temp-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1938-L1966)
- [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223)
- [xact.c#commit-flush](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1546)
- [xlog.c#group-commit-delay](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2869-L2888)
- [dependency.c#doDeletion](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1352-L1368)
- [plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)
- [relcache.c#RelationGetIndexList](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4798-L4869)
- [execIndexing.c#ExecInsertIndexTuples](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359)
- [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L903-L979)
- [lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)
- [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4662)
- [proc.c#lock-timeouts](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1240-L1292)
- [postgres.c#statement-and-transaction-timeouts](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3374-L3463)
- [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2459-L2473)
- [guc_tables.c#timeouts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2610-L2652)
- [guc_tables.c#worker-pools](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3162-L3171)
- [guc_tables.c#parallel-workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3408-L3437)
- [guc_tables.c#I/O-settings](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3108-L3159)
- [guc_tables.c#planner-and-GIN-settings](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3507-L3584)
- [guc_tables.c#WAL-size-checkpoint-and-commit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2841-L3000)
- [guc_tables.c#WAL-commit-and-compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4912-L4970)
- [utility.c#ClassifyUtilityCommandAsReadOnly](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296)
- [progress.h#create-index-phases](../../../../raw/postgres-17/src/include/commands/progress.h#L92-L114)
- [system_views.sql#pg_stat_progress_create_index](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277)
- [ref/reindex.sgml#CONCURRENTLY](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L478)
- [create_index.sql#ric-block](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165)
- [reindex-concurrently.spec](../../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40)

## Navigation

- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../../v12/questions/indexing/reindex-index-concurrently.md)
- [v17 index](../../index.md)
- [versions](../../../versions.md)
