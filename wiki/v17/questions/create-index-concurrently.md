---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: GPT-5-5-XHigh-Thinking 2026-06-18T14:10:05Z
---

# How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Preconditions and restrictions](#preconditions-and-restrictions)
  - [The three pg_index state flags](#the-three-pgindex-state-flags)
  - [Step-by-step implementation](#step-by-step-implementation)
  - [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table)
  - [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)
  - [Failure handling](#failure-handling)
  - [Test coverage](#test-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

give a comprehensive explanation of how create index concurrently is
implemented, add a section with all steps and locks required on the table, and
add a section with what has changed from postgresql 12.

## Answer

`CREATE INDEX CONCURRENTLY` (CIC) builds an index without ever taking a lock
that blocks `INSERT`/`UPDATE`/`DELETE` on the table. In PostgreSQL 17, as in
PostgreSQL 12, it splits the build across **four internal transactions**, does
**two full table scans**, and **waits out other transactions three times**. The
core lock/scan/wait shape remains recognizable from v12. The main behavioral
optimization is `PROC_IN_SAFE_IC` (added in PostgreSQL 14), which lets a CIC
build on a plain index skip waiting for other concurrent index builds in its
final snapshot wait; v17 also differs in catalog-update and security plumbing
inside the same phases. See [What changed from PostgreSQL 12](#what-changed-from-postgresql-12).

All orchestration lives in `DefineIndex()` in the concurrent branch
([indexcmds.c#DefineIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540)),
which calls helpers in `index.c` (`index_create`, `index_concurrently_build`,
`validate_index`, `index_set_state_flags`) and waits via `WaitForLockers` and
`WaitForOlderSnapshots`. The table lock used throughout is
**`ShareUpdateExclusiveLock`** — strong enough to keep out a second CIC,
`VACUUM`, `ANALYZE`, and schema changes, but weak enough to let normal DML
proceed
([lockdefs.h:36-46](../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)).

### Preconditions and restrictions

| Restriction | Where enforced |
|---|---|
| Cannot run inside a transaction block (`BEGIN; ... COMMIT;`) | `PreventInTransactionBlock(isTopLevel, "CREATE INDEX CONCURRENTLY")` ([utility.c:1460-1462](../../../raw/postgres-17/src/backend/tcop/utility.c#L1460-L1462)) |
| Temporary tables silently fall back to a non-concurrent build | [indexcmds.c:612](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L612) |
| Partitioned tables cannot be built concurrently | [indexcmds.c:729](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L729) |
| System catalog tables cannot be indexed concurrently | [index.c:852-853](../../../raw/postgres-17/src/backend/catalog/index.c#L852-L853) |
| Exclusion constraints cannot be built concurrently | [index.c:861-862](../../../raw/postgres-17/src/backend/catalog/index.c#L861-L862) |

### The three pg_index state flags

CIC is driven by three boolean flags on the index's `pg_index` row, set by
`UpdateIndexRelation`
([index.c:645-648](../../../raw/postgres-17/src/backend/catalog/index.c#L645-L648)):

- `indislive` — the index exists and must be maintained. CIC sets this `true`
  from the start.
- `indisready` — new tuples (from `INSERT`/non-HOT `UPDATE`) must be inserted
  into the index.
- `indisvalid` — the planner may use the index to answer queries.

A normal `CREATE INDEX` is born with all three `true`. CIC instead creates the
catalog row with `indisvalid = false` and `indisready = false`
(`!concurrent && !invalid` and `!concurrent` at
[index.c:1048-1049](../../../raw/postgres-17/src/backend/catalog/index.c#L1048-L1049)),
then flips `indisready`, and finally `indisvalid`, using `index_set_state_flags`.
In v17, that helper edits a writable copy of the `pg_index` tuple and stores it
with transactional `CatalogTupleUpdate`, so other sessions hear about the flag
change after the transaction commits
([index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3440-L3518)).

### Step-by-step implementation

The concurrent branch runs as four transactions, separated by
`CommitTransactionCommand()` / `StartTransactionCommand()` pairs.

**Transaction 1 — create the catalog entry (no build).** `index_create` makes
the `pg_index`/`pg_class` rows with `INDEX_CREATE_CONCURRENT` and
`INDEX_CREATE_SKIP_BUILD`, so the index has a catalog identity but no data and is
marked not-ready/not-valid. CIC also computes `safe_index` — true only when the
index has no expressions and no predicate
([indexcmds.c:1128-1130](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1128-L1130)).
Before committing, it takes a **session-level** `ShareUpdateExclusiveLock` on the
table so it survives the upcoming commits and nobody can drop the table or index
([indexcmds.c:1599-1603](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1599-L1603)).
The commit makes the empty index visible so other backends stop making
incompatible HOT updates.

**Wait 1, then Transaction 2 — first scan, build.** After the commit (and, if
`safe_index`, a call to `set_indexsafe_procflags()` —
[indexcmds.c:1605-1607](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1605-L1607)),
CIC waits for every transaction that could still have the table open without the
new index, using `WaitForLockers(heaplocktag, ShareLock, true)`
([indexcmds.c:1626-1642](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1626-L1642)).
It then takes a fresh snapshot and calls `index_concurrently_build`, which scans
the heap, builds the index, and sets `indisready = true`
([indexcmds.c:1663-1666](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1663-L1666),
[index.c#index_concurrently_build](../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1539)).
The commit publishes `indisready` so new writers start maintaining the index.

**Wait 2, then Transaction 3 — second scan, validate.** After another
`set_indexsafe_procflags()` (if safe) it waits again with
`WaitForLockers(..., ShareLock, ...)`
([indexcmds.c:1677-1689](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1677-L1689)),
registers a **reference snapshot**, and calls `validate_index`
([indexcmds.c:1706-1712](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1706-L1712)).
`validate_index` collects the TIDs already in the index, sorts them, scans the
heap, and inserts any tuple visible to the reference snapshot but missing from
the index
([index.c#validate_index](../../../raw/postgres-17/src/backend/catalog/index.c#L3296-L3324)).
Before committing, CIC saves the snapshot's `xmin` as `limitXmin` and **drops the
snapshot** to avoid deadlocking against other CIC runs
([indexcmds.c:1721-1724](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1721-L1724)).

**Wait 3, then Transaction 4 — mark valid.** In a fresh transaction (so no
snapshot is held; the `Assert(MyProc->xmin == InvalidTransactionId)` enforces
this), and after a final `set_indexsafe_procflags()` if safe, CIC calls
`WaitForOlderSnapshots(limitXmin, true)` to wait out any transaction whose
snapshot predates the reference snapshot
([indexcmds.c:1737-1752](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1737-L1752)).
Then it sets `indisvalid = true`, sends a relcache invalidation on the parent
table so cached plans re-plan to use the new index, and releases the session lock
([indexcmds.c:1757-1772](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1772)).
This last transaction commits in the caller when the utility command finishes.

### All steps and locks required on the table

Two distinct lock layers act on the **heap table**:

1. A **transaction-level** `ShareUpdateExclusiveLock`, acquired by `table_open`
   at the start of the command
   ([indexcmds.c:678-679](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L678-L679))
   and re-acquired inside `index_concurrently_build` and `validate_index`;
   released at each `CommitTransactionCommand`.
2. A **session-level** `ShareUpdateExclusiveLock`
   (`LockRelationIdForSession`) that spans the gaps between transactions so the
   table cannot be dropped mid-build; released by `UnlockRelationIdForSession`
   at the very end.

`ShareUpdateExclusiveLock` is the same level `VACUUM (non-FULL)` and `ANALYZE`
use ([lockdefs.h:36-46](../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)).
Its conflict row determines what it lets through and what it blocks
([lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L103)):

| Other lock (command) | Conflicts with CIC's `ShareUpdateExclusiveLock`? |
|---|---|
| `AccessShareLock` (`SELECT`) | No — reads continue |
| `RowShareLock` (`SELECT FOR UPDATE/SHARE`) | No |
| `RowExclusiveLock` (`INSERT`/`UPDATE`/`DELETE`) | No — **writes continue** |
| `ShareUpdateExclusiveLock` (another CIC, `VACUUM`, `ANALYZE`) | **Yes** — only one at a time |
| `ShareLock` (plain `CREATE INDEX`) | **Yes** |
| `ShareRowExclusiveLock` / `ExclusiveLock` / `AccessExclusiveLock` (most `ALTER TABLE`, `DROP`) | **Yes** — schema changes blocked |

Because `ShareUpdateExclusiveLock` is **self-conflicting**
([lock.c:77-81](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L81)),
only one concurrent build can run on a given table at a time.

The three "waits" are **not** table locks. `WaitForLockers` calls
`WaitForLockersMultiple`, which obtains the current conflicting lock holders and
waits on their virtual transaction IDs; it explicitly does not try to acquire the
relation locks itself
([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L900),
[lmgr.c:914-923](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L914-L923),
[lmgr.c:935-952](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L935-L952)).
Waits 1 and 2 pass `ShareLock`, which conflicts with `RowExclusiveLock`, so they
wait out all current **writers**
([lock.c:82-86](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L86)).
Wait 3 (`WaitForOlderSnapshots`) waits on transactions holding old snapshots,
excluding autovacuum workers, lazy `VACUUM`, and — present in v17 but absent in
v12 — other safe concurrent index builds
(`PROC_IS_AUTOVACUUM | PROC_IN_VACUUM | PROC_IN_SAFE_IC`)
([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L439-L442)).

End-to-end table lock timeline:

| Phase | Table (heap) locks | Wait performed | Catalog effect |
|---|---|---|---|
| Txn 1: create catalog row | Txn `ShareUpdateExclusiveLock` + session `ShareUpdateExclusiveLock` taken ([:678-679](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L678-L679), [:1599](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1599)) | — | `indislive=t`, `indisready=f`, `indisvalid=f` |
| (commit 1) | Txn lock released; session lock held | — | empty index visible |
| Txn 2: build | Heap `ShareUpdateExclusiveLock`; index `RowExclusiveLock` ([index.c:1496-1509](../../../raw/postgres-17/src/backend/catalog/index.c#L1496-L1509)) | Wait 1: `WaitForLockers(ShareLock)` waits out writers | first scan; then `indisready=t` |
| (commit 2) | Txn lock released; session lock held | — | `indisready` visible |
| Txn 3: validate | Heap `ShareUpdateExclusiveLock`; index `RowExclusiveLock` ([index.c:3324](../../../raw/postgres-17/src/backend/catalog/index.c#L3324)) | Wait 2: `WaitForLockers(ShareLock)` waits out writers | second scan inserts missing tuples |
| (commit 3) | Txn lock released; session lock held | — | reference snapshot dropped |
| Txn 4: mark valid | Session `ShareUpdateExclusiveLock` still held | Wait 3: `WaitForOlderSnapshots(limitXmin)` (ignores safe CIC/RIC) | `indisvalid=t`; relcache inval; session lock released |

Throughout, the **only** lock the table ever carries is
`ShareUpdateExclusiveLock` (transaction-level within each phase, session-level
across the gaps). DML conflicts with none of these — the whole point of CIC.

### What changed from PostgreSQL 12

The CIC algorithm — four transactions, two scans, three waits, the
`ShareUpdateExclusiveLock` footprint, and the `indislive`/`indisready`/`indisvalid`
progression shape — remains recognizable from v12. (For the v12 trace, see
[How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../../v12/questions/create-index-concurrently.md).)
PostgreSQL 17 differs in one snapshot-wait optimization and in catalog/security
plumbing around the same phases.

**1. `PROC_IN_SAFE_IC`: a CIC build can be ignored by other concurrent builds
(PostgreSQL 14).** In v12, the final snapshot wait `WaitForOlderSnapshots`
excluded only autovacuum and lazy `VACUUM`, so two CIC builds running on
different tables would wait on each other's snapshots, wasting time and risking
deadlock. PostgreSQL 14 added the `PROC_IN_SAFE_IC` status flag
([proc.h:57-62](../../../raw/postgres-17/src/include/storage/proc.h#L57-L62)):
a CIC (or REINDEX CONCURRENTLY) on a **plain** index — no expressions, no partial
predicate — advertises it via `set_indexsafe_procflags()`
([indexcmds.c#set_indexsafe_procflags](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4455-L4487)),
called once per internal transaction after each commit
([:1605-1607](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1605-L1607),
[:1677-1679](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1677-L1679),
[:1737-1739](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1737-L1739)).
`WaitForOlderSnapshots` then adds `PROC_IN_SAFE_IC` to the processes it ignores
([:439-442](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L439-L442)),
because such a process "won't examine any data outside the table they're
indexing"
([:412-419](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L412-L419)).
The v17 docs spell this out: after the second scan the build waits for snapshots
predating it "including transactions used by any phase of concurrent index builds
on other tables, **if the indexes involved are partial or have columns that are
not simple column references**"
([ref/create_index.sgml:627-643](../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L627-L643)).
Introduced by commit `c98763bf` (2020-11-25, "Avoid spurious waits in concurrent
indexing"; first shipped in PostgreSQL 14), extended to REINDEX CONCURRENTLY by
`f9900df5` (2021-01-15, "Avoid spurious wait in concurrent reindex").

Scope of the optimization: it only shortens **Wait 3** (the snapshot wait) and
only when the *other* builds are on safe indexes. Waits 1 and 2 use
`WaitForLockers`, which is lock-based (`GetLockConflicts`) and is **not** affected
by `PROC_IN_SAFE_IC` — they still wait out all writers. The v17 `multiple-cic`
isolation test deliberately uses **partial** indexes (`WHERE ...`), which are not
safe, so the two builds still interact and one is expected to wait
([multiple-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43)).

**2. A `VACUUM`-ignores-CIC attempt was added and then reverted — so this did
NOT change.** PostgreSQL 14 also tried to let `VACUUM` advance its xmin horizon
past safe CIC/RIC processes, but that was reverted by commit `e28bb885`
(2022-05-31, "Revert changes to CONCURRENTLY that \"sped up\" Xmin advance"),
which is present in the v17 tree, because it could let a concurrently-built index
miss heap tuples that were HOT-pruned during the build. So in v17, exactly as in
v12, a running CIC still holds back `VACUUM`'s horizon; `PROC_IN_SAFE_IC` only
affects other *concurrent index* operations' snapshot waits, not `VACUUM`.

**3. `index_set_state_flags` is transactional in v17 (PostgreSQL 14).**
PostgreSQL 14 commit `83158f74d3a` (2020-09-14, "Make
index_set_state_flags() transactional") changed the state-flag helper used by
CIC. In v17, `index_set_state_flags` fetches a writable copy of the `pg_index`
tuple, mutates `indisready` or `indisvalid`, and stores the result with
`CatalogTupleUpdate`
([index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3440-L3518)).
That means the phase-2 ready flip and phase-4 valid flip follow normal
transaction commit/abort semantics in v17. It does not change the number of CIC
transactions, scans, waits, or table locks.

**4. The flag lives in `PGPROC`, not `PGXACT` (mechanical, PostgreSQL 14).**
v12 kept per-backend vacuum state in `MyPgXact->vacuumFlags` (the `PGXACT`
array). PostgreSQL 14's snapshot-scalability work (commit `5788e258`, 2020-08-14)
moved that state to `PGPROC`/`ProcGlobal`, and `set_indexsafe_procflags` now sets
`MyProc->statusFlags` and mirrors it into `ProcGlobal->statusFlags[]`
([indexcmds.c:4483-4485](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4483-L4485)).
This is why DefineIndex's final-phase assertion reads `MyProc->xmin` in v17 where
v12 read `MyPgXact->xmin`.

**5. The concurrent build runs index/predicate functions under the owner with a
restricted search path (security hardening, post-v12).** v17's
`index_concurrently_build` switches to the table owner's userid and calls
`RestrictSearchPath()` before building
([index.c:1503-1507](../../../raw/postgres-17/src/backend/catalog/index.c#L1503-L1507));
v12's `index_concurrently_build` did neither. This does not change the lock or
transaction structure.

### Failure handling

If the build or validation fails, CIC leaves behind an **invalid** index: it has
a catalog row but `indisvalid = false`, so the planner ignores it while it still
adds write overhead. The documented recovery is to `DROP INDEX` and retry, or
rebuild with `REINDEX INDEX CONCURRENTLY`
([ref/create_index.sgml:645-667](../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667)).
For a unique index, the uniqueness constraint is enforced from the start of the
second scan, so violations can surface in other sessions before the index is
usable, and a failed second scan leaves an invalid index that still enforces
uniqueness
([ref/create_index.sgml:669-677](../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L669-L677)).

### Test coverage

- Regression: `create_index.sql` builds empty-table, unique, partial, and
  expression indexes concurrently, defaults the index name, and asserts that CIC
  fails inside a `BEGIN; ... COMMIT;` block
  ([create_index.sql:488-525](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L488-L525)).
- Isolation: `multiple-cic.spec` runs two concurrent builds (on partial indexes)
  simultaneously
  ([multiple-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43)),
  and `prepared-transactions-cic.spec` verifies CIC waits correctly for a prepared
  (two-phase-commit) transaction
  ([prepared-transactions-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and the recent
  `wiki/log.md` entries (navigation only).
- Pinned checkout `raw/postgres-17/` at commit
  `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- `DefineIndex` concurrent branch, `WaitForOlderSnapshots`, and
  `set_indexsafe_procflags` in `src/backend/commands/indexcmds.c`.
- `UpdateIndexRelation`/`index_create`, `index_concurrently_build`,
  `validate_index`, and `index_set_state_flags` in `src/backend/catalog/index.c`.
- `PROC_IN_SAFE_IC` in `src/include/storage/proc.h`; the `LockConflicts` table in
  `src/backend/storage/lmgr/lock.c`; lock-mode definitions in
  `src/include/storage/lockdefs.h`.
- `doc/src/sgml/ref/create_index.sgml` (CONCURRENTLY narrative).
- Tests: `src/test/regress/sql/create_index.sql`,
  `src/test/isolation/specs/multiple-cic.spec`,
  `src/test/isolation/specs/prepared-transactions-cic.spec`.
- v12 vs v17 deltas established from the `raw/postgres-17/` checkout's own commit
  history (`c98763bf`, `f9900df5`, `e28bb885`, `83158f74d3a`, `5788e258`), each
  verified present with `git show` / `git tag --contains`.

## Evidence Map

| Claim | Source |
|---|---|
| Table lock is `ShareUpdateExclusiveLock` for concurrent | [indexcmds.c:678-679](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L678-L679) |
| `safe_index` true only for non-expression, non-partial indexes | [indexcmds.c:1128-1130](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1128-L1130) |
| Four-transaction structure; session lock; three waits | [indexcmds.c:1599-1772](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1599-L1772) |
| Build sets `indisready`; validate inserts missing tuples | [index.c:1482-1539](../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1539), [index.c:3296-3402](../../../raw/postgres-17/src/backend/catalog/index.c#L3296-L3402) |
| v17 state-flag flips use transactional `CatalogTupleUpdate` | [index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3440-L3518) |
| `index_set_state_flags` transactional change came in PG14 | commit `83158f74d3a` (2020-09-14) |
| `WaitForLockers` reads current lock holders and waits on VXIDs without taking the relation lock itself | [lmgr.c#WaitForLockersMultiple](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L900), [lmgr.c:914-923](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L914-L923), [lmgr.c:935-952](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L935-L952) |
| `WaitForLockers(ShareLock)` waits out writers; `ShareLock` vs `RowExclusiveLock` | [lock.c:82-86](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L82-L86) |
| `ShareUpdateExclusiveLock` conflict set + self-conflict | [lock.c:77-81](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L81) |
| Wait 3 ignores autovacuum, VACUUM, and safe CIC (`PROC_IN_SAFE_IC`) | [indexcmds.c:439-442](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L439-L442) |
| `PROC_IN_SAFE_IC` flag definition | [proc.h:57-62](../../../raw/postgres-17/src/include/storage/proc.h#L57-L62) |
| `set_indexsafe_procflags` sets `MyProc->statusFlags` | [indexcmds.c:4455-4487](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4455-L4487) |
| Doc: snapshot wait ignores other builds unless partial/expressional | [ref/create_index.sgml:627-643](../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L627-L643) |
| PROC_IN_SAFE_IC added PG14 | commit `c98763bf` (2020-11-25), `f9900df5` (2021-01-15) |
| VACUUM-ignores-CIC reverted (present in v17) | commit `e28bb885` (2022-05-31) |
| PGXACT->PGPROC statusFlags move (PG14) | commit `5788e258` (2020-08-14) |
| v17 restricts search path during build | [index.c:1503-1507](../../../raw/postgres-17/src/backend/catalog/index.c#L1503-L1507) |
| Tests | [create_index.sql:488-525](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L488-L525), [multiple-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43), [prepared-transactions-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37) |

## Open Questions

- The exact tuple-visibility rule of the first build scan lives inside
  `index_build` → `table_index_build_scan` (via `indexInfo->ii_Concurrent`),
  summarized here rather than traced line-by-line.
- PostgreSQL 17 also refined REINDEX CONCURRENTLY's own `safe` determination for
  indexes with predicates/expressions (commit `cd6b2ae3`, "Fix waits of REINDEX
  CONCURRENTLY for indexes with predicates or expressions"); that is REINDEX
  CONCURRENTLY scope, not `CREATE INDEX CONCURRENTLY`, and was not traced here.
- The v12 behavioral claims in [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)
  are anchored to the v17 checkout's commit history and the companion
  [v12 page](../../v12/questions/create-index-concurrently.md); they are not
  re-cited against the PostgreSQL 12 source checkout here, per the
  one-version-per-page citation rule.

## Source References

- [utility.c#T_IndexStmt](../../../raw/postgres-17/src/backend/tcop/utility.c#L1452-L1462)
- [indexcmds.c#DefineIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L1777)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L397-L479)
- [indexcmds.c#set_indexsafe_procflags](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4455-L4487)
- [index.c#UpdateIndexRelation](../../../raw/postgres-17/src/backend/catalog/index.c#L630-L648)
- [index.c#index_create](../../../raw/postgres-17/src/backend/catalog/index.c#L697-L709)
- [index.c#index_create-flags](../../../raw/postgres-17/src/backend/catalog/index.c#L1043-L1049)
- [index.c#index_concurrently_build](../../../raw/postgres-17/src/backend/catalog/index.c#L1482-L1539)
- [index.c#validate_index](../../../raw/postgres-17/src/backend/catalog/index.c#L3233-L3402)
- [index.c#index_set_state_flags](../../../raw/postgres-17/src/backend/catalog/index.c#L3440-L3518)
- [proc.h#PROC_IN_SAFE_IC](../../../raw/postgres-17/src/include/storage/proc.h#L54-L78)
- [lmgr.c#WaitForLockersMultiple](../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L988)
- [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L103)
- [lockdefs.h#lockmodes](../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)
- [ref/create_index.sgml#CONCURRENTLY](../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L612-L702)
- [create_index.sql#concurrent-indexes](../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L488-L525)
- [multiple-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43)
- [prepared-transactions-cic.spec](../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37)

## Navigation

- [v17 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
- [v12: CREATE INDEX CONCURRENTLY](../../v12/questions/create-index-concurrently.md)
