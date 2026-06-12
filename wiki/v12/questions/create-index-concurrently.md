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
  - [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table)
  - [All operations that can block CREATE INDEX CONCURRENTLY](#all-operations-that-can-block-create-index-concurrently)
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
  See `## Open Questions`.

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

#### The same states under REINDEX INDEX CONCURRENTLY (the `_ccnew` / `_ccold` names)

`REINDEX INDEX CONCURRENTLY` reuses the exact same four-state build machine, but
it runs it on a **new copy** it creates next to the original, then swaps the two.
Its six phases are: create the copy, build it, validate it, swap names, mark the
old one dead, drop the old one
([indexcmds.c#reindex-phases](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955)).
The copy is named `<original>_ccnew` while it is being built
([indexcmds.c#ccnew-name](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2993-L2998)),
and the swap then gives the rebuilt copy the original's name while renaming the
old index to `<original>_ccold`
([indexcmds.c#ccold-name](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3230-L3241),
[index.c#swap-names](../../../raw/postgres-12/src/backend/catalog/index.c#L1490-L1492)).
So the four states apply to the `_ccnew` copy, and the original is left untouched
(still valid, if it was valid) until the swap.

Take this index to rebuild:

| Command | Index being reindexed | Transient copy it builds |
|---|---|---|
| `REINDEX INDEX CONCURRENTLY concur_reindex_ind5` | `concur_reindex_ind5` | `concur_reindex_ind5_ccnew` |

`\d` then shows these names after each state of the `_ccnew` copy's build:

| `_ccnew` copy state | Names visible in `\d` afterward |
|---|---|
| no copy (failure before the copy's first commit) | only `concur_reindex_ind5` (original untouched) |
| invalid, not ready (build-scan failure, e.g. a duplicate caught by the unique build sort) | `concur_reindex_ind5` + a leftover `concur_reindex_ind5_ccnew` **INVALID** ([create_index.out:2323-2333](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2323-L2333)) |
| invalid, ready (second-scan failure) | `concur_reindex_ind5` + `concur_reindex_ind5_ccnew` **INVALID** (ready, so the copy also enforces uniqueness) |
| valid, then swap (success) | one `concur_reindex_ind5` (now the rebuilt copy, marked valid); the old index becomes `concur_reindex_ind5_ccold`, is marked dead, and is dropped ([indexcmds.c#reindex-phases](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955)) |

The names are identical for the two `INVALID` rows because `indisready` is not
shown by `\d` — only the build's write/uniqueness cost differs (see the cost
table below). In the cited regression the original `concur_reindex_ind5` is also
`INVALID`, but only because that example built it from an already-failed CIC; a
reindex whose `_ccnew` build fails never invalidates a healthy original, since
the swap (the only step that marks the old index invalid) is reached only after
the copy is fully built and validated
([indexcmds.c#swap-marks-old-invalid](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3201-L3241)).
A failed `_ccnew` is just an invalid index, cleaned up like any other —
`DROP INDEX concur_reindex_ind5_ccnew`, then retry once the underlying problem is
gone
([create_index.out:2335-2350](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2335-L2350)).

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
- **A retried concurrent rebuild can stack invalid indexes.**
  `REINDEX INDEX CONCURRENTLY` of an already-invalid unique index adds a second,
  also-invalid `_ccnew` index
  ([create_index.out#ccnew](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2317-L2333));
  the spare is dropped, and after the duplicate is removed the original is
  repaired with a non-concurrent `REINDEX INDEX`
  ([create_index.out#repair](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2335-L2350)).

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

This page does not state a definitive crash-recovery outcome for every
instruction boundary. Ordinary ERROR/cancel rollback rules are not enough by
themselves, because `index_set_state_flags` uses a non-transactional in-place
catalog update that "will not roll back on error", and `heap_inplace_update`
overwrites the tuple and writes a WAL record in a critical section
([index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3316-L3324),
[heapam.c#heap_inplace_update](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5692-L5774)).
The unresolved crash windows are listed under `## Open Questions`.

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
  ([create_index.out#concurrent-invalid](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1382-L1417)).

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
  the `INDEX_DROP_*` branches of `index_set_state_flags`, plus the
  `REINDEX INDEX CONCURRENTLY` phase loop (`ReindexRelationConcurrently`, the
  `_ccnew`/`_ccold` naming, and `index_concurrently_swap`) in
  `src/backend/catalog/index.c` and `src/backend/commands/indexcmds.c`;
  `heap_inplace_update` in `src/backend/access/heap/heapam.c` for the scoped
  crash-recovery open question;
  `LockRelationIdForSession` in
  `src/backend/storage/lmgr/lmgr.c` and `ProcReleaseLocks` in
  `src/backend/storage/lmgr/proc.c`; and the invalid-index regression evidence in
  `src/test/regress/expected/create_index.out`.
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
| Failure before commit 1 leaves no index (commit-1 boundary) | [indexcmds.c:1318-1320](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1318-L1320) |
| Planner ignores `indisvalid = false` indexes (executor still inserts if `indisready`) | [plancat.c:200-210](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210) |
| Not-ready index is opened + `RowExclusiveLock`ed but receives no entries, and its unique check is skipped | [execIndexing.c:185-192](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L185-L192), [execIndexing.c:330-332](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332), [execIndexing.c:537-539](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L537-L539) |
| Every `indislive` index counts for HOT-safety; not-live indexes are omitted from the index list | [relcache.c:4388-4395](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4388-L4395), [relcache.c:4861-4870](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4861-L4870) |
| Build sets `indisready` as its last action (build-scan vs validate-scan split); state-ladder asserts | [index.c:1426-1438](../../../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438), [index.c:3353-3396](../../../raw/postgres-12/src/backend/catalog/index.c#L3353-L3396) |
| Build-scan dup is "could not create unique index ... is duplicated"; concurrent second-scan dup is "duplicate key ... already exists" | [tuplesort.c:4048-4056](../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4048-L4056), [nbtinsert.c:563-568](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568) |
| Regression: failed unique build left INVALID, retained through `VACUUM FULL`, fixed by REINDEX; retried rebuild stacks an invalid `_ccnew` | [create_index.out:1383-1417](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1383-L1417), [create_index.out:1400-1406](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1400-L1406), [create_index.out:2317-2350](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2317-L2350) |
| Named example index per `pg_index` state: `concur_index7` (none), `concur_index3` (invalid, not ready), `concur_index1`/`concur_index2` (valid) | [create_index.out:1391-1395](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1391-L1395), [create_index.out:1413-1420](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1413-L1420) |
| `REINDEX INDEX CONCURRENTLY`: six phases; builds `<orig>_ccnew`, swap renames it to the original and the old to `<orig>_ccold` (new marked valid, old invalid); failed `_ccnew` left INVALID | [indexcmds.c:2941-2955](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955), [indexcmds.c:2993-2998](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2993-L2998), [indexcmds.c:3201-3241](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3201-L3241), [index.c:1490-1492](../../../raw/postgres-12/src/backend/catalog/index.c#L1490-L1492), [create_index.out:2323-2350](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2323-L2350) |
| A failed CIC releases its session lock (removed on `ereport(ERROR)`; abort releases session locks) | [lmgr.c:356-363](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L356-L363), [proc.c:772-798](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L772-L798) |
| `DROP INDEX CONCURRENTLY` is retryable: `INDEX_DROP_CLEAR_VALID` does not assert its starting flags | [index.c:3367-3383](../../../raw/postgres-12/src/backend/catalog/index.c#L3367-L3383) |
| Crash/immediate-shutdown outcome is left open because state-flag updates are non-transactional in-place WAL-logged overwrites | [index.c:3316-3324](../../../raw/postgres-12/src/backend/catalog/index.c#L3316-L3324), [heapam.c:5746-5774](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774) |
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
- Waits 1 and 2 ignore prepared transactions:
  [lmgr.c:890-894](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L890-L894)
  asserts they "certainly aren't going to do anything anymore". But a
  transaction that wrote the table and then ran `PREPARE TRANSACTION` still
  holds `RowExclusiveLock` and commits its writes later, at `COMMIT PREPARED`
  ([lock.c:2873-2876](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876)).
  Whether skipping such a transaction in the writer waits is actually safe for
  index correctness was not assessed from this checkout; this page records
  only what the pinned source does and claims.
- Whether walsenders or other non-transaction backends holding an old xmin
  (for example via `hot_standby_feedback` or replication slots) can appear in
  the Wait 3 set was not traced; `GetCurrentVirtualXIDs` requires a valid
  virtual transaction ID
  ([procarray.c:2537-2539](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2537-L2539)),
  which such backends may not advertise.
- The exact crash / immediate-shutdown outcome was not traced through crash
  recovery. This matters because `index_set_state_flags` is explicitly
  non-transactional and will not roll back on error
  ([index.c:3316-3324](../../../raw/postgres-12/src/backend/catalog/index.c#L3316-L3324)),
  while `heap_inplace_update` WAL-logs the overwrite in a critical section
  ([heapam.c:5746-5774](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774)).
  The exact flag state after a crash around `INDEX_CREATE_SET_READY`,
  `INDEX_CREATE_SET_VALID`, and their surrounding commit boundaries was not
  isolated.

## Source References

- [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L1473)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)
- [index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1399-L1439)
- [index.c#validate_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3112-L3298)
- [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)
- [lmgr.c#WaitForLockers](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)
- [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)
- [lock.c#GetLockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2804-L2821)
- [lockdefs.h#lockmodes](../../../raw/postgres-12/src/include/storage/lockdefs.h#L36-L46)
- [utility.c#CIC-dispatch](../../../raw/postgres-12/src/backend/tcop/utility.c#L1305-L1321)
- [procarray.c#GetCurrentVirtualXIDs](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2467-L2548)
- [proc.c#ProcSleep-autovacuum-cancel](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)
- [proc.h#vacuumFlags](../../../raw/postgres-12/src/include/storage/proc.h#L53-L56)
- [twophase.c#MarkAsPreparingGuts](../../../raw/postgres-12/src/backend/access/transam/twophase.c#L446-L490)
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
- [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L3260)
- [index.c#index_concurrently_swap](../../../raw/postgres-12/src/backend/catalog/index.c#L1441-L1515)
- [heapam.c#heap_inplace_update](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L5692-L5774)
- [tuplesort.c#unique-violation](../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4040-L4056)
- [nbtinsert.c#_bt_check_unique](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568)
- [lmgr.c#LockRelationIdForSession](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L356-L383)
- [proc.c#ProcReleaseLocks](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L772-L798)
- [create_index.out#concurrent-invalid](../../../raw/postgres-12/src/test/regress/expected/create_index.out#L1382-L1417)

## Navigation

- [v12 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
