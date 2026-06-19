---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Preconditions and restrictions](#preconditions-and-restrictions)
  - [The three pg_index state flags](#the-three-pgindex-state-flags)
  - [Step-by-step implementation](#step-by-step-implementation)
  - [The first build scan's tuple-visibility rule](#the-first-build-scans-tuple-visibility-rule)
  - [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table)
  - [How concurrent index builds interact with each other](#how-concurrent-index-builds-interact-with-each-other)
  - [All operations that can block CREATE INDEX CONCURRENTLY](#all-operations-that-can-block-create-index-concurrently)
  - [Is skipping prepared transactions in the writer waits safe?](#is-skipping-prepared-transactions-in-the-writer-waits-safe)
  - [Can walsenders or replication-slot xmin holders appear in the Wait 3 set?](#can-walsenders-or-replication-slot-xmin-holders-appear-in-the-wait-3-set)
  - [Why two scans and three waits](#why-two-scans-and-three-waits)
  - [Failure scenarios and the outcome on the table](#failure-scenarios-and-the-outcome-on-the-table)
  - [Test coverage](#test-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

give a comprehensive explanation of how create index concurrently is
implemented, add a section with all steps and locks required on the table.

Follow-up (2026-06-10): also add all the possible operations that could block
a `CREATE INDEX CONCURRENTLY`.

Follow-up (2026-06-10): does a running `pg_dump` block it? How about another
open transaction that will not commit for 1 hour? How about a session that is
idle in transaction?

Follow-up (2026-06-12): also add a new section with a comprehensive list of all
failure scenarios and the outcome on the table, like invalid indexes, etc.

## Answer

`CREATE INDEX CONCURRENTLY` (CIC) builds an index without ever taking a lock
that blocks `INSERT`/`UPDATE`/`DELETE` on the table. It pays for that with more
total work: it splits the build across **four internal transactions**, does
**two full table scans**, and **waits out other transactions three times**. The
whole dance exists to keep the index correct under concurrent writes, since the
build cannot freeze the table.

All of the orchestration lives in `DefineIndex()` in the concurrent path: it
creates not-built catalog entries with `INDEX_CREATE_CONCURRENT` and
`INDEX_CREATE_SKIP_BUILD`, then runs the post-commit build, validation, and
mark-valid phases
([indexcmds.c#flags](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L974-L986),
[indexcmds.c#concurrent-phases](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1472)).
Those phases call helpers in `index.c` (`index_create`,
`index_concurrently_build`, `validate_index`, `index_set_state_flags`) and wait
via `WaitForLockers` / `WaitForOlderSnapshots`.

The table lock used throughout is **`ShareUpdateExclusiveLock`** — strong enough
to keep out a second CIC, `VACUUM`, `ANALYZE`, and schema changes, but weak
enough to let normal DML proceed
([lockdefs.h:36-46](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46)).

### Preconditions and restrictions

Before any build work, the command is rejected in several cases:

| Restriction | Where enforced |
|---|---|
| Cannot run inside a transaction block (`BEGIN; ... COMMIT;`) | `PreventInTransactionBlock(isTopLevel, "CREATE INDEX CONCURRENTLY")` ([utility.c:1307-1309](../../../raw/postgres-12/src/backend/tcop/utility.c#L1307-L1309)) |
| Temporary tables silently fall back to a non-concurrent build | [indexcmds.c#temp-fallback](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499) |
| Partitioned tables cannot be built concurrently | [indexcmds.c#partitioned-error](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L604-L616) |
| System catalog tables cannot be indexed concurrently | [index.c#system-catalog](../../../raw/postgres-12/src/backend/catalog/index.c#L813-L817) |
| Exclusion constraints cannot be built concurrently | [index.c#exclusion](../../../raw/postgres-12/src/backend/catalog/index.c#L823-L826) |

The reason CIC cannot run in a transaction block is structural: the
implementation must itself commit several times, which is impossible inside a
user-opened transaction.

### The three pg_index state flags

CIC is driven by three boolean flags on the index's `pg_index` row, all set by
`UpdateIndexRelation`
([index.c#UpdateIndexRelation](../../../raw/postgres-12/src/backend/catalog/index.c#L612-L615)):

- `indislive` — the index exists and must be maintained. CIC sets this `true`
  from the start.
- `indisready` — new tuples (from `INSERT`/non-HOT `UPDATE`) must be inserted
  into the index.
- `indisvalid` — the planner may use the index to answer queries.

A normal `CREATE INDEX` is born with all three `true`. CIC instead creates the
catalog row with `indisvalid = false` and `indisready = false`
(`!concurrent && !invalid` and `!concurrent` respectively at
[index.c:990-996](../../../raw/postgres-12/src/backend/catalog/index.c#L990-L996)),
then flips `indisready`, and finally `indisvalid`, at carefully chosen points.
The transitions are applied by `index_set_state_flags` using a
**non-transactional in-place update** so they cannot roll back
([index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)).

### Step-by-step implementation

The concurrent branch of `DefineIndex` runs as four transactions. The commit /
start boundaries are the `CommitTransactionCommand()` / `StartTransactionCommand()`
pairs in the source.

**Transaction 1 — create the catalog entry (no build).** `index_create` makes
the `pg_index`/`pg_class` rows with `INDEX_CREATE_CONCURRENT` and
`INDEX_CREATE_SKIP_BUILD`, so the index has a catalog identity but no data, and
is marked not-ready/not-valid
([indexcmds.c#flags](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L974-L986),
[indexcmds.c#index_create-call](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1005-L1014)).
Before committing, CIC takes a **session-level** `ShareUpdateExclusiveLock` on
the table so it survives across the upcoming commits and nobody can drop the
table or index
([indexcmds.c#session-lock](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1320)).
The commit makes the empty index visible so other backends stop making
incompatible HOT updates.

**Wait 1**, then **Transaction 2 — first scan, build the index.** CIC waits for
every transaction that could still have the table open without the new index in
its list, using `WaitForLockers(heaplocktag, ShareLock, true)`
([indexcmds.c#wait1](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1328-L1346)).
It then takes a fresh snapshot and calls `index_concurrently_build`, which scans
the heap, builds the index, and sets `indisready = true`
([indexcmds.c#build](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1366-L1370),
[index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439)).
This scan indexes only tuples valid as of the scan's snapshot; rows written by
still-running or later transactions are handled differently (see below). The
commit publishes `indisready` so new writers start maintaining the index.

**Wait 2**, then **Transaction 3 — second scan, validate.** CIC waits again with
`WaitForLockers(..., ShareLock, ...)` so that every in-flight transaction now
sees the index as ready-for-inserts
([indexcmds.c#wait2](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1382-L1389)).
It registers a **reference snapshot**, then calls `validate_index`
([indexcmds.c#validate](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1391-L1412)).
`validate_index` collects all TIDs already in the index (via an
`index_bulk_delete` callback that only records, never deletes), sorts them, then
scans the heap and "merge-joins" against that sorted TID list, inserting any
tuple that is visible to the reference snapshot but missing from the index
([index.c#validate_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3176-L3298)).
Before committing, CIC saves the reference snapshot's `xmin` as `limitXmin` and
**drops the snapshot** — deliberately, to avoid deadlocking against other CIC
runs that would otherwise wait on it
([indexcmds.c#drop-snapshot](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1414-L1424)).

**Wait 3**, then **Transaction 4 — mark valid.** In a fresh transaction (so no
snapshot is held), CIC calls `WaitForOlderSnapshots(limitXmin, true)` to wait
out any transaction whose snapshot predates the reference snapshot and could
therefore expect to see rows the index does not contain
([indexcmds.c#wait3](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1437-L1448)).
Then it sets `indisvalid = true`, sends a relcache invalidation on the parent
table so cached plans get re-planned to use the new index, and releases the
session lock
([indexcmds.c#set-valid](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1450-L1472)).
This last transaction's commit happens in the caller when the utility command
finishes.

The canonical narrative of why this ordering is correct is in the
`validate_index` header comment
([index.c#validate_index-overview](../../../raw/postgres-12/src/backend/catalog/index.c#L3112-L3174))
and the user docs
([ref/create_index.sgml#concurrent-narrative](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L572)).

### The first build scan's tuple-visibility rule

The first scan (Transaction 2) indexes **exactly the heap tuples that are live
according to one fresh MVCC snapshot taken at the start of that transaction**,
after Wait 1. It does *not* run the time-qual logic a normal `CREATE INDEX`
uses, and it never indexes recently-dead tuples. That rule lives in the heap
table AM, not in `DefineIndex`; the steps below trace it from the concurrent
flag down to the per-tuple visibility test.

**Where the choice is made.** `index_concurrently_build` rebuilds the
`IndexInfo`, sets `ii_Concurrent = true`, then calls `index_build`
([index.c#ii_Concurrent](../../../raw/postgres-12/src/backend/catalog/index.c#L1421-L1427)).
`index_build` calls the access method's `ambuild`
([index.c#ambuild-call](../../../raw/postgres-12/src/backend/catalog/index.c#L2902-L2903)),
and every core index AM's `ambuild` scans the heap through
`table_index_build_scan` — B-tree
([nbtsort.c#build-scan](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L489-L491)),
hash ([hash.c:166](../../../raw/postgres-12/src/backend/access/hash/hash.c#L166)),
GiST ([gistbuild.c:196](../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L196)),
SP-GiST ([spginsert.c:126](../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L126)),
GIN ([gininsert.c:382](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L382)),
and BRIN ([brin.c:723](../../../raw/postgres-12/src/backend/access/brin/brin.c#L723)).
So the visibility rule is AM-independent: it is decided once in the heap AM and
inherited by every index type built concurrently. `table_index_build_scan`
dispatches to the heap AM's `heapam_index_build_range_scan` and always passes
`anyvisible = false`
([tableam.h#table_index_build_scan](../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533),
[heapam_handler.c#tableam-routine](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2644)).

**The MVCC-vs-`SnapshotAny` fork.** Inside `heapam_index_build_range_scan`,
`ii_Concurrent` picks the snapshot. `OldestXmin` is left invalid for a
concurrent build because the `GetOldestXmin` call is guarded by
`!indexInfo->ii_Concurrent`; a non-concurrent build instead computes
`OldestXmin` and scans with `SnapshotAny`
([heapam_handler.c#snapshot-choice](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1212-L1223)).
With `OldestXmin` invalid, the serial build registers a regular MVCC
transaction snapshot (`RegisterSnapshot(GetTransactionSnapshot())`) and begins
the scan with it via `table_beginscan_strat`
([heapam_handler.c#mvcc-snapshot](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1233-L1246)).
This is the transaction-2 MVCC snapshot `DefineIndex` set up for: just before
calling the build it pushes a fresh `GetTransactionSnapshot()` and comments that
it will "build the index using all tuples that are visible in this snapshot"
([indexcmds.c#build-snapshot](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1358-L1370)).
A parallel B-tree CIC build (CIC passes `parallel = true` to `index_build`)
does not register its own snapshot; it inherits the MVCC snapshot from the
parallel heap scan, chosen by the same `ii_Concurrent` criteria
([heapam_handler.c#parallel-snapshot](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1248-L1260)).

**What "visible to the snapshot" means, line by line.** The scan calls
`heap_getnext`, which in page-at-a-time mode filters each page in `heapgetpage`:
every normal line pointer is passed to `HeapTupleSatisfiesVisibility`, and only
tuples that pass are recorded in `rs_vistuples`
([heapam.c#heapgetpage](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L444-L453)).
For an MVCC snapshot, `HeapTupleSatisfiesVisibility` dispatches to
`HeapTupleSatisfiesMVCC`
([heapam_visibility.c#dispatch](../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1690-L1696)).
So by the time a tuple reaches the build loop, its MVCC visibility is already
decided. The build loop reflects that: because `snapshot != SnapshotAny`, it
takes the `else` branch — "heap_getnext did the time qual check" — marks the
tuple alive, counts it, and (after any partial-index predicate) hands it to the
AM callback
([heapam_handler.c#mvcc-branch](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1595-L1615)).

**What the concurrent scan therefore skips.** The entire
`switch (HeapTupleSatisfiesVacuum(...))` time-qual block runs only on the
`SnapshotAny` path and is never entered for a concurrent build
([heapam_handler.c#vacuum-switch](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1364-L1388)).
Two consequences follow directly:

- **No recently-dead tuples.** A normal build indexes
  `HEAPTUPLE_RECENTLY_DEAD` tuples to preserve MVCC semantics for old
  snapshots; the concurrent build cannot reach that case, so it omits them
  ([heapam_handler.c#recently-dead](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1402-L1428)).
  CIC compensates with Wait 3, which waits out every transaction old enough to
  have needed those omitted rows before it sets `indisvalid`.
- **No `ii_BrokenHotChain` from the scan.** `ii_BrokenHotChain` is set only
  inside that `SnapshotAny` switch
  ([heapam_handler.c#broken-hot](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1418-L1423)),
  so a concurrent build never flags a broken HOT chain during the first scan,
  and `index_build` accordingly skips the `indcheckxmin` write for concurrent
  builds
  ([index.c#concurrent-indcheckxmin](../../../raw/postgres-12/src/backend/catalog/index.c#L2939-L2952)).

**Why MVCC and not `HeapTupleSatisfiesVacuum`.** The `validate_index` header
gives the reason this page previously only summarized: using
`HeapTupleSatisfiesVacuum` could make two versions of the same
concurrently-updated row both look valid, producing a bogus unique-index
failure; one MVCC snapshot sees exactly one version
([index.c#validate_index-why-mvcc](../../../raw/postgres-12/src/backend/catalog/index.c#L3124-L3132)).
The price is that the first scan "does not contain any tuples added to the
table while we built the index," which is exactly why CIC then sets
`indisready`, waits again, and runs the second `validate_index` scan to
backfill the rest
([index.c#validate_index-second-scan](../../../raw/postgres-12/src/backend/catalog/index.c#L3134-L3168)).

### All steps and locks required on the table

Two distinct lock layers act on the **heap table**:

1. A **transaction-level** `ShareUpdateExclusiveLock`, acquired by `table_open`
   at the start of the command and re-acquired inside `index_concurrently_build`
   and `validate_index`; it is released at each `CommitTransactionCommand`.
2. A **session-level** `ShareUpdateExclusiveLock`
   (`LockRelationIdForSession`) that spans the gaps between transactions so the
   table cannot be dropped mid-build; released by `UnlockRelationIdForSession`
   at the very end.

`ShareUpdateExclusiveLock` is the same level `VACUUM (non-FULL)` and `ANALYZE`
use ([lockdefs.h:36-46](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46)).
Its conflict row shows what it lets through and what it blocks
([lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)):

| Other lock (command) | Conflicts with CIC's `ShareUpdateExclusiveLock`? |
|---|---|
| `AccessShareLock` (`SELECT`) | No — reads continue |
| `RowShareLock` (`SELECT FOR UPDATE/SHARE`) | No |
| `RowExclusiveLock` (`INSERT`/`UPDATE`/`DELETE`) | No — **writes continue** |
| `ShareUpdateExclusiveLock` (another CIC, `VACUUM`, `ANALYZE`) | **Yes** — only one at a time |
| `ShareLock` (plain `CREATE INDEX`) | **Yes** |
| `ShareRowExclusiveLock` / `ExclusiveLock` / `AccessExclusiveLock` (most `ALTER TABLE`, `DROP`) | **Yes** — schema changes blocked |

Because `ShareUpdateExclusiveLock` is **self-conflicting**, only one concurrent
build can run on a given table at a time
([lock.c#self-conflict](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L194-L196)).

The three "waits" are **not** table locks. `WaitForLockers` does not acquire any
lock on the table; it reads the current set of conflicting lock holders with
`GetLockConflicts` and then sleeps on each one's virtual transaction ID until it
ends. New transactions that start after the holder list is taken are not waited
for ([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)).
Waits 1 and 2 pass `ShareLock`, which conflicts with `RowExclusiveLock`, so they
wait out all current **writers**
([lock.c#ShareLock-row](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86)).
Wait 3 (`WaitForOlderSnapshots`) instead waits on transactions holding old
snapshots, excluding autovacuum workers and lazy `VACUUM` (which cannot be
confused by missing index entries), but it does **not** exclude other CIC
operations
([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)).

End-to-end table lock timeline:

| Phase | Table (heap) locks | Wait performed | Catalog effect |
|---|---|---|---|
| Txn 1: create catalog row | Txn `ShareUpdateExclusiveLock` + session `ShareUpdateExclusiveLock` taken ([indexcmds.c:563-564](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L563-L564), [:1316](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1316)) | — | `indislive=t`, `indisready=f`, `indisvalid=f` |
| (commit 1) | Txn lock released; session lock held | — | empty index now visible |
| Txn 2: build | Heap `ShareUpdateExclusiveLock`; index `RowExclusiveLock` ([index.c:1411-1414](../../../raw/postgres-12/src/backend/catalog/index.c#L1411-L1414)) | Wait 1: `WaitForLockers(ShareLock)` waits out writers | first scan; then `indisready=t` |
| (commit 2) | Txn lock released; session lock held | — | `indisready` visible |
| Txn 3: validate | Heap `ShareUpdateExclusiveLock`; index `RowExclusiveLock` ([index.c:3204-3206](../../../raw/postgres-12/src/backend/catalog/index.c#L3204-L3206)) | Wait 2: `WaitForLockers(ShareLock)` waits out writers | second scan inserts missing tuples |
| (commit 3) | Txn lock released; session lock held | — | reference snapshot dropped |
| Txn 4: mark valid | Session `ShareUpdateExclusiveLock` still held | Wait 3: `WaitForOlderSnapshots(limitXmin)` | `indisvalid=t`; relcache inval; session lock released |

Throughout, the **only** lock the table ever carries is
`ShareUpdateExclusiveLock` (transaction-level within each phase, session-level
across the gaps). DML conflicts with none of these, which is the whole point of
CIC.

### How concurrent index builds interact with each other

PostgreSQL 12 does **not** have a special safe-concurrent-index-build exclusion
in `WaitForOlderSnapshots`: the call passes only `PROC_IS_AUTOVACUUM |
PROC_IN_VACUUM` as its exclusion mask
([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L348)).
Concurrent index builders interact through two ordinary mechanisms: relation
locks and database-wide old-snapshot waits.

- **Same table: the lock serializes them before the snapshot wait.**
  `DefineIndex` opens the heap with `ShareUpdateExclusiveLock` for a concurrent
  build and then keeps a session-level `ShareUpdateExclusiveLock` across the
  phase commits
  ([indexcmds.c#heap-lockmode](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L563-L564),
  [indexcmds.c#session-lock](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1316)).
  That lock conflicts with itself
  ([lock.c#SUE-self-conflict](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81)),
  and the v12 docs state that only one concurrent index build can occur on a
  table at a time
  ([ref/create_index.sgml#one-concurrent-build-per-table](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L614-L621)).
  So a second CIC on the same heap waits at the initial table-lock acquisition;
  it does not reach a same-table mutual `WaitForOlderSnapshots` cycle.

- **Different tables: writer waits are table-local, but the old-snapshot wait is
  database-wide.** Waits 1 and 2 call `WaitForLockers(heaplocktag, ShareLock,
  true)` for the target heap only
  ([indexcmds.c#wait1](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1328-L1346),
  [indexcmds.c#wait2](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1381-L1389)).
  `WaitForLockersMultiple` waits only for the lock holders it collects from the
  passed lock tags; it does not acquire the heap lock itself, and later holders
  are outside that wait set
  ([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L860)).
  Wait 3 is different: `WaitForOlderSnapshots` calls
  `GetCurrentVirtualXIDs(limitXmin, true, false, PROC_IS_AUTOVACUUM |
  PROC_IN_VACUUM, ...)`, so it filters by same database, nonzero `xmin`, and
  `xmin <= limitXmin`, not by table
  ([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L348),
  [procarray.c#GetCurrentVirtualXIDs-filters](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2471-L2490),
  [procarray.c#GetCurrentVirtualXIDs-loop](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2508-L2540)).
  The only v12 exclusion mask there is autovacuum plus manual lazy `VACUUM`
  (`PROC_IS_AUTOVACUUM | PROC_IN_VACUUM`)
  ([proc.h#vacuumFlags](../../../raw/postgres-12/src/include/storage/proc.h#L53-L63)).
  Therefore a different-table CIC, or a `REINDEX CONCURRENTLY` build using the
  same helper, can be in the Wait 3 set if it is in the same database and is
  still advertising an old `xmin`.

- **The deadlock-avoidance step clears the builder's own advertised `xmin` before
  waiting.** After validation, CIC saves the reference snapshot's `xmin`, pops
  and unregisters that snapshot, commits, starts a new transaction, asserts that
  `MyPgXact->xmin` is invalid, and only then calls `WaitForOlderSnapshots`
  ([indexcmds.c#drop-reference-snapshot](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1414-L1424),
  [indexcmds.c#no-xmin-before-wait](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1426-L1438),
  [indexcmds.c#wait3](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1440-L1448)).
  The source comment names other CIC processes as the reason to drop the
  reference snapshot before waiting
  ([indexcmds.c#drop-reference-snapshot](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1415-L1419)).
  This does not make other builders invisible. It makes the waiting builder stop
  being an old-snapshot holder before it waits on anyone else.

- **`REINDEX CONCURRENTLY` follows the same boundary.**
  `ReindexRelationConcurrently` takes analogous `ShareUpdateExclusiveLock`
  relation locks, records heap lock tags for the writer waits, registers and
  drops a reference snapshot for validation, commits into a new transaction, and
  then calls `WaitForOlderSnapshots`
  ([indexcmds.c#RIC-locks](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2957-L3077),
  [indexcmds.c#RIC-build-validate-wait](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3080-L3198)).
  So the snapshot-wait interaction is not a CIC-only code path.

- **The direct inter-CIC isolation test covers different tables.**
  `multiple-cic.spec` creates two tables, starts two simultaneous CIC commands,
  and uses an advisory-lock interleaving to make the first command wait while
  the second command runs
  ([multiple-cic.spec](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40)).
  The expected output shows the first CIC waiting, the second CIC executing, and
  then the first completing
  ([multiple-cic.out](../../../raw/postgres-12/src/test/isolation/expected/multiple-cic.out#L3-L19)).

Resolved conclusion: the old open question was too narrow. In the traced v12
paths, inter-builder behavior consists of same-table serialization by
`ShareUpdateExclusiveLock`, same-database old-snapshot waiting through
`GetCurrentVirtualXIDs`, and the final-phase rule that the builder must clear
its own advertised `xmin` before calling `WaitForOlderSnapshots`
([indexcmds.c#heap-lockmode](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L563-L564),
[procarray.c#GetCurrentVirtualXIDs-loop](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2508-L2540),
[indexcmds.c#no-xmin-before-wait](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1426-L1448)).
The traced `DefineIndex`, `WaitForLockers`, `WaitForOlderSnapshots`,
`GetCurrentVirtualXIDs`, and `ReindexRelationConcurrently` paths show those
mechanisms, not a separate v12 CIC-specific exclusion flag or side channel
([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L860),
[indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L348),
[indexcmds.c#RIC-build-validate-wait](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3080-L3198)).

### All operations that can block CREATE INDEX CONCURRENTLY

CIC itself can be made to wait at four points: the initial table-lock
acquisition, the two `WaitForLockers(ShareLock)` waits, and the
`WaitForOlderSnapshots` wait. Each point has a different set of blockers.

#### Point 1 — acquiring `ShareUpdateExclusiveLock` at command start

The command's first action is to look up and lock the table in
`ShareUpdateExclusiveLock` mode
([utility.c#CIC-lockmode](../../../raw/postgres-12/src/backend/tcop/utility.c#L1311-L1326)).
CIC queues until every holder of a conflicting mode —
`ShareUpdateExclusiveLock`, `ShareLock`, `ShareRowExclusiveLock`,
`ExclusiveLock`, or `AccessExclusiveLock`
([lock.c#SUE-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81))
— ends its transaction. The v12 operations that take those modes on a table:

| Lock mode held by another session | Operations (v12) |
|---|---|
| `ShareUpdateExclusiveLock` | `VACUUM` (non-FULL), autovacuum, `ANALYZE`, another `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY`, `CREATE STATISTICS`, certain `ALTER TABLE`/`ALTER INDEX` variants ([mvcc.sgml#SHARE-UPDATE-EXCLUSIVE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L912-L936)) |
| `ShareLock` | plain `CREATE INDEX` ([mvcc.sgml#SHARE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L938-L956)) |
| `ShareRowExclusiveLock` | `CREATE TRIGGER`, some `ALTER TABLE` forms ([mvcc.sgml#SHARE-ROW-EXCLUSIVE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L958-L978)) |
| `ExclusiveLock` | `REFRESH MATERIALIZED VIEW CONCURRENTLY` ([mvcc.sgml#EXCLUSIVE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L980-L1000)) |
| `AccessExclusiveLock` | `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL`, `REFRESH MATERIALIZED VIEW` (non-concurrent), many `ALTER TABLE`/`ALTER INDEX` forms, and `LOCK TABLE` without an explicit mode ([mvcc.sgml#ACCESS-EXCLUSIVE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1002-L1030)) |

`LOCK TABLE` naming any of these five modes explicitly blocks the acquisition
the same way, per the conflict table
([lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L105)).
Plain reads and writes never block this point: `AccessShareLock`,
`RowShareLock`, and `RowExclusiveLock` are absent from
`ShareUpdateExclusiveLock`'s conflict mask
([lock.c#SUE-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81)).

Two special blockers at this point:

- **Autovacuum** holding the table's `ShareUpdateExclusiveLock` normally
  yields: after CIC has waited `deadlock_timeout`, the lock manager detects
  `DS_BLOCKED_BY_AUTOVACUUM` and sends the worker `SIGINT`
  ([proc.c#autovacuum-cancel](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)).
  The exception is an anti-wraparound autovacuum
  (`PROC_VACUUM_FOR_WRAPAROUND`), which is never cancelled and must finish
  ([proc.c#wraparound-exception](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1319-L1324)).
- **Prepared transactions** keep their locks after `PREPARE TRANSACTION`; the
  locks are transferred to the primary lock table and stay held until
  `COMMIT PREPARED` / `ROLLBACK PREPARED`
  ([lock.c#prepared-locks](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876)).
  A prepared transaction holding a conflicting mode blocks CIC indefinitely.

The session-level lock taken just before commit 1 never blocks, because the
same lock is already held at transaction level
([indexcmds.c#session-lock-noblock](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1310)).

#### Points 2 and 3 — Wait 1 (before build) and Wait 2 (before validation)

`WaitForLockers(heaplocktag, ShareLock, true)` collects the virtual
transaction IDs of transactions **currently holding** table locks that
conflict with `ShareLock`, then sleeps on each one until its whole transaction
ends ([lmgr.c#wait-loop](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L896-L918)).
`ShareLock` conflicts with `RowExclusiveLock` plus every mode from
`ShareUpdateExclusiveLock` up
([lock.c#ShareLock-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86)),
but CIC's own `ShareUpdateExclusiveLock` already keeps out every holder at
that level or above, so in practice these waits block on one thing: **open
transactions that hold `RowExclusiveLock`**, i.e. any transaction that ran
`INSERT`, `UPDATE`, `DELETE`, or any other data-modifying command on the table
([mvcc.sgml#ROW-EXCLUSIVE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L910)).
The wait is for the transaction, not the statement: a session that wrote one
row and then sits idle in transaction blocks CIC until it commits or rolls
back.

Not waited for at these two points:

- Plain `SELECT` (`AccessShareLock`) and `SELECT FOR UPDATE/SHARE`
  (`RowShareLock`) — neither mode is in `ShareLock`'s conflict mask
  ([lock.c#ShareLock-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86)).
- Transactions merely **waiting** for a conflicting lock; only current
  holders are reported
  ([lock.c#GetLockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2804-L2818)).
- Writers that start after the holder list is collected; they already see the
  index ([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L855-L860)).
- **Prepared transactions**: `GetLockConflicts` ignores them
  ([lock.c#GetLockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2815-L2818)),
  and `WaitForLockersMultiple` states this is fine "since they certainly
  aren't going to do anything anymore"
  ([lmgr.c#prepared-comment](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L890-L894)).
  This skip is **not** sufficient for index correctness on its own — see
  [Is skipping prepared transactions in the writer waits safe?](#is-skipping-prepared-transactions-in-the-writer-waits-safe).

These waits use real lock acquisition on each holder's virtual transaction ID
precisely so that deadlock detection applies; a cycle errors out instead of
hanging forever
([indexcmds.c#wait-deadlock-note](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1338-L1342)).

#### Point 4 — Wait 3 (before marking valid): old snapshot holders

`WaitForOlderSnapshots(limitXmin, true)` is the broadest wait. It blocks on
every backend **in the same database** whose advertised `xmin` is at or below
the reference snapshot's `xmin` — regardless of which tables that backend
touches
([indexcmds.c#wait3-filter](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L346-L348),
[procarray.c#GetCurrentVirtualXIDs](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2467-L2548)).
Operations that block here:

- Any long-running query in the same database, even a single-statement
  `SELECT` against an unrelated table — the filter is "oldest live snapshot
  older than ours", not "uses this table"
  ([procarray.c#xmin-filter](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2480-L2482),
  and the docs describe the phase as waiting for "transactions that can
  potentially see the table to release their snapshots"
  ([monitoring.sgml#wait-old-snapshots](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3699-L3708))).
- Open `REPEATABLE READ` / `SERIALIZABLE` transactions whose first snapshot
  predates the reference snapshot: the first snapshot "must live until end of
  xact"
  ([snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L336-L356)).
- `READ COMMITTED` sessions idle in transaction while still holding a
  registered snapshot (for example an open cursor); only backends whose
  `xmin` is zero are skipped
  ([procarray.c#xmin0-skip](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2525-L2526)).
- Manual `ANALYZE`, which the wait explicitly cannot exclude because it may
  run inside a transaction that does arbitrary work later
  ([indexcmds.c#analyze-not-excluded](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L322-L326)).
- Another `CREATE INDEX CONCURRENTLY` or `REINDEX CONCURRENTLY` in the same
  database while it holds a snapshot (its build or validation scan): the v12
  exclusion mask covers only vacuum flags, not other index builds
  ([indexcmds.c#wait3-filter](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L346-L348)).
- A running `pg_dump` of the same database — see the worked example below.

Not waited for at this point:

- Autovacuum workers and backends running manual lazy `VACUUM`
  (`PROC_IS_AUTOVACUUM | PROC_IN_VACUUM`
  [proc.h#vacuumFlags](../../../raw/postgres-12/src/include/storage/proc.h#L53-L56),
  filtered at
  [indexcmds.c#wait3-filter](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L346-L348)).
- Sessions attached to **other databases**, which can never see the index
  ([indexcmds.c#other-dbs](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L318-L320),
  [procarray.c#same-db](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520)).
- Backends with no live snapshot (`xmin` = 0). The wait also re-checks the
  list between sleeps and drops any backend whose `xmin` has since cleared
  ([indexcmds.c#idle-recheck](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L331-L338),
  [indexcmds.c#recheck-loop](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L357-L383)).
- Backends whose `xmin` is newer than `limitXmin`
  ([procarray.c#limitXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2532-L2533)).
- **Prepared transactions**: their dummy `PGPROC` carries
  `xmin = InvalidTransactionId` and `backendId = InvalidBackendId`
  ([twophase.c#MarkAsPreparingGuts](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472)),
  so both the xmin filter and the virtual-transaction-ID validity check skip
  them
  ([procarray.c#vxid-check](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2525-L2539)).
- **Walsenders and replication-slot xmin holders**: a standby using
  `hot_standby_feedback`, and any physical or logical replication slot, hold
  back the cluster's dead-row *removal* horizon, but never as a per-backend
  snapshot the wait can see — see
  [Can walsenders or replication-slot xmin holders appear in the Wait 3 set?](#can-walsenders-or-replication-slot-xmin-holders-appear-in-the-wait-3-set).

#### Worked example: a running pg_dump

A `pg_dump` of the **same database** blocks CIC at exactly one point — Wait 3
— but reliably, and for the dump's entire remaining duration. `pg_dump` wraps
the whole dump in a single transaction: `BEGIN` followed by
`SET TRANSACTION ISOLATION LEVEL REPEATABLE READ, READ ONLY` (or
`SERIALIZABLE, READ ONLY, DEFERRABLE` under `--serializable-deferrable`)
([pg_dump.c#dump-transaction](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1166-L1194)),
so its advertised `xmin` stays pinned from start to finish
([snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L336-L356)).
It also disables `statement_timeout`, `lock_timeout`, and
`idle_in_transaction_session_timeout` on its connection, so no timeout will
end it early
([pg_dump.c#disable-timeouts](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147)).

The other three points never see it: the only table locks `pg_dump` takes are
`ACCESS SHARE`
([pg_dump.c#lock-table](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6646-L6671)),
which is in neither `ShareUpdateExclusiveLock`'s nor `ShareLock`'s conflict
mask ([lock.c#SUE-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81),
[lock.c#ShareLock-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86))
— even when it is dumping the very table being indexed. Two boundary cases
follow from the Wait 3 filters above: a dump of **another database** is
skipped entirely, and a dump that **starts after** CIC's reference snapshot
has a newer `xmin` than `limitXmin` and is skipped too. The blocking is also
one-way: CIC never blocks `pg_dump`, since `ShareUpdateExclusiveLock` does not
conflict with `AccessShareLock`.

#### Worked example: a transaction held open for an hour (idle in transaction)

An open transaction — including a session sitting `idle in transaction` —
blocks CIC only through what it has already done, never through its age:

| What the open transaction did | Blocks CIC at | For how long |
|---|---|---|
| `BEGIN;` only, no statements yet | nothing | — |
| Wrote the **target table** (`INSERT`/`UPDATE`/`DELETE`), now idle | Wait 1 (or 2) | until it commits or rolls back ([mvcc.sgml#ROW-EXCLUSIVE](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L910), [lmgr.c#wait-loop](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L896-L918)) |
| `REPEATABLE READ`/`SERIALIZABLE`, snapshot taken before the reference snapshot, same database | Wait 3 | until transaction end ([snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L336-L356)) |
| `READ COMMITTED`, only reads (or writes to **other** tables), idle, no open cursors | nothing | — ([snapmgr.c#SnapshotResetXmin](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L989-L1028)) |
| `READ COMMITTED` holding an open cursor, or mid-statement on an old snapshot | Wait 3 | until the cursor closes / the snapshot is released |
| Took a conflicting lock on the target table (DDL, `LOCK TABLE`) | initial acquisition | until transaction end |

The writer row is the operationally painful one. A transaction that touched
even one row of the target table holds `RowExclusiveLock` until transaction
end, and `WaitForLockers` sleeps on its virtual transaction ID until the
**whole transaction** finishes — there is no recheck or escape at Waits 1/2
([lmgr.c#wait-loop](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L896-L918)).
Because Wait 1 runs before the build scan even starts, one idle-in-transaction
session that ran a single `UPDATE` an hour ago stalls CIC at its first phase
for the remaining hour.

The reads-only row is the counter-intuitive one: an idle-in-transaction
`READ COMMITTED` session that only ran `SELECT`s does **not** block CIC at
all. Once its statement finished and no registered snapshots remained,
`SnapshotResetXmin` cleared its advertised `xmin`
([snapmgr.c#SnapshotResetXmin](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L989-L1028)),
so Wait 3 skips it — and would drop it mid-wait if it went idle later
([indexcmds.c#idle-recheck](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L331-L338)).
Its leftover `AccessShareLock` matters at no blocking point. Writes to
**other** tables are equally irrelevant at Waits 1/2, because
`WaitForLockers` examines only the target table's lock tag
([indexcmds.c#wait1](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1346)).

A transaction that starts after CIC reaches a wait never extends that wait:
the Waits 1/2 holder list is a point-in-time snapshot
([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L855-L860)),
and a fresh snapshot's `xmin` is newer than `limitXmin`
([procarray.c#limitXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2532-L2533)).

#### Watching the waits

All three waits are visible in `pg_stat_progress_create_index` as the phases
`waiting for writers before build`, `waiting for writers before validation`,
and `waiting for old snapshots`, with `lockers_total`, `lockers_done`, and
`current_locker_pid` identifying who CIC is waiting on
([monitoring.sgml#create-index-phases](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3639-L3708),
[indexcmds.c#wait-progress](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L385-L395),
[lmgr.c#wait-progress](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L887-L916)).

### Is skipping prepared transactions in the writer waits safe?

**Not on its own.** Skipping prepared transactions in Waits 1 and 2 is safe for
the narrow thing the comment claims: a prepared transaction runs no more
statements, so after the wait it cannot start a new write or a new
index-incompatible HOT update
([lmgr.c#prepared-comment](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L890-L894)).
It is **not** sufficient for full index correctness. A transaction that wrote
the table and then ran `PREPARE TRANSACTION` has already made changes that are
not yet committed, are invisible to both concurrent scans, and are never
backfilled — yet they become live later at `COMMIT PREPARED` with no index
entry. The pinned source's own correctness invariant is not satisfied for such
a transaction.

**The invariant the waits are supposed to establish.** The `validate_index`
header states the rule the two writer waits enforce: CIC waits "for all
transactions that could have been modifying the table to terminate" (twice), so
the build and validate scans see a settled writer set, and it then relies on the
claim that "Any tuples committed live after the snap will be inserted into the
index by their originating transaction"
([index.c#validate_index-overview](../../../raw/postgres-12/src/backend/catalog/index.c#L3117-L3144)).
The Wait 1 comment in `DefineIndex` says the same thing concretely: after the
wait, "any updates made by transactions that didn't know about the index are now
committed or rolled back"
([indexcmds.c#wait1-hot-safety](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1348-L1364)).
A prepared transaction is the one writer that has neither "terminated" nor been
"committed or rolled back," and that will not insert its tuples "by their
originating transaction." It falls outside both halves of the invariant.

**Why a prepared transaction is skipped at every wait.** `PREPARE TRANSACTION`
builds a dummy `PGPROC` whose `xid` is still the real, in-progress XID, but whose
`xmin` and `backendId` are invalid
([twophase.c#MarkAsPreparingGuts](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472)).
Two consequences:

- Waits 1 and 2 collect conflicting lock holders with `GetLockConflicts`, which
  reads each holder's virtual transaction ID and drops an invalid VXID — the
  prepared xact's — in both the fast-path and primary-table scans
  ([lock.c#fast-path-skip](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2930-L2936),
  [lock.c#primary-table-skip](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2995-L3001)).
  The prepared xact still holds its `RowExclusiveLock` (transferred to the
  primary lock table at prepare,
  [lock.c#prepared-locks](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876)),
  but it is not waited on.
- Wait 3 (`WaitForOlderSnapshots`) filters on `xmin` and a valid VXID, both of
  which the dummy proc lacks, so it is skipped there too
  ([procarray.c#vxid-check](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2525-L2539))
  — as already noted under Point 4.

**Why the scans never index its tuples.** The prepared xact's `xid` stays in the
proc array as in-progress until `COMMIT PREPARED` removes it
([procarray.c#prepared-in-array](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L15-L18),
[twophase.c#commit-prepared-procarray](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L1514-L1534)).
Both CIC scans use ordinary MVCC snapshots, so they treat the prepared xact as
in-progress and never see the row versions it inserted. The concurrent build
takes the `else` "heap_getnext did the time qual check" branch and never runs
the `SnapshotAny` `INSERT_IN_PROGRESS` logic, the only path that indexes
in-progress tuples
([heapam_handler.c#insert-in-progress](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1429-L1494),
[heapam_handler.c#mvcc-branch](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1595-L1600)).

**Why `COMMIT PREPARED` does not fix it.** `COMMIT PREPARED` records the commit,
marks the XID committed, removes the proc from the proc array, and runs only the
two-phase resource-manager callbacks (which release locks, predicate locks, and
update stats). It does no executor work and inserts into no index
([twophase.c#FinishPreparedTransaction](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L1455-L1534)).
The index entries a transaction makes are created when it runs its DML, for the
indexes that exist then; an index created afterward gets nothing — and a
not-yet-ready index is skipped by writers in any case
([execIndexing.c#skip-not-ready](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332)).

**A concrete sequence (plain `INSERT`).** With `max_prepared_transactions > 0`:

| Step | Action | Effect on the new index |
|---|---|---|
| 1 | Session A: `BEGIN; INSERT INTO t(c) VALUES (1);` | tuple X written, `xmin` = A, in-progress |
| 2 | Session A: `PREPARE TRANSACTION 'p';` | A's lock moves to a dummy proc; X still invisible |
| 3 | Session B: `CREATE INDEX CONCURRENTLY i ON t(c);` Txn 1 | empty `i` committed (`indislive`, not ready, not valid) |
| 4 | Wait 1 | A skipped (invalid VXID); returns at once |
| 5 | Txn 2 build scan (MVCC) | A in-progress -> X invisible -> X not indexed; `indisready` set |
| 6 | Wait 2 | A skipped; returns at once |
| 7 | Txn 3 `validate_index` (reference snapshot) | A in-progress -> X invisible -> X not backfilled |
| 8 | Wait 3 | A skipped (invalid `xmin`); returns at once |
| 9 | Txn 4 | `indisvalid` set — `i` is valid but lacks X |
| 10 | `COMMIT PREPARED 'p';` | X becomes live; no index entry is ever created for it |

After step 10, `SELECT c FROM t WHERE c = 1` answered by `i` returns no row,
while a sequential scan returns X. The same outcome applies to the new live
version produced by an `UPDATE` (HOT or not) that a prepared transaction
performed before the index became ready. A prepared `DELETE` is harmless: it
leaves at most a dead index entry, which is normal and reclaimed by VACUUM. The
gap does not depend on how long the transaction stays prepared — any
`COMMIT PREPARED` after the index is built, of writes the scans could not see,
exposes it.

**Contrast: a non-concurrent `CREATE INDEX` is not exposed this way.** A plain
build takes `ShareLock` on the heap
([utility.c:1320-1321](../../../raw/postgres-12/src/backend/tcop/utility.c#L1320-L1321)),
which conflicts with the prepared xact's `RowExclusiveLock`
([lock.c#ShareLock-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86)),
so it blocks at lock acquisition until `COMMIT PREPARED` / `ROLLBACK PREPARED`,
then scans the resolved heap with `SnapshotAny`. CIC takes only
`ShareUpdateExclusiveLock`, which does not conflict with `RowExclusiveLock`, and
replaces lock-blocking with the VXID waits that skip prepared xacts — which is
exactly where the gap enters.

**Scope of this assessment.** This is a static trace of the pinned 12.2 source
paths (the three waits, the two MVCC scans, and `COMMIT PREPARED`); it was not
reproduced on a running cluster in this environment. The source itself flags the
underlying choice as questionable: `GetLockConflicts` notes that ignoring
prepared transactions "is a bit more debatable but is appropriate for current
uses of the result"
([lock.c#GetLockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2815-L2818)).
Nothing in this checkout's CIC code or `CREATE INDEX` documentation guards
against, or warns about, building an index concurrently while a conflicting
prepared transaction is outstanding.

### Can walsenders or replication-slot xmin holders appear in the Wait 3 set?

**No — not through replication's xmin-holdback machinery.** A standby using
`hot_standby_feedback`, and any physical or logical replication slot, hold back
the cluster's dead-row removal horizon, but they do so in a way
`WaitForOlderSnapshots` never observes. Three independent facts keep these
backends out of the Wait 3 set; the only walsender that can appear is one that
is, at that moment, running an ordinary in-database transaction — already
covered by the lists above.

**1. A replication slot's reserved xmin is a global, not a per-backend xmin.**
Each slot's `effective_xmin` / `effective_catalog_xmin` is aggregated across all
slots by `ReplicationSlotsComputeRequiredXmin` and stored in two *global*
`ProcArrayStruct` fields — `replication_slot_xmin` and
`replication_slot_catalog_xmin` — via `ProcArraySetReplicationSlotXmin`
([slot.c#ReplicationSlotsComputeRequiredXmin](../../../raw/postgres-12/src/backend/replication/slot.c#L701-L742),
[procarray.c#slot-xmin-fields](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L90-L93),
[procarray.c#ProcArraySetReplicationSlotXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2982-L2992)).
`GetCurrentVirtualXIDs` — the function behind Wait 3 — only ever reads each
backend's own `pgxact->xmin` in its proc-array loop and never consults those
slot globals
([procarray.c#GetCurrentVirtualXIDs-xmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520-L2523)).
So a slot can pin the removal horizon arbitrarily far back without ever
contributing a VXID to the Wait 3 set. The slot globals *are* consulted
elsewhere: `GetOldestXmin` folds them into the VACUUM removal horizon
([procarray.c#GetOldestXmin-slots](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1425-L1441)),
and `GetSnapshotData` folds them into `RecentGlobalXmin`
([procarray.c#GetSnapshotData-slots](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1727-L1741)).
Wait 3 is simply not one of those call sites.

**2. A physical walsender is filtered out by database, even when it advertises
its own xmin.** With `hot_standby_feedback` on and no slot, the walsender writes
the standby's reported xmin straight into its own `MyPgXact->xmin` "so that the
xmin will be taken into account by GetOldestXmin"
([walsender.c#hs-feedback-xmin](../../../raw/postgres-12/src/backend/replication/walsender.c#L2026-L2065)).
But a plain (physical) walsender connects to no database: `InitPostgres`
returns early for `am_walsender && !am_db_walsender` without ever setting
`MyDatabaseId`, so its `proc->databaseId` keeps the `InvalidOid` it was given at
`InitProcess`
([postinit.c#walsender-no-db](../../../raw/postgres-12/src/backend/utils/init/postinit.c#L841-L867),
[proc.c#InitProcess-databaseId](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L394-L396)).
`GetCurrentVirtualXIDs`'s same-database test is `proc->databaseId ==
MyDatabaseId`
([procarray.c#GetCurrentVirtualXIDs-db](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520)),
and — unlike `GetOldestXmin` — it has **no** `|| proc->databaseId == 0 /* always
include WalSender */` clause
([procarray.c#GetOldestXmin-walsender-clause](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1348-L1350)).
So the physical walsender is skipped by database. When it uses a slot instead,
it clears `MyPgXact->xmin` to `InvalidTransactionId` and reserves through the
slot, so there is no per-backend xmin to find at all
([walsender.c#slot-clears-xmin](../../../raw/postgres-12/src/backend/replication/walsender.c#L1872-L1909)).

**3. The VXID gate matches xmin's own lifecycle: no transaction, no xmin, no
VXID.** Even a backend that passed the database and xmin filters is recorded
only when `VirtualTransactionIdIsValid(vxid)` holds
([procarray.c#vxid-gate](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2537-L2539)),
which requires a valid `lxid`
([lock.h#VirtualTransactionIdIsValid](../../../raw/postgres-12/src/include/storage/lock.h#L69-L82)).
A backend's `lxid` is assigned in `StartTransaction`
([xact.c#StartTransaction-lxid](../../../raw/postgres-12/src/backend/access/transam/xact.c#L1981-L1994))
and cleared, together with `pgxact->xmin`, in `ProcArrayEndTransaction`
([procarray.c#ProcArrayEndTransaction](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L433-L456)).
Because the local transaction id and the advertised xmin are set and cleared as
a pair, an ordinary backend that is *not* in a transaction has both an invalid
`lxid` and a zero xmin — there is no normal "old xmin but no transaction" state
for the wait to miss. The only two code paths that set `pgxact->xmin` *outside*
a normal transaction are the physical-walsender feedback path (excluded by item
2) and the prepared-transaction dummy proc, which sets `xmin =
InvalidTransactionId` anyway
([twophase.c#MarkAsPreparingGuts](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472)).

**The one walsender that can be in the set is a logical walsender
mid-transaction.** When a logical slot is created with an exported snapshot,
`SnapBuildInitialSnapshot` runs inside a `REPEATABLE READ` transaction (it
asserts `XactIsoLevel == XACT_REPEATABLE_READ`) and sets `MyPgXact->xmin =
snap->xmin`
([snapbuild.c#SnapBuildInitialSnapshot](../../../raw/postgres-12/src/backend/replication/logical/snapbuild.c#L543-L583)).
A logical walsender connects to a real database (`replication=database` sets
`am_db_walsender`,
[postmaster.c#replication-database](../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L2103-L2124)),
so in that window it has a matching `databaseId`, an old xmin, and a valid
`lxid`, and it *is* eligible for Wait 3 — but only because it is then an
ordinary in-database `REPEATABLE READ` snapshot holder, indistinguishable from
the open `REPEATABLE READ` / `SERIALIZABLE` transactions already listed under
Point 4. It is not a "non-transaction backend." Routine logical *streaming*
(after the slot exists) protects catalogs through the slot's `catalog_xmin`
global, not through `MyPgXact->xmin`, so it falls back under item 1.

Net: the `hot_standby_feedback` / replication-slot horizon is real, but it
reaches CIC only through `GetOldestXmin` (what may be vacuumed) and
`GetSnapshotData`'s `RecentGlobalXmin`, never through `WaitForOlderSnapshots`.
Wait 3 waits on local, in-database, snapshot-holding transactions, and the
slot/feedback machinery does not, by itself, create one.

### Why two scans and three waits

The build cannot stop the world, so it has to reason about three groups of
writers: those that ran before the index existed, those that wrote while it was
ready-but-not-valid, and those holding old snapshots. The ordering — visible
empty index, wait, build, wait, validate, wait, mark valid — guarantees:

- After Wait 1, no transaction can do a HOT update that is incompatible with the
  new index, so the first build's scan is safe
  ([index.c#validate_index-overview](../../../raw/postgres-12/src/backend/catalog/index.c#L3118-L3132)).
- After `indisready` and Wait 2, every live writer inserts its own new tuples
  into the index, so `validate_index` only has to backfill tuples that existed
  but were not indexed by the first scan
  ([index.c#validate_index-overview](../../../raw/postgres-12/src/backend/catalog/index.c#L3134-L3152)).
- Wait 3 ensures no surviving transaction has a snapshot old enough to expect a
  row the index intentionally omitted, so it is finally safe to set `indisvalid`
  ([index.c#validate_index-overview](../../../raw/postgres-12/src/backend/catalog/index.c#L3163-L3168)).

For a unique index, the uniqueness constraint is enforced from the moment the
second scan begins, so violations can surface in other sessions before the index
is even usable, and a failed second scan leaves an invalid index that still
enforces uniqueness
([ref/create_index.sgml#unique-caveat](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L598-L606)).

### Failure scenarios and the outcome on the table

Because CIC commits several times, the effect of a failure on the table depends
entirely on **which internal transaction was running when it failed**. Two facts
fix every outcome:

- The catalog row is created in transaction 1 and only becomes durable at the
  first commit. **Any failure before commit 1 leaves no index at all** — the
  transaction simply rolls back
  ([indexcmds.c#commit1](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1318-L1320)).
- Every leftover after that is an **invalid** index (`indisvalid = false`), which
  the planner never uses for queries
  ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210)).
  Whether it is also **ready** (`indisready`) — and therefore whether it costs
  writes and enforces uniqueness — depends only on whether the build set
  `indisready`, which is `index_concurrently_build`'s last action
  ([index.c#build-set-ready](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438)).

#### The three persistent pg_index states

| Leftover state | `indislive` | `indisready` | `indisvalid` | How you reach it | Example (v12 regression suite) |
|---|---|---|---|---|---|
| no index | — | — | — | failure before commit 1 | `concur_index7` — rejected inside a `BEGIN; ... COMMIT;` block, so it never appears in `\d` ([create_index.out:1391-1395](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1391-L1395)) |
| invalid, not ready | `t` | `f` | `f` | failure after commit 1, before `indisready` is set | `concur_index3` — its unique build over duplicate `f2` values failed in the build scan, leaving it `INVALID` ([create_index.out:1383-1385](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1383-L1385), [create_index.out:1415](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1415)) |
| invalid, ready | `t` | `t` | `f` | failure after `indisready` is set, before `indisvalid` | none named in the v12 suite — it needs a duplicate appearing *during* the second scan, which the non-concurrent regression test cannot stage; the docs describe this case ([create_index.sgml:598-606](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L598-L606)) |
| valid (success) | `t` | `t` | `t` | `index_set_state_flags(SET_VALID)` ran | `concur_index1` / `concur_index2` — built concurrently and listed without `INVALID` ([create_index.out:1413-1420](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1413-L1420)) |

The `index_set_state_flags` asserts encode this exact ladder: `SET_READY`
requires live / not-ready / not-valid, and `SET_VALID` requires live / ready /
not-valid
([index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3353-L3366)).

#### The same states under REINDEX INDEX CONCURRENTLY

`REINDEX INDEX CONCURRENTLY` reuses this exact build/validate state machine, but
runs it on a **new copy** (`<original>_ccnew`) it creates next to the original and
then swaps in, leaving an invalid `_ccnew` (before the swap) or `_ccold` (after it)
on failure. The full six-phase walkthrough — including the two extra
`AccessExclusiveLock` "wait for readers" phases, the swap, and the per-phase failure
table — has its own page:
[How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](reindex-index-concurrently.md).

#### What each leftover costs the table

| Leftover | Planner uses it? | INSERT/UPDATE/DELETE | Uniqueness | HOT |
|---|---|---|---|---|
| invalid, **not ready** | no ([plancat.c:206-210](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L206-L210)) | index is still opened and `RowExclusiveLock`-ed on every write ([execIndexing.c#ExecOpenIndices](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L185-L192)), but **no entries are inserted** ([execIndexing.c#skip-not-ready](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332)) | **not** enforced — the unique check is skipped for not-ready indexes ([execIndexing.c#unique-skip](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L537-L539)) | still counted in HOT-safety, so an update touching its columns is forced non-HOT (extra bloat) even though it receives no entries ([relcache.c#omit-not-live](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4388-L4395), [relcache.c#HOT-all-live](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4861-L4870)) |
| invalid, **ready** | no ([plancat.c:206-210](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L206-L210)) | **entries inserted on every write** — the documented "update overhead" ([execIndexing.c#skip-not-ready](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332), [ref/create_index.sgml#invalid-overhead](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L580)) | **enforced** for a unique index, even while invalid ([ref/create_index.sgml#unique-caveat](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L598-L606), [nbtinsert.c#dup-key](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568)) | counted in HOT-safety |

A "dead" index (`indislive = false`) is a different thing: it appears only during
`DROP INDEX CONCURRENTLY`, and `RelationGetIndexList` drops it from every list so
nothing touches it
([relcache.c#omit-not-live](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4388-L4395),
[index.c#INDEX_DROP_SET_DEAD](../../../raw/postgres-12/src/backend/catalog/index.c#L3384-L3396)).
CIC never leaves this state.

#### Failure by phase

| Failure point | Example causes | Leftover on the table |
|---|---|---|
| Preconditions / parse / catalog insert (transaction 1, before commit 1) | runs inside a `BEGIN; ... COMMIT;` block; partitioned, system-catalog, or exclusion-constraint table; index-name collision; `lock_timeout` while acquiring the initial `ShareUpdateExclusiveLock`; any error inside `index_create` | **none** — transaction 1 rolls back |
| Wait 1 or the build scan (transaction 2, before `indisready` is set) | deadlock / cancel (`SIGINT`) / `statement_timeout` in `WaitForLockers`; a **pre-existing duplicate** caught by the unique build sort ([tuplesort.c#dup](../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4048-L4056)); an error evaluating an index expression or predicate; out-of-space during the build | **invalid, not ready** index |
| Wait 2, `validate_index`, or Wait 3 (after `indisready` committed, before `indisvalid`) | deadlock / cancel / timeout in `WaitForLockers` or `WaitForOlderSnapshots`; a **duplicate that appears concurrently** and is hit by the second scan's `index_insert` ([nbtinsert.c#dup-key](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568)); an expression error in the second scan | **invalid, ready** index |
| After `index_set_state_flags(SET_VALID)` | — | none possible: the flag flip is a non-transactional in-place update that cannot roll back, after which the command only sends a relcache invalidation and releases the session lock ([index.c#set-valid](../../../raw/postgres-12/src/backend/catalog/index.c#L3360-L3366), [indexcmds.c#after-valid](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1453-L1472)) |

The not-ready-vs-ready split is exactly the build-scan-vs-validation-scan split,
because `index_concurrently_build` sets `indisready` only after `index_build`
returns ([index.c#build-set-ready](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438));
the second scan in `validate_index` runs in the next transaction, with
`indisready` already true.

#### Worked regression examples

- **Pre-existing duplicate fails the build (first scan).** With two `f2 = 'b'`
  rows present, `CREATE UNIQUE INDEX CONCURRENTLY concur_index3 ON concur_heap(f2)`
  errors `could not create unique index "concur_index3" ... Key (f2)=(b) is
  duplicated`
  ([create_index.out#concur_index3](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1383-L1385)),
  and `\d` later shows the index retained but `INVALID`
  ([create_index.out#invalid-listing](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1413-L1417)).
  The error comes from the build sort
  ([tuplesort.c#dup](../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4048-L4056)),
  i.e. before `indisready` is set, so this leftover is invalid **and not ready**.
- **The leftover survives maintenance.** `VACUUM FULL` keeps the invalid index,
  and `REINDEX TABLE` re-runs the same build and fails identically until the
  duplicate row is deleted
  ([create_index.out#vacuum-reindex](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1400-L1406)).
- **Repairing the leftover.** Once the underlying cause is gone, the invalid
  index can be rebuilt. The non-concurrent path shown in the suite is
  `REINDEX TABLE`: after the duplicate row is deleted, `REINDEX TABLE concur_heap`
  clears `concur_index3`'s `INVALID` marker
  ([create_index.out#reindex-repair](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1422-L1436)).
  Rebuilding it *concurrently* runs through `REINDEX INDEX CONCURRENTLY`, which
  builds a separate `_ccnew` copy and can itself stack another invalid index if
  the cause persists, then makes the index valid once the cause is gone
  ([create_index.out#cic-repair](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2323-L2358)) — see
  [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](reindex-index-concurrently.md).

#### A failed CIC does not leak its session lock

For ordinary ERROR/cancel paths, a failure before commit 1 leaves no index; a
failure after commit 1 and before `SET_VALID` leaves an invalid index. In either
case, the session-level `ShareUpdateExclusiveLock` is released.
`LockRelationIdForSession` is documented to be removed "if an `ereport(ERROR)` occurs"
([lmgr.c#session-lock-doc](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L356-L363)),
because main-transaction **abort** releases session locks too: `ProcReleaseLocks`
calls `LockReleaseAll(DEFAULT_LOCKMETHOD, !isCommit)`, and on abort `!isCommit` is
true
([proc.c#ProcReleaseLocks](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L772-L798)).
So a failed CIC never leaves a lock that blocks `VACUUM`, `ANALYZE`, DDL, or
another CIC on the table.

#### Server crash or immediate shutdown

A crash or `immediate`-mode shutdown leaves **the same four states as an ERROR
or cancel — never a half-applied flag flip, and never a valid index that is
missing rows.** An unclean stop leaves `pg_control` in a state other than
`DB_SHUTDOWNED`, so the next startup forces `InRecovery` and replays WAL
automatically
([xlog.c#crash-recovery](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6740-L6766)).
Recovery is pure physical WAL replay: the recovered `pg_index` flags are
whatever that replay produces, and v12 runs no CIC-aware repair pass — which is
why the documented fix for an interrupted build is still a manual `DROP INDEX` /
`REINDEX`
([ref/create_index.sgml#invalid-index](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596)).

**Why a flip's durability is decoupled from its phase commit.**
`index_set_state_flags` runs with no assigned transaction id — it asserts
`GetTopTransactionIdIfAny() == InvalidTransactionId` — and writes the flag
through `heap_inplace_update`
([index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)).
`heap_inplace_update` overwrites the tuple in a critical section, emits one
`XLOG_HEAP_INPLACE` record, sets the page LSN, and returns **without** flushing
WAL
([heapam.c#heap_inplace_update](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774)).
Because the phase transaction holds no XID, `RecordTransactionCommit` writes no
commit record and takes the **asynchronous** branch — `XLogSetAsyncXactLSN`, not
`XLogFlush` — even under `synchronous_commit = on`; its own comment notes that
losing such a WAL-writing-but-xidless transaction on crash "will be irrelevant"
([xact.c#RecordTransactionCommit](../../../raw/postgres-12/src/backend/access/transam/xact.c#L1232-L1392)).
`XLogSetAsyncXactLSN` only nudges the WAL writer
([xlog.c#XLogSetAsyncXactLSN](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2630-L2670)).
So a flip turns durable only once the WAL writer, a checkpoint, or a later
synchronous commit flushes WAL past its LSN — possibly well after
`CommitTransactionCommand` has already returned.

This cuts both ways, but safely:

- A flip can **survive even though its phase wrote no commit record**:
  `heap_xlog_inplace` redoes the byte overwrite physically and unconditionally
  whenever its record is durable
  ([heapam.c#heap_xlog_inplace](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835)).
  ("Will not roll back on error" concerns the ERROR path, where the surrounding
  transaction aborts but the in-place row stays; a crash only asks whether the
  record was durable.)
- A flip can be **lost even though the phase commit returned**, because that
  commit was asynchronous; recovery then shows the prior, more-conservative
  flag.

**The recovered state is monotone.** SET_READY (Txn 2) has a lower LSN than
SET_VALID (Txn 4)
([index.c#build-set-ready](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438),
[indexcmds.c#set-valid](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1448-L1463)),
and `XLogFlush` flushes all WAL through a position
([xlog.c#XLogFlush](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2791-L2798)).
Durable SET_VALID therefore implies durable SET_READY, so recovered
`(indisready, indisvalid)` is always `(f,f)`, `(t,f)`, or `(t,t)`, never
`(f,t)`:

| Last record durable at crash | Recovered `pg_index` | Failure-table row |
|---|---|---|
| commit-1 catalog row not yet durable | (no row) | no index |
| catalog row only | `indislive=t, indisready=f, indisvalid=f` | invalid, not ready |
| + SET_READY (`XLOG_HEAP_INPLACE`) | `indisready=t, indisvalid=f` | invalid, ready |
| + SET_VALID (`XLOG_HEAP_INPLACE`) | `indisready=t, indisvalid=t` | valid |

**A recovered valid index is always complete.** The B-tree build writes its
pages outside shared buffers and `smgrimmedsync`s the index file before the
build transaction may commit, and WAL-logs the built pages only when
`wal_level >= replica`; either way the index data is durable no later than the
SET_READY flip
([nbtsort.c#build-durability](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L34-L44),
[nbtsort.c#use-wal](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L580),
[nbtsort.c#immedsync](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1288-L1307));
a permanent index always needs WAL
([rel.h#RelationNeedsWAL](../../../raw/postgres-12/src/include/utils/rel.h#L519-L520)).
The `validate_index` backfill inserts go through shared buffers and are
WAL-logged ahead of SET_VALID, and the WAL-before-data rule forces `XLogFlush`
up to a buffer's LSN before that buffer can reach disk
([bufmgr.c#FlushBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2712-L2736)).
So durable SET_VALID implies the build and the backfill are durable too; no
crash window can expose a valid index that is missing rows.

#### Recovery

The documented fix is to `DROP INDEX` (optionally `CONCURRENTLY`) the invalid
index and retry, or rebuild it with `REINDEX INDEX CONCURRENTLY`
([ref/create_index.sgml#invalid-index](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596)).
A `DROP INDEX CONCURRENTLY` that itself fails partway can simply be re-run: its
`INDEX_DROP_CLEAR_VALID` step deliberately does not assert its starting flags, so
the drop is retryable
([index.c#drop-clear-valid](../../../raw/postgres-12/src/backend/catalog/index.c#L3367-L3383)).

### Test coverage

- Functional coverage is in the regression suite: `create_index.sql` builds
  empty-table, unique, expression, and partial indexes concurrently, defaults
  the index name, and asserts that CIC fails inside a `BEGIN; ... COMMIT;` block
  ([create_index.sql#cic-block](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L467-L520)).
- The concurrency itself is exercised by an isolation test that runs two CIC
  operations simultaneously and uses advisory locks to interleave them
  ([multiple-cic.spec](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40)).
- Failure outcomes are exercised too: `create_index.sql` builds a unique index
  concurrently over duplicate rows and checks that the failed build is left
  `INVALID`, survives `VACUUM FULL`, and is repaired only after the duplicate is
  removed
  ([create_index.out#concurrent-invalid](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1382-L1436)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v12/index.md`, and the last ~20
  `wiki/log.md` entries (navigation only).
- Pinned checkout `raw/postgres-12/` at commit
  `45b88269a353ad93744772791feb6d01bc7e1e42` ("Stamp 12.2.").
- `DefineIndex` concurrent branch and `WaitForOlderSnapshots` in
  `src/backend/commands/indexcmds.c`.
- `index_create`/`UpdateIndexRelation`, `index_concurrently_build`,
  `validate_index`, and `index_set_state_flags` in
  `src/backend/catalog/index.c`.
- `WaitForLockers`/`WaitForLockersMultiple` in
  `src/backend/storage/lmgr/lmgr.c`; the `LockConflicts` table and self-conflict
  note in `src/backend/storage/lmgr/lock.c`; lock-mode definitions in
  `src/include/storage/lockdefs.h`.
- `PreventInTransactionBlock` dispatch in `src/backend/tcop/utility.c`.
- `doc/src/sgml/ref/create_index.sgml` (CONCURRENTLY narrative).
- For the blocking-operations section: `GetLockConflicts` in
  `src/backend/storage/lmgr/lock.c`; `GetCurrentVirtualXIDs` in
  `src/backend/storage/ipc/procarray.c`; the autovacuum-cancel branch of
  `ProcSleep` in `src/backend/storage/lmgr/proc.c`; `vacuumFlags` in
  `src/include/storage/proc.h`; `MarkAsPreparingGuts` in
  `src/backend/access/transam/twophase.c`; `GetTransactionSnapshot` in
  `src/backend/utils/time/snapmgr.c`; the lock-mode/command table in
  `doc/src/sgml/mvcc.sgml`; the `pg_stat_progress_create_index` phase table in
  `doc/src/sgml/monitoring.sgml`.
- For the worked examples: the dump-transaction setup, timeout disabling, and
  `LOCK TABLE ... IN ACCESS SHARE MODE` statements in
  `src/bin/pg_dump/pg_dump.c`; `SnapshotResetXmin` in
  `src/backend/utils/time/snapmgr.c`.
- For the failure-scenarios section: the planner's invalid-index skip in
  `src/backend/optimizer/util/plancat.c`; `ExecOpenIndices` /
  `ExecInsertIndexTuples` ready-for-inserts handling in
  `src/backend/executor/execIndexing.c`; `RelationGetIndexList` index-list
  filtering and HOT-attribute collection in
  `src/backend/utils/cache/relcache.c`; the unique-violation `ereport`s in
  `src/backend/utils/sort/tuplesort.c` (build sort) and
  `src/backend/access/nbtree/nbtinsert.c` (`_bt_check_unique`); `index_drop` and
  the `INDEX_DROP_*` branches of `index_set_state_flags` in
  `src/backend/catalog/index.c`;
  `heap_inplace_update` in `src/backend/access/heap/heapam.c`;
  `LockRelationIdForSession` in
  `src/backend/storage/lmgr/lmgr.c` and `ProcReleaseLocks` in
  `src/backend/storage/lmgr/proc.c`; and the invalid-index regression evidence in
  `src/test/regress/expected/create_index.out`.
- For the crash / immediate-shutdown recovery trace: `heap_inplace_update` and
  its redo `heap_xlog_inplace` (`XLOG_HEAP_INPLACE`) in
  `src/backend/access/heap/heapam.c`; the xidless-transaction asynchronous-commit
  path of `RecordTransactionCommit` in `src/backend/access/transam/xact.c`;
  `XLogFlush`, `XLogSetAsyncXactLSN`, and the `StartupXLOG` crash-recovery
  determination in `src/backend/access/transam/xlog.c`; the WAL-before-data rule
  in `FlushBuffer` in `src/backend/storage/buffer/bufmgr.c`; the B-tree build's
  WAL gating and final `smgrimmedsync` in
  `src/backend/access/nbtree/nbtsort.c`; the `index_concurrently_build`
  SET_READY and `DefineIndex` SET_VALID commit boundaries in
  `src/backend/catalog/index.c` and `src/backend/commands/indexcmds.c`; and the
  `RelationNeedsWAL` macro in `src/include/utils/rel.h`.
- Tests: `src/test/regress/sql/create_index.sql`,
  `src/test/isolation/specs/multiple-cic.spec`, and
  `src/test/isolation/expected/multiple-cic.out`.
- For the inter-builder follow-up: same-checkout source history for commits
  `c3d09b3bd23`, `54eff5311d7`, and `1dec82068b3`; the
  `ReindexRelationConcurrently` adjacent caller in
  `src/backend/commands/indexcmds.c`; and the `GetCurrentVirtualXIDs` filter
  contract in `src/backend/storage/ipc/procarray.c`.
- For the first-build-scan tuple-visibility trace: `index_concurrently_build`
  and `index_build` in `src/backend/catalog/index.c`; the `ambuild` heap-scan
  callers `nbtsort.c`, `hash.c`, `gistbuild.c`, `spginsert.c`, `gininsert.c`,
  and `brin.c` under `src/backend/access/`; the `table_index_build_scan`
  inline wrapper and `index_build_range_scan` callback in
  `src/include/access/tableam.h`; `heapam_index_build_range_scan` and the
  `TableAmRoutine` registration in
  `src/backend/access/heap/heapam_handler.c`; `heapgetpage` in
  `src/backend/access/heap/heapam.c`; `HeapTupleSatisfiesVisibility` /
  `HeapTupleSatisfiesMVCC` in `src/backend/access/heap/heapam_visibility.c`;
  the build-snapshot setup in `DefineIndex` in
  `src/backend/commands/indexcmds.c`; and the `validate_index` header comment
  in `src/backend/catalog/index.c`.
- For the prepared-transaction writer-wait safety assessment: the
  `validate_index` correctness narrative and the Wait 1 HOT-safety comment in
  `src/backend/catalog/index.c` and `src/backend/commands/indexcmds.c`;
  `GetLockConflicts`'s invalid-VXID skip (fast-path and primary-table loops) in
  `src/backend/storage/lmgr/lock.c`; `MarkAsPreparingGuts` and
  `FinishPreparedTransaction` in `src/backend/access/transam/twophase.c`; the
  prepared-xacts-in-the-proc-array note in
  `src/backend/storage/ipc/procarray.c`; the `SnapshotAny`
  `INSERT_IN_PROGRESS` branch versus the MVCC `else` branch in
  `heapam_index_build_range_scan` in
  `src/backend/access/heap/heapam_handler.c`; and the not-ready-index insert
  skip in `src/backend/executor/execIndexing.c`.
- For the walsender / replication-slot Wait 3 assessment: `GetCurrentVirtualXIDs`
  versus `GetOldestXmin` versus `GetSnapshotData`, the
  `replication_slot_xmin`/`replication_slot_catalog_xmin` global fields, and
  `ProcArrayEndTransaction`/`ProcArraySetReplicationSlotXmin` in
  `src/backend/storage/ipc/procarray.c`; `ReplicationSlotsComputeRequiredXmin`
  in `src/backend/replication/slot.c`; `ProcessStandbyHSFeedbackMessage` and
  `PhysicalReplicationSlotNewXmin` in `src/backend/replication/walsender.c`; the
  physical-walsender no-database early return in `InitPostgres` in
  `src/backend/utils/init/postinit.c` and the `InitProcess` `databaseId`
  initialization in `src/backend/storage/lmgr/proc.c`; the
  `VirtualTransactionIdIsValid`/`GET_VXID_FROM_PGPROC` macros in
  `src/include/storage/lock.h`; `StartTransaction`'s `lxid` assignment in
  `src/backend/access/transam/xact.c`; `SnapBuildInitialSnapshot` in
  `src/backend/replication/logical/snapbuild.c`; and the `replication=database`
  startup-packet parsing in `src/backend/postmaster/postmaster.c`.

## Evidence Map

| Claim | Source |
|---|---|
| Table lock is `ShareUpdateExclusiveLock` for concurrent, `ShareLock` otherwise | [indexcmds.c:563-564](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L563-L564), [utility.c:1320-1321](../../../raw/postgres-12/src/backend/tcop/utility.c#L1320-L1321) |
| CIC cannot run in a transaction block | [utility.c:1307-1309](../../../raw/postgres-12/src/backend/tcop/utility.c#L1307-L1309) |
| Temp tables fall back to non-concurrent | [indexcmds.c:489-499](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499) |
| Partitioned / system-catalog / exclusion restrictions | [indexcmds.c:604-616](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L604-L616), [index.c:813-817](../../../raw/postgres-12/src/backend/catalog/index.c#L813-L817), [index.c:823-826](../../../raw/postgres-12/src/backend/catalog/index.c#L823-L826) |
| Catalog row created not-ready/not-valid, `indislive` true | [index.c:612-615](../../../raw/postgres-12/src/backend/catalog/index.c#L612-L615), [index.c:990-996](../../../raw/postgres-12/src/backend/catalog/index.c#L990-L996) |
| Session lock taken before first commit; released at end | [indexcmds.c:1307-1320](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1320), [indexcmds.c:1465-1468](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1465-L1468) |
| Wait 1 / build / set indisready (txn 2) | [indexcmds.c:1328-1379](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1328-L1379), [index.c:1399-1439](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439) |
| Wait 2 / reference snapshot / validate (txn 3) | [indexcmds.c:1382-1424](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1382-L1424), [index.c:3176-3298](../../../raw/postgres-12/src/backend/catalog/index.c#L3176-L3298) |
| Wait 3 / set indisvalid / relcache inval (txn 4) | [indexcmds.c:1437-1472](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1437-L1472) |
| `index_set_state_flags` is non-transactional in-place | [index.c:3331-3403](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403) |
| `WaitForLockers` waits on VXIDs, takes no lock; `ShareLock` conflicts with `RowExclusiveLock` | [lmgr.c:850-949](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949), [lock.c:83-86](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86) |
| `ShareUpdateExclusiveLock` conflict set and self-conflict | [lock.c:78-81](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81), [lock.c:194-196](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L194-L196) |
| Wait 3 excludes autovacuum and lazy VACUUM only | [indexcmds.c:339-402](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402) |
| Same-table concurrent index builds serialize on self-conflicting `ShareUpdateExclusiveLock` | [indexcmds.c:563-564](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L563-L564), [indexcmds.c:1307-1316](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1316), [lock.c:78-81](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81), [ref/create_index.sgml:614-621](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L614-L621) |
| Different-table concurrent builders can meet in the same-database old-snapshot wait, while Waits 1/2 remain heap-lock-tag-local | [indexcmds.c:1328-1346](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1328-L1346), [indexcmds.c:1381-1389](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1381-L1389), [indexcmds.c:339-348](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L348), [procarray.c:2471-2540](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2471-L2540) |
| CIC clears its own advertised xmin before Wait 3 to avoid inter-CIC deadlock | [indexcmds.c:1414-1448](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1414-L1448) |
| `REINDEX CONCURRENTLY` shares the same concurrent-build snapshot-wait boundary | [indexcmds.c:2957-3077](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2957-L3077), [indexcmds.c:3080-3198](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3080-L3198) |
| `multiple-cic` tests two simultaneous CIC commands on different tables completing | [multiple-cic.spec:1-40](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40), [multiple-cic.out:3-19](../../../raw/postgres-12/src/test/isolation/expected/multiple-cic.out#L3-L19) |
| Correctness narrative (two scans, three waits) | [index.c:3112-3174](../../../raw/postgres-12/src/backend/catalog/index.c#L3112-L3174), [ref/create_index.sgml:545-572](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L572) |
| First build scan sets `ii_Concurrent` then builds via `index_build` -> `ambuild` -> `table_index_build_scan` (all core AMs) | [index.c:1421-1427](../../../raw/postgres-12/src/backend/catalog/index.c#L1421-L1427), [index.c:2902-2903](../../../raw/postgres-12/src/backend/catalog/index.c#L2902-L2903), [nbtsort.c:489-491](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L489-L491), [hash.c:166](../../../raw/postgres-12/src/backend/access/hash/hash.c#L166), [gistbuild.c:196](../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L196), [spginsert.c:126](../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L126), [gininsert.c:382](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L382), [brin.c:723](../../../raw/postgres-12/src/backend/access/brin/brin.c#L723) |
| `table_index_build_scan` passes `anyvisible = false` and dispatches to `heapam_index_build_range_scan` | [tableam.h:1512-1533](../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533), [heapam_handler.c:2644](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2644) |
| Concurrent build keeps `OldestXmin` invalid and uses an MVCC snapshot; non-concurrent uses `SnapshotAny` + `GetOldestXmin` | [heapam_handler.c:1212-1223](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1212-L1223), [heapam_handler.c:1233-1246](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1233-L1246), [indexcmds.c:1358-1370](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1358-L1370) |
| Heap scan applies the MVCC visibility test itself (`heapgetpage` -> `HeapTupleSatisfiesVisibility` -> `HeapTupleSatisfiesMVCC`); build loop trusts it via the `else` branch | [heapam.c:444-453](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L444-L453), [heapam_visibility.c:1690-1696](../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1690-L1696), [heapam_handler.c:1595-1615](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1595-L1615) |
| Concurrent scan skips the `SnapshotAny` `HeapTupleSatisfiesVacuum` switch, so it never indexes `RECENTLY_DEAD` tuples nor sets `ii_BrokenHotChain`; `index_build` skips `indcheckxmin` for concurrent | [heapam_handler.c:1364-1388](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1364-L1388), [heapam_handler.c:1402-1428](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1402-L1428), [index.c:2939-2952](../../../raw/postgres-12/src/backend/catalog/index.c#L2939-L2952) |
| MVCC (not `HeapTupleSatisfiesVacuum`) avoids bogus unique failures; omitted tuples are backfilled by the second `validate_index` scan | [index.c:3124-3132](../../../raw/postgres-12/src/backend/catalog/index.c#L3124-L3132), [index.c:3134-3168](../../../raw/postgres-12/src/backend/catalog/index.c#L3134-L3168) |
| Invalid index left on failure; unique-constraint caveat | [ref/create_index.sgml:574-606](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L606) |
| Failure before commit 1 leaves no index (commit-1 boundary) | [indexcmds.c:1318-1320](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1318-L1320) |
| Planner ignores `indisvalid = false` indexes (executor still inserts if `indisready`) | [plancat.c:200-210](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210) |
| Not-ready index is opened + `RowExclusiveLock`ed but receives no entries, and its unique check is skipped | [execIndexing.c:185-192](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L185-L192), [execIndexing.c:330-332](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332), [execIndexing.c:537-539](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L537-L539) |
| Every `indislive` index counts for HOT-safety; not-live indexes are omitted from the index list | [relcache.c:4388-4395](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4388-L4395), [relcache.c:4861-4870](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4861-L4870) |
| Build sets `indisready` as its last action (build-scan vs validate-scan split); state-ladder asserts | [index.c:1426-1438](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438), [index.c:3353-3396](../../../raw/postgres-12/src/backend/catalog/index.c#L3353-L3396) |
| Build-scan dup is "could not create unique index ... is duplicated"; concurrent second-scan dup is "duplicate key ... already exists" | [tuplesort.c:4048-4056](../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4048-L4056), [nbtinsert.c:563-568](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568) |
| Regression: failed unique build left INVALID, retained through `VACUUM FULL`, then made valid by non-concurrent `REINDEX TABLE` once the duplicate is deleted | [create_index.out:1383-1417](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1383-L1417), [create_index.out:1400-1406](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1400-L1406), [create_index.out:1422-1436](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1422-L1436) |
| Named example index per `pg_index` state: `concur_index7` (none), `concur_index3` (invalid, not ready), `concur_index1`/`concur_index2` (valid) | [create_index.out:1391-1395](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1391-L1395), [create_index.out:1413-1420](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1413-L1420) |
| A failed CIC releases its session lock (removed on `ereport(ERROR)`; abort releases session locks) | [lmgr.c:356-363](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L356-L363), [proc.c:772-798](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L772-L798) |
| `DROP INDEX CONCURRENTLY` is retryable: `INDEX_DROP_CLEAR_VALID` does not assert its starting flags | [index.c:3367-3383](../../../raw/postgres-12/src/backend/catalog/index.c#L3367-L3383) |
| Crash / `immediate` shutdown triggers automatic WAL-replay recovery; recovered `pg_index` flags are whatever physical replay produces (no CIC repair pass), so the documented fix stays manual `DROP`/`REINDEX` | [xlog.c:6740-6766](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6740-L6766), [ref/create_index.sgml:574-596](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596) |
| Flag flips run with no XID and are written by `heap_inplace_update` (`XLOG_HEAP_INPLACE`, page LSN set, no `XLogFlush`) | [index.c:3331-3403](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403), [heapam.c:5746-5774](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774) |
| An xidless WAL-writing transaction commits asynchronously (`XLogSetAsyncXactLSN`, not `XLogFlush`) regardless of `synchronous_commit`, so a flip's durability is decoupled from its phase commit | [xact.c:1232-1392](../../../raw/postgres-12/src/backend/access/transam/xact.c#L1232-L1392), [xlog.c:2630-2670](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2630-L2670) |
| `heap_xlog_inplace` redoes the flip physically and unconditionally whenever its record is durable, so a flip can survive a phase that wrote no commit record | [heapam.c:8797-8835](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835) |
| `XLogFlush` flushes all WAL through a position, so durable SET_VALID implies durable SET_READY; recovered `(indisready, indisvalid)` is monotone, never `(f,t)` | [xlog.c:2791-2798](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2791-L2798), [index.c:1426-1438](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438), [indexcmds.c:1448-1463](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1448-L1463) |
| A recovered valid index is complete: the build `smgrimmedsync`s its file before commit and WAL-logs pages when `wal_level >= replica`, and validate-scan WAL precedes SET_VALID under the WAL-before-data rule | [nbtsort.c:1288-1307](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1288-L1307), [nbtsort.c:580](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L580), [rel.h:519-520](../../../raw/postgres-12/src/include/utils/rel.h#L519-L520), [bufmgr.c:2712-2736](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2712-L2736) |
| Tests | [create_index.sql:467-520](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L467-L520), [multiple-cic.spec:1-40](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40) |
| Initial lock acquisition point and its conflict set | [utility.c:1311-1326](../../../raw/postgres-12/src/backend/tcop/utility.c#L1311-L1326), [lock.c:78-81](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81) |
| Which v12 commands take each conflicting lock mode | [mvcc.sgml:890-1030](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L1030) |
| Autovacuum blocking CIC is sent SIGINT after `deadlock_timeout`, except anti-wraparound | [proc.c:1308-1375](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375), [proc.c:1319-1324](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1319-L1324) |
| Prepared transactions hold table locks until COMMIT/ROLLBACK PREPARED but are ignored by Waits 1/2 | [lock.c:2873-2876](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876), [lock.c:2815-2818](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2815-L2818), [lmgr.c:890-894](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L890-L894) |
| Waits 1/2 sleep until each holder's whole transaction ends; lock waiters and later writers are skipped | [lmgr.c:896-918](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L896-L918), [lock.c:2804-2807](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2804-L2807), [lmgr.c:855-860](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L855-L860) |
| Wait 3 filters: same database, xmin ≤ limitXmin, xmin=0 skipped, vacuum flags excluded | [indexcmds.c:346-348](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L346-L348), [procarray.c:2508-2541](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2508-L2541), [proc.h:53-56](../../../raw/postgres-12/src/include/storage/proc.h#L53-L56) |
| REPEATABLE READ / SERIALIZABLE first snapshot lives until transaction end | [snapmgr.c:336-356](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L336-L356) |
| Prepared transactions invisible to Wait 3 (invalid xmin and backendId) | [twophase.c:465-472](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472), [procarray.c:2525-2539](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2525-L2539) |
| The three waits are visible in `pg_stat_progress_create_index` | [monitoring.sgml:3639-3708](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3639-L3708), [indexcmds.c:385-395](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L385-L395) |
| `pg_dump` runs one `REPEATABLE READ, READ ONLY` (or `SERIALIZABLE, READ ONLY, DEFERRABLE`) transaction, disables the three timeouts, and takes only `ACCESS SHARE` table locks | [pg_dump.c:1166-1194](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1166-L1194), [pg_dump.c:1140-1147](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147), [pg_dump.c:6646-6671](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6646-L6671) |
| An idle backend with no active or registered snapshots clears its advertised xmin | [snapmgr.c:989-1028](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L989-L1028) |
| Skipping prepared xacts in the writer waits is not sufficient for index correctness: it violates the stated "wait for all modifying transactions to terminate" / "inserted by their originating transaction" invariant | [index.c:3117-3144](../../../raw/postgres-12/src/backend/catalog/index.c#L3117-L3144), [indexcmds.c:1348-1364](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1348-L1364) |
| Prepared xacts are skipped at the writer waits because their dummy proc has an invalid VXID, dropped by `GetLockConflicts` | [twophase.c:465-472](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472), [lock.c:2930-2936](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2930-L2936), [lock.c:2995-3001](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2995-L3001) |
| A prepared xact's tuples stay in-progress (invisible to the MVCC build/validate scans) until `COMMIT PREPARED`; the concurrent build never runs the `SnapshotAny` `INSERT_IN_PROGRESS` path that would index them | [procarray.c:15-18](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L15-L18), [twophase.c:1514-1534](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L1514-L1534), [heapam_handler.c:1429-1494](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1429-L1494), [heapam_handler.c:1595-1600](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1595-L1600) |
| `COMMIT PREPARED` records the commit and releases locks but performs no executor work and inserts into no index | [twophase.c:1455-1534](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L1455-L1534) |
| A non-concurrent `CREATE INDEX` instead blocks at lock acquisition on a prepared xact's `RowExclusiveLock`, because its `ShareLock` conflicts with it | [utility.c:1320-1321](../../../raw/postgres-12/src/backend/tcop/utility.c#L1320-L1321), [lock.c:83-86](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86) |
| A replication slot's reserved xmin is a global `procArray->replication_slot_xmin`/`replication_slot_catalog_xmin` (set by `ProcArraySetReplicationSlotXmin`), which `GetCurrentVirtualXIDs` never reads — so a slot never puts a VXID in the Wait 3 set | [procarray.c:90-93](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L90-L93), [procarray.c:2982-2992](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2982-L2992), [slot.c:701-742](../../../raw/postgres-12/src/backend/replication/slot.c#L701-L742), [procarray.c:2520-2523](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520-L2523) |
| The slot xmin globals are consulted by `GetOldestXmin` and `GetSnapshotData` (`RecentGlobalXmin`), not by `WaitForOlderSnapshots` | [procarray.c:1425-1441](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1425-L1441), [procarray.c:1727-1741](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1727-L1741) |
| A physical walsender sets its own `MyPgXact->xmin` from `hot_standby_feedback` (no slot) or clears it and reserves via the slot | [walsender.c:2026-2065](../../../raw/postgres-12/src/backend/replication/walsender.c#L2026-L2065), [walsender.c:1872-1909](../../../raw/postgres-12/src/backend/replication/walsender.c#L1872-L1909) |
| A physical walsender connects to no database (`proc->databaseId` stays `InvalidOid`); `GetCurrentVirtualXIDs`'s same-db test lacks `GetOldestXmin`'s "always include WalSender" clause, so it is filtered by database | [postinit.c:841-867](../../../raw/postgres-12/src/backend/utils/init/postinit.c#L841-L867), [proc.c:394-396](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L394-L396), [procarray.c:2520](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520), [procarray.c:1348-1350](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1348-L1350) |
| Wait 3 records a backend only with a valid VXID; `lxid` and `xmin` are set in `StartTransaction` and cleared together in `ProcArrayEndTransaction`, so an idle (non-transaction) backend has neither | [procarray.c:2537-2539](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2537-L2539), [lock.h:69-82](../../../raw/postgres-12/src/include/storage/lock.h#L69-L82), [xact.c:1981-1994](../../../raw/postgres-12/src/backend/access/transam/xact.c#L1981-L1994), [procarray.c:433-456](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L433-L456) |
| A logical walsender sets `MyPgXact->xmin` only inside the `REPEATABLE READ` initial-snapshot transaction (slot creation/export) and connects to a real database; routine streaming uses the slot's catalog_xmin global | [snapbuild.c:543-583](../../../raw/postgres-12/src/backend/replication/logical/snapbuild.c#L543-L583), [postmaster.c:2103-2124](../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L2103-L2124) |

## Open Questions

None for source behavior at the pinned PostgreSQL 12 commit. The former
crash / immediate-shutdown question is resolved inline under
[Server crash or immediate shutdown](#server-crash-or-immediate-shutdown): the
flag flips are xidless `heap_inplace_update` writes that commit asynchronously,
`heap_xlog_inplace` redoes them physically on recovery, and the recovered state
is monotone, so a crash always lands on one of the four documented leftovers and
never on a valid-but-incomplete index. This conclusion is a static source trace
of the pinned 12.2 checkout, not a live crash reproduction in this environment.

## Source References

- [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L1473)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)
- [index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439)
- [index.c#index_build](../../../raw/postgres-12/src/backend/catalog/index.c#L2824-L2952)
- [tableam.h#table_index_build_scan](../../../raw/postgres-12/src/include/access/tableam.h#L1485-L1533)
- [heapam_handler.c#heapam_index_build_range_scan](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1150-L1703)
- [heapam.c#heapgetpage](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L461)
- [heapam_visibility.c#HeapTupleSatisfiesVisibility](../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1679-L1718)
- [index.c#validate_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3112-L3298)
- [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)
- [lmgr.c#WaitForLockers](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)
- [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)
- [lock.c#GetLockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2804-L2821)
- [lockdefs.h#lockmodes](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46)
- [utility.c#CIC-dispatch](../../../raw/postgres-12/src/backend/tcop/utility.c#L1305-L1321)
- [procarray.c#GetCurrentVirtualXIDs](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2467-L2548)
- [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2739-L3342)
- [proc.c#ProcSleep-autovacuum-cancel](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)
- [proc.h#vacuumFlags](../../../raw/postgres-12/src/include/storage/proc.h#L53-L63)
- [twophase.c#MarkAsPreparingGuts](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L446-L490)
- [twophase.c#FinishPreparedTransaction](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L1455-L1618)
- [procarray.c#prepared-xacts-in-array](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L13-L18)
- [snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L305-L373)
- [snapmgr.c#SnapshotResetXmin](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L989-L1028)
- [pg_dump.c#dump-transaction](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1194)
- [pg_dump.c#lock-table](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6646-L6671)
- [mvcc.sgml#table-level-locks](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L1039)
- [monitoring.sgml#create-index-phases](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3630-L3709)
- [ref/create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L631)
- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210)
- [execIndexing.c#ready-for-inserts](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L185-L539)
- [relcache.c#RelationGetIndexList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4348-L4435)
- [index.c#index_drop](../../../raw/postgres-12/src/backend/catalog/index.c#L2007-L2166)
- [heapam.c#heap_inplace_update](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5692-L5774)
- [heapam.c#heap_xlog_inplace](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835)
- [xact.c#RecordTransactionCommit](../../../raw/postgres-12/src/backend/access/transam/xact.c#L1206-L1401)
- [xlog.c#XLogFlush](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2791-L2798)
- [xlog.c#XLogSetAsyncXactLSN](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2630-L2670)
- [xlog.c#StartupXLOG-crash-recovery](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6740-L6766)
- [bufmgr.c#FlushBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2712-L2736)
- [nbtsort.c#build-durability](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1288-L1307)
- [rel.h#RelationNeedsWAL](../../../raw/postgres-12/src/include/utils/rel.h#L519-L520)
- [tuplesort.c#unique-violation](../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4040-L4056)
- [nbtinsert.c#_bt_check_unique](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568)
- [lmgr.c#LockRelationIdForSession](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L356-L383)
- [proc.c#ProcReleaseLocks](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L772-L798)
- [create_index.out#concurrent-invalid](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1382-L1436)
- [multiple-cic.spec](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40)
- [multiple-cic.out](../../../raw/postgres-12/src/test/isolation/expected/multiple-cic.out#L3-L19)
- [procarray.c#GetOldestXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1306-L1443)
- [procarray.c#GetSnapshotData-slots](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1700-L1743)
- [procarray.c#ProcArraySetReplicationSlotXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2976-L2993)
- [procarray.c#ProcArrayEndTransaction](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L433-L464)
- [slot.c#ReplicationSlotsComputeRequiredXmin](../../../raw/postgres-12/src/backend/replication/slot.c#L695-L742)
- [walsender.c#ProcessStandbyHSFeedbackMessage](../../../raw/postgres-12/src/backend/replication/walsender.c#L1872-L2066)
- [postinit.c#InitPostgres-walsender](../../../raw/postgres-12/src/backend/utils/init/postinit.c#L841-L867)
- [proc.c#InitProcess](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L386-L398)
- [lock.h#VirtualTransactionId](../../../raw/postgres-12/src/include/storage/lock.h#L55-L82)
- [xact.c#StartTransaction](../../../raw/postgres-12/src/backend/access/transam/xact.c#L1981-L1996)
- [snapbuild.c#SnapBuildInitialSnapshot](../../../raw/postgres-12/src/backend/replication/logical/snapbuild.c#L543-L583)
- [postmaster.c#ProcessStartupPacket-replication](../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L2103-L2124)

## Navigation

- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](reindex-index-concurrently.md)
- [v12 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
