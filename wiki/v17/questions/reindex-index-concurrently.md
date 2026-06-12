---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: claude-opus-4-8 2026-06-12T20:57:06Z
---

# How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [How it differs from CREATE INDEX CONCURRENTLY](#how-it-differs-from-create-index-concurrently)
  - [Dispatch, preconditions, and restrictions](#dispatch-preconditions-and-restrictions)
  - [The six phases](#the-six-phases)
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
([indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3437-L4295)):

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
([lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)).

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
| Extra waits | none beyond the snapshot wait | two `AccessExclusiveLock` waits that **wait out readers** of the old index before set-dead and drop ([indexcmds.c:4180-4216](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4216)) |
| Names | the index keeps its name | new copy takes the original's name; the old index is renamed `<name>_ccold` ([index.c#swap-names](../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1593), [indexcmds.c#ccold](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4124-L4129)) |
| On failure | leaves the **target** index invalid | leaves an invalid `_ccnew` (before swap) or `_ccold` (after swap); a healthy original is preserved until the swap |

The "wait out readers" difference is the operationally important one. CIC's two
lock-based waits use `ShareLock` conflict checks and do not wait for
`AccessShareLock` readers (its old-snapshot wait can still wait for a plain
`SELECT` holding an old snapshot). RIC adds two later `AccessExclusiveLock`
conflict checks on the heap lock tag, so current table lock holders — including
`AccessShareLock` readers — can delay marking the old index dead and dropping it
([lock.c#AccessExclusive-conflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102),
[reindex.sgml#concurrent-steps](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L382-L443)).

### Dispatch, preconditions, and restrictions

`REINDEX` is dispatched from `standard_ProcessUtility` into `ExecReindex`, which
parses the `CONCURRENTLY`, `VERBOSE`, and `TABLESPACE` options into a
`ReindexParams` bitmask and then routes by object kind
([utility.c:1566-1567](../../../raw/postgres-17/src/backend/tcop/utility.c#L1566-L1567),
[indexcmds.c#ExecReindex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781)).
Because the concurrent path commits many times, it cannot run inside a
transaction block:

```c
if (concurrently)
    PreventInTransactionBlock(isTopLevel, "REINDEX CONCURRENTLY");
```

([indexcmds.c:2720-2722](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722)).
`REINDEX` is also forbidden during recovery, but in v17 that is enforced
generically: `ClassifyUtilityCommandAsReadOnly` returns only
`COMMAND_OK_IN_READ_ONLY_TXN` (not `COMMAND_OK_IN_RECOVERY`) for a `ReindexStmt`,
so `standard_ProcessUtility` calls `PreventCommandDuringRecovery`
([utility.c#classify-reindex](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296),
[utility.c:581-582](../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582)).

`REINDEX INDEX` routes through `ReindexIndex`, which locks the index in
`ShareUpdateExclusiveLock` for the concurrent case (vs `AccessExclusiveLock` for
a plain reindex). It then dispatches in three ways
([indexcmds.c#ReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2788-L2834)):

- a **partitioned** index goes to `ReindexPartitions` (new since v12);
- a concurrent, non-temporary index goes to `ReindexRelationConcurrently`;
- otherwise (including a temporary index, even if `CONCURRENTLY` was requested) it
  falls back to the non-concurrent `reindex_index`.

`REINDEX TABLE` behaves the same way via `ReindexTable`, emitting
`table "..." has no indexes that can be reindexed concurrently` when nothing
qualifies
([indexcmds.c#ReindexTable](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2918-L2966)).

The full restriction set, with where each is enforced:

| Restriction | Behavior | Where enforced |
|---|---|---|
| Inside a transaction block | error: `REINDEX CONCURRENTLY cannot run inside a transaction block` | [indexcmds.c:2720-2722](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722) |
| During recovery (standby) | error: `cannot execute REINDEX during recovery` | [utility.c:280-296](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296), [utility.c:581-582](../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582) |
| Temporary table/index | falls back to a **non-concurrent** reindex | [indexcmds.c:2824-2833](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2824-L2833), [indexcmds.c:2940-2958](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2940-L2958) |
| System catalog target, or `REINDEX SYSTEM CONCURRENTLY` | error: `cannot reindex system catalogs concurrently` | [indexcmds.c:3516-3519](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3516-L3519), [indexcmds.c:3646-3649](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3646-L3649) |
| Partitioned table or index | **supported** — recurses into leaf partitions | [indexcmds.c#ReindexPartitions](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301) |
| Exclusion-constraint index named directly | error: `concurrent index creation for exclusion constraints is not supported` | [index.c:1328-1331](../../../raw/postgres-17/src/backend/catalog/index.c#L1328-L1331) |
| Exclusion-constraint index reached via a table | warning, skipped | [indexcmds.c:3555-3560](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3555-L3560) |
| Invalid index reached via a table | warning, skipped | [indexcmds.c:3548-3554](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3548-L3554) |
| Invalid index named directly | **allowed** (this is how you repair one) | [indexcmds.c:3695-3701](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3695-L3701) |
| Invalid index on a TOAST table named directly | error: `cannot reindex invalid index on TOAST table` | [indexcmds.c:3656-3660](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3656-L3660) |

When the target is a table, matview, or toast relation, RIC gathers every valid,
non-exclusion index plus the relation's **toast** indexes and processes them all
together; when the target is an index, it processes just that one
([indexcmds.c#gather-indexes](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3496-L3716)).
A directly named partitioned table/index never reaches the concurrent core —
`ReindexRelationConcurrently` errors with `cannot reindex this type of relation
concurrently` if one ever did
([indexcmds.c:3708-3714](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3708-L3714)).

For a **partitioned** target, `ReindexPartitions` re-checks the transaction-block
rule, collects the physical leaf partitions with `find_all_inheritors` under
`ShareLock`, and hands them to `ReindexMultipleInternal`, which reindexes each
leaf **in its own transaction**; a concurrent run calls
`ReindexRelationConcurrently` per leaf with `REINDEXOPT_MISSING_OK`
([indexcmds.c#ReindexPartitions](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301),
[indexcmds.c#ReindexMultipleInternal](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3311-L3410)).

### The six phases

The phase comment in the source is the canonical map
([indexcmds.c#phase-overview](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3737-L3751)).
Each phase loops over **all** indexes before the next phase begins.

#### Phase 1: create the catalog copies and take session locks

For each index, RIC switches to the table owner's userid and calls
`RestrictSearchPath()` (so index/predicate functions run safely), determines
whether the index is `safe` (no expressions, no partial predicate), chooses the
temporary name `<orig>_ccnew`, picks the target tablespace, and calls
`index_concurrently_create_copy`
([indexcmds.c:3786-3836](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3786-L3836)).
That helper rebuilds the new index's `IndexInfo` from the old index's catalog
rows and creates the catalog entry with
`INDEX_CREATE_SKIP_BUILD | INDEX_CREATE_CONCURRENT` — so it has identity but no
data and is **not ready, not valid**; exclusion constraints are rejected here
([index.c#index_concurrently_create_copy](../../../raw/postgres-17/src/backend/catalog/index.c#L1298-L1469),
[index.c:1442-1462](../../../raw/postgres-17/src/backend/catalog/index.c#L1442-L1462)).

RIC then takes a **session-level** `ShareUpdateExclusiveLock` on the old index,
the new index, and the heap table(s) (including toast), saving a `LOCKTAG` per
heap for the later waits, and commits
([indexcmds.c#index-locks](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3863-L3868),
[indexcmds.c#heap-locks](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3902-L3934)).
These session locks survive the upcoming commits so nobody can drop the relations
mid-rebuild. No `PROC_IN_SAFE_IC` flag is set here because this transaction takes
no snapshot
([indexcmds.c:3940-3943](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3940-L3943)).

#### Phase 2: wait, then build each new index

**Wait 1** is `WaitForLockersMultiple(lockTags, ShareLock, true)`: it waits out
the current transactions holding locks that conflict with `ShareLock`, so no
running writer can still have the table open without the new index in its index
list
([indexcmds.c:3955-3957](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3955-L3957)).
Then, in a **separate transaction per index**, RIC advertises `PROC_IN_SAFE_IC`
if the index is safe, takes a fresh snapshot, and calls
`index_concurrently_build`, which scans the heap, builds the new copy, and sets
`indisready = true`
([indexcmds.c:3960-3997](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3960-L3997),
[index.c#index_concurrently_build](../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1540)).
This is the same build CIC uses.

#### Phase 3: wait, validate, then wait for old snapshots

**Wait 2** is another `WaitForLockersMultiple(lockTags, ShareLock, true)`, so
every current `ShareLock`-conflicting transaction finishes before validation
assumes new writers see the new copy as ready
([indexcmds.c:4014-4016](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4014-L4016)).
Then, per new index, RIC registers a **reference snapshot**, runs `validate_index`
to backfill any tuples the build scan missed, saves the snapshot's `xmin` as
`limitXmin`, commits, and in a fresh transaction does **Wait 3** =
`WaitForOlderSnapshots(limitXmin, true)` to wait out transactions whose snapshot
predates the reference snapshot
([indexcmds.c:4019-4088](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4019-L4088)).
At the end of phase 3 the new copy contains every interesting tuple but is
**still invalid** — its `indisvalid` is not flipped here.

#### Phase 4: swap each old index with its new copy

In a single transaction (which sets `PROC_IN_SAFE_IC` unconditionally because it
only manipulates the catalog), RIC chooses a name `<orig>_ccold` for the old
index, calls `index_concurrently_swap`, invalidates the table's relcache, and
issues a `CommandCounterIncrement` so the next index in the loop sees the new
names
([indexcmds.c:4102-4163](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4163)).
`index_concurrently_swap` does the heavy lifting in one transaction
([index.c#index_concurrently_swap](../../../raw/postgres-17/src/backend/catalog/index.c#L1549-L1809)):

- swaps the `relname`s — the new copy takes the original's name, the old index
  becomes `<orig>_ccold` — and swaps `relispartition`
  ([index.c:1591-1601](../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1601));
- copies the constraint flags `indisprimary`, `indisexclusion`, `indimmediate`
  from old to new, and preserves `indisreplident` / `indisclustered` on the new
  ([index.c:1625-1636](../../../raw/postgres-17/src/backend/catalog/index.c#L1625-L1636));
- marks the **new index valid and the old index invalid** at the same time via the
  **transactional** `CatalogTupleUpdate`, clearing `indisclustered` /
  `indisreplident` on the old
  ([index.c:1642-1648](../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1648));
- moves constraints and their triggers, the comment, the partition inheritance
  link, and all dependencies from the old index to the new
  ([index.c:1653-1792](../../../raw/postgres-17/src/backend/catalog/index.c#L1653-L1792));
- copies the cumulative statistics and the planner column statistics from old to
  new via `pgstat_copy_relation_stats` and `CopyStatistics`
  ([index.c:1794-1798](../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798)).

The relcache invalidation makes every session re-plan against the rebuilt index
after this commit.

#### Phase 5: wait for readers, then mark old indexes dead

**Wait 4** is `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c:4180-4182](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4182)).
Because `AccessExclusiveLock` conflicts with every lock mode including
`AccessShareLock`, this waits out current table lock holders, including
**readers**. The wait is conservative: a reader need not actually use the old
index to be in the wait set, because the wait is on the heap relation lock tag
([lock.c#AccessExclusive-conflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)).
Then `index_concurrently_set_dead` transfers predicate locks to the heap and sets
`indisready = false, indislive = false` (dead) on each old index
([indexcmds.c:4184-4196](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4184-L4196),
[index.c#index_concurrently_set_dead](../../../raw/postgres-17/src/backend/catalog/index.c#L1820-L1854)).

#### Phase 6: wait for readers, then drop old indexes

After another `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c:4214-4216](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4216)),
RIC drops the old indexes with
`performMultipleDeletions(objects, DROP_RESTRICT, PERFORM_DELETION_CONCURRENT_LOCK | PERFORM_DELETION_INTERNAL)`
([indexcmds.c:4218-4244](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4218-L4244)).
`PERFORM_DELETION_CONCURRENT_LOCK` becomes `concurrent_lock_mode` in the
dependency deletion path, so `index_drop` takes `ShareUpdateExclusiveLock` (not
`AccessExclusiveLock`) on the heap and index while **skipping** its own
`if (concurrent)` set-dead/two-wait branch — RIC already did the dead-marking and
reader waits itself
([dependency.c#doDeletion-index](../../../raw/postgres-17/src/backend/catalog/dependency.c#L1356-L1368),
[index.c#index_drop-lockmode](../../../raw/postgres-17/src/backend/catalog/index.c#L2154-L2196)).
Finally RIC releases all the session locks
([indexcmds.c:4249-4254](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4249-L4254)).

### All steps and locks required on the table

Two lock layers act on each **heap table**, exactly as in CIC: a
**transaction-level** `ShareUpdateExclusiveLock` re-taken inside each helper, and
a **session-level** `ShareUpdateExclusiveLock` that spans the commit gaps. RIC
additionally holds session `ShareUpdateExclusiveLock` on both the **old and new
index** relations
([indexcmds.c#session-locks](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3928-L3934)).
DML never conflicts with any of these — `ShareUpdateExclusiveLock` does not
conflict with `RowExclusiveLock`
([lock.c#SUE-conflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80)).

The waits are **not** table locks. `WaitForLockersMultiple` reads the current set
of conflicting lock holders and sleeps on each holder's virtual transaction ID
until it ends; later transactions are not waited for
([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L903-L979)).
The conflict mode passed to the wait decides who is waited out:

| Phase | Relation locks held by RIC | Wait | Conflict mode | Who it waits out |
|---|---|---|---|---|
| 1: create copies | transaction, then session `ShareUpdateExclusiveLock` on heap and indexes | — | — | — |
| 2: before build | session `ShareUpdateExclusiveLock`; build later takes heap `ShareUpdateExclusiveLock` and new-index `RowExclusiveLock` | Wait 1 | `ShareLock` | current holders conflicting with `ShareLock` — open writers and stronger table operations ([lock.c:82-85](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L85)) |
| 3: before validation | session `ShareUpdateExclusiveLock` | Wait 2 | `ShareLock` | same `ShareLock` conflict set |
| 3: after validation | session `ShareUpdateExclusiveLock` | Wait 3 | `WaitForOlderSnapshots` | same-database **old-snapshot** holders, ignoring autovacuum, lazy `VACUUM`, and safe concurrent index builds ([indexcmds.c:433-442](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442)) |
| 5: before set-dead | session `ShareUpdateExclusiveLock` | Wait 4 | `AccessExclusiveLock` | current table lock holders, including **readers** ([lock.c:98-102](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)) |
| 6: before drop | session `ShareUpdateExclusiveLock` | Wait 5 | `AccessExclusiveLock` | current table lock holders, including **readers** |

So RIC has the same three CIC waits (`ShareLock`, `ShareLock`, old snapshots)
**plus** two reader-capable waits that CIC does not have. Those two reader waits
exist because the old index may still be usable by transactions that planned
before the swap, so RIC cannot remove it until current table lock holders that
might touch it have finished
([reindex.sgml#concurrent-steps](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L382-L443)).

Because `ShareUpdateExclusiveLock` is self-conflicting, only one concurrent build
(CIC or RIC), `VACUUM`, or `ANALYZE` can run on a given table at a time
([lock.c#self-conflict](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L78)).

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
([index.c#swap-mark-valid](../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1648)).
The old index is set dead by `index_set_state_flags(INDEX_DROP_SET_DEAD)`, whose
assert requires the index already be invalid — which the swap guarantees
([index.c#INDEX_DROP_SET_DEAD](../../../raw/postgres-17/src/backend/catalog/index.c#L3500-L3514)).
In v17, **all** `index_set_state_flags` writes are made with the transactional
`CatalogTupleUpdate`, so they roll back with their transaction and become visible
via cache invalidation on commit
([index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521)).
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
([index.c:1591-1648](../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1648)).
The source comment states the intent — mark new valid and old invalid "at the
same time to make sure we only get constraint violations from the indexes with
the correct names"
([indexcmds.c:4096-4100](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4096-L4100)).
So a pre-swap abort discards the `_ccnew` copy and leaves the original valid (even
if bloated); a post-swap failure can only happen once the rebuilt `index_name` is
already valid. The invalid leftover is always a **differently named** index
(`_ccnew` or `_ccold`), never the bare `index_name`. The one exception is when
`index_name` was **already invalid before** RIC ran (the repair case) — then a new
pre-swap failure leaves it invalid only because it started that way.

Invalid leftovers are ignored for query planning: `get_relation_info` closes and
skips an index whose `indisvalid` is false
([plancat.c#skip-invalid-index](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)).
They can still matter for **writes**, because `RelationGetIndexList` omits only
indexes with `indislive = false`, and executor insertion skips only indexes whose
`ii_ReadyForInserts` is false. So a ready-but-invalid `_ccnew` or `_ccold` can
still impose write/HOT-safety overhead even though the planner will not choose it
([relcache.c#RelationGetIndexList](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4800-L4869),
[execIndexing.c#ExecInsertIndexTuples](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359)).
RIC's failure paths are ordinary ERROR/cancel/timeout aborts. Each phase loop
runs inside a transaction precisely so that an abort cleans up the session locks
rather than leaking them
([indexcmds.c:3967-3972](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3967-L3972)).

A unique `_ccnew` that reaches phase 3 enforces uniqueness while it is ready but
invalid, so a duplicate appearing during the second scan surfaces as a uniqueness
violation — which is how the documented "leaves an invalid index" failure arises
([reindex.sgml#failure-recovery](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L446-L478)).

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
([index.c#swap-names](../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1593),
[index.c#swap-mark-valid](../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1648)).
The whole phase-4 loop runs inside one transaction, so the rename and the
validity flip commit or roll back together
([indexcmds.c#phase4-txn](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4163)).
The source comment states the intent: mark the new valid and the old invalid "at
the same time to make sure we only get constraint violations from the indexes
with the correct names"
([indexcmds.c:4096-4100](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4096-L4100)).

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
([plancat.c#skip-invalid-index](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)),
and in neither committed state is `index_name` the invalid one. A leftover
`_ccnew`/`_ccold` that is ready but invalid can still add write/HOT overhead until
you drop it, as the
[failure table](#failure-scenarios-and-the-outcome-on-the-table) notes, but it
does not remove your usable index.

Unlike v12, v17 makes **every** `pg_index` flag write transactional — the
set-ready (phase 2), the swap's validity flip (phase 4), and the set-dead (phase
5) all end in `CatalogTupleUpdate`, not the old in-place `heap_inplace_update`
([index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521);
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
([create_index.out#both-invalid](../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2716-L2734)).
Dropping the `_ccnew` spare, deleting the duplicate, and re-running RIC makes
`concur_reindex_ind5` valid again
([create_index.out#repair](../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2736-L2760)).

### Watching the phases

RIC reports through `pg_stat_progress_create_index` with `command` =
`REINDEX CONCURRENTLY`
([indexcmds.c#progress-command](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3810-L3816)).
The wait phases map to the view's `phase` text via the integer codes in
`progress.h`
([progress.h#phases](../../../raw/postgres-17/src/include/commands/progress.h#L92-L114),
[system_views.sql#create-index-view](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277)):

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
([indexcmds.c:4214-4215](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4215)).

### What changed from PostgreSQL 12

The six-phase, five-wait shape, the `ShareUpdateExclusiveLock` footprint, and the
`indislive`/`indisready`/`indisvalid` progression are **unchanged** from v12. (For
the v12 trace, see
[How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../v12/questions/reindex-index-concurrently.md).)
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
([indexcmds.c#ReindexPartitions](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301),
[indexcmds.c#ReindexMultipleInternal](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3364-L3374),
[reindex.sgml#partitions](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334)).
Two earlier v13 fixes to the swap support this: `relispartition` is now swapped
(`d80be6f2f6c`) and partition dependencies are swapped (`68ac9cf2499`)
([index.c:1595-1601](../../../raw/postgres-17/src/backend/catalog/index.c#L1595-L1601),
[index.c:1769-1792](../../../raw/postgres-17/src/backend/catalog/index.c#L1769-L1792)).

**2. `REINDEX (TABLESPACE ...)` (PostgreSQL 14).** Commit `c5b286047cd` added a
`TABLESPACE` option; `ExecReindex` parses it into `params.tablespaceOid`, and the
phase-1 copy is created in the target tablespace (toast indexes are not moved).
For partitioned targets, only leaf-partition tablespaces are updated
([indexcmds.c:2732-2750](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2732-L2750),
[indexcmds.c:3825-3836](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3825-L3836),
[reindex.sgml#tablespace](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334)).

**3. `index_set_state_flags` is now transactional (PostgreSQL 14).** In v12 the
set-ready and set-dead flag writes used `heap_inplace_update` — a non-transactional
in-place overwrite that could not roll back. Commit `83158f74d3a` ("Make
index_set_state_flags() transactional") switched it to `CatalogTupleUpdate`, so in
v17 those flips obey MVCC, roll back with their transaction, and publish via cache
invalidation on commit
([index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3445-L3518)).
The phase-4 validity flip was already transactional in v12; this change makes the
phase-2 (`SET_READY`) and phase-5 (`SET_DEAD`) writes transactional as well.

**4. `PROC_IN_SAFE_IC` lets other concurrent builds ignore a running RIC
(PostgreSQL 14).** v12's RIC did not advertise itself, so two concurrent index
operations on different tables would wait on each other's snapshots. PostgreSQL 14
(commit `f9900df5`, "Avoid spurious wait in concurrent reindex") made RIC call
`set_indexsafe_procflags()` in phases 2 and 3 when the index is safe (no
expressions, no partial predicate) and unconditionally in phase 4
([indexcmds.c:3974-3976](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3974-L3976),
[indexcmds.c:4034-4036](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4034-L4036),
[indexcmds.c:4109](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4109)).
`WaitForOlderSnapshots` then excludes `PROC_IN_SAFE_IC` processes from the
snapshot wait, alongside autovacuum and lazy `VACUUM`
([indexcmds.c:433-442](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442),
[indexcmds.c#set_indexsafe_procflags](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4487)).
This only shortens **Wait 3** (the snapshot wait); the lock-based Waits 1, 2, 4,
and 5 are unaffected.

**5. Progress-view phase fix (PostgreSQL 14).** Commit `e66bcfb4c66` ("Fix
progress reporting of REINDEX CONCURRENTLY") corrected phase 6 to set `WAIT_5`
instead of reusing `WAIT_4`, so `pg_stat_progress_create_index` now shows
`waiting for readers before dropping` during the final drop wait
([indexcmds.c:4214-4215](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4215),
[system_views.sql:1276-1277](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1276-L1277)).

**6. Statistics preservation reworked (PostgreSQL 15).** v12's swap hand-copied a
few cumulative counters into `newClassRel->pgstat_info->t_counts`. After the
PostgreSQL 15 shared-memory statistics rework (commit `8ea7963fc74`, "pgstat: add
pgstat_copy_relation_stats()"), the swap calls `pgstat_copy_relation_stats` to
move the cumulative counters and `CopyStatistics` to copy the planner's
`pg_statistic` column statistics (which matters for expression indexes)
([index.c:1794-1798](../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798)).

**7. `MAINTAIN` privilege (PostgreSQL 17).** v12 required table ownership (or
superuser) to reindex. PostgreSQL 17 (commit `ecb0fd33720`, "Reintroduce MAINTAIN
privilege and pg_maintain predefined role") lets a non-owner reindex if granted
`MAINTAIN`; the index lookup callback checks `ACL_MAINTAIN`
([indexcmds.c#RangeVarCallbackForReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2887-L2896)).

**8. Security hardening and testability (post-v12).** v17 switches to the table
owner's userid and calls `RestrictSearchPath()` before running index/predicate
functions, both when creating the copy and inside `index_concurrently_build`
([indexcmds.c:3786-3790](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3786-L3790),
[index.c:1503-1507](../../../raw/postgres-17/src/backend/catalog/index.c#L1503-L1507));
v12 did neither. v17 also adds `reindex-conc-index-safe` / `-not-safe` injection
points for deterministic testing
([indexcmds.c:3796-3801](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3796-L3801)).

**9. Dispatch refactor (PostgreSQL 14, mechanical).** v12 dispatched REINDEX
inline in `standard_ProcessUtility` and passed an `options` int. v17 routes
through `ExecReindex` with `ReindexStmt` / `ReindexParams` structs and a
`REINDEXOPT_*` bitmask, and prevents REINDEX during recovery generically via
`ClassifyUtilityCommandAsReadOnly` rather than an explicit
`PreventCommandDuringRecovery("REINDEX")`
([indexcmds.c#ExecReindex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781),
[utility.c:280-296](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296)).

### Test coverage

- Functional coverage is in the `REINDEX CONCURRENTLY` block of
  `create_index.sql`: it reindexes tables, matviews, a single index, and toast
  indexes; checks dependency and comment preservation; rebuilds expression /
  predicate indexes; exercises clustered and replica-identity indexes; and tests
  the exclusion error-vs-skip
  ([create_index.sql#ric-block](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L976)).
- Partitioned coverage: the same file reindexes leaf partition tables and indexes
  concurrently, reindexes a **partitioned index** concurrently
  (`REINDEX INDEX CONCURRENTLY concur_reindex_part_index`), and asserts the
  TABLE-vs-INDEX object-type errors
  ([create_index.sql#partitions](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L978-L1116)).
- Restriction and repair coverage: system-catalog rejection, `REINDEX SYSTEM
  CONCURRENTLY`, and the invalid-index repair (an index left invalid by a failed
  CIC, then repaired by RIC)
  ([create_index.sql#system-and-repair](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L1131-L1165)).
- Isolation coverage: `reindex-concurrently.spec` runs one reading transaction,
  one writing transaction, and a `REINDEX TABLE CONCURRENTLY` session across six
  permutations
  ([reindex-concurrently.spec](../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and the last ~20
  `wiki/log.md` entries (navigation only).
- The sibling page
  [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
  (shared build/validate core and `PROC_IN_SAFE_IC` discussion) and the v12
  counterpart
  [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../v12/questions/reindex-index-concurrently.md)
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
- Tests: `src/test/regress/sql/create_index.sql` and
  `src/test/isolation/specs/reindex-concurrently.spec`.
- v12 vs v17 deltas established from the `raw/postgres-17/` checkout's own commit
  history (`a6642b3ae06`, `c5b286047cd`, `83158f74d3a`, `f9900df5`,
  `e66bcfb4c66`, `8ea7963fc74`, `ecb0fd33720`, `d80be6f2f6c`, `68ac9cf2499`),
  each verified present with `git show` and first-release-tagged with
  `git tag --contains` against `REL_NN_0`.

## Evidence Map

| Claim | Source |
|---|---|
| RIC runs as six phases (create copy, build, catch up, swap, set dead, drop), each phase looping all indexes | [indexcmds.c:3737-3751](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3737-L3751) |
| `CONCURRENTLY` cannot run in a transaction block; recovery blocked via read-only classification | [indexcmds.c:2720-2722](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2720-L2722), [utility.c:280-296](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296), [utility.c:581-582](../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582) |
| `ExecReindex` parses options into `ReindexParams` and dispatches by kind | [indexcmds.c:2693-2781](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781) |
| `ReindexIndex` locks SUE for concurrent; partitioned -> `ReindexPartitions`; temp -> non-concurrent | [indexcmds.c:2788-2834](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2788-L2834) |
| Partitioned target recurses into leaf partitions, each reindexed concurrently in its own transaction | [indexcmds.c:3217-3301](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301), [indexcmds.c:3364-3374](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3364-L3374) |
| Directly named partitioned relation never reaches the concurrent core | [indexcmds.c:3708-3714](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3708-L3714) |
| System catalog rejected; invalid/exclusion via table skipped; invalid named directly allowed | [indexcmds.c:3516-3519](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3516-L3519), [indexcmds.c:3548-3560](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3548-L3560), [indexcmds.c:3695-3701](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3695-L3701) |
| Exclusion index named directly errors in `index_concurrently_create_copy` | [index.c:1328-1331](../../../raw/postgres-17/src/backend/catalog/index.c#L1328-L1331) |
| Phase 1 creates `_ccnew` not-ready/not-valid via the skip-build + concurrent create flags | [index.c:1395](../../../raw/postgres-17/src/backend/catalog/index.c#L1395), [index.c:1442-1462](../../../raw/postgres-17/src/backend/catalog/index.c#L1442-L1462) |
| Session SUE taken on old index, new index, and heap(s) | [indexcmds.c:3863-3934](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3863-L3934) |
| Phase 2: Wait 1 (`ShareLock`) then per-index build sets `indisready` | [indexcmds.c:3955-3997](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3955-L3997), [index.c:1482-1540](../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1540) |
| Phase 3: Wait 2 (`ShareLock`), `validate_index`, then Wait 3 (`WaitForOlderSnapshots`) | [indexcmds.c:4014-4088](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4014-L4088) |
| Phase 4 swap: new valid + old invalid via transactional `CatalogTupleUpdate`, names/constraints/deps/stats moved | [indexcmds.c:4102-4163](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4102-L4163), [index.c:1549-1809](../../../raw/postgres-17/src/backend/catalog/index.c#L1549-L1809) |
| Swap copies cumulative + planner stats via `pgstat_copy_relation_stats` + `CopyStatistics` | [index.c:1794-1798](../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798) |
| Phase 5: Wait 4 (`AccessExclusiveLock`) then `index_concurrently_set_dead` | [indexcmds.c:4180-4196](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4196), [index.c:1820-1854](../../../raw/postgres-17/src/backend/catalog/index.c#L1820-L1854) |
| Phase 6: Wait 5 (`AccessExclusiveLock`) then `performMultipleDeletions` drops old | [indexcmds.c:4214-4244](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4244) |
| Drop uses `PERFORM_DELETION_CONCURRENT_LOCK` -> `index_drop` takes SUE, skips its own concurrent branch | [dependency.c:1356-1368](../../../raw/postgres-17/src/backend/catalog/dependency.c#L1356-L1368), [index.c:2154-2196](../../../raw/postgres-17/src/backend/catalog/index.c#L2154-L2196) |
| `AccessExclusiveLock` conflicts with `AccessShareLock`, so phases 5/6 wait out readers | [lock.c:98-102](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102) |
| `ShareLock` waits (Waits 1/2) wait out `RowExclusiveLock` writers and stronger (not other `ShareLock` holders) | [lock.c:82-85](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L85) |
| `ShareUpdateExclusiveLock` self-conflicts; does not conflict with `RowExclusiveLock` | [lock.c:77-80](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80) |
| Wait 3 ignores autovacuum, lazy VACUUM, and safe concurrent builds (`PROC_IN_SAFE_IC`) | [indexcmds.c:433-442](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442) |
| RIC advertises `PROC_IN_SAFE_IC` in phases 2/3 (if safe) and 4 (unconditional) | [indexcmds.c:3974-3976](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3974-L3976), [indexcmds.c:4034-4036](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4034-L4036), [indexcmds.c:4109](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4109) |
| `index_set_state_flags` is transactional (`CatalogTupleUpdate`); `INDEX_DROP_SET_DEAD` asserts not valid | [index.c:3449-3521](../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521) |
| Invalid indexes skipped by planner but can still affect writes while live/ready | [plancat.c:260-266](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266), [relcache.c:4800-4869](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4800-L4869), [execIndexing.c:356-359](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359) |
| A RIC failure never leaves a healthy `index_name` invalid: phase 4 renames the rebuilt copy to `index_name` and flips validity in one transaction; the invalid leftover is always `_ccnew`/`_ccold`. `index_name` is invalid only if it was already invalid (repair case) | [index.c:1591-1648](../../../raw/postgres-17/src/backend/catalog/index.c#L1591-L1648), [indexcmds.c:4096-4163](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4096-L4163), [create_index.out:2716-2760](../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2716-L2760) |
| Progress: command `REINDEX CONCURRENTLY`; phase 6 correctly sets `WAIT_5` (fixed vs v12) | [indexcmds.c:4214-4215](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4214-L4215), [progress.h:92-114](../../../raw/postgres-17/src/include/commands/progress.h#L92-L114), [system_views.sql:1264-1277](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277) |
| v14 added partitioned REINDEX (`a6642b3ae06`), TABLESPACE (`c5b286047cd`), transactional flags (`83158f74d3a`), PROC_IN_SAFE_IC for RIC (`f9900df5`), progress fix (`e66bcfb4c66`) | checkout git history; [reindex.sgml:319-334](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334) |
| v15 stats rework (`8ea7963fc74`); v17 MAINTAIN privilege (`ecb0fd33720`) | [index.c:1794-1798](../../../raw/postgres-17/src/backend/catalog/index.c#L1794-L1798), [indexcmds.c:2887-2896](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2887-L2896) |
| Docs: two scans, wait for transactions, six steps, `_ccnew`/`_ccold` recovery, partitions | [reindex.sgml:366-478](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L366-L478) |
| Tests: functional, partitioned, restriction/repair, isolation | [create_index.sql:882-1165](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165), [reindex-concurrently.spec:1-40](../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40) |

## Open Questions

- The exact crash / immediate-shutdown outcome at each instruction boundary was
  not traced through crash recovery. In v17 the swap validity flip and the
  set-ready / set-dead flips all go through transactional `CatalogTupleUpdate`
  ([index.c:3518](../../../raw/postgres-17/src/backend/catalog/index.c#L3518)), so
  the v12 "non-transactional in-place" recovery hazard no longer applies to these
  flag writes; the precise recovered catalog state mid-rebuild was still not
  isolated here.
- The sibling [v17 CIC page](create-index-concurrently.md) currently describes
  `index_set_state_flags` as a "non-transactional in-place update". That matches
  v12 but not v17 source, where the function ends in `CatalogTupleUpdate`
  ([index.c:3445-3518](../../../raw/postgres-17/src/backend/catalog/index.c#L3445-L3518));
  the two pages should be reconciled the next time the CIC page is revised.
- Whether two indexes sharing a long base name can collide on the `_ccnew` /
  `_ccold` names is mitigated by `ChooseRelationName` plus the per-iteration
  `CommandCounterIncrement`
  ([indexcmds.c:4152-4159](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4152-L4159)),
  but the exact truncation/uniqueness behavior of `ChooseRelationName` was not
  traced.
- The first build scan's exact tuple-visibility rule lives inside `index_build` ->
  `table_index_build_scan` (via `ii_Concurrent`), summarized from the shared CIC
  core rather than traced line-by-line into the table AM
  ([index.c:1518-1522](../../../raw/postgres-17/src/backend/catalog/index.c#L1518-L1522)).

## Source References

- [indexcmds.c#ExecReindex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2693-L2781)
- [indexcmds.c#ReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2788-L2834)
- [indexcmds.c#ReindexTable](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2918-L2966)
- [indexcmds.c#ReindexPartitions](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3217-L3301)
- [indexcmds.c#ReindexMultipleInternal](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3311-L3410)
- [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3437-L4295)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442)
- [indexcmds.c#set_indexsafe_procflags](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4487)
- [index.c#index_concurrently_create_copy](../../../raw/postgres-17/src/backend/catalog/index.c#L1298-L1469)
- [index.c#index_concurrently_build](../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1540)
- [index.c#index_concurrently_swap](../../../raw/postgres-17/src/backend/catalog/index.c#L1549-L1809)
- [index.c#index_concurrently_set_dead](../../../raw/postgres-17/src/backend/catalog/index.c#L1820-L1854)
- [index.c#index_drop](../../../raw/postgres-17/src/backend/catalog/index.c#L2114-L2196)
- [index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521)
- [dependency.c#doDeletion](../../../raw/postgres-17/src/backend/catalog/dependency.c#L1352-L1368)
- [plancat.c#skip-invalid-index](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)
- [relcache.c#RelationGetIndexList](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4798-L4869)
- [execIndexing.c#ExecInsertIndexTuples](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359)
- [lmgr.c#WaitForLockersMultiple](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L903-L979)
- [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)
- [utility.c#ClassifyUtilityCommandAsReadOnly](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296)
- [progress.h#create-index-phases](../../../raw/postgres-17/src/include/commands/progress.h#L92-L114)
- [system_views.sql#pg_stat_progress_create_index](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277)
- [ref/reindex.sgml#CONCURRENTLY](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L478)
- [create_index.sql#ric-block](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165)
- [reindex-concurrently.spec](../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40)

## Navigation

- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../v12/questions/reindex-index-concurrently.md)
- [v17 index](../index.md)
- [versions](../../versions.md)
