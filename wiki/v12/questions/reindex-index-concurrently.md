---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [How it differs from CREATE INDEX CONCURRENTLY](#how-it-differs-from-create-index-concurrently)
  - [Dispatch, preconditions, and restrictions](#dispatch-preconditions-and-restrictions)
  - [The six phases](#the-six-phases)
  - [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table)
  - [State flags for the old and new index](#state-flags-for-the-old-and-new-index)
  - [Failure scenarios and the outcome on the table](#failure-scenarios-and-the-outcome-on-the-table)
  - [Multiple indexes in one command](#multiple-indexes-in-one-command)
  - [Watching the phases](#watching-the-phases)
  - [Test coverage](#test-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

give a comprehensive explanation of how REINDEX INDEX CONCURRENTLY is
implemented in PostgreSQL 12, including the steps, locks, and failure scenarios

(Filed 2026-06-12 by splitting the prior combined page
[How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](create-index-concurrently.md)
into one page per command.)

## Answer

`REINDEX INDEX CONCURRENTLY` (RIC) rebuilds an index without ever blocking
`INSERT`/`UPDATE`/`DELETE`, and without making the index unavailable to queries
in between. It does this by building a **brand-new copy** of the index next to
the original, catching that copy up to concurrent writes, atomically **swapping**
the copy in for the original, and only then marking the original dead and
dropping it. The build and catch-up reuse the exact `CREATE INDEX CONCURRENTLY`
(CIC) machinery; the swap-and-drop tail is what makes a reindex.

All of it runs in `ReindexRelationConcurrently()`, which lays out **six phases**
across many short transactions
([indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2738-L2955)):

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
([lockdefs.h:36-46](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46),
[lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81)).

### How it differs from CREATE INDEX CONCURRENTLY

RIC is CIC plus a swap and a drop. The build (`index_concurrently_build`) and
the validation (`validate_index`) functions are literally the same as CIC's; see
the [CIC page](create-index-concurrently.md) for that shared core. The
differences:

| Aspect | CREATE INDEX CONCURRENTLY | REINDEX INDEX CONCURRENTLY |
|---|---|---|
| Target | a new index that did not exist | a fresh copy `<name>_ccnew` of an existing index |
| Phases | build, validate, mark valid (3 waits) | create copy, build, validate, swap, set-dead, drop (**5 waits**) |
| Extra waits | none beyond the snapshot wait | two `AccessExclusiveLock` waits that **wait out readers** of the old index before set-dead and drop ([indexcmds.c:3272-3304](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3272-L3304)) |
| Names | the index keeps its name | new copy takes the original's name; the old index is renamed `<name>_ccold` ([index.c#swap-names](../../../raw/postgres-12/src/backend/catalog/index.c#L1490-L1492), [indexcmds.c#ccold](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3230-L3235)) |
| On failure | leaves the **target** index invalid | leaves an invalid `_ccnew` (before swap) or `_ccold` (after swap); a healthy original is preserved until the swap |

The "wait out readers" difference is the operationally important one: CIC's three
waits never block on plain `SELECT`, but RIC's phases 5 and 6 must wait for every
transaction that could still be **reading through the old index** to finish before
it can mark that index dead and drop it
([reindex.sgml#concurrent-steps](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L334-L359)).

### Dispatch, preconditions, and restrictions

`REINDEX ... CONCURRENTLY` is dispatched in `standard_ProcessUtility`. Because the
implementation commits many times, it cannot run inside a transaction block, and
it is forbidden during recovery
([utility.c#reindex-dispatch](../../../raw/postgres-12/src/backend/tcop/utility.c#L773-L807)):

```c
if (stmt->concurrent)
    PreventInTransactionBlock(isTopLevel, "REINDEX CONCURRENTLY");
PreventCommandDuringRecovery("REINDEX");
```

`REINDEX INDEX` routes through `ReindexIndex`, which locks the index in
`ShareUpdateExclusiveLock` for the concurrent case (vs `AccessExclusiveLock` for a
plain reindex) and then calls `ReindexRelationConcurrently` — unless the index is
on a **temporary** table, in which case it silently does a non-concurrent
`reindex_index` instead
([indexcmds.c#ReindexIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2336-L2382)).
`REINDEX TABLE` behaves the same way via `ReindexTable`, emitting
`table "..." has no indexes that can be reindexed concurrently` when nothing
qualifies
([indexcmds.c#ReindexTable](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2458-L2499)).

The full restriction set, with where each is enforced:

| Restriction | Behavior | Where enforced |
|---|---|---|
| Inside a transaction block | error: `REINDEX CONCURRENTLY cannot run inside a transaction block` | [utility.c:777-779](../../../raw/postgres-12/src/backend/tcop/utility.c#L777-L779) |
| During recovery (standby) | error: `REINDEX` cannot run during recovery | [utility.c:782](../../../raw/postgres-12/src/backend/tcop/utility.c#L782) |
| Temporary table/index | falls back to a **non-concurrent** reindex | [indexcmds.c:2377-2381](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2377-L2381), [indexcmds.c:2477-2491](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2477-L2491) |
| System catalogs | error: `cannot reindex system catalogs concurrently` | [indexcmds.c:2804-2807](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2804-L2807), [indexcmds.c:2897-2900](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2897-L2900), [indexcmds.c:2530-2533](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2530-L2533) |
| Partitioned table/index | warning, skipped (no-op) | [indexcmds.c:2917-2923](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2917-L2923) |
| Exclusion-constraint index named directly | error: `concurrent index creation for exclusion constraints is not supported` | [index.c:1268-1271](../../../raw/postgres-12/src/backend/catalog/index.c#L1268-L1271) |
| Exclusion-constraint index reached via a table | warning, skipped | [indexcmds.c:2825-2830](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2825-L2830) |
| Invalid index reached via a table | warning, skipped | [indexcmds.c:2819-2824](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2819-L2824) |
| Invalid index named directly | **allowed** (this is how you repair one) | [indexcmds.c:2908-2912](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2908-L2912) |

When the target is a table, matview, or toast relation, RIC gathers every valid,
non-exclusion index plus the relation's **toast** indexes, processing them all
together; when the target is an index, it processes just that one
([indexcmds.c#gather-indexes](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2784-L2916)).

### The six phases

The phase comment in the source is the canonical map
([indexcmds.c#phase-overview](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955)).
Each phase loops over **all** indexes before the next phase begins.

#### Phase 1: create the catalog copies and take session locks

For each index, RIC chooses a temporary name `<orig>_ccnew`
([indexcmds.c#ccnew](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2993-L2998))
and calls `index_concurrently_create_copy`, which rebuilds the new index's
`IndexInfo` from the old index's catalog rows (class, options, reloptions,
expressions, predicates) and creates the catalog entry with
`INDEX_CREATE_SKIP_BUILD | INDEX_CREATE_CONCURRENT` — so it has identity but no
data and is **not ready, not valid**
([index.c#index_concurrently_create_copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1240-L1388)).
Exclusion constraints are rejected here
([index.c:1268-1271](../../../raw/postgres-12/src/backend/catalog/index.c#L1268-L1271)).

RIC then takes a **session-level** `ShareUpdateExclusiveLock` on the old index,
the new index, and the heap table (including toast)
([indexcmds.c#session-locks](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3068-L3074)),
saves a `LOCKTAG` per heap for the later waits
([indexcmds.c#locktags](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3042-L3066)),
then commits. These session locks survive the upcoming commits so nobody can drop
the relations mid-rebuild.

#### Phase 2: wait, then build each new index

**Wait 1** is `WaitForLockersMultiple(lockTags, ShareLock, true)`: it waits out
every transaction that could still have the table open without the new index in
its index list
([indexcmds.c#wait1](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3090-L3093)).
Then, in a **separate transaction per index**, RIC takes a fresh snapshot and calls
`index_concurrently_build`, which scans the heap, builds the new copy, and sets
`indisready = true`
([indexcmds.c#build-loop](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3095-L3128),
[index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439)).
This is the same build CIC uses.

#### Phase 3: wait, validate, then wait for old snapshots

**Wait 2** is another `WaitForLockersMultiple(lockTags, ShareLock, true)`, so that
every in-flight transaction now sees the new copy as ready-for-inserts
([indexcmds.c#wait2](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3139-L3142)).
Then, per new index, RIC registers a **reference snapshot**, runs `validate_index`
to backfill any tuples the build scan missed, saves the snapshot's `xmin` as
`limitXmin`, drops the snapshot, commits, and in a fresh transaction does **Wait 3**
= `WaitForOlderSnapshots(limitXmin, true)` to wait out transactions whose snapshot
predates the reference snapshot
([indexcmds.c#validate-loop](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3144-L3199)).
At the end of phase 3 the new copy contains every interesting tuple but is **still
invalid** — its `indisvalid` is not flipped here.

#### Phase 4: swap each old index with its new copy

In a single transaction, RIC chooses a name `<orig>_ccold` for the old index and
calls `index_concurrently_swap`, then invalidates the table's relcache and issues
a `CommandCounterIncrement` so the next index in the loop sees the new names
([indexcmds.c#swap-loop](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3212-L3262)).
`index_concurrently_swap` does the heavy lifting atomically
([index.c#index_concurrently_swap](../../../raw/postgres-12/src/backend/catalog/index.c#L1447-L1716)):

- swaps the `relname`s — the new copy takes the original's name, the old index
  becomes `<orig>_ccold`
  ([index.c:1490-1492](../../../raw/postgres-12/src/backend/catalog/index.c#L1490-L1492));
- copies the constraint flags `indisprimary`, `indisexclusion`, `indimmediate`
  from old to new
  ([index.c:1524-1529](../../../raw/postgres-12/src/backend/catalog/index.c#L1524-L1529));
- marks the **new index valid and the old index invalid** at the same time, and
  clears `indisclustered` on the old
  ([index.c:1531-1534](../../../raw/postgres-12/src/backend/catalog/index.c#L1531-L1534));
- moves constraints and their triggers, the comment, the partition inheritance
  link, and all dependencies from the old index to the new
  ([index.c:1542-1680](../../../raw/postgres-12/src/backend/catalog/index.c#L1542-L1680));
- copies the cumulative statistics counters (scans, tuples returned/fetched,
  blocks fetched/hit) from old to new so per-index stats survive the rebuild
  ([index.c#stats-copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1683-L1705)).

The relcache invalidation makes every session re-plan against the rebuilt index
after this commit.

#### Phase 5: wait for readers, then mark old indexes dead

**Wait 4** is `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c#wait4](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3272-L3274)).
Because `AccessExclusiveLock` conflicts with every lock mode including
`AccessShareLock`, this waits out **readers** — any transaction that might still
run a query through the old index — not just writers
([lock.c#AccessExclusive-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L99-L103)).
Then `index_concurrently_set_dead` transfers predicate locks to the heap and sets
`indisready = false, indislive = false` (dead) on each old index
([indexcmds.c#set-dead-loop](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3276-L3290),
[index.c#index_concurrently_set_dead](../../../raw/postgres-12/src/backend/catalog/index.c#L1727-L1761)).

#### Phase 6: wait for readers, then drop old indexes

After another `WaitForLockersMultiple(lockTags, AccessExclusiveLock, true)`
([indexcmds.c#wait5](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3302-L3304)),
RIC drops the old indexes with
`performMultipleDeletions(objects, DROP_RESTRICT, PERFORM_DELETION_CONCURRENT_LOCK | PERFORM_DELETION_INTERNAL)`
([indexcmds.c#drop-loop](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3306-L3329)).
`PERFORM_DELETION_CONCURRENT_LOCK` tells `index_drop` to take
`ShareUpdateExclusiveLock` (not `AccessExclusiveLock`) on the index and to skip its
own set-dead/wait logic, because RIC already did the dead-marking and the reader
waits itself
([index.c#index_drop-lockmode](../../../raw/postgres-12/src/backend/catalog/index.c#L2001-L2048)).
Finally RIC releases all the session locks
([indexcmds.c#release-session-locks](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3337-L3342)).

### All steps and locks required on the table

Two lock layers act on each **heap table**, exactly as in CIC: a
**transaction-level** `ShareUpdateExclusiveLock` re-taken inside each helper, and a
**session-level** `ShareUpdateExclusiveLock` that spans the commit gaps. RIC
additionally holds session `ShareUpdateExclusiveLock` on both the **old and new
index** relations
([indexcmds.c#session-locks](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3068-L3074)).
DML never conflicts with any of these
([lock.c#SUE-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81)).

The waits are **not** table locks. `WaitForLockersMultiple` reads the current set
of conflicting lock holders and sleeps on each holder's virtual transaction ID
until it ends; later transactions are not waited for
([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)).
The conflict mode passed to the wait decides who is waited out:

| Phase | Heap lock held | Wait | Conflict mode | Who it waits out |
|---|---|---|---|---|
| 1: create copies | txn + session `ShareUpdateExclusiveLock` | — | — | — |
| 2: build | `ShareUpdateExclusiveLock`; new index `RowExclusiveLock` | Wait 1 | `ShareLock` | open **writers** (`RowExclusiveLock`) ([lock.c:83-86](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86)) |
| 3: validate | `ShareUpdateExclusiveLock` | Wait 2 | `ShareLock` | open **writers** |
| 3: catch up | — | Wait 3 | `WaitForOlderSnapshots` | same-database **old-snapshot** holders ([indexcmds.c:339-402](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)) |
| 5: set dead | `ShareUpdateExclusiveLock` | Wait 4 | `AccessExclusiveLock` | **all** lock holders, including **readers** ([lock.c:99-103](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L99-L103)) |
| 6: drop | index `ShareUpdateExclusiveLock` | Wait 5 | `AccessExclusiveLock` | **all** lock holders, including **readers** |

So RIC has the same three CIC waits (writers, writers, old snapshots) **plus** two
reader waits that CIC does not have. Those two reader waits exist because the old
index is still being used for queries until it is marked dead, so RIC cannot
remove it until every backend that might read through it has finished
([reindex.sgml#concurrent-steps](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L334-L359)).

Because `ShareUpdateExclusiveLock` is self-conflicting, only one concurrent build
(CIC or RIC), `VACUUM`, or `ANALYZE` can run on a given table at a time
([lock.c#self-conflict](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L194-L196)).

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

The new copy's flags are set by the shared `index_set_state_flags` ladder
(`INDEX_CREATE_SET_READY`); its `indisvalid` is flipped by the swap, not by
`index_set_state_flags`
([index.c#swap-mark-valid](../../../raw/postgres-12/src/backend/catalog/index.c#L1531-L1534)).
The old index is set dead by `index_set_state_flags(INDEX_DROP_SET_DEAD)`, whose
assert requires the index already be invalid — which the swap guarantees
([index.c#INDEX_DROP_SET_DEAD](../../../raw/postgres-12/src/backend/catalog/index.c#L3384-L3396)).
All `index_set_state_flags` writes are **non-transactional in-place** updates that
cannot roll back
([index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)).

### Failure scenarios and the outcome on the table

Like CIC, RIC commits many times, so the outcome depends on **which phase failed**.
The single most important property: a RIC failure on a **healthy** index never
leaves you without a working index, because the swap (phase 4) is the only step
that touches the original, and it runs only after the new copy is fully built and
validated.

| Failure point | Leftover on the table | Recovery |
|---|---|---|
| Phase 1 (gather / create copies / initial lock) | **none** — phase 1's single transaction rolls back, so no `_ccnew` persists; original untouched | retry |
| Phase 2 (Wait 1 or build) | one or more **invalid `_ccnew`** copies (the failing one **not ready**, any earlier-built ones **ready**); original still valid | `DROP INDEX <name>_ccnew`, then retry |
| Phase 3 (Wait 2 / validate / Wait 3) | **invalid, ready `_ccnew`** copy; original still valid | `DROP INDEX <name>_ccnew`, then retry |
| Phase 4 (swap), before its commit | swap transaction rolls back: `_ccnew` still invalid, original still valid and normally named | `DROP INDEX <name>_ccnew`, then retry |
| Phase 5 or 6 (after swap committed) | the **new** index is already valid and carries the original name; the **old** `_ccold` index lingers (invalid, possibly dead) | `DROP INDEX <name>_ccold` |

The leftover is invalid because `plancat` never offers an `indisvalid = false`
index to the planner, and because RIC's failure paths are ordinary
ERROR/cancel/timeout aborts. Each phase loop runs inside a transaction precisely so
that an abort cleans up session locks rather than leaking them
([indexcmds.c#abort-cleanup](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3105-L3110)).

The regression suite stages the phase-2 case directly. Starting from an index left
invalid by a failed CIC over duplicate data,
`REINDEX INDEX CONCURRENTLY concur_reindex_ind5` fails its build with
`could not create unique index "concur_reindex_ind5_ccnew" ... Key (c1)=(1) is
duplicated`, leaving **both** the original and a new `_ccnew` marked `INVALID`
([create_index.out#ccnew-invalid](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2323-L2333)).
The spare is removed with `DROP INDEX concur_reindex_ind5_ccnew`, and after the
duplicate row is deleted, `REINDEX INDEX CONCURRENTLY` repairs the original — `\d`
then shows it without `INVALID`
([create_index.out#repair](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2335-L2358)).
The docs describe the same `_ccnew` / `_ccold` leftovers and recommend dropping the
invalid index and retrying
([reindex.sgml#failure-recovery](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)).

A unique `_ccnew` that reaches phase 3 enforces uniqueness while it is ready but
invalid, just like any ready index — so a duplicate appearing during the second
scan surfaces as a `duplicate key value violates unique constraint` error from
`_bt_check_unique`
([nbtinsert.c#dup-key](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568)).

### Multiple indexes in one command

`REINDEX TABLE CONCURRENTLY` (and reindex of a matview or toast relation) rebuilds
every qualifying index of the relation in one run. RIC processes **each phase for
all indexes before moving to the next phase**, and runs each index's build and
validation in its own transaction, to bound how long any single transaction stays
open
([indexcmds.c#phase-overview](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955),
[reindex.sgml#steps-loop](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L299-L302)).
The regression suite confirms a table reindex preserves index dependencies and
comments and rebuilds toast indexes, with the `pg_depend` listing identical before
and after
([create_index.out#deps-preserved](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2052-L2091),
[create_index.out#comment-preserved](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2110-L2115)).

### Watching the phases

RIC reports through `pg_stat_progress_create_index` with `command` =
`REINDEX CONCURRENTLY`
([indexcmds.c#progress-command](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2984-L2991)).
The wait phases map to the view's `phase` text via the integer codes in
`progress.h`
([progress.h#phases](../../../raw/postgres-12/src/include/commands/progress.h#L73-L82),
[system_views.sql#create-index-view](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1004-L1016)):
`waiting for writers before build`, `waiting for writers before validation`,
`waiting for old snapshots`, and `waiting for readers before marking dead`. See
`## Open Questions` for the unused `waiting for readers before dropping` text.

### Test coverage

- Functional coverage is in the main regression suite, in the `REINDEX CONCURRENTLY`
  block of `create_index.sql`: it reindexes tables, matviews, a single index, and
  toast indexes; checks dependency and comment preservation; rebuilds
  expression/predicate indexes and verifies their definitions are unchanged; and
  exercises the no-index `NOTICE`
  ([create_index.sql#ric-block](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L786-L905)).
- Restriction coverage: transaction-block rejection, system-catalog rejection,
  partitioned no-op/skip, exclusion-constraint error-vs-skip, invalid-index
  skip-vs-allow, and temporary-table fallback
  ([create_index.out#restrictions](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2278-L2296),
  [create_index.out#exclusion](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2029-L2032),
  [create_index.out#temp](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2439-L2460)).
- Failure coverage: the invalid `_ccnew` leftover, its manual drop, and the eventual
  repair
  ([create_index.out#invalid-handling](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2314-L2358)).
- **Test absence:** v12 ships an isolation spec for concurrent *creation*
  (`multiple-cic.spec`) but **no** dedicated isolation spec for `REINDEX
  CONCURRENTLY`; its concurrency is covered only indirectly through the functional
  tests above
  ([multiple-cic.spec](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v12/index.md`, and the last ~20
  `wiki/log.md` entries (navigation only).
- The sibling page
  [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](create-index-concurrently.md),
  which documents the shared build/validate core and the wait semantics this page
  reuses.
- Pinned checkout `raw/postgres-12/` at commit
  `45b88269a353ad93744772791feb6d01bc7e1e42` ("Stamp 12.2.").
- `ReindexRelationConcurrently`, `ReindexIndex`, `ReindexTable`,
  `ReindexMultipleTables`, and `WaitForOlderSnapshots` in
  `src/backend/commands/indexcmds.c`.
- `index_concurrently_create_copy`, `index_concurrently_build`,
  `index_concurrently_swap`, `index_concurrently_set_dead`, `index_drop`, and
  `index_set_state_flags` in `src/backend/catalog/index.c`.
- `WaitForLockersMultiple` in `src/backend/storage/lmgr/lmgr.c`; the `LockConflicts`
  table in `src/backend/storage/lmgr/lock.c`; lock-mode definitions in
  `src/include/storage/lockdefs.h`.
- REINDEX dispatch and `PreventInTransactionBlock`/`PreventCommandDuringRecovery`
  in `src/backend/tcop/utility.c`.
- The CREATE INDEX progress phase codes in `src/include/commands/progress.h` and the
  `pg_stat_progress_create_index` view in
  `src/backend/catalog/system_views.sql`; the phase descriptions in
  `doc/src/sgml/monitoring.sgml`.
- `doc/src/sgml/ref/reindex.sgml` (CONCURRENTLY steps, restrictions, recovery).
- Tests: `src/test/regress/sql/create_index.sql`,
  `src/test/regress/expected/create_index.out`, and the absence of a
  REINDEX-specific spec under `src/test/isolation/specs/`.

## Evidence Map

| Claim | Source |
|---|---|
| RIC runs as six phases (create copy, build, catch up, swap, set dead, drop), each phase looping all indexes | [indexcmds.c:2941-2955](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955) |
| CONCURRENTLY cannot run in a transaction block or during recovery | [utility.c:773-807](../../../raw/postgres-12/src/backend/tcop/utility.c#L773-L807) |
| `REINDEX INDEX` locks `ShareUpdateExclusiveLock` for concurrent; temp falls back to non-concurrent | [indexcmds.c:2336-2382](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2336-L2382) |
| `REINDEX TABLE` no-qualifying-index NOTICE | [indexcmds.c:2477-2499](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2477-L2499) |
| System catalogs rejected with error | [indexcmds.c:2804-2807](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2804-L2807), [indexcmds.c:2530-2533](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2530-L2533) |
| Partitioned table/index warns and skips | [indexcmds.c:2917-2923](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2917-L2923) |
| Invalid index skipped via table, allowed when named directly | [indexcmds.c:2819-2824](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2819-L2824), [indexcmds.c:2908-2912](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2908-L2912) |
| Exclusion index: error if named directly, skip if via table | [index.c:1268-1271](../../../raw/postgres-12/src/backend/catalog/index.c#L1268-L1271), [indexcmds.c:2825-2830](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2825-L2830) |
| Toast indexes are gathered and rebuilt with the table | [indexcmds.c:2844-2888](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2844-L2888) |
| Phase 1 creates `_ccnew` copy not-ready/not-valid via `index_concurrently_create_copy` | [indexcmds.c:2993-3003](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2993-L3003), [index.c:1240-1388](../../../raw/postgres-12/src/backend/catalog/index.c#L1240-L1388) |
| Session `ShareUpdateExclusiveLock` taken on old index, new index, and heap(s) | [indexcmds.c:3042-3074](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3042-L3074) |
| Phase 2: Wait 1 (`ShareLock`) then per-index build sets `indisready` | [indexcmds.c:3090-3128](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3090-L3128), [index.c:1399-1439](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439) |
| Phase 3: Wait 2 (`ShareLock`), `validate_index`, then Wait 3 (`WaitForOlderSnapshots`) | [indexcmds.c:3139-3199](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3139-L3199) |
| Phase 4 swap: new valid + old invalid, names swapped, constraints/triggers/comment/deps/stats moved | [indexcmds.c:3212-3262](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3212-L3262), [index.c:1447-1716](../../../raw/postgres-12/src/backend/catalog/index.c#L1447-L1716) |
| Old index renamed `_ccold`; new takes original name | [indexcmds.c:3230-3235](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3230-L3235), [index.c:1490-1492](../../../raw/postgres-12/src/backend/catalog/index.c#L1490-L1492) |
| Per-index cumulative stats copied old->new during swap | [index.c:1683-1705](../../../raw/postgres-12/src/backend/catalog/index.c#L1683-L1705) |
| Phase 5: Wait 4 (`AccessExclusiveLock`) then `index_concurrently_set_dead` (clears indisready+indislive) | [indexcmds.c:3272-3290](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3272-L3290), [index.c:1727-1761](../../../raw/postgres-12/src/backend/catalog/index.c#L1727-L1761) |
| Phase 6: Wait (`AccessExclusiveLock`) then `performMultipleDeletions` drops old | [indexcmds.c:3302-3329](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3302-L3329) |
| Drop uses `PERFORM_DELETION_CONCURRENT_LOCK` -> `index_drop` takes SUE, skips its own set-dead | [index.c:2001-2048](../../../raw/postgres-12/src/backend/catalog/index.c#L2001-L2048) |
| `AccessExclusiveLock` conflicts with `AccessShareLock`, so phases 5/6 wait out readers | [lock.c:99-103](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L99-L103) |
| `ShareLock` waits (Waits 1/2) wait out writers (`RowExclusiveLock`) | [lock.c:83-86](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86) |
| `WaitForLockersMultiple` waits on VXIDs, not later transactions | [lmgr.c:850-949](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949) |
| `index_set_state_flags` non-transactional in-place; `INDEX_DROP_SET_DEAD` asserts not valid | [index.c:3331-3403](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403) |
| Abort inside each phase loop cleans up session locks | [indexcmds.c:3105-3110](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3105-L3110) |
| Docs: two table scans, wait for transactions, six steps, `_ccnew`/`_ccold`, recovery | [reindex.sgml:283-390](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L283-L390) |
| Regression: failed `_ccnew` left INVALID, dropped, then repaired | [create_index.out:2323-2358](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2323-L2358) |
| Regression: deps and comments preserved across reindex | [create_index.out:2052-2115](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2052-L2115) |
| Regression: txn-block, system-catalog, partitioned, exclusion, temp restrictions | [create_index.out:2029-2032](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2029-L2032), [create_index.out:2278-2296](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2278-L2296), [create_index.out:2439-2460](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2439-L2460) |
| Progress command is `REINDEX CONCURRENTLY`; phase codes/text | [indexcmds.c:2984-2991](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2984-L2991), [progress.h:73-82](../../../raw/postgres-12/src/include/commands/progress.h#L73-L82), [system_views.sql:1004-1016](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1004-L1016) |

## Open Questions

- **The "waiting for readers before dropping" phase string is never emitted in
  v12.** `progress.h` defines `PROGRESS_CREATEIDX_PHASE_WAIT_5 = 9`, which the
  `pg_stat_progress_create_index` view and `monitoring.sgml` map to
  `waiting for readers before dropping`
  ([progress.h:81-82](../../../raw/postgres-12/src/include/commands/progress.h#L81-L82),
  [system_views.sql:1014-1015](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1014-L1015),
  [monitoring.sgml:3721-3730](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3721-L3730)).
  But phase 6 sets `PROGRESS_CREATEIDX_PHASE_WAIT_4` (value 8 =
  `waiting for readers before marking dead`), the same code phase 5 uses
  ([indexcmds.c:3272-3274](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3272-L3274),
  [indexcmds.c:3302-3304](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3302-L3304)).
  Per the source, the drop wait is reported as "waiting for readers before marking
  dead" and the `WAIT_5` text is unused. Source wins; this is recorded as a
  source-vs-view/docs discrepancy rather than a behavioral claim that the "before
  dropping" text appears.
- The first build scan's exact tuple-visibility rule lives inside `index_build` ->
  `table_index_build_scan` (via `ii_Concurrent`), which this page summarizes from
  the shared CIC core rather than tracing line-by-line into the table AM
  ([index.c:1426-1427](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1427)).
- The exact crash / immediate-shutdown outcome at each instruction boundary was not
  traced through crash recovery; as on the CIC page, `index_set_state_flags` is a
  non-transactional WAL-logged in-place overwrite, so the recovered flag state
  around the set-ready / swap / set-dead boundaries was not isolated
  ([index.c:3316-3324](../../../raw/postgres-12/src/backend/catalog/index.c#L3316-L3324)).
- Whether reindexing multiple indexes that share a long base name can collide on
  the `_ccnew` / `_ccold` names is mitigated by `ChooseRelationName` plus the
  per-iteration `CommandCounterIncrement`
  ([indexcmds.c:3250-3257](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3250-L3257)),
  but the exact truncation/uniqueness behavior of `ChooseRelationName` was not
  traced.

## Source References

- [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2715-L3382)
- [indexcmds.c#ReindexIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2336-L2382)
- [indexcmds.c#ReindexTable](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2458-L2499)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)
- [index.c#index_concurrently_create_copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1240-L1388)
- [index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439)
- [index.c#index_concurrently_swap](../../../raw/postgres-12/src/backend/catalog/index.c#L1447-L1716)
- [index.c#index_concurrently_set_dead](../../../raw/postgres-12/src/backend/catalog/index.c#L1719-L1761)
- [index.c#index_drop](../../../raw/postgres-12/src/backend/catalog/index.c#L2001-L2048)
- [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)
- [lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)
- [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)
- [lockdefs.h#lockmodes](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46)
- [utility.c#reindex-dispatch](../../../raw/postgres-12/src/backend/tcop/utility.c#L773-L807)
- [progress.h#create-index-phases](../../../raw/postgres-12/src/include/commands/progress.h#L73-L82)
- [system_views.sql#pg_stat_progress_create_index](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1004-L1016)
- [monitoring.sgml#create-index-phases](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3630-L3731)
- [ref/reindex.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L283-L414)
- [nbtinsert.c#_bt_check_unique](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568)
- [create_index.sql#ric-block](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L786-L905)
- [create_index.out#ric-coverage](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2005-L2360)
- [multiple-cic.spec](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40)

## Navigation

- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](create-index-concurrently.md)
- [v12 index](../index.md)
- [versions](../../versions.md)
