---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)

## Question

give a comprehensive explanation of how create index concurrently is
implemented, add a section with all steps and locks required on the table.

## Answer

`CREATE INDEX CONCURRENTLY` (CIC) builds an index without ever taking a lock
that blocks `INSERT`/`UPDATE`/`DELETE` on the table. It pays for that with more
total work: it splits the build across **four internal transactions**, does
**two full table scans**, and **waits out other transactions three times**. The
whole dance exists to keep the index correct under concurrent writes, since the
build cannot freeze the table.

All of the orchestration lives in `DefineIndex()` in the concurrent branch
([indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L440)),
which calls helpers in `index.c` (`index_create`, `index_concurrently_build`,
`validate_index`, `index_set_state_flags`) and waits via `WaitForLockers` /
`WaitForOlderSnapshots`.

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

### Failure handling

If the build or validation fails (deadlock, uniqueness violation, expression
error), the command leaves behind an **invalid** index: it has a catalog row but
`indisvalid = false`, so the planner ignores it for queries, yet it still adds
write overhead. The documented recovery is to `DROP INDEX` and retry, or rebuild
with `REINDEX INDEX CONCURRENTLY`
([ref/create_index.sgml#invalid-index](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596)).

### Test coverage

- Functional coverage is in the regression suite: `create_index.sql` builds
  empty-table, unique, expression, and partial indexes concurrently, defaults
  the index name, and asserts that CIC fails inside a `BEGIN; ... COMMIT;` block
  ([create_index.sql#cic-block](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L467-L520)).
- The concurrency itself is exercised by an isolation test that runs two CIC
  operations simultaneously and uses advisory locks to interleave them
  ([multiple-cic.spec](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40)).

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
- Tests: `src/test/regress/sql/create_index.sql` and
  `src/test/isolation/specs/multiple-cic.spec`.

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
| Correctness narrative (two scans, three waits) | [index.c:3112-3174](../../../raw/postgres-12/src/backend/catalog/index.c#L3112-L3174), [ref/create_index.sgml:545-572](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L572) |
| Invalid index left on failure; unique-constraint caveat | [ref/create_index.sgml:574-606](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L606) |
| Tests | [create_index.sql:467-520](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L467-L520), [multiple-cic.spec:1-40](../../../raw/postgres-12/src/test/isolation/specs/multiple-cic.spec#L1-L40) |

## Open Questions

- The first build scan's exact tuple-visibility rule lives inside
  `index_build` → `table_index_build_scan` (via `indexInfo->ii_Concurrent`),
  which this page summarizes from the `validate_index` header comment rather
  than tracing line-by-line into the table AM.
  ([index.c:1426-1427](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1427))
- In PostgreSQL 12, `WaitForOlderSnapshots` does not exclude other concurrent
  index builds from its wait set; CIC instead relies on dropping its reference
  snapshot before waiting to avoid mutual deadlock. Whether this is the only
  inter-CIC interaction was not exhaustively traced.

## Source References

- [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L1473)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)
- [index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439)
- [index.c#validate_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3112-L3298)
- [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)
- [lmgr.c#WaitForLockers](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)
- [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)
- [lockdefs.h#lockmodes](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46)
- [utility.c#CIC-dispatch](../../../raw/postgres-12/src/backend/tcop/utility.c#L1305-L1321)
- [ref/create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L631)

## Navigation

- [v12 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
