---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Working model and logic map](#working-model-and-logic-map)
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
- [Measurement Script](#measurement-script)
  - [Usage and non-default conditions](#usage-and-non-default-conditions)
  - [The script](#the-script)
  - [Last run](#last-run)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Give a comprehensive explanation of how REINDEX INDEX CONCURRENTLY is
implemented in PostgreSQL 17, including the steps, locks, failure scenarios,
and what changed since PostgreSQL 12.

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

[`REINDEX INDEX CONCURRENTLY`](../../../glossary.md#reindex) (RIC) rebuilds a
physical index while allowing ordinary `INSERT`/`UPDATE`/`DELETE` relation locks.
For an initially valid index, it preserves a valid index under the original
name across its commits. A partitioned target first goes through a separate
discovery stage that can block writes; see the [dispatch section](#dispatch-preconditions-and-restrictions).
These are relation-lock and catalog-validity guarantees, not a guarantee that a
writer never waits on an index operation or another queued lock request
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104),
[lock.c#queued-conflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L1032-L1043),
[indexcmds.c#partition-discovery](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3310),
[index.c#swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665)).
It builds a **brand-new copy** of the index next to the original,
catches that copy up to concurrent writes, atomically **swaps** the copy in for
the original, and only then marks the original dead and drops it. The build and
catch-up reuse the exact [`CREATE INDEX CONCURRENTLY`](../../../glossary.md#concurrently) (CIC) machinery; the
swap-and-drop tail is what makes a reindex.

All of it runs in `ReindexRelationConcurrently()`, which lays out **six phases**
across many short transactions
([indexcmds.c#ReindexRelationConcurrently](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3453-L4313)):

1. create the new index copies in the catalog (not built),
2. build each new copy,
3. let each new copy catch up and validate it,
4. swap each new copy in for its old index (new becomes valid, old invalid),
5. mark the old indexes dead, and
6. drop the old indexes.

The concurrent core holds a table **[`ShareUpdateExclusiveLock`](../../../glossary.md#shareupdateexclusivelock)** — the same level
CIC, `VACUUM` (non-FULL), and `ANALYZE` use, weak enough to let DML proceed but
strong enough to keep out a second concurrent build, `VACUUM`, `ANALYZE`, and
schema changes requiring a conflicting lock
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)).

The algorithm is the same six-phase shape PostgreSQL 12 used, but several things
changed since v12 — most visibly, RIC now works on **partitioned** indexes and
tables, can move indexes to a new **[tablespace](../../../glossary.md#tablespace)**, advertises [`PROC_IN_SAFE_IC`](../../../glossary.md#proc_in_safe_ic)
so other concurrent builds can ignore it, makes its [`pg_index`](../../../glossary.md#pg_index) flag writes
**transactional**, and fixes a progress-view phase bug. See
[What changed from PostgreSQL 12](#what-changed-from-postgresql-12).

### Working model and logic map

The old index remains available while the replacement is incomplete. The
replacement's catalog flags tell other sessions when to maintain it and when to
plan queries with it. Each change becomes visible after its transaction commits
([index.c#concurrent-build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1526-L1557),
[index.c#swap-flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1665)).

| Value | Meaning and origin | Where it goes next |
|---|---|---|
| Old/new index OIDs and heap OID | Object identities saved in `ReindexIndexInfo` and cross-transaction lists | Each phase reopens the same objects; names change at the swap ([indexcmds.c#cross-transaction-state](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3455-L3503), [indexcmds.c#new-index-state](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3862-L3895)). |
| `indislive`, `indisready`, `indisvalid` | Stored [`pg_index`](../../../glossary.md#pg_index) flags: alive, maintained by writes, eligible for queries | The relcache, executor, and planner consume the committed state ([relcache.c#index-list](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4802-L4871), [execIndexing.c#ready-check](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359), [plancat.c#validity](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L284)). |
| Build and reference [snapshots](../../../glossary.md#snapshot) | Separate [MVCC](../../../glossary.md#mvcc) views taken for the first scan and validation | The first scan indexes its visible rows; validation fills gaps for its later reference snapshot ([heapam_handler.c#build-snapshot](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1279), [indexcmds.c#reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4056-L4082)). |
| `limitXmin` | Saved `xmin` of the validation snapshot, not a fresh value at wait time | Wait 3 excludes transactions that could still require older row versions ([indexcmds.c#save-and-wait](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4080-L4106)). |
| Heap `LOCKTAG`s and session locks | Object identities for waits, plus live locks retained across commits | Waits collect existing conflicting holders; session locks prevent concurrent removal ([indexcmds.c#session-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3914-L3954), [lmgr.c#wait-set](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973)). |

```text
Target
  temporary? -> ordinary, non-concurrent REINDEX
  partitioned? -> discover leaves under ShareLock; commit discovery
                 -> run the physical-index/table path for each surviving leaf
  physical non-temporary target:
    Phase 1: create not-ready, invalid copies; take session locks; COMMIT
    Wait 1: existing ShareLock-conflicting holders; COMMIT
    Phase 2, per copy: first snapshot -> build -> set ready; COMMIT
    Wait 2: existing ShareLock-conflicting holders; COMMIT
    Phase 3, per copy:
      reference snapshot -> index TID scan/sort + heap gap-fill; COMMIT
      saved reference xmin -> Wait 3 for older snapshots; COMMIT
    Phase 4: rename + new-valid/old-invalid + dependency swap; COMMIT
    Phase 5: Wait 4 for existing table holders -> old not-ready/dead; COMMIT
    Phase 6: Wait 5 for existing table holders -> drop old; COMMIT
    release session locks; start caller's final transaction

Abort before creation commits -> no new copy from that attempt
Abort after creation, before swap commits -> invalid _ccnew; old unchanged
Abort after swap, before drop commits -> valid replacement + invalid _ccold
```

The dispatch branches are in
[ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2838-L2850)
and [ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3310).
The phase and commit edges are in the
[build/validation sequence](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3944-L4106)
and [swap/retirement sequence](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4275).
The abort branches follow those commit boundaries and the
[transactional swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665).
`_ccnew` and `_ccold` denote generated names: `ChooseRelationName` can adjust
them for length or collisions
([indexcmds.c#copy-name](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3834-L3839),
[indexcmds.c#old-name](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4142-L4147)).

### How it differs from CREATE INDEX CONCURRENTLY

RIC is CIC plus a swap and a drop. The build (`index_concurrently_build`) and the
validation (`validate_index`) are literally the same functions CIC uses; see the
[v17 CIC page](create-index-concurrently.md) for that shared core. The
differences:

| Aspect | CREATE INDEX CONCURRENTLY | REINDEX INDEX CONCURRENTLY |
|---|---|---|
| Target | a new index that did not exist | a fresh copy `<name>_ccnew` of an existing index |
| Phases / waits | build, validate, mark valid (3 waits) | create copy, build, validate, swap, set-dead, drop (**5 waits**) |
| Extra waits | none beyond the snapshot wait | two `AccessExclusiveLock` waits that **wait out readers** of the old index before set-dead and drop ([indexcmds.c:4198-4234](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4198-L4234)) |
| Names | the index keeps its name | new copy takes the original's name; the old index is renamed `<name>_ccold` ([index.c#swap-names](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1610), [indexcmds.c#ccold](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4142-L4147)) |
| On failure | leaves the **target** index invalid | no new copy survives an abort before phase 1 commits; later failures can leave an [invalid](../../../glossary.md#invalid-index) `_ccnew` (before the swap commits) or `_ccold` (after it commits); see the [failure table](#failure-scenarios-and-the-outcome-on-the-table) for the commit boundaries and evidence |

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
([utility.c:1567-1568](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1567-L1568),
[indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2709-L2797)).
Because the concurrent path commits many times, it cannot run inside a
transaction block:

```c
if (concurrently)
    PreventInTransactionBlock(isTopLevel, "REINDEX CONCURRENTLY");
```

([indexcmds.c:2736-2738](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2736-L2738)).
`REINDEX` is also forbidden during recovery, but in v17 that is enforced
generically: `ClassifyUtilityCommandAsReadOnly` returns only
`COMMAND_OK_IN_READ_ONLY_TXN` (not `COMMAND_OK_IN_RECOVERY`) for a `ReindexStmt`,
so `standard_ProcessUtility` calls `PreventCommandDuringRecovery`
([utility.c#classify-reindex](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296),
[utility.c:581-582](../../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582)).

`REINDEX INDEX` routes through `ReindexIndex`, which locks the index in
`ShareUpdateExclusiveLock` for the concurrent case (vs `AccessExclusiveLock` for
a plain reindex). It then dispatches in three ways
([indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2850)):

- a **partitioned** index goes to `ReindexPartitions` (new since v12);
- a concurrent, non-temporary index goes to `ReindexRelationConcurrently`;
- otherwise (including a temporary index, even if `CONCURRENTLY` was requested) it
  falls back to the non-concurrent `reindex_index`.

`REINDEX TABLE` behaves the same way via `ReindexTable`, emitting
`table "..." has no indexes that can be reindexed concurrently` when nothing
qualifies
([indexcmds.c#ReindexTable](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2934-L2982)).

The full restriction set, with where each is enforced:

| Restriction | Behavior | Where enforced |
|---|---|---|
| Inside a transaction block | error: `REINDEX CONCURRENTLY cannot run inside a transaction block` | [indexcmds.c:2736-2738](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2736-L2738) |
| During recovery (standby) | error: `cannot execute REINDEX during recovery` | [utility.c:280-296](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296), [utility.c:581-582](../../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582) |
| Temporary table/index | falls back to a **non-concurrent** reindex | [indexcmds.c:2840-2849](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2840-L2849), [indexcmds.c:2956-2974](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2956-L2974) |
| System catalog target, or `REINDEX SYSTEM CONCURRENTLY` | error: `cannot reindex system catalogs concurrently` | [indexcmds.c:3532-3535](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3532-L3535), [indexcmds.c:3662-3665](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3662-L3665) |
| Partitioned table or index | **supported** — recurses into leaf partitions | [indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3233-L3317) |
| Exclusion-constraint index named directly | error: `concurrent index creation for exclusion constraints is not supported` | [index.c:1338-1341](../../../../raw/postgres-17/src/backend/catalog/index.c#L1338-L1341) |
| Exclusion-constraint index reached via a table | warning, skipped | [indexcmds.c:3571-3576](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3571-L3576) |
| Invalid index reached via a table | warning, skipped | [indexcmds.c:3564-3570](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3564-L3570) |
| Invalid index named directly | **allowed** (this is how you repair one) | [indexcmds.c:3711-3717](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3711-L3717) |
| Invalid index on a TOAST table named directly | error: `cannot reindex invalid index on TOAST table` | [indexcmds.c:3672-3676](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3672-L3676) |

When the target is a table, [materialized view](../../../glossary.md#materialized-view), or [TOAST](../../../glossary.md#toast) relation, RIC gathers every valid,
non-exclusion index plus the relation's valid **TOAST** indexes and processes them all
together; when the target is an index, it processes just that one
([indexcmds.c#gather-indexes](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3512-L3732)).
A directly named partitioned table/index never reaches the concurrent core —
`ReindexRelationConcurrently` errors with `cannot reindex this type of relation
concurrently` if one ever did
([indexcmds.c:3724-3730](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3724-L3730)).

For a **partitioned** target, `ReindexPartitions` re-checks the transaction-block
rule, collects the physical leaf partitions with `find_all_inheritors` under
`ShareLock`, and hands them to `ReindexMultipleInternal`, which reindexes each
leaf separately; a concurrent run calls
`ReindexRelationConcurrently` per leaf with `REINDEXOPT_MISSING_OK`
([indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3233-L3317),
[indexcmds.c#ReindexMultipleInternal](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3327-L3426)).

This discovery is an important exception to the concurrent core's lock behavior.
`find_all_inheritors` takes `ShareLock` on descendant relations before any leaf
rebuild starts. For a partitioned table, these are table locks; for a partitioned
index, these are index locks. Both can block writes because writers acquire
`RowExclusiveLock` on the table and its indexes. Discovery releases its locks
at the initial commit in `ReindexMultipleInternal`; each concurrent leaf rebuild
then uses the six-phase, multiple-transaction core. Therefore a partitioned RIC
is neither free of write-blocking discovery nor one transaction per leaf
([pg_inherits.c#child-locks](../../../../raw/postgres-17/src/backend/catalog/pg_inherits.c#L204-L230),
[execIndexing.c#writer-index-lock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L190-L204),
[lock.c#ShareLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L86),
[indexcmds.c#discovery-commit-and-leaf-call](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3327-L3390)).

### The six phases

The phase comment in the source is the canonical map
([indexcmds.c#phase-overview](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3753-L3767)).
Each phase loops over **all** indexes before the next phase begins.

#### Phase 1: create the catalog copies and take session locks

For each index, RIC switches to the table owner's userid and calls
`RestrictSearchPath()` (so index/predicate functions run safely), determines
whether the index is `safe` (no expressions, no partial predicate), chooses the
temporary name `<orig>_ccnew`, picks the target tablespace, and calls
`index_concurrently_create_copy`
([indexcmds.c:3802-3852](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3802-L3852)).
That helper rebuilds the new index's `IndexInfo` from the old index's catalog
rows and creates the catalog entry with
`INDEX_CREATE_SKIP_BUILD | INDEX_CREATE_CONCURRENT` — so it has identity but no
data and is **not ready, not valid**; exclusion constraints are rejected here
([index.c#index_concurrently_create_copy](../../../../raw/postgres-17/src/backend/catalog/index.c#L1306-L1486),
[index.c:1459-1479](../../../../raw/postgres-17/src/backend/catalog/index.c#L1459-L1479)).
The helper adds a third flag, `INDEX_CREATE_DEFERRABLE`, when the old index's
`pg_index.indimmediate` is false, so the copy of a deferrable unique or
primary-key index is itself created with `indimmediate = false` and does not
force immediate constraint checks when concurrent writers begin maintaining it
after its phase-2 build commits. Before that commit it is not ready, so the
executor skips it. Afterward, `indimmediate = false` selects a tentative unique
check; the original deferred constraint still governs when the write must be
valid. This matters through validation, not only during the build phase
([index.c:1348-1352](../../../../raw/postgres-17/src/backend/catalog/index.c#L1348-L1352),
[index.h#INDEX_CREATE_DEFERRABLE](../../../../raw/postgres-17/src/include/catalog/index.h#L68),
[index.c#UpdateIndexRelation-immediate](../../../../raw/postgres-17/src/backend/catalog/index.c#L1050-L1057),
[index.c#publish-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1547-L1557),
[execIndexing.c#ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359),
[execIndexing.c#unique-mode](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L402-L428)).
That flag arrived in 17.11 with commit `28269fed661` ("Fix propagation of
indimmediate flag in index_create_copy()", back-patched through v14); before it
the helper passed no deferrable information, so every copy was immediate.

RIC then takes a **session-level** `ShareUpdateExclusiveLock` on the old index,
the new index, and the heap table(s) (including toast), saving a `LOCKTAG` per
heap for the later waits, and commits
([indexcmds.c#index-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3879-L3884),
[indexcmds.c#heap-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3918-L3950)).
These session locks survive the upcoming commits so nobody can drop the relations
mid-rebuild. Phase 1 ends by popping its active snapshot and committing. The
next transaction performs Wait 1 without taking a snapshot; the source's
"no snapshot" explanation for omitting `PROC_IN_SAFE_IC` refers to that
wait-only transaction, not to phase 1
([indexcmds.c#creation-to-wait-boundary](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L3974)).

#### Phase 2: wait, then build each new index

**Wait 1** is `WaitForLockersMultiple(lockTags, ShareLock, true)`: it waits out
the current transactions holding locks that conflict with `ShareLock`, so no
running writer can still have the table open without the new index in its index
list
([indexcmds.c:3971-3973](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3971-L3973)).
Then, in a **separate transaction per index**, RIC advertises `PROC_IN_SAFE_IC`
if the index is safe, takes a fresh snapshot, and calls
`index_concurrently_build`, which scans the heap, builds the new copy, and sets
`indisready = true`
([indexcmds.c:3976-4013](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3976-L4013),
[index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1499-L1557)).
This is the same build CIC uses.

#### Phase 3: wait, validate, then wait for old snapshots

**Wait 2** is another `WaitForLockersMultiple(lockTags, ShareLock, true)`, so
every current `ShareLock`-conflicting transaction finishes before validation
assumes new writers see the new copy as ready
([indexcmds.c:4032-4034](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4032-L4034)).
Then, per new index, RIC registers a **reference snapshot** (the MVCC view defining
which rows the validation must cover), runs `validate_index`
to backfill any tuples the build scan missed, saves the snapshot's `xmin` as
`limitXmin`, commits, and in a fresh transaction does **Wait 3** =
`WaitForOlderSnapshots(limitXmin, true)` to wait out transactions whose snapshot
predates the reference snapshot
([indexcmds.c:4037-4106](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4037-L4106)).
At the end of phase 3 the new copy contains every interesting tuple but is
**still invalid** — its `indisvalid` is not flipped here.

#### Phase 4: swap each old index with its new copy

In a single transaction (which sets `PROC_IN_SAFE_IC` unconditionally because it
only manipulates the catalog), RIC chooses a name `<orig>_ccold` for the old
index, calls `index_concurrently_swap`, invalidates the table's relcache, and
issues a `CommandCounterIncrement` so the next index in the loop sees the new
names
([indexcmds.c:4120-4181](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4181)).
`index_concurrently_swap` does the heavy lifting in one transaction
([index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1566-L1826)):

- swaps the `relname`s — the new copy takes the original's name, the old index
  becomes `<orig>_ccold` — and swaps `relispartition`
  ([index.c:1608-1618](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1618));
- copies the constraint flags `indisprimary`, `indisexclusion`, `indimmediate`
  from old to new, and preserves `indisreplident` / `indisclustered` on the new
  ([index.c:1642-1653](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1653));
- marks the **new index valid and the old index invalid** at the same time via the
  **transactional** `CatalogTupleUpdate`, clearing `indisclustered` /
  `indisreplident` on the old
  ([index.c:1659-1665](../../../../raw/postgres-17/src/backend/catalog/index.c#L1659-L1665));
- moves constraints and their triggers, the comment, the partition inheritance
  link, and all dependencies from the old index to the new
  ([index.c:1670-1809](../../../../raw/postgres-17/src/backend/catalog/index.c#L1670-L1809));
- copies the cumulative statistics and the planner column statistics from old to
  new via `pgstat_copy_relation_stats` and `CopyStatistics`
  ([index.c:1811-1815](../../../../raw/postgres-17/src/backend/catalog/index.c#L1811-L1815)).

The [relcache](../../../glossary.md#relcache) invalidation makes sessions refresh
affected cached plans after this commit; already-running users of the old index
are handled by the following reader waits
([indexcmds.c#invalidation-and-reader-wait](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4163-L4200)).

#### Phase 5: wait for readers, then mark old indexes dead

**Wait 4** is `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c:4198-4200](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4198-L4200)).
Because `AccessExclusiveLock` conflicts with every lock mode including
`AccessShareLock`, this waits out current table lock holders, including
**readers**. The wait is conservative: a reader need not actually use the old
index to be in the wait set, because the wait is on the heap relation lock tag
([lock.c#AccessExclusive-conflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)).
Then `index_concurrently_set_dead` transfers [predicate locks](../../../glossary.md#predicate-lock) to the heap and sets
`indisready = false, indislive = false` (dead) on each old index
([indexcmds.c:4202-4214](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4202-L4214),
[index.c#index_concurrently_set_dead](../../../../raw/postgres-17/src/backend/catalog/index.c#L1837-L1871)).

#### Phase 6: wait for readers, then drop old indexes

After another `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c:4232-4234](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4232-L4234)),
RIC drops the old indexes with
`performMultipleDeletions(objects, DROP_RESTRICT, PERFORM_DELETION_CONCURRENT_LOCK | PERFORM_DELETION_INTERNAL)`
([indexcmds.c:4236-4262](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4236-L4262)).
`PERFORM_DELETION_CONCURRENT_LOCK` becomes `concurrent_lock_mode` in the
dependency deletion path, so `index_drop` takes `ShareUpdateExclusiveLock` (not
`AccessExclusiveLock`) on the heap and index while **skipping** its own
`if (concurrent)` set-dead/two-wait branch — RIC already did the dead-marking and
reader waits itself
([dependency.c#doDeletion-index](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1356-L1368),
[index.c#index_drop-lockmode](../../../../raw/postgres-17/src/backend/catalog/index.c#L2171-L2213)).
Finally RIC releases all the session locks
([indexcmds.c:4267-4272](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4267-L4272)).

### GUCs that affect REINDEX INDEX CONCURRENTLY performance

[GUCs](../../../glossary.md#guc) are server configuration settings. Their
[contexts](../../../glossary.md#guc-context) determine whether a change can be
made in a session, needs a reload, or needs a restart; the tables below name
the scope for each setting.

For a typical [B-tree](../../../glossary.md#b-tree) RIC, examine `maintenance_work_mem`,
`max_parallel_maintenance_workers`, `min_parallel_table_scan_size`,
`max_parallel_workers`, and `max_worker_processes` first. The first setting
controls build and validation memory. The other four determine whether the first
heap scan requests parallel workers and whether those workers can launch. Only
B-tree and [BRIN](../../../glossary.md#brin) support [parallel builds](../../../glossary.md#parallel-index-build) in v17
([index.c#parallel-AM-boundary](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
[planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7018)).

RIC reuses the CIC build and validation functions, but it has more commit and
wait exposure. Within one `ReindexRelationConcurrently` invocation over `N`
indexes, the source has six fixed internal commits plus three per index: one
after each build, one after each validation scan, and one after each old-snapshot
wait. A one-index RIC therefore executes nine internal commits, then starts the
transaction that the utility-command caller finishes. It also has five blocker
waits, including two reader waits that CIC lacks. Here "five" counts the named
wait points for one index: with `N` indexes there are `N + 4` wait calls because
Wait 3 runs once per index
([indexcmds.c#build-and-validation-commits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4106),
[indexcmds.c#swap-set-dead-drop-commits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4275),
[postgres.c#command-end-commit](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2807)).

This section uses a bounded meaning of “performance impact”: the setting must
change a core RIC phase, a shipped access method, worker availability, heap or
temporary I/O, WAL/commit latency, or a command wait. Index expressions and
third-party access methods can execute arbitrary code and read additional GUCs;
the core dispatch invokes the access method's `ambuild` callback, so no
core-source list can cover those additions
([amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L98-L108),
[index.c#ambuild-dispatch](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3054)).

**Application scope used below.** “Session” means a `PGC_USERSET` setting, or a
`PGC_SUSET` setting changed by an authorized role, with no reload or restart.
Although those contexts can also be transaction-local, a session value is the
usable per-run scope: RIC cannot run inside an explicit transaction, and its own
commits end transaction-local values. Parallel workers restore the leader's
serialized GUC state
([indexcmds.c#transaction-block-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2736-L2738),
[ref/set.sgml#SET-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L35-L60),
[ref/set.sgml#LOCAL](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L100-L117),
[parallel.c#GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L376-L385),
[parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1449-L1458)).

**Build, scan, and storage settings.**

| GUC | Exact RIC effect and boundary | Default and application |
|---|---|---|
| `maintenance_work_mem` | The main direct control. Phase 2 passes through the AM's build algorithm; phase 3 uses this setting for the serial encoded-TID validation sort. The automatic B-tree/BRIN worker request also reserves at least 32 MB per participant, including the leader ([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1533-L1539), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3434), [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)). | 64 MB; session; no reload or restart ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2460-L2474)). |
| `max_parallel_maintenance_workers` | Caps the workers requested for a B-tree or BRIN first build. Zero disables parallel utility workers. It is a request cap, not a guarantee that workers launch ([planner.c#worker-request-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L7018)). | 2; session; no reload or restart ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)). |
| `min_parallel_table_scan_size` | The automatic model rejects a heap below this threshold, then adds workers as heap size crosses successive threefold thresholds. A table `parallel_workers` reloption bypasses both this size model and the 32 MB request test, but remains capped by `max_parallel_maintenance_workers` ([allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4278), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L7012)). | 8 MB; session; no reload or restart ([guc_tables.c#parallel-scan-thresholds](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3539)). |
| `max_parallel_workers`; `max_worker_processes` | The first caps live parallel workers when RIC registers them. The second sizes the shared background-worker slot array. Pool pressure can launch fewer workers than requested; B-tree and BRIN then continue with the participants that launched or fall back to serial execution ([bgworker.c#worker-slot-pool](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L146-L175), [bgworker.c#parallel-and-slot-caps](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L969-L1043), [nbtsort.c#parallel-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594)). | Both default to 8. `max_parallel_workers` is session-scoped; `max_worker_processes` is `postmaster` context and needs restart ([guc_tables.c#max_worker_processes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3163-L3172), [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3438)). |
| `effective_cache_size` | Only [GiST](../../../glossary.md#gist)'s buffered build reads it. Together with `maintenance_work_mem`, it limits the buffered subtree depth; it does not reserve cache or control the other shipped AMs ([gistbuild.c#gistInitBuffering](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L757)). | 524,288 blocks, normally 4 GB at 8 kB per block; session; no reload or restart ([cost.h#DEFAULT_EFFECTIVE_CACHE_SIZE](../../../../raw/postgres-17/src/include/optimizer/cost.h#L31-L35), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)). |
| `shared_buffers` | Its `NBuffers` value changes two decisions: each heap scan gets the bulk-read path only when the heap exceeds `NBuffers / 4`, and a permanent hash-index build caps its sort-selection threshold at `NBuffers`. More shared buffers can therefore move either threshold; it is not a monotonic RIC speed control ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L496), [hash.c#hashbuild-sort-choice](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)). | 16,384 blocks, normally 128 MB; `postmaster` context; restart ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2257-L2270)). |
| `io_combine_limit` | Both heap scans use v17's sequential read stream. This setting caps adjacent blocks combined into one read, and each scan captures it when its stream starts ([heapam.c#sequential-read-stream](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1233-L1259), [read_stream.c#combined-read-limit](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L360-L376), [read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L522-L537)). | Normally 128 kB on an 8 kB build, subject to the platform maximum; session; no reload or restart ([guc_tables.c#io_combine_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3138-L3150), [bufmgr.h#DEFAULT_IO_COMBINE_LIMIT](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L170)). |
| `effective_io_concurrency` | The heap stream selects the heap tablespace's effective-I/O value because the heap AM passes `READ_STREAM_SEQUENTIAL`, not `READ_STREAM_MAINTENANCE`. Sequential mode disables explicit prefetch advice and drives look-ahead toward `io_combine_limit`, so this is a secondary source-visible boundary, not a primary RIC lever ([read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L407-L478), [read_stream.c#sequential-advice-boundary](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L508-L536), [read_stream.c#sequential-distance](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L704-L728)). | 1 when prefetch support is compiled in, otherwise 0; session; no reload or restart. A tablespace reloption overrides the GUC ([bufmgr.h#I/O-concurrency-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L166), [guc_tables.c#effective_io_concurrency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3109-L3121), [spccache.c#tablespace-I/O-override](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L207-L223)). |
| `synchronize_seqscans` | On a sufficiently large heap, the first scan can start at another synchronized scan's location and wrap around; it still reads every page. B-tree, hash, GiST, SP-GiST, Bloom, and parallel BRIN permit this. GIN and serial BRIN require physical order and disable it. Validation always disables it ([heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496), [gininsert.c#no-syncscan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L378-L384), [brin.c#serial-scan-order](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1217-L1224), [heapam_handler.c#validation-order](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1805)). | On; session; no reload or restart ([guc_tables.c#synchronize_seqscans](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1765-L1774)). |
| `temp_tablespaces`; `temp_file_limit` | `temp_tablespaces` places tuplesort spills, parallel worker tapes, and GiST buffered-build files. `temp_file_limit` does not throttle RIC: each process errors when its accounted temporary files cross the limit. Such an error occurs before the swap and can leave an invalid `_ccnew` while preserving the original index ([tuplesort.c#PrepareTempTablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1966), [fileset.c#parallel-temp-tablespaces](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L35-L70), [gistbuildbuffers.c#temporary-buffer-file](../../../../raw/postgres-17/src/backend/access/gist/gistbuildbuffers.c#L40-L58), [fd.c#temp-file-limit](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L2211-L2236), [indexcmds.c#pre-swap-phases](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3962-L4117)). | `temp_tablespaces` defaults to the database default tablespace and is session-scoped. `temp_file_limit` defaults to unlimited (`-1`), is `PGC_SUSET`, and is session-scoped for an authorized role; neither needs reload or restart ([guc_tables.c#temp_file_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2504-L2513), [guc_tables.c#temp_tablespaces](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4148-L4156)). |
| Writer-side `gin_pending_list_limit`; `work_mem` | Neither sizes the initial [GIN](../../../glossary.md#gin) build. After phase 2 makes the new index ready, concurrent writers can append to its pending list. Their `gin_pending_list_limit`, unless the copied index reloption overrides it, decides when regular cleanup starts; regular cleanup uses the writer's `work_mem`. Phase 3 forces cleanup with the RIC backend's `maintenance_work_mem`, so writer settings can change how much work validation inherits ([index.c#copy-reloptions](../../../../raw/postgres-17/src/backend/catalog/index.c#L1362-L1367), [index.c#apply-reloptions](../../../../raw/postgres-17/src/backend/catalog/index.c#L1459-L1479), [gin_private.h#pending-list-limit](../../../../raw/postgres-17/src/include/access/gin_private.h#L23-L45), [ginfast.c#regular-cleanup-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828), [ginvacuum.c#validation-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)). | Both default to 4 MB and are session-scoped with no reload or restart; the relevant values are in concurrent writer sessions ([guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2458), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)). |

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
([indexcmds.c#RIC-tablespace-selection](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3841-L3852),
[guc_tables.c#default_tablespace](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4137-L4146)).
RIC processes each index build and validation in its own transaction rather than
building all selected indexes in parallel, so one command does not multiply
`maintenance_work_mem` by its number of indexes at a single instant; a B-tree or
BRIN build can still divide that one operation's budget among its parallel
participants
([indexcmds.c#per-index-builds](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3962-L4013),
[indexcmds.c#per-index-validations](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4025-L4107),
[ref/create_index.sgml#parallel-memory](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L807-L844)).

**[WAL](../../../glossary.md#wal), [checkpoints](../../../glossary.md#checkpoint), and commit latency.** Permanent RIC builds generate WAL.
B-tree and sorted GiST bulk writing log forced page images; GiST, GIN, and
SP-GiST can log the built page range after the first build
([bulk_write.c#bulk-WAL](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L310),
[gistbuild.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L328-L337),
[gininsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L408-L417),
[spginsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L132-L141)).

| GUC | Exact RIC effect and boundary | Default and application |
|---|---|---|
| `synchronous_commit`; `synchronous_standby_names` | WAL-writing internal transactions can flush locally and wait for a synchronous standby at commit. `synchronous_commit = off` permits eligible commits to be asynchronous and disables synchronous-replication waits, but phase 6's non-temporary relation deletion still forces a local WAL flush. Wait-only transactions that assign no XID or write no WAL have nothing to flush ([indexcmds.c#phase-6-drop](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4232-L4262), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.c#sync-rep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1537-L1546), [syncrep.h#SyncRepRequested](../../../../raw/postgres-17/src/include/replication/syncrep.h#L18-L25)). | `synchronous_commit` defaults to `on`; session; no reload or restart. `synchronous_standby_names` defaults empty; `sighup` context; reload ([guc_tables.c#synchronous_commit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4925-L4933), [guc_tables.c#synchronous_standby_names](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4571-L4580)). |
| `wal_compression` | Forced and ordinary full-page images can be compressed, trading CPU for fewer WAL bytes when pages compress well; the result is page- and codec-dependent ([xloginsert.c#full-page-compression](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L695), [xloginsert.c#forced-page-images](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1169-L1223)). | Off; `PGC_SUSET`; session for an authorized role; no reload or restart ([guc_tables.c#wal_compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4976-L4983)). |
| `wal_buffers`; `wal_sync_method` | These are generic WAL-path controls: the first sizes shared WAL buffering for RIC's WAL stream; the second selects the local method used to force WAL to disk. Neither changes worker count or scan shape ([guc_tables.c#wal_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2891-L2900), [guc_tables.c#wal_sync_method](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5026-L5033)). | `wal_buffers` defaults to `-1` (automatic), is `postmaster` context, and needs restart. `wal_sync_method` uses the platform default, is `sighup` context, and needs reload. |
| `max_wal_size`; `checkpoint_timeout`; `checkpoint_completion_target`; `checkpoint_flush_after` | The first two influence automatic checkpoint frequency; the latter two pace checkpoint writes and writeback. If a checkpoint starts during an AM bulk write, `smgr_bulk_finish()` immediately syncs the relation because the checkpoint missed earlier writes. These cluster-wide effects are not monotonic ([bulk_write.c#checkpoint-crossing-sync](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223), [guc_tables.c#checkpoint-size-time-flush](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2842-L2888), [guc_tables.c#checkpoint_completion_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3916-L3924)). | Normally 1 GB, 5 minutes, `0.9`, and a platform default respectively; all are `sighup` context and need reload, not restart. |
| `commit_delay`; `commit_siblings` | When enabled and enough other transactions are active, the WAL flush path deliberately sleeps before group commit. This can improve aggregate throughput while adding latency to affected RIC commits; defaults add no delay ([xlog.c#group-commit-delay](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2869-L2888)). | `commit_delay = 0`; `PGC_SUSET`, authorized session. `commit_siblings = 5`; session. Neither needs reload or restart ([guc_tables.c#commit-delay](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2980-L3001)). |

Disabling `fsync` or `full_page_writes` can reduce generic durability I/O, but it
is not a safe production RIC tuning method. Both are `sighup` settings and need
reload. In addition, `REGBUF_FORCE_IMAGE` makes bulk-build page images
independent of `full_page_writes`, so turning that setting off does not remove
those records
([guc_tables.c#fsync-and-full_page_writes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1096-L1107),
[guc_tables.c#full_page_writes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1156-L1168),
[xloginsert.c#force-image-wins](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L610)).

**Wait and cancellation settings.** These settings do not make either scan
faster. They bound or expose the initial relation lock and RIC's five waits:

| GUC | RIC behavior | Default and application |
|---|---|---|
| `statement_timeout` | Covers the client statement across all internal commits, scans, and five waits. RIC calls `StartTransactionCommand` directly, so it does not restart the statement timer; the normal command-end path disables it ([postgres.c#statement-timeout-lifetime](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2807), [postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5278)). | 0 (disabled); session; no reload or restart ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)). |
| `transaction_timeout` | Starts for every internal transaction and is disabled at each commit. It is a per-internal-transaction cap, not a whole-RIC cap; expiry terminates the connection with `FATAL` ([xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2168-L2178), [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2307-L2317), [postgres.c#transaction-timeout-error](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3463)). | 0 (disabled); session; no reload or restart ([guc_tables.c#transaction_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653)). |
| `lock_timeout` | Applies to initial relation-lock acquisition and to each individual VXID lock used by all five waits. It restarts for each lock acquisition, so it is not an aggregate limit for a wait phase, blocker set, or whole command ([indexcmds.c#initial-index-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2823-L2827), [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L972), [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L492), [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4662), [proc.c#lock-timeout-timer](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1273-L1307)). | 0 (disabled); session; no reload or restart ([guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)). |
| `deadlock_timeout`; `log_lock_waits` | The first controls when a blocked lock acquisition runs deadlock detection; lowering it does not end a non-deadlocked wait. The second logs waits that survive that interval. They are detection and observability controls, not throughput controls ([proc.c#deadlock-and-lock-timeouts](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1273-L1325), [proc.c#long-wait-logging](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1491-L1525)). | 1 second and off; both are `PGC_SUSET`, so an authorized role can use session scope with no reload or restart ([guc_tables.c#deadlock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2146-L2157), [guc_tables.c#log_lock_waits](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1512-L1521)). |
| Blocker-side `idle_in_transaction_session_timeout` | It does not fire in the actively running RIC backend. In another session it can terminate an idle-in-transaction reader or writer that blocks RIC; that is blocker policy, not a scan-speed setting ([postgres.c#idle-in-transaction-timer](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4616-L4646)). | 0 (disabled); session in the blocker; no reload or restart ([guc_tables.c#idle_in_transaction_session_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2632-L2642)). |

A timeout or cancellation after phase 1 commits can leave the committed transient index
behind. Before the swap commits it is the invalid `_ccnew`; after that commit the
rebuilt index has the original name and an invalid `_ccold` can remain. Timeouts
are therefore completion policy, not free acceleration
([indexcmds.c#phase-boundaries](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4262),
[index.c#atomic-swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665)).

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
  ([heapam.c#RIC-read-stream-flags](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L425-L439)).
- The RIC backend's `work_mem`, `autovacuum_work_mem`, and `vacuum_cost_*`
  settings do not size or throttle the common validation path. Validation uses
  `maintenance_work_mem`; its forced GIN cleanup chooses that value for a normal
  RIC backend. A unique B-tree allocates a secondary `work_mem` spool during the
  first build, but the concurrent MVCC scan supplies no dead tuples and discards
  it empty. `vacuum_delay_point()` returns without delaying when
  `VacuumCostActive` is false
  ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3434),
  [ginfast.c#forced-cleanup-memory](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828),
  [nbtsort.c#secondary-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L505),
  [heapam_handler.c#concurrent-tuples-are-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1621-L1629),
  [vacuum.c#vacuum_delay_point](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2383-L2419)).
- `wal_level = minimal` and `wal_skip_threshold` do not unlock the usual
  new-relation WAL skip for a permanent RIC copy. Phase 1 commits the catalog
  creation before phase 2 builds it, so the index is no longer new in the build
  transaction and `RelationNeedsWAL()` remains true
  ([indexcmds.c#catalog-then-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3841-L4012),
  [rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L621-L631)).
- `track_activities`, `trace_sort`, `log_temp_files`, and `track_counts` expose
  progress or spills rather than changing RIC's correctness algorithm.
  `track_io_timing` and `track_wal_io_timing` add measurements but can add
  significant clock-call overhead on some systems; both are `PGC_SUSET`, default
  off, and can use authorized session scope without reload or restart
  ([backend_progress.c#track_activities](../../../../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L20-L60),
  [guc_tables.c#trace_sort](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1659-L1669),
  [fd.c#temp-file-reporting](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1524-L1538),
  [pgstat_database.c#temp-file-accounting](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_database.c#L171-L185),
  [guc_tables.c#I/O-timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1420-L1436),
  [config.sgml#I/O-timing-overhead](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8451-L8499)).

In practice, tune and measure in this order: (1) `maintenance_work_mem`; (2) for
B-tree or BRIN, the full request-and-availability chain; (3) AM-specific GiST or
GIN behavior; (4) heap, temporary, and explicit output-tablespace I/O; and (5)
WAL, checkpoints, and commit confirmation. Diagnose the five blocker waits
separately, because memory and workers cannot shorten them
([ref/create_index.sgml#memory-and-parallelism](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L862),
[ref/reindex.sgml#concurrent-cost](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L366-L379),
[indexcmds.c#RIC-waits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3962-L4234)).
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

The phase rows below describe the physical concurrent core. A partitioned
target has the preceding discovery row as well; those locks end before leaf
rebuilding starts
([indexcmds.c#discovery-boundary](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3345)).

Two lock layers act on each **heap table**, exactly as in CIC: a
**transaction-level** `ShareUpdateExclusiveLock` re-taken inside each helper, and
a **session-level** `ShareUpdateExclusiveLock` that spans the commit gaps. RIC
additionally holds session `ShareUpdateExclusiveLock` on both the **old and new
index** relations
([indexcmds.c#session-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3944-L3950)).
DML never conflicts with any of these — `ShareUpdateExclusiveLock` does not
conflict with `RowExclusiveLock`
([lock.c#SUE-conflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80)).

The waits are **not** table locks. `WaitForLockersMultiple` reads the current set
of conflicting lock holders and sleeps on each holder's [virtual transaction ID](../../../glossary.md#virtual-transaction-id)
until it ends; later transactions are not waited for
([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L903-L979)).
The conflict mode passed to the wait decides who is waited out:

| Phase | Relation locks held by RIC | Wait | Conflict mode | Who it waits out |
|---|---|---|---|---|
| Partition discovery, if applicable | `ShareLock` on descendant tables or indexes, released at the discovery commit | — | actual relation lock | conflicts with writers' `RowExclusiveLock`; this precedes the concurrent core ([indexcmds.c#discovery](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3345), [execIndexing.c#writer-lock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L190-L204)) |
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
`indisvalid` — but on **two** indexes at once. For an initially live, ready,
valid original, their committed states progress like this. A directly named
invalid index instead retains its original invalid state until the swap
([indexcmds.c#invalid-direct-target](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3710-L3717),
[index.c#swap-flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1665)):

| Step | New copy (`_ccnew`) | Old index |
|---|---|---|
| after phase 1 | live, not ready, **not valid**; also `indimmediate = false` when the old index backs a deferrable constraint | live, ready, valid |
| after phase 2 build | live, **ready**, not valid | unchanged |
| after phase 3 validate | live, ready, not valid (data complete) | unchanged |
| after phase 4 swap | live, ready, **valid** (takes original name); `indimmediate` copied from the old index | **invalid** (renamed `_ccold`), `indimmediate` reset to `true` |
| after phase 5 set-dead | unchanged | **dead** (`indisready=false`, `indislive=false`) |
| after phase 6 drop | the surviving index | gone |

The new copy's `indisready` is set by the shared `index_set_state_flags`
(`INDEX_CREATE_SET_READY`); its `indisvalid` is flipped by the swap, not by
`index_set_state_flags`
([index.c#swap-mark-valid](../../../../raw/postgres-17/src/backend/catalog/index.c#L1659-L1665)).
The old index is set dead by `index_set_state_flags(INDEX_DROP_SET_DEAD)`, whose
assert requires the index already be invalid — which the swap guarantees
([index.c#INDEX_DROP_SET_DEAD](../../../../raw/postgres-17/src/backend/catalog/index.c#L3529-L3543)).
In v17, **all** `index_set_state_flags` writes are made with the transactional
`CatalogTupleUpdate`, so they roll back with their transaction and become visible
via cache invalidation on commit
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3478-L3550)).
This is a behavior change from v12 (see
[What changed from PostgreSQL 12](#what-changed-from-postgresql-12)).

A fourth flag matters when the index backs a deferrable constraint. Phase 1
gives the `_ccnew` copy `indimmediate = false` for a deferrable old index, so
its row in the table above reads live, not ready, not valid **and not
immediate** from phase 1 onward
([index.c:1348-1352](../../../../raw/postgres-17/src/backend/catalog/index.c#L1348-L1352)).
Phase 4 then copies `indimmediate` from old to new and resets the old index to
`true`, in the same catalog update as the other constraint flags
([index.c:1646](../../../../raw/postgres-17/src/backend/catalog/index.c#L1646),
[index.c#swap-constraint-flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1653)).

### Failure scenarios and the outcome on the table

RIC commits its work in stages. **The phase-1 commit decides whether the new
copy survives an abort; the phase-4 commit decides which index has the original
name.** The build starts after the creation transaction commits, and the swap
commits in a later transaction
([indexcmds.c#creation-and-build-transactions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4012),
[indexcmds.c#swap-transaction](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4110-L4182)).

| Failure point | Leftover on the table | Recovery |
|---|---|---|
| Phase 1 (gather / create copies / initial lock), **before its commit** | **none from this attempt** — the creation transaction rolls back, so no new `_ccnew` persists; original untouched ([indexcmds.c#copy-creation](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3834-L3852), [indexcmds.c#creation-commit](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3944-L3954), [storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L108-L118)) | retry |
| Phase 2 (Wait 1 or build) | one or more **invalid `_ccnew`** copies; copies whose build has not committed are **not ready**, earlier committed builds are **ready**; original state unchanged ([indexcmds.c#build-transactions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3962-L4012), [index.c#build-set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1538-L1556)) | `DROP INDEX <name>_ccnew`, then retry |
| Phase 3 (Wait 2 / validate / Wait 3) | **invalid, ready `_ccnew`** copy; original state unchanged ([indexcmds.c#validation-transactions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4008-L4106), [index.c#build-set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1547-L1556)) | `DROP INDEX <name>_ccnew`, then retry |
| Phase 4 (swap), before its commit | swap transaction rolls back: `_ccnew` still invalid, original state unchanged and normally named ([indexcmds.c#swap-transaction](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4110-L4182), [index.c#swap-names-and-validity](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665)) | `DROP INDEX <name>_ccnew`, then retry |
| Phase 5 or 6, after the swap commits and before the drop commits | the **new** index is already valid and carries the original name; the **old** `_ccold` index lingers (invalid, possibly dead) ([index.c#swap-names-and-validity](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665), [indexcmds.c#retire-and-drop-old-index](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4190-L4262)) | `DROP INDEX <name>_ccold` |

The documented recovery is to drop an invalid `_ccnew` and retry, or to drop an
invalid `_ccold` because the rebuild has already succeeded
([reindex.sgml#failure-recovery](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L466-L478)).

**A failure never leaves `index_name` itself invalid for a healthy index.** The
swap in phase 4 is the only step that touches the original index's name or its
`indisvalid` flag, and it does both together in one transaction: it renames the
rebuilt copy to `index_name`, renames the original to `index_name_ccold`, and in
the same `CatalogTupleUpdate` flips `new.indisvalid = true`,
`old.indisvalid = false`
([index.c:1608-1665](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665)).
The source comment states the intent — mark new valid and old invalid "at the
same time to make sure we only get constraint violations from the indexes with
the correct names"
([indexcmds.c:4114-4118](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4114-L4118)).

The `_ccnew` copy disappears on abort only if its phase-1 creation transaction
has not committed. Once that transaction commits, a later abort rolls back the
current transaction's work without undoing the earlier creation. A failure in
phase 2 or 3, or in phase 4 before its commit, therefore leaves an invalid
`_ccnew` alongside the unchanged original
([indexcmds.c#creation-and-build-transactions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4012),
[indexcmds.c#swap-transaction](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4110-L4182),
[index.c#previously-committed-copy](../../../../raw/postgres-17/src/backend/catalog/index.c#L1526-L1534)).

For a healthy original, an abort before the swap commits leaves `index_name`
valid, even if it is bloated. After the swap commits, that name belongs to the
valid rebuilt index; any invalid leftover has a different name, `_ccnew` or
`_ccold`
([index.c#swap-names-and-validity](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665),
[indexcmds.c#swap-commit](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4182)).
If `index_name` was already invalid, an abort before the swap commits preserves
that invalid state; see the [repair case](#can-a-failure-leave-an-invalid-index-with-the-original-index-name)
and its regression evidence below.

Invalid leftovers are ignored for query planning: `get_relation_info` closes and
skips an index whose `indisvalid` is false
([plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)).
They can still matter for **writes**, because `RelationGetIndexList` omits only
indexes with `indislive = false`, and executor insertion skips only indexes whose
`ii_ReadyForInserts` is false. So a ready-but-invalid `_ccnew` or `_ccold` can
still impose write/[HOT](../../../glossary.md#hot)-safety overhead even though the planner will not choose it
([relcache.c#RelationGetIndexList](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4802-L4871),
[execIndexing.c#ExecInsertIndexTuples](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359)).
RIC's failure paths are ordinary ERROR/cancel/timeout aborts. Each phase loop
runs inside a transaction precisely so that an abort cleans up the session locks
rather than leaking them
([indexcmds.c:3983-3988](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3983-L3988)).

An immediate unique `_ccnew` that reaches phase 3 enforces uniqueness while it is ready but
invalid, so a duplicate appearing during the second scan surfaces as a uniqueness
violation — which is how the documented "leaves an invalid index" failure arises
([reindex.sgml#failure-recovery](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L446-L478)).

### Can a failure leave an invalid index with the original index name?

**For a healthy index, no.** A `REINDEX INDEX CONCURRENTLY index_name` failure
never leaves the table with `index_name` marked invalid and therefore no usable
index. The name `index_name` remains on a **valid** index, preserving its
eligibility with respect to the planner's validity check. The
`indcheckxmin` check and normal plan selection still apply
([plancat.c#index-eligibility](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L284)).
`index_name` itself is left invalid only if it was **already
invalid before** the command ran — the repair case below.

Why the name is safe: the **swap in phase 4 is the only step that touches the
original index's name or its `indisvalid` flag**, and it does both together. In a
single transaction, `index_concurrently_swap` renames the rebuilt copy to
`index_name` and the old index to `index_name_ccold`, and in the same call flips
`new.indisvalid = true`, `old.indisvalid = false` through the transactional
`CatalogTupleUpdate`
([index.c#swap-names](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1610),
[index.c#swap-mark-valid](../../../../raw/postgres-17/src/backend/catalog/index.c#L1659-L1665)).
The whole phase-4 loop runs inside one transaction, so the rename and the
validity flip commit or roll back together
([indexcmds.c#phase4-txn](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4181)).
The source comment states the intent: mark the new valid and the old invalid "at
the same time to make sure we only get constraint violations from the indexes
with the correct names"
([indexcmds.c:4114-4118](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4114-L4118)).

So every committed outcome keeps a valid `index_name`:

| When the failure happens | What `index_name` is | Invalid leftover |
|---|---|---|
| Phase 1, **before** its creation transaction commits | the **original** index — still valid under its original name | none from this attempt; creation rolls back ([indexcmds.c#creation-commit](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3944-L3954), [storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L108-L118)) |
| After phase 1 commits, but **before** phase 4 commits | the **original** index — name and `indisvalid` untouched, still valid even if bloated | a separate `index_name_ccnew` build copy, invalid; its creation already committed ([indexcmds.c#creation-and-build-transactions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4012), [indexcmds.c#swap-transaction](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4110-L4182)) |
| Phases 5-6, **after** the swap commits and before the drop commits | the **rebuilt** index — already marked valid under `index_name` | the renamed `index_name_ccold` old index, invalid ([index.c#swap-names-and-validity](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665), [indexcmds.c#retire-and-drop-old-index](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4190-L4262)) |

An abort before the swap commits preserves the original index's name and
validity. It does **not** remove a `_ccnew` whose creation committed in phase 1.
Even an abort during the swap rolls back the renames and validity changes, not
that earlier creation. This is why preserving a healthy `index_name` and
leaving an invalid `_ccnew` are compatible outcomes
([indexcmds.c#creation-commit](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L3954),
[indexcmds.c#swap-transaction](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4182),
[index.c#swap-names-and-validity](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665)).

This directly answers the bloat worry: you do **not** go from "one bloated but
valid `index_name`" to "no usable index." Before the swap the planner still has
the original (bloated) `index_name`; after the swap it has the fresh rebuild. The
planner skips indexes whose `indisvalid` is false, and can also exclude a valid
index while its `indcheckxmin` horizon check fails. Validity therefore preserves
eligibility with respect to the invalid-index check; it does not promise that
every transaction can use the index or that the planner will choose it
([plancat.c#validity-and-xmin-checks](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L284)).
In neither committed state is `index_name` the invalid one. A leftover
`_ccnew`/`_ccold` that is ready but invalid can still add write/HOT overhead until
you drop it, as the
[failure table](#failure-scenarios-and-the-outcome-on-the-table) notes, but it
does not remove your usable index.

In v17 the set-ready (phase 2), swap validity flip (phase 4), and set-dead
(phase 5) writes all use transactional `CatalogTupleUpdate`
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3478-L3550);
see [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)). So each
phase's catalog effect commits or rolls back atomically with its transaction.
The swap was already transactional in v12: commit `83158f74d3a` changed
`index_set_state_flags`, not that swap. Protection of the original name depends
on the swap transaction itself
([index.c#transactional-swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665)).
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
([indexcmds.c#progress-command](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3826-L3832)).
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
([indexcmds.c:4232-4233](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4232-L4233)).

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
the tree to its physical leaf partitions and processes them separately;
with `CONCURRENTLY`, each leaf goes through the multiple transactions of
`ReindexRelationConcurrently`
([indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3233-L3317),
[indexcmds.c#ReindexMultipleInternal](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3380-L3390),
[reindex.sgml#partitions](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334)).
Two earlier v13 fixes to the swap support this: `relispartition` is now swapped
(`d80be6f2f6c`) and partition dependencies are swapped (`68ac9cf2499`)
([index.c:1612-1618](../../../../raw/postgres-17/src/backend/catalog/index.c#L1612-L1618),
[index.c:1786-1809](../../../../raw/postgres-17/src/backend/catalog/index.c#L1786-L1809)).

**2. `REINDEX (TABLESPACE ...)` (PostgreSQL 14).** Commit `c5b286047cd` added a
`TABLESPACE` option; `ExecReindex` parses it into `params.tablespaceOid`, and the
phase-1 copy is created in the target tablespace (toast indexes are not moved).
For partitioned targets, only leaf-partition tablespaces are updated
([indexcmds.c:2748-2766](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2748-L2766),
[indexcmds.c:3841-3852](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3841-L3852),
[reindex.sgml#tablespace](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334)).

**3. `index_set_state_flags` is now transactional (PostgreSQL 14).** In v12 the
set-ready and set-dead flag writes used `heap_inplace_update` — a non-transactional
in-place overwrite that could not roll back. Commit `83158f74d3a` ("Make
index_set_state_flags() transactional") switched it to `CatalogTupleUpdate`, so in
v17 those flips obey MVCC, roll back with their transaction, and publish via cache
invalidation on commit
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3474-L3547)).
The phase-4 validity flip was already transactional in v12; this change makes the
phase-2 (`SET_READY`) and phase-5 (`SET_DEAD`) writes transactional as well.

**4. `PROC_IN_SAFE_IC` lets other concurrent builds ignore a running RIC
(PostgreSQL 14).** v12's RIC did not advertise itself, so two concurrent index
operations on different tables would wait on each other's snapshots. PostgreSQL 14
(commit `f9900df5`, "Avoid spurious wait in concurrent reindex") made RIC call
`set_indexsafe_procflags()` in phases 2 and 3 when the index is safe (no
expressions, no partial predicate) and unconditionally in phase 4
([indexcmds.c:3990-3992](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3990-L3992),
[indexcmds.c:4052-4054](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4052-L4054),
[indexcmds.c:4127](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4127)).
`WaitForOlderSnapshots` then excludes `PROC_IN_SAFE_IC` processes from the
snapshot wait, alongside autovacuum and lazy `VACUUM`
([indexcmds.c:433-442](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442),
[indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4491-L4505)).
This only shortens **Wait 3** (the snapshot wait); the lock-based Waits 1, 2, 4,
and 5 are unaffected.

**5. Progress-view phase fix (PostgreSQL 14).** Commit `e66bcfb4c66` ("Fix
progress reporting of REINDEX CONCURRENTLY") corrected phase 6 to set `WAIT_5`
instead of reusing `WAIT_4`, so `pg_stat_progress_create_index` now shows
`waiting for readers before dropping` during the final drop wait
([indexcmds.c:4232-4233](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4232-L4233),
[system_views.sql:1276-1277](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1276-L1277)).

**6. Statistics preservation reworked (PostgreSQL 15).** v12's swap hand-copied a
few cumulative counters into `newClassRel->pgstat_info->t_counts`. After the
PostgreSQL 15 shared-memory statistics rework (commit `8ea7963fc74`, "pgstat: add
pgstat_copy_relation_stats()"), the swap calls `pgstat_copy_relation_stats` to
move the cumulative counters and `CopyStatistics` to copy the planner's
`pg_statistic` column statistics (which matters for expression indexes)
([index.c:1811-1815](../../../../raw/postgres-17/src/backend/catalog/index.c#L1811-L1815)).

**7. `MAINTAIN` privilege (PostgreSQL 17).** v12 required table ownership (or
superuser) to reindex. PostgreSQL 17 (commit `ecb0fd33720`, "Reintroduce MAINTAIN
privilege and pg_maintain predefined role") lets a non-owner reindex if granted
`MAINTAIN`; the index lookup callback checks `ACL_MAINTAIN`
([indexcmds.c#RangeVarCallbackForReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2903-L2912)).

**8. Security hardening and testability.** Commit `a117cebd63` (2022,
back-patched through v10) moved security restrictions earlier, including into
`index_concurrently_build` and the phase-1 copy loop. It did not introduce
running the AM build as the table owner: the parent version in this checkout's
history already did that inside `index_build`. The distinction is when the
protection starts, before catalog-derived expressions can be evaluated.
In v17 these earlier entry points switch to the table owner's userid
([indexcmds.c:3802-3806](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3802-L3806),
[index.c:1520-1524](../../../../raw/postgres-17/src/backend/catalog/index.c#L1520-L1524));
Commit `2af07e2f74` (v17) fixed the maintenance `search_path` to a safe value;
`59825d1639` wrapped the save/restore behavior in `RestrictSearchPath`, and
`a15b0edb5d` added missing calls in the copy loop. Thus the pinned code combines
an owner-switching fix also back-patched to v12 with v17 search-path hardening
([indexcmds.c#copy-security-context](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3795-L3806),
[index.c#build-security-context](../../../../raw/postgres-17/src/backend/catalog/index.c#L1515-L1524)).
Commit `cd6b2ae3e7` (back-patched through v14, test back-patched through v17)
made RIC's safe flag conditional on having no expressions or predicate and
added `reindex-conc-index-safe` / `-not-safe` injection points for deterministic testing
([indexcmds.c:3812-3817](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3812-L3817)),
and 17.11 adds a third, `reindex-conc-index-built`, fired at the phase 2 ->
phase 3 boundary outside a transaction, where the new copy is ready but not yet
valid
([indexcmds.c:4015](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4015)).

**9. Dispatch refactors (PostgreSQL 13 and 14).** v12 dispatched REINDEX
inline in `standard_ProcessUtility` and passed an `options` int. v17 routes
through `ExecReindex` with `ReindexStmt` / `ReindexParams` structs and a
`REINDEXOPT_*` bitmask, introduced by v14 commit `a3dc926009`.
The separate v13 commit `2eb34ac369` introduced generic command classification;
the pinned code prevents REINDEX during recovery via
`ClassifyUtilityCommandAsReadOnly` rather than an explicit
`PreventCommandDuringRecovery("REINDEX")`
([indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2709-L2797),
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
- Deferrable-constraint coverage (new in 17.11): the injection-point isolation
  test `reindex_concurrently_deferred.spec` parks a `REINDEX TABLE CONCURRENTLY`
  on `reindex-conc-index-built` and has a second session insert a duplicate
  value and resolve it inside one transaction
  ([reindex_concurrently_deferred.spec](../../../../raw/postgres-17/src/test/modules/injection_points/specs/reindex_concurrently_deferred.spec#L1-L50)).
  Its expected output records the `_ccnew` row while the reindex waits:
  `indisunique` true, `indimmediate` **false**, `indisready` true,
  `indisvalid` false
  ([reindex_concurrently_deferred.out:11-20](../../../../raw/postgres-17/src/test/modules/injection_points/expected/reindex_concurrently_deferred.out#L11-L20)).

The causal chain is: committed old index → invalid replacement definition →
snapshot build and committed readiness → reference-snapshot gap-fill and older
snapshot wait → transactional rename/validity swap → reader waits and retirement
of the old index. Creation and swap commits decide which invalid spare can
survive an abort; partition discovery is a separate write-blocking entry stage
([indexcmds.c#core-sequence](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3944-L4275),
[indexcmds.c#discovery](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3345)).

## Measurement Script

### Usage and non-default conditions

This script tests lifecycle and lock behavior with synthetic shop fixtures. It
does not measure rebuild speed or wasted space, so the common-concept bloat
measurement protocols are outside this experiment's scope.

| Item | Details |
|---|---|
| Purpose | Check partition discovery blocking, post-build deferred writes, pre-creation abort, pre-swap cancellation, and successful retry. |
| Invocation | From the repository root, save the block below as `.wiki-runtime/tmp/ric17-measure.sh`, then run `bash .wiki-runtime/tmp/ric17-measure.sh`. |
| Stages | Default: `build start measure stop`. `build` verifies the pin, builds out of tree and installs the injection extension; it reuses an existing install. `start` starts this script's stopped cluster and clears its server log. `measure` recreates disposable fixtures, coordinates sessions and writes assertions. `stop` requests fast shutdown and checks PID-file, process, socket-directory and TCP-port teardown. `clean` runs `stop` and deletes the entire sandbox, including the build. Stages may be listed explicitly. |
| Environment | `JOBS` defaults to `4` and controls build parallelism. Run from the repository root (`PWD`); `PATH` must supply Bash, the build tools, `git`, `pgrep`, and the POSIX utilities used below. The current OS login from `id -un` owns the new cluster. Connection host, port, database, role and timeout options are set by the script; no existing database is used. |
| Prerequisites | C compiler, Make, Bison and Flex for the pinned checkout; C locale, UTF8 database. Configure uses `--enable-injection-points --without-icu --without-readline --without-zlib`. The extension is installed from the same build. |
| Output | `.wiki-runtime/tmp/ric17/out/results.txt` is the assertion summary. `platform.txt` records server, settings, platform and control-file facts. Per-scenario transcripts and `server.log` accompany it. Read these before cleanup deletes them. |
| Runtime | Allow several minutes for a fresh build (estimate). The filed run with the build present took 2 seconds, including startup and shutdown; this is harness runtime, not a RIC performance benchmark. |
| Cleanup | `bash .wiki-runtime/tmp/ric17-measure.sh clean`. An EXIT trap also tries fast shutdown on failure; cleanup failure is an error and must be resolved before deleting the sandbox. |

These non-default conditions make the states deterministic:

| Condition | Default, change, scope and mechanism |
|---|---|
| Test injection points | Normally disabled at configure time. `--enable-injection-points` enables the hooks, and the script loads the test extension. The local `safe` error aborts before copy creation; the local `built` wait pauses after readiness commits. The final script wakes that wait normally and cancels only inside Wait 2 ([configure.ac#injection-points](../../../../raw/postgres-17/configure.ac#L252-L257), [indexcmds.c#safe-hook](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3812-L3817), [indexcmds.c#built-hook-and-wait](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4015-L4034), [injection_points.c#actions](../../../../raw/postgres-17/src/test/modules/injection_points/injection_points.c#L261-L287)). |
| Deferred uniqueness | Ordinary unique indexes are immediate; the fixture explicitly declares `UNIQUE DEFERRABLE INITIALLY DEFERRED`, which creates a non-immediate original. RIC copies that flag. Temporary duplicates are resolved before COMMIT; this is not evidence that permanent duplicates are allowed ([index.c#copy-deferrable](../../../../raw/postgres-17/src/backend/catalog/index.c#L1348-L1352), [execIndexing.c#unique-mode](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L402-L428), [reindex_concurrently_deferred.spec#fixture](../../../../raw/postgres-17/src/test/modules/injection_points/specs/reindex_concurrently_deferred.spec#L1-L50)). |
| Active writer | The script keeps INSERT running with `RETURNING pg_sleep(50)`. An idle transaction after INSERT would not keep its executor index lock: `ExecCloseIndices` explicitly releases it. Holding the later leaf's active writer lets discovery retain a granted ShareLock on the earlier leaf while waiting for the later leaf ([execIndexing.c#open-and-lock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L190-L204), [execIndexing.c#close-and-unlock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L240-L252), [pg_inherits.c#ordered-child-locks](../../../../raw/postgres-17/src/backend/catalog/pg_inherits.c#L194-L230)). |
| Timeouts | Both GUCs default to `0` (disabled), have user context, and need no reload/restart. The script sets connection/session `statement_timeout=60s`, `lock_timeout=5s`; partition discovery uses `45s` for its lock timeout so the probe has time to observe it. The deliberately blocked writer instead uses `10s`/`500ms` to test lock timeout before statement timeout. Other blockers are cancelled explicitly once observed ([guc_tables.c#timeouts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631), [proc.c#lock-timer](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1273-L1307)). |
| Isolated server | Autovacuum defaults on; `-c autovacuum=off` at startup prevents unrelated maintenance from competing for these tiny fixtures (sighup context; a running server would need reload). Port `55457`, a private socket directory, and empty `listen_addresses` replace the compiled port/socket defaults and `localhost` TCP listener; these are postmaster settings requiring restart. The script sets them when starting its own server ([guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457), [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2393-L2403), [guc_tables.c#listeners](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4425-L4445)). |
| Reduced dependencies | ICU, Readline and zlib normally default on in configure. They are disabled to keep this disposable build independent of external libraries; fixtures use integer keys and noninteractive SQL, and no compression is measured ([configure.ac#ICU](../../../../raw/postgres-17/configure.ac#L833-L842), [configure.ac#readline](../../../../raw/postgres-17/configure.ac#L964-L974), [configure.ac#zlib](../../../../raw/postgres-17/configure.ac#L1074-L1085)). |

### The script

```bash
#!/usr/bin/env bash
set -uo pipefail
set -e
# Disposable fixtures only. Run from the repository root.
repo=$PWD
lab=$repo/.wiki-runtime/tmp/ric17
src=$repo/raw/postgres-17
pin=786db8dcf168bd9df8f55047337525ac19118b1c
jobs=${JOBS:-4}
owner=$(id -un)
port=55457
bin=$lab/install/bin
export PGHOST=$lab/s PGPORT=$port PGDATABASE=postgres
export PGOPTIONS='-c statement_timeout=60s -c lock_timeout=5s'
sql() { "$bin/psql" -X -h "$PGHOST" -p "$port" -U "$owner" -d postgres -v ON_ERROR_STOP=1 -At "$@"; }
check() {
  local actual
  actual=$(sql -c "$2")
  test "$actual" = "$3" || { echo "FAIL $1: $actual"; return 1; }
  echo "PASS $1: $actual" | tee -a "$lab/out/results.txt"
}
until_true() {
  local attempt
  for attempt in {1..40}; do
    if test "$(sql -c "$1")" = t; then return; fi
    sleep 1
  done
  echo 'Timed out waiting for fixture state' >&2
  sql -c "SELECT /* wiki_ric_debug_activity */ pid, application_name, state, wait_event_type, wait_event, query FROM pg_stat_activity;" >&2
  sql -c "SELECT /* wiki_ric_debug_locks */ pid, relation::regclass, mode, granted FROM pg_locks WHERE locktype='relation';" >&2
  return 1
}
stop() {
  test -f "$lab/owned" || return 0
  if test -f "$lab/data/postmaster.pid"; then
    "$bin/pg_ctl" -D "$lab/data" -m fast -w -t 15 stop
  fi
  test ! -f "$lab/data/postmaster.pid"
  local process_status=0
  pgrep -f -- "$lab/data" > "$lab/processes.txt" || process_status=$?
  if test "$process_status" = 0; then
    echo 'A process still names the data directory' >&2; return 1
  fi
  test "$process_status" = 1
  test -z "$(find "$lab/s" ! -type d -print)"
  "$lab/portcheck" "$port"
}
build() {
  test "$(git -C "$src" rev-parse HEAD)" = "$pin"
  test -z "$(git -C "$src" status --porcelain --untracked-files=no)"
  if test -e "$lab" && test ! -f "$lab/owned"; then
    echo 'Refusing an unowned sandbox' >&2; return 1
  fi
  mkdir -p "$lab/build" "$lab/s" "$lab/out"
  : > "$lab/owned"
  if test ! -x "$bin/postgres"; then
    (cd "$lab/build"
     "$src/configure" --prefix="$lab/install" --without-icu --without-readline --without-zlib --enable-injection-points
     make -j "$jobs"
     make install
     make -C src/test/modules/injection_points -j "$jobs"
     make -C src/test/modules/injection_points install) > "$lab/out/build.txt" 2>&1
  fi
  cat > "$lab/portcheck.c" <<'C'
#include <arpa/inet.h>
#include <stdlib.h>
#include <unistd.h>
int main(int argc, char **argv) {
  struct sockaddr_in address = {0};
  int fd = socket(AF_INET, SOCK_STREAM, 0);
  if (argc != 2 || fd < 0) return 1;
  address.sin_family = AF_INET;
  address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
  address.sin_port = htons((unsigned short) atoi(argv[1]));
  int result = bind(fd, (struct sockaddr *) &address, sizeof(address));
  close(fd);
  return result != 0;
}
C
  cc "$lab/portcheck.c" -o "$lab/portcheck"
}
start() {
  test -f "$lab/owned"
  test ! -f "$lab/data/postmaster.pid"
  "$lab/portcheck" "$port"
  if test ! -f "$lab/data/PG_VERSION"; then
    "$bin/initdb" -D "$lab/data" -U "$owner" -A trust --no-locale -E UTF8 > "$lab/out/initdb.txt"
  fi
  : > "$lab/out/server.log"
  "$bin/pg_ctl" -D "$lab/data" -l "$lab/out/server.log" -w start \
    -o "-k $lab/s -p $port -c listen_addresses='' -c autovacuum=off"
}
measure() {
  : > "$lab/out/results.txt"
  uname -srm > "$lab/out/platform.txt"
  sql -c "SELECT /* wiki_ric_platform */ version(), current_setting('block_size');" >> "$lab/out/platform.txt"
  sql -c "SELECT /* wiki_ric_settings */ name, setting, context FROM pg_settings WHERE name IN ('autovacuum','statement_timeout','lock_timeout','port','listen_addresses','unix_socket_directories') ORDER BY name;" >> "$lab/out/platform.txt"
  "$bin/pg_controldata" "$lab/data" >> "$lab/out/platform.txt"
  sql > "$lab/out/fixtures.txt" <<'SQL'
DROP /* wiki_ric_reset */ SCHEMA IF EXISTS ric_probe CASCADE;
CREATE /* wiki_ric_schema */ SCHEMA ric_probe;
CREATE /* wiki_ric_extension */ EXTENSION IF NOT EXISTS injection_points;
CREATE /* wiki_ric_orders */ TABLE ric_probe.orders(order_id integer) PARTITION BY RANGE(order_id);
CREATE /* wiki_ric_early */ TABLE ric_probe.orders_early PARTITION OF ric_probe.orders FOR VALUES FROM (0) TO (100);
CREATE /* wiki_ric_later */ TABLE ric_probe.orders_later PARTITION OF ric_probe.orders FOR VALUES FROM (100) TO (200);
CREATE /* wiki_ric_order_index */ INDEX orders_order_id_idx ON ric_probe.orders(order_id);
CREATE /* wiki_ric_customers */ TABLE ric_probe.customers(customer_id integer UNIQUE DEFERRABLE INITIALLY DEFERRED);
INSERT /* wiki_ric_customer_data */ INTO ric_probe.customers VALUES (1),(2);
SQL
  # A writer holds the later leaf index. Discovery gets the earlier leaf's
  # ShareLock and then queues for the later leaf's ShareLock.
  PGAPPNAME=ric_blocker sql > "$lab/out/blocker.txt" 2>&1 <<'SQL' &
BEGIN /* wiki_ric_block_begin */;
INSERT /* wiki_ric_hold_later */ INTO ric_probe.orders_later VALUES (150) RETURNING pg_sleep(50);
COMMIT /* wiki_ric_block_end */;
SQL
  blocker=$!
  until_true "SELECT /* wiki_ric_blocker_ready */ EXISTS (SELECT FROM pg_stat_activity WHERE application_name='ric_blocker' AND wait_event='PgSleep');"
  PGOPTIONS='-c statement_timeout=60s -c lock_timeout=45s' PGAPPNAME=ric_partition sql -c 'REINDEX /* wiki_ric_partition */ INDEX CONCURRENTLY ric_probe.orders_order_id_idx;' > "$lab/out/partition.txt" 2>&1 &
  reindex_job=$!
  until_true "SELECT /* wiki_ric_discovery_wait */ EXISTS (SELECT FROM pg_locks l JOIN pg_stat_activity s USING(pid) WHERE s.application_name='ric_partition' AND l.locktype='relation' AND l.mode='ShareLock' AND NOT l.granted);"
  check partition_discovery_share_lock "SELECT /* wiki_ric_share_lock */ EXISTS (SELECT FROM pg_locks l JOIN pg_stat_activity s USING(pid) WHERE s.application_name='ric_partition' AND l.relation=(SELECT indexrelid FROM pg_index WHERE indrelid='ric_probe.orders_early'::regclass) AND l.mode='ShareLock' AND l.granted);" t
  if PGOPTIONS='-c statement_timeout=10s -c lock_timeout=500ms' sql -c 'INSERT /* wiki_ric_blocked_write */ INTO ric_probe.orders_early VALUES (10);' > "$lab/out/writer.txt" 2>&1; then
    echo 'Expected a blocked writer' >&2; return 1
  fi
  grep -F 'canceling statement due to lock timeout' "$lab/out/writer.txt"
  echo 'PASS partition_writer_lock_timeout' | tee -a "$lab/out/results.txt"
  sql -c "SELECT /* wiki_ric_release_blocker */ pg_cancel_backend(pid) FROM pg_stat_activity WHERE application_name='ric_blocker';" > /dev/null
  if wait "$blocker"; then echo 'Expected blocker cancellation' >&2; return 1; fi
  wait "$reindex_job"
  check partition_rebuild_valid "SELECT /* wiki_ric_partition_valid */ bool_and(indisvalid) FROM pg_index WHERE indrelid IN ('ric_probe.orders_early'::regclass,'ric_probe.orders_later'::regclass);" t
  # Local injection points affect this backend only. First stop before copy
  # creation, then stop after the build commit.
  if sql > "$lab/out/before_creation.txt" 2>&1 <<'SQL'
SELECT /* wiki_ric_local_before */ injection_points_set_local();
SELECT /* wiki_ric_fail_before */ injection_points_attach('reindex-conc-index-safe','error');
REINDEX /* wiki_ric_abort_before */ INDEX CONCURRENTLY ric_probe.customers_customer_id_key;
SQL
  then echo 'Expected injected error' >&2; return 1; fi
  grep -F 'error triggered for injection point reindex-conc-index-safe' "$lab/out/before_creation.txt"
  check no_copy_before_creation_commit "SELECT /* wiki_ric_no_copy */ count(*) FROM pg_class WHERE relnamespace='ric_probe'::regnamespace AND relname LIKE '%ccnew%';" 0
  PGAPPNAME=ric_deferred sql > "$lab/out/deferred.txt" 2>&1 <<'SQL' &
SELECT /* wiki_ric_local_built */ injection_points_set_local();
SELECT /* wiki_ric_wait_built */ injection_points_attach('reindex-conc-index-built','wait');
REINDEX /* wiki_ric_deferred */ INDEX CONCURRENTLY ric_probe.customers_customer_id_key;
SQL
  deferred_job=$!
  until_true "SELECT /* wiki_ric_built_wait */ EXISTS (SELECT FROM pg_stat_activity WHERE application_name='ric_deferred' AND wait_event='reindex-conc-index-built');"
  check built_copy_flags "SELECT /* wiki_ric_flags */ indisunique || '|' || indimmediate || '|' || indisready || '|' || indisvalid FROM pg_index WHERE indexrelid='ric_probe.customers_customer_id_key_ccnew'::regclass;" 'true|false|true|false'
  sql > "$lab/out/deferred_writes.txt" <<'SQL'
BEGIN /* wiki_ric_defer_begin */;
INSERT /* wiki_ric_duplicate */ INTO ric_probe.customers VALUES (9),(9);
UPDATE /* wiki_ric_resolve */ ric_probe.customers SET customer_id=10 WHERE ctid=(SELECT ctid FROM ric_probe.customers WHERE customer_id=9 ORDER BY ctid DESC LIMIT 1);
COMMIT /* wiki_ric_defer_commit */;
SQL
  check deferred_duplicates_resolved "SELECT /* wiki_ric_resolved */ count(*)=count(DISTINCT customer_id) FROM ric_probe.customers;" t
  # Wake the test point normally. Cancel inside the next ordinary RIC
  # transaction, at Wait 2, rather than inside the out-of-transaction hook.
  PGAPPNAME=ric_cancel_blocker sql > "$lab/out/cancel_blocker.txt" 2>&1 <<'SQL' &
BEGIN /* wiki_ric_cancel_block_begin */;
INSERT /* wiki_ric_cancel_hold */ INTO ric_probe.customers VALUES (20) RETURNING pg_sleep(50);
COMMIT /* wiki_ric_cancel_block_end */;
SQL
  cancel_blocker=$!
  until_true "SELECT /* wiki_ric_cancel_blocker_ready */ EXISTS (SELECT FROM pg_stat_activity WHERE application_name='ric_cancel_blocker' AND wait_event='PgSleep');"
  sql -c "SELECT /* wiki_ric_detach_built */ injection_points_detach('reindex-conc-index-built'); SELECT /* wiki_ric_wake_built */ injection_points_wakeup('reindex-conc-index-built');" > /dev/null
  until_true "SELECT /* wiki_ric_wait_two */ EXISTS (SELECT FROM pg_stat_progress_create_index p JOIN pg_stat_activity s USING(pid) WHERE s.application_name='ric_deferred' AND p.phase='waiting for writers before validation' AND s.wait_event='virtualxid');"
  sql -c "SELECT /* wiki_ric_cancel_built */ pg_cancel_backend(pid) FROM pg_stat_activity WHERE application_name='ric_deferred';" > /dev/null
  if wait "$deferred_job"; then echo 'Expected reindex cancellation' >&2; return 1; fi
  grep -F 'canceling statement due to user request' "$lab/out/deferred.txt"
  sql -c "SELECT /* wiki_ric_release_cancel_blocker */ pg_cancel_backend(pid) FROM pg_stat_activity WHERE application_name='ric_cancel_blocker';" > /dev/null
  if wait "$cancel_blocker"; then echo 'Expected blocker cancellation' >&2; return 1; fi
  check cancellation_preserves_original "SELECT /* wiki_ric_original_valid */ indisvalid FROM pg_index WHERE indexrelid='ric_probe.customers_customer_id_key'::regclass;" t
  check cancellation_keeps_ready_invalid_copy "SELECT /* wiki_ric_copy_left */ indisready AND NOT indisvalid FROM pg_index WHERE indexrelid='ric_probe.customers_customer_id_key_ccnew'::regclass;" t
  sql > "$lab/out/repair.txt" <<'SQL'
DROP /* wiki_ric_drop_spare */ INDEX ric_probe.customers_customer_id_key_ccnew;
REINDEX /* wiki_ric_retry */ INDEX CONCURRENTLY ric_probe.customers_customer_id_key;
SQL
  check retry_valid "SELECT /* wiki_ric_retry_valid */ indisvalid AND NOT indimmediate FROM pg_index WHERE indexrelid='ric_probe.customers_customer_id_key'::regclass;" t
  if grep -E 'PANIC|terminated by signal|Segmentation fault' "$lab/out/server.log"; then
    echo 'Unexpected server failure' >&2; return 1
  fi
}
clean() {
  stop
  test -f "$lab/owned"
  rm -r "$lab"
}
trap 'stop' EXIT
task_started=$SECONDS
if test "$#" = 0; then set -- build start measure stop; fi
for stage in "$@"; do
  case "$stage" in
    build|start|measure|stop|clean) "$stage" ;;
    *) echo "Unknown stage: $stage" >&2; exit 1 ;;
  esac
done
echo "Elapsed seconds: $((SECONDS - task_started))"
```

### Last run

2026-11-02, PostgreSQL 17.11 at
`786db8dcf168bd9df8f55047337525ac19118b1c`, Darwin 27.2.0 arm64, Apple clang
21.0.0, 64-bit; `block_size = 8192`, maximum data alignment `8`. Command:
`bash .wiki-runtime/tmp/ric17-measure.sh build start measure stop` with `JOBS=4`.
All nine assertions passed. The final server log contained no PANIC or
signal-termination record. This run reset the schema in the disposable cluster;
an earlier development run's recovery had completed before these checks.

| Check | Observed result | Engine evidence |
|---|---|---|
| Discovery locks and writer | Granted ShareLock on the earlier leaf index while discovery waited on the later leaf; the earlier leaf's writer failed with lock timeout | [indexcmds.c#discovery](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3310), [pg_inherits.c#child-locks](../../../../raw/postgres-17/src/backend/catalog/pg_inherits.c#L204-L230), [execIndexing.c#index-lock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L190-L204). |
| Partitioned rebuild after blocker cancellation | Physical leaf indexes valid | [indexcmds.c#per-leaf-call](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3377-L3390), [index.c#swap-valid](../../../../raw/postgres-17/src/backend/catalog/index.c#L1659-L1665). |
| Error before copy creation | Zero `_ccnew` relations in the fixture schema | [indexcmds.c#hook-before-create](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3812-L3852). |
| Copy at post-build hook | `indisunique=true`, `indimmediate=false`, `indisready=true`, `indisvalid=false` | [index.c#copy-deferrable](../../../../raw/postgres-17/src/backend/catalog/index.c#L1348-L1352), [index.c#ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1547-L1557), [indexcmds.c#hook](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4015-L4034). |
| Deferred writes | Duplicate `customer_id=9` values inserted and one changed to `10`; COMMIT succeeded with distinct customer IDs | [execIndexing.c#unique-mode](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L402-L428), [reindex_concurrently_deferred.spec#same-pattern](../../../../raw/postgres-17/src/test/modules/injection_points/specs/reindex_concurrently_deferred.spec#L38-L50). |
| Cancellation in ordinary Wait 2 | Original valid; separate `_ccnew` ready and invalid | [indexcmds.c#build-commit-and-wait](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4008-L4034). |
| Drop spare and retry | Original-name replacement valid and non-immediate | [index.c#swap-flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1665). |

The nine assertions split the lock and cancellation rows into their individual
checks. No old-name truncation/collision, crash-recovery matrix, planner-horizon
fixture or performance comparison was measured. The sandbox was stopped and
deleted after the run; the script above is the retained reproduction artifact.

## Context Reviewed

- 2026-11-02 review fixes: partition discovery in `pg_inherits.c`, writer
  index locks in `execIndexing.c`, snapshot/commit boundaries, planner
  `indcheckxmin`, and early security context placement. Additional history:
  `a117cebd63`, `2af07e2f74`, `59825d1639`, `a15b0edb5d`, `cd6b2ae3e7`,
  `a3dc926009`, and `2eb34ac369`; their patches and commit messages were read
  in the target checkout. No external history or another version's checkout
  was used for these changes. Targeted execution is filed under Measurement Script.

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and the last ~20
  `wiki/log.md` entries (navigation only).
- The sibling page
  [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
  (shared build/validate core and `PROC_IN_SAFE_IC` discussion) and the v12
  counterpart
  [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../../v12/questions/indexing/reindex-index-concurrently.md)
  (the v12 baseline for the change comparison).
- Pinned checkout `raw/postgres-17/` at commit
  `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11,
  `REL_17_11-7-g786db8dcf16`); repinned from
  `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on 2026-08-17.
- 2026-08-17 repin review: `28269fed661` ("Fix propagation of indimmediate flag
  in index_create_copy()", back-patched through v14) is the one commit in the
  `54eeefaed..786db8dcf16` range that changed a claim on this page. It adds the
  `INDEX_CREATE_DEFERRABLE` flag to phase 1, the `reindex-conc-index-built`
  injection point at the phase 2 -> phase 3 boundary, and the
  `reindex_concurrently_deferred` isolation test; all three are recorded above.
  No other phase, lock, wait, flag, or failure claim changed in the range.
- `ExecReindex`, `ReindexIndex`, `ReindexTable`, `ReindexMultipleTables`,
  `ReindexPartitions`, `ReindexMultipleInternal`, `ReindexRelationConcurrently`,
  `WaitForOlderSnapshots`, and `set_indexsafe_procflags` in
  `src/backend/commands/indexcmds.c`.
- `index_concurrently_create_copy`, `index_concurrently_build`,
  `index_concurrently_swap`, `index_concurrently_set_dead`, `index_drop`, and
  `index_set_state_flags` in `src/backend/catalog/index.c`.
- The creation and swap commit boundaries in `ReindexRelationConcurrently`,
  and transactional storage creation in `src/backend/catalog/storage.c`.
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
| Partition discovery can block writes before the concurrent core; its locks end at the initial multi-relation commit | [indexcmds.c#discovery](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3276-L3310), [pg_inherits.c#child-locks](../../../../raw/postgres-17/src/backend/catalog/pg_inherits.c#L204-L230), [execIndexing.c#writer-lock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L190-L204), [indexcmds.c#discovery-commit](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3327-L3345) |
| Phase 1 has an active snapshot; the following Wait-1 transaction takes no snapshot | [indexcmds.c#transaction-boundary](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L3974) |
| Validity is necessary, but `indcheckxmin` can still exclude a valid index | [plancat.c#index-checks](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L284) |
| Writers skip the copy before readiness; non-immediate uniqueness checks matter after its build commits | [index.c#ready-publication](../../../../raw/postgres-17/src/backend/catalog/index.c#L1547-L1557), [execIndexing.c#ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359), [execIndexing.c#unique-mode](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L402-L428) |
| RIC runs as six phases (create copy, build, catch up, swap, set dead, drop), each phase looping all indexes | [indexcmds.c:3753-3767](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3753-L3767) |
| `CONCURRENTLY` cannot run in a transaction block; recovery blocked via read-only classification | [indexcmds.c:2736-2738](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2736-L2738), [utility.c:280-296](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296), [utility.c:581-582](../../../../raw/postgres-17/src/backend/tcop/utility.c#L581-L582) |
| `ExecReindex` parses options into `ReindexParams` and dispatches by kind | [indexcmds.c:2709-2797](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2709-L2797) |
| `ReindexIndex` locks SUE for concurrent; partitioned -> `ReindexPartitions`; temp -> non-concurrent | [indexcmds.c:2804-2850](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2850) |
| Partitioned target recurses into leaf partitions; each concurrent leaf rebuild has its own multiple-transaction lifecycle | [indexcmds.c:3233-3317](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3233-L3317), [indexcmds.c:3380-3390](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3380-L3390), [indexcmds.c#core-commits](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4106) |
| Directly named partitioned relation never reaches the concurrent core | [indexcmds.c:3724-3730](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3724-L3730) |
| System catalog rejected; invalid/exclusion via table skipped; invalid named directly allowed | [indexcmds.c:3532-3535](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3532-L3535), [indexcmds.c:3564-3576](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3564-L3576), [indexcmds.c:3711-3717](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3711-L3717) |
| Exclusion index named directly errors in `index_concurrently_create_copy` | [index.c:1338-1341](../../../../raw/postgres-17/src/backend/catalog/index.c#L1338-L1341) |
| Phase 1 creates `_ccnew` not-ready/not-valid via the skip-build + concurrent create flags | [index.c:1412](../../../../raw/postgres-17/src/backend/catalog/index.c#L1412), [index.c:1459-1479](../../../../raw/postgres-17/src/backend/catalog/index.c#L1459-L1479) |
| Phase 1 adds `INDEX_CREATE_DEFERRABLE` for a deferrable old index, so `_ccnew` gets `indimmediate = false`; phase 4 copies `indimmediate` old -> new (17.11, `28269fed661`) | [index.c:1348-1352](../../../../raw/postgres-17/src/backend/catalog/index.c#L1348-L1352), [index.h:68](../../../../raw/postgres-17/src/include/catalog/index.h#L68), [index.c:1050-1057](../../../../raw/postgres-17/src/backend/catalog/index.c#L1050-L1057), [index.c:1642-1653](../../../../raw/postgres-17/src/backend/catalog/index.c#L1642-L1653) |
| 17.11 injection point `reindex-conc-index-built` fires between phases 2 and 3, outside a transaction, and the new isolation test observes `_ccnew` as unique, non-immediate, ready, not valid | [indexcmds.c:4015](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4015), [reindex_concurrently_deferred.spec:1-50](../../../../raw/postgres-17/src/test/modules/injection_points/specs/reindex_concurrently_deferred.spec#L1-L50), [reindex_concurrently_deferred.out:11-20](../../../../raw/postgres-17/src/test/modules/injection_points/expected/reindex_concurrently_deferred.out#L11-L20) |
| Session SUE taken on old index, new index, and heap(s) | [indexcmds.c:3879-3950](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3879-L3950) |
| Phase 2: Wait 1 (`ShareLock`) then per-index build sets `indisready` | [indexcmds.c:3971-4013](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3971-L4013), [index.c:1499-1557](../../../../raw/postgres-17/src/backend/catalog/index.c#L1499-L1557) |
| Phase 3: Wait 2 (`ShareLock`), `validate_index`, then Wait 3 (`WaitForOlderSnapshots`) | [indexcmds.c:4032-4106](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4032-L4106) |
| One invocation has six fixed internal commits plus three per index; a one-index RIC has nine before the final caller-owned transaction | [indexcmds.c:3952-4106](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4106), [indexcmds.c:4120-4275](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4275), [postgres.c:2798-2807](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2807) |
| Direct build controls are AM-dependent `maintenance_work_mem` plus the B-tree/BRIN worker request and pool chain | [index.c:2995-3005](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005), [index.c:3390-3434](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3434), [planner.c:6876-7018](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7018), [bgworker.c:969-1043](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L969-L1043) |
| Both heap scans use sequential read streams; `io_combine_limit` and the effective-I/O branch apply, not maintenance I/O advice | [heapam.c:1233-1259](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1233-L1259), [read_stream.c:407-536](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L407-L536) |
| RIC preserves the old index tablespace unless `REINDEX (TABLESPACE ...)` is explicit; `default_tablespace` does not place the copy | [indexcmds.c:3841-3852](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3841-L3852) |
| WAL/checkpoint/commit settings affect permanent builds and RIC's many eligible commits without changing its state machine | [bulk_write.c:124-223](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223), [xloginsert.c:604-695](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L695), [xact.c:1462-1546](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1546) |
| `statement_timeout` spans RIC, `transaction_timeout` restarts per internal transaction, and `lock_timeout` applies per VXID acquisition | [postgres.c:2767-2807](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2807), [xact.c:2168-2178](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2168-L2178), [lock.c:4550-4662](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4662) |
| Phase 4 swap: new valid + old invalid via transactional `CatalogTupleUpdate`, names/constraints/deps/stats moved | [indexcmds.c:4120-4181](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4120-L4181), [index.c:1566-1826](../../../../raw/postgres-17/src/backend/catalog/index.c#L1566-L1826) |
| Swap copies cumulative + planner stats via `pgstat_copy_relation_stats` + `CopyStatistics` | [index.c:1811-1815](../../../../raw/postgres-17/src/backend/catalog/index.c#L1811-L1815) |
| Phase 5: Wait 4 (`AccessExclusiveLock`) then `index_concurrently_set_dead` | [indexcmds.c:4198-4214](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4198-L4214), [index.c:1837-1871](../../../../raw/postgres-17/src/backend/catalog/index.c#L1837-L1871) |
| Phase 6: Wait 5 (`AccessExclusiveLock`) then `performMultipleDeletions` drops old | [indexcmds.c:4232-4262](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4232-L4262) |
| Drop uses `PERFORM_DELETION_CONCURRENT_LOCK` -> `index_drop` takes SUE, skips its own concurrent branch | [dependency.c:1356-1368](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1356-L1368), [index.c:2171-2213](../../../../raw/postgres-17/src/backend/catalog/index.c#L2171-L2213) |
| `AccessExclusiveLock` conflicts with `AccessShareLock`, so phases 5/6 wait out readers | [lock.c:98-102](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102) |
| `ShareLock` waits (Waits 1/2) wait out `RowExclusiveLock` writers and stronger (not other `ShareLock` holders) | [lock.c:82-85](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L85) |
| `ShareUpdateExclusiveLock` self-conflicts; does not conflict with `RowExclusiveLock` | [lock.c:77-80](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80) |
| Wait 3 ignores autovacuum, lazy VACUUM, and safe concurrent builds (`PROC_IN_SAFE_IC`) | [indexcmds.c:433-442](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442) |
| RIC advertises `PROC_IN_SAFE_IC` in phases 2/3 (if safe) and 4 (unconditional) | [indexcmds.c:3990-3992](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3990-L3992), [indexcmds.c:4052-4054](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4052-L4054), [indexcmds.c:4127](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4127) |
| `index_set_state_flags` is transactional (`CatalogTupleUpdate`); `INDEX_DROP_SET_DEAD` asserts not valid | [index.c:3478-3550](../../../../raw/postgres-17/src/backend/catalog/index.c#L3478-L3550) |
| Invalid indexes skipped by planner but can still affect writes while live/ready | [plancat.c:260-266](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266), [relcache.c:4802-4871](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4802-L4871), [execIndexing.c:356-359](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359) |
| An abort before phase 1 commits leaves no new copy; after that commit and before the swap commits, an abort leaves the already-created `_ccnew` invalid | [indexcmds.c:3952-4012](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3952-L4012), [indexcmds.c:4110-4182](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4110-L4182), [index.c:1526-1534](../../../../raw/postgres-17/src/backend/catalog/index.c#L1526-L1534), [storage.c:108-118](../../../../raw/postgres-17/src/backend/catalog/storage.c#L108-L118), [create_index.out:2722-2736](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2722-L2736) |
| A RIC failure never leaves a healthy `index_name` invalid: phase 4 renames the rebuilt copy to `index_name` and flips validity in one transaction; the invalid leftover is always `_ccnew`/`_ccold`. `index_name` is invalid only if it was already invalid (repair case) | [index.c:1608-1665](../../../../raw/postgres-17/src/backend/catalog/index.c#L1608-L1665), [indexcmds.c:4114-4181](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4114-L4181), [create_index.out:2716-2760](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2716-L2760) |
| Progress: command `REINDEX CONCURRENTLY`; phase 6 correctly sets `WAIT_5` (fixed vs v12) | [indexcmds.c:4232-4233](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4232-L4233), [progress.h:92-114](../../../../raw/postgres-17/src/include/commands/progress.h#L92-L114), [system_views.sql:1264-1277](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277) |
| v14 added partitioned REINDEX (`a6642b3ae06`), TABLESPACE (`c5b286047cd`), transactional flags (`83158f74d3a`), PROC_IN_SAFE_IC for RIC (`f9900df5`), progress fix (`e66bcfb4c66`) | checkout git history; [reindex.sgml:319-334](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L334) |
| v15 stats rework (`8ea7963fc74`); v17 MAINTAIN privilege (`ecb0fd33720`) | [index.c:1811-1815](../../../../raw/postgres-17/src/backend/catalog/index.c#L1811-L1815), [indexcmds.c:2903-2912](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2903-L2912) |
| Docs: two scans, wait for transactions, six steps, `_ccnew`/`_ccold` recovery, partitions | [reindex.sgml:366-478](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L366-L478) |
| Tests: functional, partitioned, restriction/repair, isolation | [create_index.sql:882-1165](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165), [reindex-concurrently.spec:1-40](../../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40) |

## Open Questions

- In an earlier development run of this instrumented build, cancelling while
  stopped inside `reindex-conc-index-built` produced a backend segmentation
  fault after the cancellation error. Fast shutdown then timed out; immediate
  shutdown and a restart were needed before a later clean fast shutdown.
  The source places that hook outside a transaction, and its test callback
  waits through shared-memory condition-variable state. The root cause and
  reproducibility were not established. This observation is not evidence
  about cancellation in a default build, which has no injection hooks. The
  filed script wakes the hook normally and cancels inside the following
  ordinary Wait-2 transaction instead
  ([indexcmds.c#hook-boundary](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4008-L4034),
  [injection_points.c#wait-callback](../../../../raw/postgres-17/src/test/modules/injection_points/injection_points.c#L183-L258),
  [configure.ac#default-disabled](../../../../raw/postgres-17/configure.ac#L252-L257)).

- Outside this page's authorized scope, the v17
  [deduplication page's cancellation advice](btree-deduplication-after-pg-upgrade.md#settings-this-page-touches)
  says a cancelled long RIC leaves `_ccnew`. That omits the commit conditions:
  no copy survives before creation commits; `_ccold`, rather than `_ccnew`,
  may survive after the swap commits. The [failure table](#failure-scenarios-and-the-outcome-on-the-table)
  and the pinned creation/swap/drop boundaries settle this conflict
  ([indexcmds.c#creation](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3944-L3954),
  [indexcmds.c#swap-and-drop](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4180-L4262)).
  The other page was reported without a change.

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
  [heapam.c#read-stream-flags](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L425-L439)).
- The exact crash / immediate-shutdown outcome at each instruction boundary was
  not traced through crash recovery. In v17 the swap validity flip and the
  set-ready / set-dead flips all go through transactional `CatalogTupleUpdate`
  ([index.c:3547](../../../../raw/postgres-17/src/backend/catalog/index.c#L3547)), so
  the v12 "non-transactional in-place" recovery hazard no longer applies to these
  flag writes; the precise recovered catalog state mid-rebuild was still not
  isolated here.
- Whether two indexes sharing a long base name can collide on the `_ccnew` /
  `_ccold` names is mitigated by `ChooseRelationName` plus the per-iteration
  `CommandCounterIncrement`
  ([indexcmds.c:4170-4177](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4170-L4177)),
  but the exact truncation/uniqueness behavior of `ChooseRelationName` was not
  traced.
- The first build scan's exact tuple-visibility rule lives inside `index_build` ->
  `table_index_build_scan` (via `ii_Concurrent`), summarized from the shared CIC
  core rather than traced line-by-line into the table AM
  ([index.c:1535-1539](../../../../raw/postgres-17/src/backend/catalog/index.c#L1535-L1539)).

## Source References

- [indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2709-L2797)
- [pg_inherits.c#child-locks](../../../../raw/postgres-17/src/backend/catalog/pg_inherits.c#L194-L230)
- [execIndexing.c#writer-index-locks](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L190-L204)
- [execIndexing.c#writer-index-unlock](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L240-L252)
- [plancat.c#validity-and-xmin](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L284)
- [configure.ac#injection-build-option](../../../../raw/postgres-17/configure.ac#L252-L257)
- [pg_config.h.in#USE_INJECTION_POINTS](../../../../raw/postgres-17/src/include/pg_config.h.in#L715-L716)
- [injection_points.c#wait-and-actions](../../../../raw/postgres-17/src/test/modules/injection_points/injection_points.c#L183-L287)
- [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2850)
- [indexcmds.c#ReindexTable](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2934-L2982)
- [indexcmds.c#ReindexPartitions](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3233-L3317)
- [indexcmds.c#ReindexMultipleInternal](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3327-L3426)
- [indexcmds.c#ReindexRelationConcurrently](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3453-L4313)
- [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L433-L442)
- [indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4491-L4505)
- [index.c#index_concurrently_create_copy](../../../../raw/postgres-17/src/backend/catalog/index.c#L1306-L1486)
- [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1499-L1557)
- [index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1566-L1826)
- [index.c#index_concurrently_set_dead](../../../../raw/postgres-17/src/backend/catalog/index.c#L1837-L1871)
- [index.c#index_drop](../../../../raw/postgres-17/src/backend/catalog/index.c#L2131-L2213)
- [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3478-L3550)
- [storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L108-L171)
- [index.c#index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L2959-L3069)
- [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3262-L3452)
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
- [heapam.c#scan-strategy-and-read-stream](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L417-L496)
- [heapam.c#heap_beginscan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1151-L1262)
- [read_stream.c#read_stream_begin_relation](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L379-L570)
- [tuplesort.c#temp-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1938-L1966)
- [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223)
- [xact.c#commit-flush](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1546)
- [xlog.c#group-commit-delay](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2869-L2888)
- [dependency.c#doDeletion](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1352-L1368)
- [plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L260-L266)
- [relcache.c#RelationGetIndexList](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4800-L4871)
- [execIndexing.c#ExecInsertIndexTuples](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L356-L359)
- [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L903-L979)
- [lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)
- [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4662)
- [proc.c#lock-timeouts](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1273-L1325)
- [postgres.c#statement-and-transaction-timeouts](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3374-L3463)
- [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2460-L2474)
- [guc_tables.c#timeouts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2653)
- [guc_tables.c#worker-pools](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3163-L3172)
- [guc_tables.c#parallel-workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3438)
- [guc_tables.c#I/O-settings](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3109-L3160)
- [guc_tables.c#planner-and-GIN-settings](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3585)
- [guc_tables.c#WAL-size-checkpoint-and-commit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2842-L3001)
- [guc_tables.c#WAL-commit-and-compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4925-L4983)
- [utility.c#ClassifyUtilityCommandAsReadOnly](../../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L296)
- [progress.h#create-index-phases](../../../../raw/postgres-17/src/include/commands/progress.h#L92-L114)
- [system_views.sql#pg_stat_progress_create_index](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1264-L1277)
- [ref/reindex.sgml#CONCURRENTLY](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L319-L478)
- [create_index.sql#ric-block](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L882-L1165)
- [create_index.out#failed-rebuild-and-repair](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2716-L2760)
- [reindex-concurrently.spec](../../../../raw/postgres-17/src/test/isolation/specs/reindex-concurrently.spec#L1-L40)
- [reindex_concurrently_deferred.spec](../../../../raw/postgres-17/src/test/modules/injection_points/specs/reindex_concurrently_deferred.spec#L1-L50)
- [reindex_concurrently_deferred.out](../../../../raw/postgres-17/src/test/modules/injection_points/expected/reindex_concurrently_deferred.out#L1-L41)
- [index.h#INDEX_CREATE_DEFERRABLE](../../../../raw/postgres-17/src/include/catalog/index.h#L68)

## Navigation

- [Wiki Glossary (unverified)](../../../glossary.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../../v12/questions/indexing/reindex-index-concurrently.md)
- [v17 index](../../index.md)
- [versions](../../../versions.md)
