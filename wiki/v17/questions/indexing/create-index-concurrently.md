---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Mental model](#mental-model)
  - [Logic map](#logic-map)
  - [Preconditions and restrictions](#preconditions-and-restrictions)
  - [The three pg_index state flags](#the-three-pg_index-state-flags)
  - [Step-by-step implementation](#step-by-step-implementation)
  - [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table)
  - [Failure handling](#failure-handling)
  - [How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping)
  - [GUCs that affect CIC performance](#gucs-that-affect-cic-performance)
  - [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)
  - [Test coverage](#test-coverage)
  - [Final causal summary](#final-causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Give a comprehensive explanation of how `CREATE INDEX CONCURRENTLY` is
implemented, add a section with all steps and locks required on the table, and
add a section with what has changed from PostgreSQL 12.

Follow-up: Investigate how `maintenance_work_mem` is used during `CREATE INDEX
CONCURRENTLY`, and at what point increasing it stops improving the index
creation process.

Follow-up: What GUCs have a performance impact on it?

## Answer

`CREATE INDEX CONCURRENTLY` (CIC, see
[CONCURRENTLY](../../../glossary.md#concurrently)) builds an index while other
sessions keep inserting, updating and deleting rows in the table. A plain
`CREATE INDEX` takes `ShareLock`, which blocks those writes for the whole
build. CIC takes the weaker
[`ShareUpdateExclusiveLock`](../../../glossary.md#shareupdateexclusivelock)
instead, so writes continue
([indexcmds.c#lock-mode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L663-L679),
[lockdefs.h#lock-modes](../../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)).

The price is more work. Because CIC cannot lock writers out, it has to make
every other session learn about the new index in two steps, and it has to wait
for the sessions that have not learned yet. That takes **four transactions, two
full scans of the table, and three waits for other transactions**
([ref/create_index.sgml#concurrent-phases](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L627-L643),
[index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)):

1. Transaction 1 creates the index's [catalog](../../../glossary.md#catalog)
   rows, marked "not ready, not valid", and commits. From then on every session
   knows the index exists. Wait 1 lets the transactions that started earlier
   finish. Then the first scan builds the index.
2. Transaction 2 marks the index "ready" and commits. From then on every new
   write also inserts into the index. Wait 2 lets the earlier writers finish.
   Then the second scan adds the rows the first scan could not see.
3. Wait 3 lets every transaction finish whose
   [snapshot](../../../glossary.md#snapshot) is older than the second scan.
   Only then does transaction 4 mark the index "valid", so queries may use it.

All of this is driven by one function, `DefineIndex()`, whose concurrent path
calls `index_create`, `index_concurrently_build`, `validate_index` and
`index_set_state_flags`, and waits through `WaitForLockers` and
`WaitForOlderSnapshots`
([indexcmds.c#DefineIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L539-L1793)).

This page describes PostgreSQL 17.11. The four-transaction shape is the same
as in PostgreSQL 12; what differs is listed under
[What changed from PostgreSQL 12](#what-changed-from-postgresql-12).

### Mental model

Five ideas explain why CIC is built the way it is.

**1. HOT updates force the index to be announced before it is built.** A
[HOT](../../../glossary.md#hot) update writes a new row version without adding
index entries. That is allowed only when the update changes no column that an
index pointing at individual rows references
([README.HOT#HOT-update-requirement](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L136-L147)).
A session that does not yet know about the new index could HOT-update a column
the new index covers. CIC's index entry would then point at a row chain whose live
value differs from the entry, and there is no clean way to remove such a wrong
entry later. Therefore every writer must know about the index before the first
scan starts
([README.HOT#CIC-broken-chain-example](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L378-L386)).

**2. Three flags tell other sessions how to treat the index.** The index's
[`pg_index`](../../../glossary.md#pg_index) row carries `indislive`,
`indisready` and `indisvalid`. Together they say whether writers must respect
the index for HOT, whether writers must insert into it, and whether queries
may read it. See
[The three pg_index state flags](#the-three-pg_index-state-flags).

**3. A flag change reaches other sessions only when it commits.**
`index_set_state_flags()` updates the catalog row transactionally, and the
[invalidation message](../../../glossary.md#invalidation-message) for it is
sent at commit
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550)).
This is why CIC needs several transactions: each state other sessions must see
needs its own commit. A commit also throws away everything CIC built in
memory; only the relation IDs survive
([indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)).

**4. A commit releases ordinary locks, so CIC also holds a session-level
lock.** A [session-level lock](../../../glossary.md#session-level-lock) belongs
to the session, not to a transaction, so it survives the internal commits and
keeps the table and the index from being dropped between them
([indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)).

**5. CIC waits for other transactions; it never blocks them.** It waits in two
different ways that read two different pieces of shared state. Waits 1 and 2
ask the lock manager who holds a conflicting lock on the table right now.
Wait 3 asks the [ProcArray](../../../glossary.md#procarray) which transactions
hold an old snapshot. In both cases CIC then sleeps on each such transaction's
[virtual transaction ID](../../../glossary.md#virtual-transaction-id) (VXID)
until that transaction ends
([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973),
[indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497)).

The values that carry these ideas through the command:

| Value | Meaning | Source | Live/current or stored | Later use |
|---|---|---|---|---|
| `concurrent` | Whether this command takes the concurrent path at all | `stmt->concurrent` and the table is not temporary ([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)) | Computed once, at the start of `DefineIndex` | Chooses the lock mode, the `index_create` flags, and whether `DefineIndex` continues past the catalog step |
| Table lock mode | `ShareUpdateExclusiveLock` for CIC, `ShareLock` for a plain build | Chosen in `ProcessUtilitySlow` and again, identically, in `DefineIndex` ([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480), [indexcmds.c#lock-mode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L663-L679)) | Live in the shared lock table | Decides which other commands run alongside the build |
| `indislive`, `indisready`, `indisvalid` | The index's state as other sessions must treat it | Written by `index_create`, changed by `index_set_state_flags` ([index.c#index_create-state](../../../../raw/postgres-17/src/backend/catalog/index.c#L1042-L1057), [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550)) | Stored in the catalog. Another session sees a change only after the changing transaction commits | Read by the relcache (live), the executor (ready) and the planner (valid) |
| `safe_index` | True when the index has no expression column and no predicate | Computed in transaction 1 ([indexcmds.c#safe_index](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1144-L1146)) | A local variable that survives the internal commits | Decides whether transactions 2 to 4 set `PROC_IN_SAFE_IC` |
| `PROC_IN_SAFE_IC` | A per-backend status flag that says "this backend only builds a plain index" | Set by `set_indexsafe_procflags` ([indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4505)) | Live in shared memory. It is reset at every transaction end, so it must be set again in each transaction | Other concurrent builds skip this backend in their wait 3 |
| `heaprelid`, `heaplocktag` | The table's identity for locking | Saved before the first commit ([indexcmds.c#save-lock-identity](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1589-L1592)) | Local variables that survive the commits | The session lock, both `WaitForLockers` calls, the final invalidation |
| Build snapshot | The [MVCC](../../../glossary.md#mvcc) snapshot of the first scan | Taken after wait 1 ([indexcmds.c#first-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1682), [heapam_handler.c#build-scan-snapshot](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1270)) | Current as of the start of the first scan; gone at commit 2 | Decides which rows the first build indexes |
| Reference snapshot | The MVCC snapshot of the second scan | Taken after wait 2 ([indexcmds.c#reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1707-L1728)) | Current as of the start of validation; dropped before commit 3 | Decides which rows must be in the index |
| `limitXmin` | The reference snapshot's `xmin`: every transaction older than it had ended when the snapshot was taken | Copied out before the snapshot is dropped ([indexcmds.c#drop-reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1730-L1751)) | A stored copy of a past value. It does not move while wait 3 runs | The threshold of wait 3 |
| Index [TID](../../../glossary.md#tid) sort | Every heap TID the index already contains, sorted | Collected and sorted in `validate_index` ([index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)) | Built from the index's content at the start of validation | Merged against the second heap scan to find missing rows |

The two snapshots are easy to confuse. The build snapshot only limits what the
first scan reads. The reference snapshot is the one whose `xmin` becomes
`limitXmin`, so only it decides how long wait 3 lasts.

### Logic map

The command is a linear chain with a few early exits. Keys in brackets refer to
the list below the diagram.

```text
Statement start
  [A] inside a transaction block?        -- yes --> ERROR
  [B] temporary table?                   -- yes --> plain (non-concurrent) build
  [C] lock the table: ShareUpdateExclusiveLock

Transaction 1  (the statement's own transaction)
  [D] checks; partitioned table or system catalog?  -- yes --> ERROR
  [E] index_create: catalog rows only; indisready = f, indisvalid = f
  [F] take a session-level ShareUpdateExclusiveLock on the table
  COMMIT 1 ........ every session now sees the index, so HOT-safety applies

Transaction 2
  [G] set PROC_IN_SAFE_IC if the index is plain
  [H] WAIT 1: each transaction that holds a conflicting lock on the table
  [I] build snapshot -> first heap scan -> access method build
  [J] indisready = t
  COMMIT 2 ........ every new write now inserts into the index

Transaction 3
  [G] set PROC_IN_SAFE_IC again
  [K] WAIT 2: lock holders again
  [L] reference snapshot -> collect the index's TIDs -> sort ->
      second heap scan -> insert rows visible to the snapshot but missing
  [M] limitXmin = reference snapshot's xmin; drop the snapshot
  COMMIT 3

Transaction 4
  [G] set PROC_IN_SAFE_IC again
  [N] WAIT 3: each transaction in this database whose xmin <= limitXmin,
      except autovacuum, VACUUM and safe concurrent index builds
  [O] indisvalid = t; relcache invalidation for the table
  [P] release the session-level lock
  COMMIT 4 (the statement's normal commit) ........ queries may use the index
```

- [A] [utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463)
- [B] [indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)
- [C] [utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480),
  [indexcmds.c#lock-mode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L663-L679)
- [D] [indexcmds.c#partitioned-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L709-L731),
  [index.c#concurrent-restrictions](../../../../raw/postgres-17/src/backend/catalog/index.c#L852-L869)
- [E] [indexcmds.c#index_create-flags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1181-L1199),
  [index.c#index_create-state](../../../../raw/postgres-17/src/backend/catalog/index.c#L1042-L1057),
  [index.c#index_create-skip-build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1256-L1281)
- [F] [indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)
- [G] [indexcmds.c#safe-flag-txn-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1621-L1623),
  [indexcmds.c#safe-flag-txn-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1693-L1695),
  [indexcmds.c#safe-flag-txn-4](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1753-L1755)
- [H] [indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658)
- [I] [indexcmds.c#first-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1682),
  [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557)
- [J] [index.c#set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1551-L1556)
- [K] [indexcmds.c#wait-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1697-L1705)
- [L] [indexcmds.c#reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1707-L1728),
  [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)
- [M] [indexcmds.c#drop-reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1730-L1751)
- [N] [indexcmds.c#wait-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1768),
  [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497)
- [O] [indexcmds.c#mark-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1770-L1783)
- [P] [indexcmds.c#release-session-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1785-L1792),
  [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821)

Each wait exists for one reason, and each makes one statement true:

| Wait | Runs in | Reads | Waits for | True afterwards |
|---|---|---|---|---|
| 1 | Transaction 2, before any snapshot | Lock table | Transactions that hold a lock on the table conflicting with `ShareLock` at that moment | No transaction still has the table open without knowing the index. So no new HOT chain can mix different key values ([indexcmds.c#build-rationale](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1660-L1676)) |
| 2 | Transaction 3, before any snapshot | Lock table | The same kind of lock holders, looked up again | Every transaction that can still write will insert into the index ([indexcmds.c#wait-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1697-L1705)) |
| 3 | Transaction 4, before any snapshot | ProcArray | Transactions whose snapshot `xmin` is at or before `limitXmin` | Nobody is left who could still see a row that validation rightly skipped ([indexcmds.c#wait-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1768)) |

### Preconditions and restrictions

CIC refuses some situations and silently changes behavior in one.

| Condition / state | Behavior | Consequence |
|---|---|---|
| The statement runs inside a transaction block | `PreventInTransactionBlock` raises `CREATE INDEX CONCURRENTLY cannot run inside a transaction block` ([utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463), [create_index.out#cic-in-transaction](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1427-L1431)) | CIC commits internally, so it cannot be part of a larger transaction. The docs state the same difference from a regular build ([ref/create_index.sgml#one-at-a-time](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L685-L693)) |
| The table is temporary | `DefineIndex` clears `concurrent` and runs a plain build ([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)) | No error and no CIC phases; see the explanation below the table |
| The table is partitioned | Error `cannot create index on partitioned table ... concurrently` ([indexcmds.c#partitioned-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L709-L731)) | Build each partition's index concurrently, then create the partitioned index, which is then a metadata-only step ([ref/create_index.sgml#partitioned](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L695-L702)) |
| The table is a system catalog | Error `concurrent index creation on system catalog tables is not supported` ([index.c#concurrent-restrictions](../../../../raw/postgres-17/src/backend/catalog/index.c#L852-L869)) | The source gives the reason: catalog code tends to release locks before committing |
| Another session holds a conflicting lock on the table, for example a second CIC or `VACUUM` | The statement queues for the table lock ([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480)) | Only one concurrent build runs on a table at a time ([ref/create_index.sgml#one-at-a-time](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L685-L693)) |
| The invoking role lacks `USAGE` on a type an index expression or predicate depends on | Error before `index_create()` ([indexcmds.c#type-usage-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L931-L945), [dependency.c#CheckUsageOnTypesInSingleRelExpr](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1740-L1766)) | New in 17.11; details below |

**Why a temporary table does not get a concurrent build.** By default, the
`CONCURRENTLY` keyword selects the four-transaction path. The cause of the
different behavior is the table's persistence: for a temporary table
`DefineIndex` sets `concurrent = false` before anything else uses the option.
The source gives the reason: other backends cannot access a temporary
relation, so the stronger lock of a plain build harms nobody
([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)).
The transaction-block check is different: it tests the statement's keyword, not
the computed flag, so CIC on a temporary table inside a transaction block still
fails
([create_index.sql#temp-tables](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L536-L559),
[create_index.out#temp-in-transaction](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1500-L1502)).

**Exclusion constraints.** `index_create()` also rejects a concurrent build for
an exclusion constraint, but its own comment says that `CREATE INDEX` has no
syntax to ask for one; only `REINDEX` can reach that check
([index.c#concurrent-restrictions](../../../../raw/postgres-17/src/backend/catalog/index.c#L852-L869)).
It is therefore not a restriction a CIC user can hit.

**The type `USAGE` check (17.11).** Commit `d1c8aa0b09` ("Check for USAGE
privilege on types used by stored expressions.", 2026-08-10, back-patched
through 14) added this check. It fails the statement **before**
`index_create()`, whose header now states the contract that the caller must
check `USAGE` for the types `indexInfo->ii_{Expressions,Predicate}` depend on
([index.c#index_create-contract](../../../../raw/postgres-17/src/backend/catalog/index.c#L727-L728)).
The check runs only when `DefineIndex` is called with `check_rights = true`. A
user-issued `CREATE INDEX`, concurrent or not, arrives that way from
`ProcessUtilitySlow`
([utility.c#IndexStmt-DefineIndex](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1540-L1553)).
The `ALTER TABLE ... ATTACH PARTITION` path that clones a parent index into the
new partition passes `check_rights = false`, so it skips the check
([tablecmds.c#attach-partition-clone-index](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18999-L19016)).
The regression test `CREATE INDEX ON test9a ((a::priv_testdomain1));` fails
with `permission denied for type public.priv_testdomain1`
([privileges.sql:959](../../../../raw/postgres-17/src/test/regress/sql/privileges.sql#L959),
[privileges.out#index-type-usage](../../../../raw/postgres-17/src/test/regress/expected/privileges.out#L1357-L1358)).
A companion block asserts that rebuilding stored expressions that already exist
needs no `USAGE`
([privileges.sql#rebuilds-need-no-usage](../../../../raw/postgres-17/src/test/regress/sql/privileges.sql#L1030-L1038),
[privileges.out#rebuilds-need-no-usage](../../../../raw/postgres-17/src/test/regress/expected/privileges.out#L1417-L1428)).

### The three pg_index state flags

Each flag answers one question for one reader.

| Flag | Question it answers | Who reads it | Effect when false |
|---|---|---|---|
| `indislive` | Is the index alive at all? | The [relcache](../../../glossary.md#relcache), when it builds a table's index list ([relcache.c#RelationGetIndexList-live](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4864-L4871)) | The index is being dropped. It is left out of the index list, so nothing searches it, inserts into it, or counts it for HOT-safety ([catalogs.sgml#indislive](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L4479-L4487)) |
| `indisready` | Must writes insert into the index? | The executor, through `IndexInfo.ii_ReadyForInserts` ([index.c#BuildIndexInfo-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L2434-L2447), [execIndexing.c#skip-not-ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L357-L359)) | `INSERT` and `UPDATE` skip the index ([catalogs.sgml#indisready](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L4468-L4477)) |
| `indisvalid` | May queries use the index? | The [planner](../../../glossary.md#planner), in `get_relation_info` ([plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267)) | The planner ignores the index. Writes still maintain it if it is ready ([catalogs.sgml#indisvalid](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L4443-L4454)) |

The header comments say the same in three lines
([pg_index.h#state-flags](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)).

One combination matters most for CIC: **live but not ready**. Such an index
receives no inserts, yet it still limits HOT. The relcache computes the set of
HOT-blocking columns from every index in the index list, "even if they are not
indisready or indisvalid", precisely because an index that CIC has just started
must count
([relcache.c#RelationGetIndexAttrBitmap-all-indexes](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5325-L5334),
[README.HOT#CIC-not-ready-entry](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L368-L376)).

A plain `CREATE INDEX` on an ordinary table creates the row with all three
flags true. CIC creates
it with `indisvalid = false` and `indisready = false`: `index_create` passes
`!concurrent && !invalid` and `!concurrent` to `UpdateIndexRelation`, which
always writes `indislive = true`
([index.c#index_create-state](../../../../raw/postgres-17/src/backend/catalog/index.c#L1042-L1057),
[index.c#UpdateIndexRelation-flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L632-L659)).
CIC then turns the other two on, one commit apart:

| Event | State before | State after | Effect on other sessions |
|---|---|---|---|
| Commit 1 (`index_create`) | No index | live = t, ready = f, valid = f | The index limits HOT updates. Nobody inserts into it or reads it |
| Commit 2 (`INDEX_CREATE_SET_READY`) | live = t, ready = f, valid = f | live = t, ready = t, valid = f | New writes insert into it, and a unique index starts rejecting duplicates. Nobody reads it |
| Commit 3 | unchanged | unchanged | None. Validation changes the index's contents, not its flags |
| Commit 4 (`INDEX_CREATE_SET_VALID`) | live = t, ready = t, valid = f | live = t, ready = t, valid = t | Queries may use it |

`index_set_state_flags` asserts exactly these transitions: it sets `indisready`
only on a live, not-ready, not-valid index, and `indisvalid` only on a live,
ready, not-valid one. It fetches a writable copy of the row and stores it with
`CatalogTupleUpdate`, so the change follows normal commit and abort rules and
other sessions hear about it when the transaction commits
([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550)).

### Step-by-step implementation

The concurrent path of `DefineIndex` runs as four transactions separated by
three `CommitTransactionCommand()` / `StartTransactionCommand()` pairs. Each
wait happens at the start of the next transaction, before that transaction has
taken any snapshot.

#### Transaction 1: catalog entry and session lock

**Input.** The parsed statement and the table, already locked with
`ShareUpdateExclusiveLock` by `ProcessUtilitySlow`
([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480)).

**Decision.** `DefineIndex` computes `concurrent`, runs the checks listed under
[Preconditions and restrictions](#preconditions-and-restrictions), and computes
`safe_index`, which is true only when the index has no expression column and no
predicate
([indexcmds.c#safe_index](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1144-L1146)).

**Output.** Because `concurrent` is set, `DefineIndex` passes
`INDEX_CREATE_CONCURRENT` and `INDEX_CREATE_SKIP_BUILD` to `index_create`
([indexcmds.c#index_create-flags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1181-L1199)).
`index_create` therefore makes the catalog rows, marks the index not ready and
not valid, and skips the build
([index.c#index_create-state](../../../../raw/postgres-17/src/backend/catalog/index.c#L1042-L1057),
[index.c#index_create-skip-build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1256-L1281)).
A non-concurrent command returns at this point; a concurrent one continues
([indexcmds.c#non-concurrent-return](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1572-L1587)).

Before committing, CIC takes the session-level `ShareUpdateExclusiveLock` on
the table. This cannot block, because the transaction already holds the same
lock. The source notes that it takes no session lock on the index, because
nothing can change the index's state while the table lock is held
([indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)).

**Consequence.** Commit 1 makes the empty index visible. Every transaction that
opens the table from now on includes the index in its HOT-safety decisions, so
it no longer makes HOT updates that change the new index's key
([README.HOT#CIC-not-ready-entry](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L368-L376)).

#### Transaction 2: wait 1 and the first build

**Input.** An index that everyone can see but nobody maintains.

**Wait 1.** Transactions that opened the table before commit 1 may still be
running with the old index list. CIC waits for them with
`WaitForLockers(heaplocktag, ShareLock, true)`. It does not need to worry about
transactions that open the table later, because those will see the index. The
source also says why this is a real lock wait instead of a sleep loop: if one
of those transactions is itself blocked on CIC's table, the lock manager can
detect the [deadlock](../../../glossary.md#deadlock) and report it
([indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658)).

**Calculation.** CIC then takes the build snapshot and calls
`index_concurrently_build`
([indexcmds.c#first-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1682)).
That function re-locks the table and the index, switches to the table owner's
identity with a restricted `search_path`, rebuilds the `IndexInfo` that was
lost at commit 1, sets `ii_Concurrent`, and calls `index_build`, which calls
the [access method](../../../glossary.md#access-method)'s build routine
([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557)).

The first heap scan differs from a plain build's scan in one important way. A
plain build reads with `SnapshotAny` and also indexes recently dead rows. A
concurrent build reads with a regular MVCC snapshot and indexes only what is
live in it
([heapam_handler.c#build-scan-snapshot](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1270),
[heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629)).
The source explains why: with concurrent updates, the other method could see
two versions of the same row as valid and report a bogus uniqueness failure.
Skipping recently dead rows is safe because the index will not become valid
until every transaction that could see them is gone
([index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)).
For a row that is part of a HOT chain, the scan computes the key from the live
row version but stores the TID of the chain's root, as every index does
([heapam_handler.c#build-scan-root-tid](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1663-L1702),
[README.HOT#CIC-first-build](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L388-L394)).

**Output.** An index that contains every row the build snapshot saw, and, at
the end of `index_concurrently_build`, `indisready = true`
([index.c#set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1551-L1556)).

**Consequence.** Commit 2 publishes `indisready`. Every transaction that opens
the table afterwards inserts into the index. Rows that were inserted or updated
while the first scan ran are not in the index yet; that is what transaction 3
fixes.

#### Transaction 3: wait 2 and validation

**Input.** An index that is complete as of the build snapshot and maintained by
all new transactions.

**Wait 2.** Transactions that opened the table before commit 2 may still write
without inserting into the index. CIC waits for them with the same
`WaitForLockers` call
([indexcmds.c#wait-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1697-L1705)).

**Calculation.** CIC registers the reference snapshot and calls
`validate_index`
([indexcmds.c#reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1707-L1728)).
Validation has three parts
([index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)):

1. It collects every heap TID the index currently holds. It does this by
   calling the access method's bulk-delete routine with a callback that records
   each TID and never asks for a deletion
   ([index.c#validate_index_callback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3454-L3466)).
   The source says this should be faster than a plain index scan, and that not
   every access method supports a full-index scan.
2. It sorts those TIDs with a
   [tuplesort](../../../glossary.md#tuplesort). The memory this sort may use is
   the subject of
   [How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping).
3. It scans the [heap](../../../glossary.md#heap) with the reference snapshot
   and merges the scan against the sorted TIDs. The scan must read from block
   zero forward to match the sort order. For each visible row it finds the HOT
   chain's root TID and checks whether that TID is in the sorted list. If it is
   missing, the scan evaluates the predicate and the index expressions and
   inserts the row
   ([heapam_handler.c#heapam_index_validate_scan](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1747-L1986)).

For a unique index, the inserts in part 3 ask the access method to check
uniqueness. The row being inserted may already be dead or about to be deleted,
so the access method must recheck that the row is live before it reports a
violation
([heapam_handler.c#validate-insert](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1951-L1971),
[index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)).

Two groups of rows need no work here, and the source says why. Rows committed
after the reference snapshot were inserted into the index by their own
transaction, because of wait 2. Rows that were already dead to the reference
snapshot need no entry, because wait 3 will outlast everyone who could still
see them
([index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)).

**Output.** An index that contains every row visible to the reference snapshot,
and `limitXmin`, the snapshot's `xmin`. CIC copies `limitXmin` and then drops
the snapshot. It must drop it before waiting, or it would deadlock against
another CIC that sees this snapshot as one it must wait for. Even then, other
registered snapshots or the catalog snapshot could keep this backend's `xmin`
set, so CIC commits and starts a fresh transaction to be sure
([indexcmds.c#drop-reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1730-L1751)).

**Consequence.** After commit 3 the index is complete for every transaction
whose snapshot is at least as new as the reference snapshot. It is not yet
safe for older snapshots.

#### Transaction 4: wait 3 and marking the index valid

**Input.** `limitXmin` and a backend that advertises no `xmin`; an assertion
checks the latter
([indexcmds.c#wait-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1768)).

**Wait 3.** A transaction with a snapshot older than the reference snapshot
may still see a row that was deleted just before the reference snapshot. That
row is not in the index, so such a transaction must not use the index.
`WaitForOlderSnapshots(limitXmin, true)` waits until no such transaction is
left. It asks `GetCurrentVirtualXIDs` for the backends in the current database
whose `xmin` is set and is at or before `limitXmin`
([indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497),
[procarray.c#GetCurrentVirtualXIDs](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L3296-L3378)).

Three kinds of backend are left out of that list, because missing index entries
cannot hurt them: autovacuum workers, backends running a manual lazy
[`VACUUM`](../../../glossary.md#vacuum), and backends that carry
[`PROC_IN_SAFE_IC`](../../../glossary.md#proc_in_safe_ic), that is, other
concurrent builds of plain indexes. A manual `ANALYZE` is not left out, because
its transaction might do arbitrary work later. Before each individual wait the
function looks the list up again and drops every transaction that no longer
appears, so it does not wait for a session that has gone idle with no snapshot
([indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497)).

**Output.** CIC sets `indisvalid = true`. The catalog update makes every
backend rebuild its relcache entry for the index itself. CIC also sends a
relcache invalidation for the table, which forces cached plans to be replanned
so that existing sessions start considering the index
([indexcmds.c#mark-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1770-L1783)).
Finally it releases the session-level lock
([indexcmds.c#release-session-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1785-L1792)).

**Consequence.** `DefineIndex` returns with transaction 4 still open. The
statement's normal end-of-command commit ends it and publishes `indisvalid`
([postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821)).
The source does not set `indcheckxmin` on this path, because wait 3 has already
outlasted every transaction that would have to avoid the index
([index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3071-L3124),
[README.HOT#CIC-no-indcheckxmin](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L410-L414)).
The reference page words this differently; see
[Open Questions](#open-questions).

#### Key interaction: two kinds of waiting

Waits 1 and 2 use the lock table, while wait 3 uses snapshots. Because these
are derived from different shared state, the same blocker can stop one kind of
wait and not the other:

| Blocker | Stops waits 1 and 2? | Stops wait 3? | Why |
|---|---|---|---|
| A transaction that wrote to the table and is still open | Yes | Only if it also holds a snapshot with `xmin` at or before `limitXmin` | Waits 1 and 2 look for holders of locks on this table. Wait 3 looks only at `xmin` |
| A long query on another table in the same database | No | Yes, if its snapshot is old enough | It holds no lock on this table, but its snapshot could see rows validation skipped |
| A session idle in a transaction that holds no snapshot and no lock on the table | No | No | `GetCurrentVirtualXIDs` is called with `excludeXmin0 = true`, and the session is not a lock holder |
| A [prepared transaction](../../../glossary.md#two-phase-commit) that wrote to the table | Yes | No | `WaitForLockersMultiple` notes that prepared transactions are reported and awaited ([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973)). Its stand-in process entry carries no `xmin` ([twophase.c:462](../../../../raw/postgres-17/src/backend/access/transam/twophase.c#L462)) |
| Another CIC or `REINDEX CONCURRENTLY` on a plain index of another table | No | No | It carries `PROC_IN_SAFE_IC`, which wait 3 excludes |
| Another CIC on a partial or expression index of another table | No | Yes, while it holds a snapshot | It does not carry the flag, because its expressions could read other tables ([indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4505)) |

The flag matters in one direction only. It is the **other** backend's flag that
lets CIC skip it; CIC's own index may be partial or expressional and CIC still
skips flagged backends. The flag also changes nothing for waits 1 and 2,
because those never look at status flags
([indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497),
[lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973)).

`set_indexsafe_procflags` is called at the start of transactions 2, 3 and 4,
never in transaction 1. It must be repeated because the flag is one of the
status flags that are reset at transaction end, and it may only be set before
the backend has an XID or an `xmin`
([indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4505),
[proc.h#statusFlags](../../../../raw/postgres-17/src/include/storage/proc.h#L54-L78)).
Parallel build workers take the flag over from the leader when they install
its `xmin` in `ProcArrayInstallRestoredXmin`
([procarray.c#restored-xmin-flags](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2644-L2651)).

### All steps and locks required on the table

CIC uses one [lock mode](../../../glossary.md#lock-mode) on the table, held in
two ways.

1. A **transaction-level** `ShareUpdateExclusiveLock`. `ProcessUtilitySlow`
   acquires it first, through `RangeVarGetRelidExtended`, and this is the only
   acquisition that can block. Its comment says the lock mode must match what
   `DefineIndex` uses, to avoid a lock upgrade
   ([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480)).
   `DefineIndex`, `index_concurrently_build` and `validate_index` each take the
   same lock again with `table_open`; those calls cannot block, because a lock
   the backend already holds is granted locally
   ([lock.c#already-held](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L876-L889),
   [indexcmds.c#lock-mode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L663-L679),
   [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557),
   [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)).
   Each commit releases the transaction-level lock.
2. A **session-level** `ShareUpdateExclusiveLock`, taken just before commit 1
   and released just before the statement ends. It bridges the gaps between
   the transactions
   ([indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619),
   [indexcmds.c#release-session-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1785-L1792)).

`ShareUpdateExclusiveLock` is the mode that `VACUUM` (without `FULL`),
`ANALYZE` and CIC share
([lockdefs.h#lock-modes](../../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)).
Its row in the conflict table decides what runs alongside the build
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L59-L104)):

| Other lock (typical command) | Conflicts with CIC's lock? |
|---|---|
| `AccessShareLock` (`SELECT`) | No. Reads continue |
| `RowShareLock` (`SELECT ... FOR UPDATE/SHARE`) | No |
| `RowExclusiveLock` (`INSERT`, `UPDATE`, `DELETE`) | No. **Writes continue** |
| `ShareUpdateExclusiveLock` (another CIC, `VACUUM`, `ANALYZE`) | **Yes.** The mode conflicts with itself, so one such command runs at a time |
| `ShareLock` (plain `CREATE INDEX`) | **Yes** |
| `ShareRowExclusiveLock`, `ExclusiveLock`, `AccessExclusiveLock` (most `ALTER TABLE` forms, `DROP TABLE`) | **Yes.** Schema changes wait |

Because the mode conflicts with itself and CIC holds it for the whole command,
no `VACUUM` or `ANALYZE` of the table can run until CIC finishes
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L59-L104),
[lockdefs.h#lock-modes](../../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)).
In the other direction, if CIC's first lock request queues behind an
[autovacuum](../../../glossary.md#autovacuum) worker that is not protecting
against wraparound, CIC cancels that worker once `deadlock_timeout` has passed
([proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493));
see [GUCs that affect CIC performance](#gucs-that-affect-cic-performance).

**The three waits are not table locks.** `WaitForLockers` calls
`WaitForLockersMultiple`, which asks `GetLockConflicts` for the transactions
that currently hold a lock conflicting with the given mode, and then waits on
each one's VXID. Its comment is explicit: it does not try to acquire the lock
itself, and it does not wait for anyone who takes a conflicting lock after the
list was made
([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973)).
`GetLockConflicts` reports holders only, never transactions that are merely
waiting for such a lock, and never the caller itself
([lock.c#GetLockConflicts-contract](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L2884-L2902)).

Waits 1 and 2 pass `ShareLock`. `ShareLock` conflicts with `RowExclusiveLock`
and with the four stronger modes plus `ShareUpdateExclusiveLock`
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L59-L104)).
CIC's own lock already keeps every other session out of all of those except
`RowExclusiveLock`. Therefore the holders these waits find are the
transactions that have the table open for writing.

Each individual wait is a real lock acquisition: `VirtualXactLock(vxid, true)`
takes a `ShareLock` on the other transaction's VXID, which that transaction
holds exclusively until it ends, and then releases it at once
([lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663)).

The complete timeline, with the locks on the index as well:

| Step | Locks on the table | Locks on the new index | Wait | Catalog effect |
|---|---|---|---|---|
| Statement start | Transaction-level `ShareUpdateExclusiveLock` ([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480)) | none; it does not exist yet | Queues only if another session holds a conflicting lock | none |
| Transaction 1 | Transaction-level lock; session-level lock added before the commit ([indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)) | `AccessExclusiveLock`, held to commit; nobody else can see the index yet ([index.c#new-index-lock](../../../../raw/postgres-17/src/backend/catalog/index.c#L1001-L1006)) | none | Row created: live, not ready, not valid |
| Commit 1 | Session-level lock only | none | none | The empty index becomes visible |
| Transaction 2 | Session-level lock plus transaction-level lock ([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557)) | `RowExclusiveLock` | Wait 1, then the first scan and build | `indisready = true` at the end |
| Commit 2 | Session-level lock only | none | none | `indisready` becomes visible |
| Transaction 3 | Session-level lock plus transaction-level lock ([index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)) | `RowExclusiveLock` | Wait 2, then the index scan, the sort and the second heap scan | none; missing rows are inserted |
| Commit 3 | Session-level lock only | none | none | none |
| Transaction 4 | Session-level lock, released at the end ([indexcmds.c#release-session-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1785-L1792)) | none | Wait 3 | `indisvalid = true`, visible at the statement's commit |

Parallel build workers open the table and the index with the same modes as
the leader in transaction 2, `ShareUpdateExclusiveLock` and `RowExclusiveLock`
([nbtsort.c#worker-lock-modes](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1778-L1792)).

So the table never carries a lock stronger than `ShareUpdateExclusiveLock`
because of CIC, and `INSERT`, `UPDATE` and `DELETE` conflict with none of the
locks above.

### Failure handling

A failure leaves different things behind depending on which commit it follows.
The reference page describes only the general case: the command fails, an
"invalid" index stays, queries ignore it, and it "will still consume update
overhead"
([ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667)).
The flags make the cases distinguishable:

| Failure happens in | State before | State after | Effect on later work |
|---|---|---|---|
| Transaction 1 (a check, a lock timeout, a name conflict) | No index | No index. The catalog rows were never committed | Nothing to clean up |
| Transaction 2 (wait 1, or the first build, for example a duplicate key in a unique index) | live, not ready, not valid | The same: an [invalid index](../../../glossary.md#invalid-index) that is also not ready | Writes do not insert into it ([execIndexing.c#skip-not-ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L357-L359)). It still limits HOT updates of its columns ([relcache.c#RelationGetIndexAttrBitmap-all-indexes](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5325-L5334)). Queries ignore it |
| Transaction 3 (wait 2, validation) or transaction 4 (wait 3) | live, ready, not valid | The same: invalid but ready | Every write inserts into it, and a unique index keeps enforcing uniqueness ([ref/create_index.sgml#unique-caveat](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L669-L677)). Queries ignore it ([plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267)) |

Three further facts follow from the mechanism:

- **The session lock does not outlive a failure.** Transaction abort releases
  session-level locks too
  ([proc.c#ProcReleaseLocks](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L794-L821)).
- **A unique index can reject writes before it is usable.** Once commit 2 has
  made the index ready, new transactions insert into it, and those inserts
  check uniqueness
  ([index.c#set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1551-L1556)).
  So other sessions can get violations while the build is still running, and
  an invalid index left by a later failure keeps enforcing uniqueness
  ([ref/create_index.sgml#unique-caveat](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L669-L677)).
- **Errors in index expressions or predicates behave the same way**
  ([ref/create_index.sgml#expression-partial](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L679-L683)).

`psql`'s `\d` marks such an index `INVALID`. The documented recovery is to drop
it and run CIC again, or to rebuild it with `REINDEX INDEX CONCURRENTLY`
([ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667)).
The regression test shows the whole cycle: a unique CIC fails in the first
build with `could not create unique index`, `\d` lists the index as `INVALID`,
and a later `REINDEX TABLE`, after the duplicate rows are gone, repairs it
([create_index.out#failed-unique-build](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1418-L1421),
[create_index.out#invalid-index-repaired](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1448-L1484)).

A timeout or a cancel is a failure like any other. If it arrives after commit 1
it leaves an invalid index behind; the timeouts that can fire are listed under
[GUCs that affect CIC performance](#gucs-that-affect-cic-performance).

### How maintenance_work_mem is used and where increases stop helping

There is **no single `maintenance_work_mem` value at which every CIC stops
getting faster**. That is surprising, because the setting looks like one knob.
The reason is that the command reads
[`maintenance_work_mem`](../../../glossary.md#maintenance_work_mem)
in several separate places, and each place has its own point where more memory
stops helping:

1. In transaction 2, the first build uses the setting to cap the number of
   workers that a
   [parallel index build](../../../glossary.md#parallel-index-build)
   requests. Then the index
   [access method](../../../glossary.md#access-method)
   uses the setting in its own way, or not at all
   ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
   [index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3053)).
2. In transaction 3, validation sorts every
   [TID](../../../glossary.md#tid)
   that it finds in the new index in one serial sort. For a
   [GIN](../../../glossary.md#gin)
   index, validation first empties the
   [pending list](../../../glossary.md#pending-list)
   with the same budget
   ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399),
   [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)).

An increase helps only while it changes one of these things:

- the worker request keeps one more worker, which needs 32 MB per participant
  ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012));
- an access-method threshold moves: the hash sort choice, the GiST buffering
  step, or the point where GIN dumps its collected keys
  ([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166),
  [gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741),
  [gininsert.c#build-memory-test](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L311));
- a sort keeps more tuples in memory, so it needs fewer runs and less merging,
  or fits in memory altogether
  ([tuplesort.c#header-memory-model](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L72)).

Once none of these changes any more, extra memory cannot shorten the rest of
the command: the first
[heap](../../../glossary.md#heap)
scan, the index scan and the second heap scan of validation, the building and
writing of index pages, and wait 1, wait 2 and wait 3
([indexcmds.c#wait-1-to-mark-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1773),
[index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)).

#### Memory values and where they come from

The table lists every value that the rest of this section uses. `M` is short
for the session's `maintenance_work_mem`.

| Value | Meaning | Source | Live/current or stored | Later use |
|---|---|---|---|---|
| `maintenance_work_mem` (M) | Budget in kB that each reader turns into a byte limit or a threshold | [GUC](../../../glossary.md#guc) of `user` [context](../../../glossary.md#guc-context); boot value 65536 kB (64 MB), minimum 64 kB ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)) | Live. Each reader takes the value its own process has when it runs. A parallel worker has the value restored from the leader ([parallel.c#serialize-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L382-L385), [parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1455-L1457)) | Worker cap, first build, validation sort, GIN forced cleanup |
| [`work_mem`](../../../glossary.md#work_mem) | Budget of the [B-tree](../../../glossary.md#b-tree) dead-tuple sort and of an unforced GIN pending-list cleanup | GUC of `user` context; boot value 4096 kB (4 MB) ([guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2458)) | Live | Second B-tree sort ([nbtsort.c#secondary-spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L471)); regular GIN cleanup ([ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)) |
| `max_parallel_maintenance_workers` | Upper bound of the worker request | GUC of `user` context; boot value 2, range 0 to 1024 ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)) | Live | Decisions W1, W3 and W4 of the worker request |
| Table setting `parallel_workers` | A per-table worker count that replaces the size model and the memory cap | Table [storage parameter](../../../glossary.md#storage-parameter); the reloption default `-1` means "not set" ([reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382)); loaded as `rel_parallel_workers` ([plancat.c#rel_parallel_workers](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L204-L205)) | Stored with the table. It changes only when someone sets or resets the storage parameter, for example with `ALTER TABLE` ([ref/create_index.sgml#parallel_workers](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L835-L845)) | Decision W3 |
| Requested workers W (`ii_ParallelWorkers`) | Workers the build asks for; the leader is not counted | Result of `plan_create_index_workers()` ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005), [planner.c#plan_create_index_workers-header-comment](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L6892)) | Calculated once, at the start of the first build | Passed to the B-tree or [BRIN](../../../glossary.md#brin) build as its request ([nbtsort.c#parallel-attempt-and-coordination](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L404), [brin.c#begin-parallel-call](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1153-L1163)) |
| Planned participants (`scantuplesortstates`) | W + 1, because the leader also scans and sorts | Calculated before the launch and copied into the shared build state ([nbtsort.c:1426](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1426), [nbtsort.c:1507](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1507)) | Stored. It does not change when fewer workers start | Divisor of each worker's share ([nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829)) |
| Launched participants (`nparticipanttuplesorts`) | Workers that really started, plus the leader | Set right after `LaunchParallelWorkers()` ([nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594)) | Live. Known only after the launch | Divisor of the leader's own scan share ([nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)); number of runs the leader merges ([nbtsort.c#parallel-attempt-and-coordination](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L404)) |
| `sortmem` | One participant's share of M, in kB | M divided by the planned or by the launched count ([nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829), [nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)) | Calculated once per participant | The budget of that participant's sort |
| `allowedMem`, `availMem` | Byte limit of one sort, and what is left of it | `Max(workMem, 64) × 1024` when the sort starts ([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)); every stored tuple is charged ([tuplesort.c#charge-tuple](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1196-L1198)) | Live counter inside one sort | Decides when the sort switches to tapes ([tuplesort.c#puttuple-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1233-L1294)) |
| `NBuffers` | Number of shared buffers | Set by [`shared_buffers`](../../../glossary.md#shared_buffers); see [GUCs that affect CIC performance](#gucs-that-affect-cic-performance) | Read when a hash build starts | Caps the hash sort threshold ([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)) |
| [`effective_cache_size`](../../../glossary.md#effective_cache_size) | Assumed size of the data caches, in blocks | GUC; see [GUCs that affect CIC performance](#gucs-that-affect-cic-performance) | Read during a GiST build | GiST switch to buffering and `levelStep` ([gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900), [gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741)) |
| GIN `allocatedMemory` | Bytes tracked for GIN's in-memory collection of keys | Added up by the collection's own allocations ([ginbulk.c#ginAllocEntryAccumulator](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L83-L106), [ginbulk.c#posting-list-growth](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L39-L52)) | Live counter | Compared with M after each heap tuple of the build ([gininsert.c#build-memory-test](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L311)), or with the cleanup budget ([ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000)) |
| GIN cleanup `workMemory` | Budget of one pending-list cleanup | Chosen at the start of each cleanup ([ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)) | Calculated per cleanup | Flush test of that cleanup ([ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000)) |
| N | Number of TIDs that `ambulkdelete` reports from the new index | One callback call per reported TID ([index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404), [index.c#validate_index_callback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3454-L3466)) | Live. Produced in transaction 3; it includes entries that other sessions inserted after the index became ready for inserts ([index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)) | Input size of the validation sort |
| `SortTuple` slot | In-memory unit of a sort: a tuple pointer, the first key, a null flag and a tape number | Struct definition ([tuplesort.h#SortTuple](../../../../raw/postgres-17/src/include/utils/tuplesort.h#L117-L153)) | Fixed by the build of the server | N slots are what the validation sort charges |

Two pairs of values need care.

*Planned and launched participants.* They have similar names but different
origins. The planned count is calculated before any worker starts and is stored
in the shared build state. The launched count exists only after the launch. They
differ exactly when fewer workers start than were requested. That matters,
because a worker divides M by the planned count and the leader divides M by the
launched count; see
[Worker-count formula and worked example](#worker-count-formula-and-worked-example).

*The table's `parallel_workers`.* It is a stored value, but storing does not make
it stale: every build reads the value the table has at that moment
([plancat.c#rel_parallel_workers](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L204-L205)).
It surprises only someone who set it earlier for another purpose. The
documentation therefore suggests resetting it after tuning an index build,
because the same setting affects all parallel scans of the table
([ref/create_index.sgml#reset-parallel_workers-tip](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L847-L855)).

#### Logic map: where the setting is read

The first map shows where the four transactions read the setting. `M` is the
session's `maintenance_work_mem` in kB. The keys in brackets are explained, with
their sources, below the map.

```text
Transaction 1   create the catalog entry                  M is not read
Transaction 2   wait 1, then the first build
  [A] index_build(): can the access method build in parallel?
        yes (B-tree, BRIN) -> request W workers (second map; reads M at [W5])
        no                 -> W = 0
  [B] ambuild() of the access method:
        B-tree          [C] sort of all index tuples: M, or shares of M
        hash            [D] M as a size threshold, then M as a sort budget
        GiST            [E] sorted: M / buffered: M limits levelStep / plain: -
        GIN             [F] in-memory collection is dumped when it reaches M
        BRIN            [G] serial: - / parallel: summary sorts, shares of M
        SP-GiST, Bloom  [H] M is not read
      commit of transaction 2
Transaction 3   wait 2, then validation
  [I] validate_index() starts one serial sort of TIDs with M
  [J] ambulkdelete() reports the index's TIDs into that sort
        GIN first empties its pending list with M; BRIN reports nothing
  [K] sort, then merge against the second heap scan; insert missing tuples
  [L] the sort is ended; commit of transaction 3
Transaction 4   wait 3, mark the index valid               M is not read
```

- **[A]** `index_build()` asks for workers only when the access method sets
  `amcanbuildparallel`. B-tree and BRIN set it
  ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
  [nbtree.c:120](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L120),
  [brin.c:266](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L266)).
- **[B]** The build itself is the access method's `ambuild` callback
  ([index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3053)).
- **[C]** B-tree: the serial or leader sort and the participants' shares
  ([nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431),
  [nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829),
  [nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)).
- **[D]** Hash: the threshold and the sort
  ([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166),
  [hashsort.c#_h_spoolinit](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L56-L93)).
- **[E]** GiST: the choice of the method, the sorted build and the `levelStep`
  loop of the buffered build
  ([gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248),
  [gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338),
  [gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741)).
- **[F]** GIN: the test after each heap tuple
  ([gininsert.c#build-memory-test](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L311)).
- **[G]** BRIN: the parallel and the serial branch
  ([brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245)).
- **[H]** SP-GiST and Bloom: builds without any use of M
  ([spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L148),
  [blinsert.c#blbuild](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L117-L158)).
- **Commit and wait 2** separate the first build from validation
  ([indexcmds.c#build-to-validation](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1728)).
- **[I]** The sort is created
  ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399)).
- **[J]** The index is scanned through `ambulkdelete`
  ([index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404),
  [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602),
  [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)).
- **[K]** and **[L]** The sort is performed, merged with the heap and ended
  ([index.c#validate_index-sort-and-merge](../../../../raw/postgres-17/src/backend/catalog/index.c#L3406-L3434)).
- **Transactions 1 and 4** contain no access-method build and no sort; see
  [Step-by-step implementation](#step-by-step-implementation).

The second map shows how the first build arrives at its worker count and at the
memory share of each participant. This is the only place where the setting
changes how many processes work on the command.

```text
plan_create_index_workers(): how many workers W does the first build request?

[W1] standalone backend, or max_parallel_maintenance_workers = 0 ?
        yes -> W = 0
[W2] temporary table, or an index expression or predicate that is not
     parallel safe ?
        yes -> W = 0   (parallel_workers on the table cannot override this)
[W3] table storage parameter parallel_workers set (not -1) ?
        yes -> W = min(parallel_workers, max_parallel_maintenance_workers)
               stop here: no size test, no memory test
[W4] size model: W = workers for the table's size, at most
     max_parallel_maintenance_workers
[W5] memory cap: while W > 0 and M / (W + 1) < 32768 kB: W = W - 1

Inside the B-tree or BRIN build:
[W6] W = 0                                   -> serial build, sort budget M
[W7] no DSM segment, or no worker launched   -> serial build, sort budget M
[W8] L >= 1 workers launched                 -> L + 1 participants scan and sort
        each worker's share   = M / (W + 1)
        leader's scan share   = M / (L + 1)
        leader's final merge  = M
```

- **[W1]** No parallelism in a standalone backend or with a cap of zero
  ([planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913)).
- **[W2]** The safety test
  ([planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971)).
  A temporary table never reaches this point in a real CIC, because
  `DefineIndex()` has already turned the request into a non-concurrent build
  ([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)).
- **[W3]** The table's setting replaces the rest of the calculation. Its
  reloption default is `-1`, "not set"
  ([planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985),
  [reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382)).
- **[W4]** The size model
  ([planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998),
  [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4187-L4279)).
  Its thresholds belong to `min_parallel_table_scan_size`; see
  [GUCs that affect CIC performance](#gucs-that-affect-cic-performance).
- **[W5]** The memory cap
  ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)).
- **[W6]** The build tries to start workers only when W is above zero; otherwise
  it creates one serial sort with the full M
  ([nbtsort.c#parallel-attempt-and-coordination](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L404),
  [nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431)).
- **[W7]** The two fallbacks, in B-tree
  ([nbtsort.c#no-DSM-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1489-L1497),
  [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594))
  and in BRIN
  ([brin.c#no-DSM-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2435-L2443),
  [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525)).
  "DSM" is
  [dynamic shared memory](../../../glossary.md#dynamic-shared-memory),
  the segment that the leader and its workers share.
- **[W8]** The planned count, the two shares and the leader's merge sort, in
  B-tree
  ([nbtsort.c:1426](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1426),
  [nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829),
  [nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720),
  [nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431))
  and in BRIN
  ([brin.c:2383](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2383),
  [brin.c#worker-share](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2911-L2919),
  [brin.c#_brin_leader_participate_as_worker](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2764-L2783),
  [brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245)).

#### Setting and application scope

Three settings decide the memory paths of this section. All three have `user`
context, so a session can change them with `SET` and needs neither a reload nor
a restart.

| Setting | Context | Boot value | How to apply it for one CIC run |
|---|---|---|---|
| `maintenance_work_mem` | `user` | 65536 kB = 64 MB; minimum 64 kB; maximum `MAX_KILOBYTES` ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474), [guc.h#MAX_KILOBYTES](../../../../raw/postgres-17/src/include/utils/guc.h#L20-L26)) | Session-level `SET`; no reload, no restart |
| `max_parallel_maintenance_workers` | `user` | 2; range 0 to 1024 ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)) | Session-level `SET`; no reload, no restart |
| `work_mem` | `user` | 4096 kB = 4 MB; minimum 64 kB ([guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2458)) | Session-level `SET`; no reload, no restart |

**Only the session form of `SET` works for a CIC.** A `user`-context setting can
normally also be changed for one transaction with `SET LOCAL`. For a CIC that
form is useless, for three reasons. `SET LOCAL` lasts only until the end of the
current transaction
([ref/set.sgml#SET-LOCAL-lifetime](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L54-L61)).
Outside a transaction block it emits a warning and has no effect
([ref/set.sgml#LOCAL](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L109-L120)).
And CIC refuses to run inside a transaction block
([utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463)).
A session-level `SET`, in contrast, stays in force until the session ends once
its transaction has committed
([ref/set.sgml#SET-persists-after-commit](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L45-L52)),
so it survives the three commits inside the command.

**Parallel workers use the leader's value.** A worker is a separate
[background worker](../../../glossary.md#background-worker)
process and does not read the session's settings by itself. The leader
serializes its GUC state into the shared segment, and each worker restores it
before it does any work
([parallel.c#serialize-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L382-L385),
[parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1455-L1457)).
Therefore the session value also reaches the workers of a parallel B-tree or
BRIN build.

**The value is a budget, not a reservation.** Nothing is allocated in advance.
A sort charges each tuple against its byte limit as the tuple arrives
([tuplesort.c#charge-tuple](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1196-L1198)).
GIN compares its counter with the setting only after it has processed a whole
heap tuple, so a GIN build can exceed the nominal value by the entries of one
heap tuple
([gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314)).

**More is not free.** The documentation warns that a value larger than the
memory that is really available drives the machine into swapping
([ref/create_index.sgml#maintenance_work_mem-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L804)).
It also warns against setting the *default* too high, because autovacuum may
allocate up to `autovacuum_max_workers` times this memory
([config.sgml#maintenance_work_mem-autovacuum](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L1942-L1948)).
A default is also what every other session uses for its own maintenance
commands; the documentation assumes that few of them run at the same time
([config.sgml#maintenance_work_mem-usage](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L1929-L1941)).
A session-level value applies to this session and to the workers it launches,
and to nothing else
([ref/set.sgml#current-session-only](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L32-L43)).

#### How a tuplesort uses its budget

Most readers of the setting hand it to a
[tuplesort](../../../glossary.md#tuplesort),
PostgreSQL's general sort module. Four rules of that module explain every
"stops helping" point in the rest of this section.

1. **Budget.** The caller passes a budget in kB. The sort turns it into a byte
   limit and never uses less than 64 kB:
   `allowedMem = Max(workMem, 64) × 1024`. The source gives the reason for the
   floor: a parallel caller may divide its memory among many workers and leave
   each with very little
   ([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)).
2. **Loading.** Each incoming tuple is charged against the limit. While the
   tuples fit, they stay in an array in memory
   ([tuplesort.c#charge-tuple](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1196-L1198),
   [tuplesort.c#puttuple-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1233-L1294)).
3. **The input fits.** If the input ends before the limit is exceeded, a serial
   sort runs one quicksort in memory and writes nothing to disk
   ([tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434),
   [tuplesort.c#header-memory-model](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L72)).
4. **The input does not fit.** When the limit is exceeded, the sort switches to
   "tapes", which are
   [temporary files](../../../glossary.md#temporary-file).
   It sorts what is in memory, writes it out as one sorted *run*, and repeats
   that for the rest of the input. At the end it merges the runs
   ([tuplesort.c#puttuple-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1233-L1294),
   [tuplesort.c#performsort-TSS_BUILDRUNS](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1451-L1465),
   [tuplesort.c#header-memory-model](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L72)).

In case 4 more memory still helps, in three ways. Runs get longer, so there are
fewer of them. The number of runs merged at once grows with the budget, between
6 and 500
([tuplesort.c#tuplesort_merge_order](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1797-L1847),
[tuplesort.c#tape-constants](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L165-L180)).
And each input tape gets a larger read-ahead buffer
([tuplesort.c#header-memory-model](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L72)).
In case 3 more memory changes nothing for that sort: it already runs as one
in-memory quicksort.

One ceiling is independent of the budget. The in-memory array of a sort holds at
most `INT_MAX` tuples
([tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)).
An input larger than that can never be sorted in memory in one piece.

A parallel build uses the same module in three roles. The caller selects the
role with the "coordinate" argument
([tuplesort.c#coordinate-roles](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L717-L742)).

| Role of the sort | Who creates it | What `tuplesort_performsort()` does | Where more memory stops helping |
|---|---|---|---|
| Serial (no coordinate) | Serial B-tree, hash and sorted GiST builds, and the validation sort ([nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431), [hashsort.c#_h_spoolinit](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L56-L93), [gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399)) | Input fit: one in-memory quicksort. Otherwise: writes the last run, then merges ([tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434), [tuplesort.c#performsort-TSS_BUILDRUNS](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1451-L1465)) | When the whole input fits in memory |
| Worker | Every participant that scans the table in a parallel build, including the leader's own scan ([nbtsort.c#_bt_parallel_scan_and_sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1849-L1963), [nbtsort.c#_bt_leader_participate_as_worker](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1683-L1734)) | Always writes its tuples to a tape as exactly one run, even when everything fit in memory and even when it received no tuple ([tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434), [tuplesort.c#worker_nomergeruns](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L3078-L3093), [tuplesort.c#dumptuples-empty-run-rule](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L2352-L2361), [tuplesort.c#header-parallel-sorts](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L82-L88)) | When each participant's input fits its share, so that no participant has to merge several runs of its own ([tuplesort.c#performsort-TSS_BUILDRUNS](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1451-L1465)) |
| Leader | The backend that started the parallel build ([nbtsort.c#parallel-attempt-and-coordination](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L404)) | Takes over one tape per participant and merges them ([tuplesort.c#leader_takeover_tapes](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L3095-L3160)) | Does not load tuples; it needs only merge buffers |

Therefore temporary-file activity during a parallel build does not show, by
itself, that more memory would help: every participant writes one run in any
case.

#### Transaction 2: the first build, by access method

`index_concurrently_build()` opens the table and the index and calls
`index_build()`
([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557)).
`index_build()` first decides the worker request, as in the second map, and
then calls the access method's `ambuild` callback
([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
[index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3053)).
The callback receives the table, the index and an `IndexInfo` struct. It has no
memory argument
([amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105)),
so each shipped access method that wants the budget reads the global variable
itself
([miscadmin.h:269](../../../../raw/postgres-17/src/include/miscadmin.h#L269)).
The shipped access methods differ along the same two questions: what do they do
with M, and where does that gain end?

| Index AM | What the first build does with M | Where that gain stops |
|---|---|---|
| B-tree | Sorts every index tuple before it builds pages. Serial: one sort with M. Parallel: each participant sorts its part with a share of M, and the leader merges ([nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431), [nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829), [nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)) | When the request has every worker it can use and no sort produces more runs than it must. The validation sort is a separate question |
| [Hash](../../../glossary.md#hash-index) | First uses M as a size threshold that decides whether to sort at all. A chosen sort is serial with M ([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166), [hashsort.c#_h_spoolinit](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L56-L93)) | Not monotonic: a larger M can switch the sort off. With the sort on, like any serial sort |
| [GiST](../../../glossary.md#gist) | Sorted build: one serial sort with M. Buffered build: M is one of two limits on `levelStep`. Plain inserts: M is not read ([gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338), [gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741)) | Sorted: like any serial sort. Buffered: when `effective_cache_size`, not M, stops the next `levelStep`. Plain inserts: no gain |
| GIN | Collects keys and their TID lists in memory and dumps them into the index when the collection reaches M ([gininsert.c#build-memory-test](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L311), [gininsert.c#final-dump](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L386-L397)) | When the whole input reaches the single final dump, or earlier, when dumping no longer dominates the build |
| [SP-GiST](../../../glossary.md#sp-gist) | Does not read M. It inserts tuple by tuple and resets a per-tuple [memory context](../../../glossary.md#memory-context) ([spginsert.c#spgistBuildCallback](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L39-L67), [spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L148)) | No gain in transaction 2 |
| BRIN | Serial: does not read M. Parallel: each participant sorts its range summaries with a share of M, and the leader merges with M ([brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245), [brin.c#_brin_parallel_scan_and_build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2785-L2847)) | When the worker count and the summary sorts no longer change |
| Bloom ([contrib](../../../glossary.md#contrib)) | Does not read M. It fills one cached page at a time and resets a per-tuple memory context ([blinsert.c#bloomBuildCallback](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L70-L115), [blinsert.c#blbuild](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L117-L158)) | No gain in transaction 2 |

The details behind each row follow.

**B-tree.** A serial build creates one sort for all index tuples and gives it
the full M
([nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431)).
A parallel build splits the heap scan among the participants. Each participant
sorts what it scanned with its share and writes one run. The leader's sort then
merges these runs
([nbtsort.c#parallel-attempt-and-coordination](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L404),
[nbtsort.c#_bt_parallel_scan_and_sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1849-L1963),
[tuplesort.c#leader_takeover_tapes](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L3095-L3160)).
The leader's merge sort is created with the full M. That does not double the
memory, and the source explains why: the leader's sort allocates only a small
fixed amount until its `tuplesort_performsort()` runs, and by then the workers
have freed almost all of theirs
([nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431)).

A unique B-tree index adds a second sort, called spool2. It receives
[dead tuples](../../../glossary.md#dead-tuple),
so that they stay out of the uniqueness check. Its budget is `work_mem`, or, in
a parallel participant, the smaller of `work_mem` and that participant's share
([nbtsort.c#secondary-spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L471),
[nbtsort.c#worker-spool2](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1886-L1908)).
In a CIC this second sort stays empty:

- *Default.* A plain `CREATE UNIQUE INDEX` scans the table with `SnapshotAny`
  and judges each tuple itself, because it must also index recently dead
  tuples. Those reach the callback marked as not alive and go to spool2
  ([heapam_handler.c#build-scan-snapshot-serial-and-parallel](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1283),
  [heapam_handler.c#recently-dead-case](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1431-L1457),
  [nbtsort.c#_bt_build_callback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L573-L600)).
- *Cause.* A CIC builds with `ii_Concurrent` set, so both the serial and the
  parallel scan use an
  [MVCC](../../../glossary.md#mvcc)
  [snapshot](../../../glossary.md#snapshot)
  ([heapam_handler.c#build-scan-snapshot-serial-and-parallel](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1283),
  [nbtsort.c#concurrent-snapshot](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1428-L1438)).
- *Mechanism.* Under an MVCC snapshot the scan has already filtered the tuples,
  and the code marks every tuple that reaches the callback as alive
  ([heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629)).
  The callback therefore sends every tuple to the first sort
  ([nbtsort.c#_bt_build_callback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L573-L600)).
  No participant reports a dead tuple, and the leader destroys the unused
  spool2
  ([nbtsort.c#spool2-discard](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L500-L506)).

So `work_mem` does not influence the B-tree build of a CIC.

**Hash.** The build first decides whether to sort at all:

```text
num_buckets = initial number of buckets, from the estimated row count
threshold   = M * 1024 / BLCKSZ                 (a number of blocks)
threshold   = min(threshold, NBuffers)          index is not temporary
              min(threshold, NLocBuffer)        temporary index: never a real CIC
num_buckets >= threshold ?
   yes -> collect all tuples, sort them by bucket with budget M,
          then insert them in bucket order
   no  -> insert each tuple as the heap scan delivers it; M is not used again
```

The inputs and the test are in `hashbuild()`
([hash.c#initial-buckets](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L133-L137),
[hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)),
the sort is a serial tuplesort with the full M
([hashsort.c#_h_spoolinit](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L56-L93),
[hash.c#sort-and-insert](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L179-L184)),
and the temporary case cannot be a real CIC
([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)).
[`BLCKSZ`](../../../glossary.md#blcksz) is the size of one [page](../../../glossary.md#page).

The threshold and the sort use M for different purposes, and that produces a
surprise. The threshold treats M as a size: "is the initial index larger than
M?" The sort treats M as its budget. Because of the first use, raising M can
lift the threshold above the bucket count. The build then stops sorting and
inserts in scan order, which is the case the source warns about: without
locality of access, an index that is bigger than the available memory thrashes
([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)).
Therefore a larger M is not always faster for a hash index. The threshold stops
rising with M once `NBuffers` is the smaller term. If the bucket count still
reaches that cap, the sort stays selected, and more memory can still reduce its
runs and its merge work.

**GiST.** A GiST build chooses among three
[build methods](../../../glossary.md#gist-build-method):

```text
[G1] index storage parameter buffering = on ?
        yes -> go to [G3]; the sorted build is not considered
[G2] does every key column's operator class have a sortsupport function ?
        yes -> sorted build: one serial sort with budget M
[G3] insert tuple by tuple into an initially empty index; M is not read yet
        buffering = off  -> stay with plain inserts
        buffering = auto -> every 256 tuples: is the index larger than
                            effective_cache_size ? if so, try [G4]
        buffering = on   -> once 4096 tuples are in, try [G4]
[G4] gistInitBuffering(): find the largest levelStep for which
        pages of one subtree       <= effective_cache_size / 4   and
        pages on its lowest level  <= M * 1024 / BLCKSZ
     levelStep >= 1 -> buffered build for the rest of the scan
     levelStep  = 0 -> buffering is disabled; plain inserts continue
```

- **[G1]** The storage parameter is mapped to a build mode
  ([gistbuild.c#buffering-mode-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L226)).
- **[G2]** The sortsupport test and the sorted build
  ([gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248),
  [gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338)).
  An
  [operator class](../../../glossary.md#operator-class)
  with a sortsupport function is what the documentation names as the condition
  for the sorted method
  ([gist.sgml#sorted-method](../../../../raw/postgres-17/doc/src/sgml/gist.sgml#L1200-L1205)).
- **[G3]** The switch test and its two tuple counts
  ([gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900),
  [gistbuild.c#BUFFERING_MODE_SWITCH_CHECK_STEP](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L51-L52),
  [gistbuild.c#BUFFERING_MODE_TUPLE_SIZE_STATS_TARGET](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L54-L60)).
- **[G4]** The rationale, the loop, and the two outcomes
  ([gistbuild.c#levelStep-rationale](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L716),
  [gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741),
  [gistbuild.c#buffering-refused](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L749-L758),
  [gistbuild.c#buffering-started](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L773-L776)).

The `buffering` branch at [G1] departs from the default:

- *Default.* `buffering` is `auto`
  ([reloptions.c#buffering](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L521-L531),
  [ref/create_index.sgml#buffering](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L483-L502)).
  The sorted build is used when it is possible. Otherwise the build inserts
  tuple by tuple and switches to the buffered method when the index outgrows
  `effective_cache_size`
  ([gist.sgml#default-build-choice](../../../../raw/postgres-17/doc/src/sgml/gist.sgml#L1226-L1234)).
- *Cause.* The index was created with the storage parameter `buffering = on`.
- *Mechanism.* `gistbuild()` maps `on` to a mode for which it skips the
  sortsupport test, so [G2] is never reached
  ([gistbuild.c#buffering-mode-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L226),
  [gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248)).
  `buffering = off` does not have that effect: a possible sorted build is still
  used
  ([gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248),
  [ref/create_index.sgml#buffering](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L483-L502)).

In the buffered method M is a limit, not a budget for one allocation. A higher
`levelStep` means fewer buffer-emptying steps. `levelStep` grows until either a
subtree no longer fits a quarter of `effective_cache_size` or its lowest level
needs more pages than M holds
([gistbuild.c#levelStep-rationale](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L716),
[gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741)).
More M therefore helps only until `effective_cache_size` is the limit that
stops the loop. The calculation ignores the memory of the buffers' hash table,
as the source notes
([gistbuild.c#levelStep-rationale](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L716)).
A buffered build also creates its own temporary file and calls the opclass's
`penalty` function more often, which costs CPU, so its elapsed time need not
improve steadily with M
([gistbuildbuffers.c#temporary-file](../../../../raw/postgres-17/src/backend/access/gist/gistbuildbuffers.c#L53-L57),
[gist.sgml#buffered-method-costs](../../../../raw/postgres-17/doc/src/sgml/gist.sgml#L1216-L1224)).

**GIN.** The build collects keys and their TID lists in an in-memory red-black
tree
([ginbulk.c#ginInitBA](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L108-L121)).
After every heap tuple it compares the tracked size with M and, when the size
has reached M, writes the whole collection into the index and starts an empty
one. One final dump handles what is left when the scan ends
([gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314),
[gininsert.c#final-dump](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L386-L397)).
A larger M means fewer dumps, and the documentation calls GIN build time very
sensitive to the setting
([gin.sgml#maintenance_work_mem](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L584-L593)).
The gain ends when the whole input reaches the single final dump. How much
memory that takes depends on the extracted keys and the length of their TID
lists, not simply on the size of the heap or of the finished index, because one
row usually contributes many keys
([gin/README#inverted-index-structure](../../../../raw/postgres-17/src/backend/access/gin/README#L17-L26)).

**BRIN.** A BRIN index is a
[summarizing index](../../../glossary.md#summarizing-index):
it keeps summaries of block ranges and stores no entry per heap tuple
([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)).
A serial build scans the table once and writes one summary after another; it
creates no sort and does not read M. A parallel build lets each participant
build summaries for the blocks it scanned, sort them with its share, and hand
them to the leader, whose merge sort has the full M
([brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245),
[brin.c#_brin_parallel_scan_and_build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2785-L2847),
[brin.c#worker-share](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2911-L2919),
[brin.c#_brin_leader_participate_as_worker](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2764-L2783)).
The worker request is the same one B-tree uses, including the 32 MB rule. A
source comment notes that this rule suits B-tree but is stricter than BRIN
needs
([brin.c#begin-parallel-call](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1153-L1163)).

**SP-GiST and Bloom.** Neither build reads M, so transaction 2 gains nothing
from it
([spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L148),
[blinsert.c#blbuild](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L117-L158)).
Both indexes still benefit in transaction 3, because their `ambulkdelete`
reports TIDs to the validation sort; see
[Transaction 3](#transaction-3-the-validation-sort-and-gin-cleanup) below.

#### Worker-count formula and worked example

**Terms.** `M` is `maintenance_work_mem` in kB. `W` is the number of workers
that step W4 proposed; the leader is not counted. `32768` is 32 MB in kB. The
division is C integer division, which drops the remainder.

**Formula.** Step W5 is this loop
([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)):

```text
while W > 0 and M / (W + 1) < 32768:
    W = W - 1
```

**Meaning.** `W + 1` is the number of participants, because the leader sorts
too. The loop removes workers until an even share of M is at least 32 MB for
every participant. So W workers survive only if `M >= 32768 * (W + 1)`. The
documentation says the same in words: each worker needs a 32 MB share, and a
further 32 MB share must remain for the leader
([ref/create_index.sgml#32MB-shares](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L819-L833)).

**Worked example (derived from the source, not measured).**

| `maintenance_work_mem` | M in kB | Test for one more worker | Largest W the cap allows |
|---|---|---|---|
| 32 MB | 32768 | 32768 / 2 = 16384, below 32768 | 0 |
| 64 MB (boot value) | 65536 | 65536 / 2 = 32768 passes; 65536 / 3 = 21845 fails | 1 |
| 96 MB | 98304 | 98304 / 3 = 32768 passes; 98304 / 4 = 24576 fails | 2 |
| 128 MB | 131072 | 131072 / 4 = 32768 passes; 131072 / 5 = 26214 fails | 3 |
| 160 MB | 163840 | 163840 / 5 = 32768 passes; 163840 / 6 = 27306 fails | 4 |

The table shows two things. First, with both boot values, 64 MB and a cap of
two workers
([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474),
[guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)),
a build requests at most one worker: the size model may propose two, and the
memory cap removes one. Reaching the two workers that the default cap allows
needs 96 MB. Second, 96 MB is then the last value at which memory changes the
worker count. It is not the plateau of the command, because the first-build
sorts and the validation sort can still spill above 96 MB.

**Shares.** Once workers are launched, three formulas divide M. `L` is the
number of workers that really started.

```text
each worker's share   = M / (W + 1)     planned participants
leader's scan share   = M / (L + 1)     launched participants
leader's final merge  = M
```

A worker divides by the planned count that was stored before the launch
([nbtsort.c:1426](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1426),
[nbtsort.c:1507](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1507),
[nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829)).
The leader divides by the launched count, which it knows after the launch
([nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594),
[nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)).
Each share then passes through the 64 kB floor of the sort
([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)).
BRIN uses the same three formulas
([brin.c:2383](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2383),
[brin.c#worker-share](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2911-L2919),
[brin.c#_brin_leader_participate_as_worker](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2764-L2783),
[brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245)).

**Worked example (derived from the source, not measured).** M = 96 MB = 98304
kB, W = 2:

| Workers launched (L) | Each worker's share | Leader's scan share | Sum of the scan shares | Leader's final merge |
|---|---|---|---|---|
| 2 | 98304 / 3 = 32768 kB | 98304 / 3 = 32768 kB | 98304 kB | 98304 kB |
| 1 | 98304 / 3 = 32768 kB | 98304 / 2 = 49152 kB | 81920 kB | 98304 kB |
| 0 | no worker | no parallel build: one serial sort with 98304 kB | 98304 kB | the same sort |

**Key interaction.** A worker uses the planned count, W + 1. That is the count
the memory cap in step W5 tested, and it was stored before the launch. The
leader uses the launched count, L + 1, which it learns only after the launch.
Because the two counts come from different moments, they differ whenever fewer
workers start than were requested. A launched worker then keeps the smaller
planned share while the leader takes a larger one. Therefore part of M stays
unused during the scan: 16384 kB in the second row. A source comment describes
the same effect from the leader's side
([nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)).

Two settings change these formulas, and both depart from the default.

The table's `parallel_workers`:

- *Default.* The storage parameter is not set, so the size model and the memory
  cap decide
  ([reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382),
  [planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998),
  [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)).
- *Cause.* The table has `parallel_workers` set.
- *Mechanism.* Step W3 takes that number, caps it with
  `max_parallel_maintenance_workers` and skips W4 and W5; the source says it
  deliberately considers no other factor, memory included
  ([planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985)).
  A value of 0 disables parallel index builds on the table
  ([ref/create_index.sgml#parallel_workers](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L835-L845)).
  Steps W1 and W2 run before W3, so the setting cannot force workers for a
  parallel-unsafe expression or predicate
  ([planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913),
  [planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971)).
  Because W5 is skipped, shares can be far below 32 MB; only the 64 kB floor of
  the sort remains
  ([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)).

Leader participation:

- *Default.* The leader takes part in the scan and the sorting. The macro that
  would change this is not defined; it exists only inside a comment
  ([nbtsort.c#DISABLE_LEADER_PARTICIPATION](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L69-L73),
  [nbtsort.c#leaderparticipates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1410-L1415)).
- *Cause.* A server compiled with `DISABLE_LEADER_PARTICIPATION` defined, which
  the source describes as a debugging aid.
- *Mechanism.* The flag `leaderparticipates` is then false. The planned count is
  W instead of W + 1, the launched count is L instead of L + 1, and the leader
  only merges
  ([nbtsort.c#leaderparticipates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1410-L1415),
  [nbtsort.c:1426](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1426),
  [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594)).
  BRIN has the same switch
  ([brin.c#leaderparticipates](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2367-L2372)).

**What the budget does not cover.** The documentation and a source comment
describe the setting as the limit for the whole build
([ref/create_index.sgml#parallel-index-build](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L806-L817),
[config.sgml#parallel-utility-memory-note](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2887-L2898),
[nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431)).
The code keeps the main sorts within it, with these additions: every
participant's share is raised to at least 64 kB
([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700));
a unique build adds the `work_mem`-sized second sort in every participant,
which the source allows explicitly
([nbtsort.c#worker-spool2](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1886-L1908));
and the shared state that coordinates the workers lives in the DSM segment,
outside any sort's limit
([nbtsort.c#shared-state-estimate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1440-L1447)).
The difference between the wording and the code is recorded under
[Open Questions](#open-questions).

#### Transaction 3: the validation sort and GIN cleanup

Validation runs after the commit of transaction 2 and after wait 2. It uses M
in one place for every access method and in a second place for GIN only. In
order:

1. **Start the sort.** *Input:* the session's M. `validate_index()` starts a
   sort of `int8` values with the full M and no coordinate, so it is a serial
   sort
   ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399),
   [tuplesort.c#coordinate-roles](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L717-L742)).
   It sorts TIDs encoded as `int8` rather than TID structs, because `int8` is
   passed by value on most platforms and that makes the sort faster
   ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399)).
   *Consequence:* validation never uses parallel workers. The documentation
   states the same: in a CIC only the first table scan runs in parallel
   ([ref/create_index.sgml#CIC-first-scan-only](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L857-L862)).
2. **GIN only: empty the pending list.** Before a GIN index reports any TID, it
   moves its pending list into the main index, with M as the budget. The
   details are below.
3. **Collect the TIDs.** `validate_index()` calls the access method's
   `ambulkdelete` with a callback that never asks for a deletion. The callback
   encodes each reported TID and puts it into the sort
   ([index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404),
   [index.c#validate_index_callback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3454-L3466)).
   *Output:* N items in the sort. What an access method reports decides N; see
   the table below.
4. **Sort.** *Decision:* by the rules of
   [How a tuplesort uses its budget](#how-a-tuplesort-uses-its-budget), the sort
   is one in-memory quicksort if all N items fit, and an external sort
   otherwise
   ([index.c#validate_index-sort-and-merge](../../../../raw/postgres-17/src/backend/catalog/index.c#L3406-L3434),
   [tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434),
   [tuplesort.c#performsort-TSS_BUILDRUNS](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1451-L1465)).
5. **Merge with the heap.** The second heap scan reads the table in block order
   under the reference snapshot and steps through the sorted TIDs alongside it.
   A visible heap tuple whose root TID is missing from the sorted list is
   passed to `index_insert()`
   ([heapam_handler.c#heapam_index_validate_scan](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1747-L1986),
   [heapam_handler.c#validation-merge-loop](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1875-L1909),
   [heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974)).
6. **Release.** `validate_index()` ends the sort before it returns, and the
   transaction commits afterwards
   ([index.c#validate_index-sort-and-merge](../../../../raw/postgres-17/src/backend/catalog/index.c#L3406-L3434),
   [indexcmds.c#wait-1-to-mark-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1773)).

What `ambulkdelete` reports in step 3:

| Index AM | What it reports during validation | Effect on N |
|---|---|---|
| B-tree | The heap TID of every leaf tuple, and each TID inside a [deduplicated](../../../glossary.md#deduplication) posting-list tuple ([nbtree.c#regular-tuple-callback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1245-L1255), [nbtree.c#btreevacuumposting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1396-L1449)) | One item per indexed heap TID |
| Hash | The heap TID of every index tuple ([hash.c#bucket-cleanup-callback](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L739-L748)) | One item per index tuple |
| GiST | The heap TID of every leaf tuple ([gistvacuum.c#callback-loop](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L336-L354)) | One item per index tuple |
| SP-GiST | The heap TID of every live leaf tuple ([spgvacuum.c#live-tuple-callback](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L153-L181), [spgvacuum.c#spgbulkdelete](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L908-L932)) | One item per live leaf tuple |
| GIN | Every TID in every [posting list](../../../glossary.md#posting-list) of an entry page and in every [posting tree](../../../glossary.md#posting-tree) leaf ([ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L457-L569), [gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L734-L864), [ginvacuum.c#ginVacuumItemPointers](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L39-L84), [gin/README#inverted-index-structure](../../../../raw/postgres-17/src/backend/access/gin/README#L17-L26)) | One item per key and TID: a heap row that is indexed under k keys adds k items |
| BRIN | Nothing; the function never calls the callback ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)) | N = 0, the sort stays empty |
| Bloom | The heap TID of every stored tuple ([blvacuum.c#tuple-loop](../../../../raw/postgres-17/contrib/bloom/blvacuum.c#L88-L107)) | One item per index tuple |

Two consequences of step 5 are worth stating. For BRIN the sort is empty, so
*every* visible heap tuple that satisfies the index predicate counts as missing
and is passed to `index_insert()`; M has no influence on that work
([heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974)).
For GIN the inserted tuples take the normal insert path. With `fastupdate` on,
which is its default
([reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)),
they go to the pending list, and a list that outgrows its limit triggers a
regular cleanup that runs with the `work_mem` of this
[backend](../../../glossary.md#backend),
not with M
([gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530),
[ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471),
[ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)).
So the CIC session's own `work_mem` and pending-list limit apply during
validation, as well as those of concurrent writers; the settings are described
under [GUCs that affect CIC performance](#gucs-that-affect-cic-performance).

**The GIN cleanup in step 2.** It departs from the cleanup that GIN runs by
default:

- *Default.* A cleanup that an ordinary insert starts is not forced. It gives
  up if another cleanup holds the lock, it uses `work_mem`, and it stops at the
  page that was the tail of the list when it started
  ([ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471),
  [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828),
  [ginfast.c#old-tail-rule](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L881-L888)).
- *Cause.* Validation calls `ambulkdelete` without a statistics struct, and GIN
  takes a missing struct as the first call of a vacuum pass
  ([index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404),
  [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)).
- *Mechanism.* `ginbulkdelete()` then calls `ginInsertCleanup()` with
  `forceCleanup` true and with `full_clean` set to "this is not an autovacuum
  worker". A CIC session is not an autovacuum worker, so `full_clean` is true
  ([ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)).
  A forced cleanup waits for the lock and takes `maintenance_work_mem` as its
  budget; only an
  [autovacuum](../../../glossary.md#autovacuum)
  worker with `autovacuum_work_mem` set would use that setting instead
  ([ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)).

The forced cleanup then works in batches:

1. If the pending list is empty, it returns before it allocates anything
   ([ginfast.c#empty-list-return](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L835-L841),
   [ginfast.c#cleanup-init](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L859-L870)).
2. Otherwise it reads pending pages into an in-memory collection. It flushes the
   collection into the main index at the end of the list, or earlier when a
   page ends a complete row and the tracked size has reached the budget
   ([ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000)).
3. After a flush it resets its memory context and starts the next batch
   ([ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000)).
4. Because `full_clean` is true, the rule that stops at the old tail does not
   apply. The cleanup continues until the list is empty, including entries that
   other sessions appended while it ran
   ([ginfast.c#old-tail-rule](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L881-L888),
   [ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000)).
5. At the end it deletes its memory context
   ([ginfast.c#cleanup-context-deleted](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1022-L1024)).

The gain from M in this step ends when the pending list is empty or small
enough for one batch. How long the list is depends on what other sessions
appended after the index became ready for inserts.

**When does the validation sort fit in memory?** For this one sort the source
supports a calculation, because the sorted items have a fixed size.

*Terms.* `N` is the number of TIDs from step 3. `S` is the size of one
`SortTuple` slot. `B` is the sort's byte limit, `M × 1024`
([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)).

*Why only the slots count.* The sort is a datum sort of a pass-by-value type.
For such a type the sort keeps the value inside the slot and allocates no
separate tuple
([tuplesortvariants.c#tuplesort_begin_datum](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L583-L661),
[tuplesortvariants.c#tuplesort_putdatum](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L820-L867),
[tuplesort.h#SortTuple](../../../../raw/postgres-17/src/include/utils/tuplesort.h#L117-L153)).
The only memory it charges is the slot array itself
([tuplesort.c#memtuples-charge](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L808-L812),
[tuplesort.c#charge-tuple](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1196-L1198)).
The array grows by doubling and then takes one last, smaller step, so that it
can use the limit almost fully
([tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)).

*Formula.*

```text
the sort stays in memory while   N * S <= B   (approximately)
```

*Meaning.* The validation sort fits when the slots for all reported TIDs fit
the budget. The index's size on disk does not enter the calculation; only the
number of TIDs does.

*Worked example (derived from the source, not measured).* A slot has four
fields: a pointer, a `Datum`, a `bool` and an `int`
([tuplesort.h#SortTuple](../../../../raw/postgres-17/src/include/utils/tuplesort.h#L117-L153)).
On a 64-bit build with 8-byte pointers and 8-byte alignment that is
8 + 8 + 1 + 4 bytes plus 3 bytes of padding, so S = 24 bytes. The source does
not state this number; it follows from the struct layout.

| `maintenance_work_mem` | B in bytes | Largest N that fits, B / 24 |
|---|---|---|
| 64 MB (boot value) | 67,108,864 | about 2.79 million TIDs |
| 1 GB | 1,073,741,824 | about 44.7 million TIDs |

Read the other way round, one million TIDs need 24,000,000 bytes, about 23 MB.

*Limits of the calculation.*

| Condition | Effect on the calculation |
|---|---|
| GIN index | N counts one item per key and TID, so N can be many times the number of heap rows ([ginvacuum.c#ginVacuumItemPointers](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L39-L84)) |
| BRIN index | N = 0; there is nothing to fit ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)) |
| More than `INT_MAX` TIDs | The slot array cannot hold them at any budget, so the sort is external ([tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)). `INT_MAX` slots of 24 bytes are about 48 GB, a budget that only builds with the larger `MAX_KILOBYTES` accept ([guc.h#MAX_KILOBYTES](../../../../raw/postgres-17/src/include/utils/guc.h#L20-L26)) |
| A build where `int8` is passed by reference | Each item also has a separately allocated copy, so the formula does not describe that build ([tuplesortvariants.c#tuplesort_putdatum](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L820-L867), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399)) |
| Concurrent writers | N includes the entries that other sessions inserted after the index became ready, so it can exceed the row count at the first build ([index.c#validate_index-overview](../../../../raw/postgres-17/src/backend/catalog/index.c#L3261-L3323)) |
| Rounding | The formula ignores the allocator's per-chunk overhead and the size of the array's last growth step ([tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)) |

For the first-build sorts no such constant exists. Each B-tree, hash or GiST
tuple is charged its own size in addition to its slot
([tuplesort.c#charge-tuple](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1196-L1198)),
so their fit point depends on the indexed data.

#### Memory branches and exceptional cases

| Condition / state | Behavior | Consequence |
|---|---|---|
| The table is temporary (default: `CONCURRENTLY` is honored) | `DefineIndex()` clears the concurrent flag, because no other backend can access a temporary table ([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)) | An ordinary build: no validation sort. A hash build caps its threshold with `NLocBuffer` instead of `NBuffers` ([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)) |
| Standalone backend, or `max_parallel_maintenance_workers = 0` (boot value 2) | W = 0 before any other test ([planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913), [guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)) | Serial build; the first-build sort has all of M |
| An index expression or predicate is not parallel safe | W = 0, even when the table sets `parallel_workers` ([planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971)) | Serial build |
| The table sets `parallel_workers` (default: not set) | W = min(setting, `max_parallel_maintenance_workers`); no size test and no memory test; 0 disables parallel builds ([planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985), [ref/create_index.sgml#parallel_workers](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L835-L845)) | Shares can be far below 32 MB |
| No DSM segment is available, or no worker starts | The build leaves parallel mode again ([nbtsort.c#no-DSM-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1489-L1497), [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594), [brin.c#no-DSM-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2435-L2443), [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525)) | Serial build with all of M |
| Fewer workers start than were requested | A started worker keeps M / (W + 1); the leader's scan share becomes M / (L + 1) ([nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829), [nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720)) | Part of M is unused during the scan |
| The server was compiled with `DISABLE_LEADER_PARTICIPATION` (default: not defined) | The leader does not scan or sort; the planned count is W ([nbtsort.c#DISABLE_LEADER_PARTICIPATION](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L69-L73), [nbtsort.c#leaderparticipates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1410-L1415), [nbtsort.c:1426](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1426)) | Each worker's share is M / W |
| A share is below 64 kB | The sort raises it to 64 kB ([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)) | The sum of the shares can exceed M |
| Unique B-tree index | A second sort is created with `work_mem`, or with min(share, `work_mem`) in a participant ([nbtsort.c#secondary-spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L471), [nbtsort.c#worker-spool2](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1886-L1908)) | In a CIC it stays empty and is discarded ([heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629), [nbtsort.c#spool2-discard](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L500-L506)) |
| Hash: M is raised until the threshold exceeds the bucket count | The build stops sorting ([hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166)) | Tuples are inserted in scan order, which the source expects to thrash when the index is bigger than the available memory |
| GiST index with `buffering = on` (default `auto`) | The sorted build is skipped; buffering is tried once 4096 tuples are in ([gistbuild.c#buffering-mode-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L226), [gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248), [gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900), [gistbuild.c#BUFFERING_MODE_TUPLE_SIZE_STATS_TARGET](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L54-L60)) | M matters only as a limit on `levelStep` |
| GiST: the cache assumption or M is too small for `levelStep` 1 | Buffering is disabled for the rest of the build ([gistbuild.c#buffering-refused](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L749-L758)) | Plain inserts continue; M is not read again |
| GIN: the TID list of one key holds more than `INT_MAX` slots and must grow | The build or the cleanup fails with "posting list is too long" and the hint to reduce `maintenance_work_mem` ([ginbulk.c#posting-list-growth](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L39-L52)) | A larger M is not always safer for one extremely frequent key |
| GIN: the pending list is empty when validation starts | The forced cleanup returns before it allocates anything ([ginfast.c#empty-list-return](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L835-L841)) | No cleanup memory is used |
| BRIN index | `ambulkdelete` reports no TID, so the validation sort is empty ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)) | Validation passes every visible heap tuple to `index_insert()`; M plays no role ([heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974)) |
| A sort would need more than `INT_MAX` tuples in memory | The in-memory array cannot grow further ([tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)) | That sort is external at any M |
| Any participant of a parallel build | It always writes one run to a temporary file ([tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434)) | Temporary files appear even when memory is ample |
| M is larger than the memory that is really free | The machine swaps ([ref/create_index.sgml#maintenance_work_mem-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L804)) | The build gets slower, not faster |

#### The two memory windows

The first build and validation use memory at different times. The events that
open and close each window:

| Event | State before | State after | Effect on later calculation |
|---|---|---|---|
| Session-level `SET maintenance_work_mem`, before the CIC | Old session value | New session value, which persists across the command's commits ([ref/set.sgml#SET-persists-after-commit](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L45-L52)) | Every reader below uses it |
| Transaction 2: `index_build()` computes W | No request | W is fixed for this build ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005)) | The planned count and the worker shares follow from it |
| Transaction 2: the access method builds | No build memory | Sorts, the GIN collection or GiST buffers hold up to their budgets | None yet |
| Transaction 2: `ambuild` returns | Build memory in use | Released. For example, B-tree ends its sorts and its parallel context, hash and sorted GiST end their sorts, and GIN deletes its memory contexts ([nbtsort.c#btbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L289-L349), [hash.c#sort-and-insert](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L179-L184), [gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338), [gininsert.c#build-contexts-deleted](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L399-L400)) | No first-build memory reaches transaction 3 |
| Commit of transaction 2, then wait 2 | First window closed | Second window not yet open ([indexcmds.c#build-to-validation](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1728)) | The two windows cannot overlap |
| Transaction 3: GIN forced cleanup | The pending list may hold entries | The list is empty and the cleanup's memory context is deleted ([ginfast.c#cleanup-context-deleted](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1022-L1024)) | Those entries are now in the main index and are reported as TIDs |
| Transaction 3: the validation sort is created, filled, sorted and merged | No sort | Up to M in use, or runs on disk ([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399), [index.c#validate_index-sort-and-merge](../../../../raw/postgres-17/src/backend/catalog/index.c#L3406-L3434)) | Decides which heap tuples are missing from the index |
| Transaction 3: `tuplesort_end()` | Sort in use | Released before `validate_index()` returns ([index.c#validate_index-sort-and-merge](../../../../raw/postgres-17/src/backend/catalog/index.c#L3406-L3434)) | Transaction 4 uses no build memory and no sort memory |

`index_concurrently_build()` returns only after `index_build()` and the
access-method callback are complete and the index is marked ready
([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557)).
CIC then commits, performs wait 2, and only after that lets
`validate_index()` create its sort
([indexcmds.c#build-to-validation](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1728)).
Therefore the sort memory of the command at any moment is that of one window,
not the sum of both.

#### How to find the practical plateau

The source gives thresholds, not timings. To find the value at which a
particular build stops improving, keep the data, the index definition, the
concurrent workload, the storage state and the worker availability comparable,
raise the session value in steps, and watch the two stages that use memory.

| What to look at | Where to look | What it shows | Setting it depends on |
|---|---|---|---|
| Which stage takes the time | `pg_stat_progress_create_index.phase` ([system_views.sql#pg_stat_progress_create_index](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1256-L1289), [monitoring.sgml#create-index-phases](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L6018-L6132)) | `building index`, `index validation: scanning index`, `index validation: sorting tuples`, `index validation: scanning table`, and one phase for each of the three waits. A B-tree build appends its sub-phase to `building index`: `scanning table`, `sorting live tuples`, `sorting dead tuples` or `loading tuples in tree` ([monitoring.sgml#building-index-phase](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L6048-L6058), [nbtutils.c#btbuildphasename](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4604-L4625), [nbtree.c:138](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L138)) | [Progress reporting](../../../glossary.md#progress-reporting) needs `track_activities`: `superuser` context, boot value on ([backend_progress.c#pgstat_progress_start_command](../../../../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L20-L40), [backend_progress.c#pgstat_progress_update_param](../../../../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L42-L61), [guc_tables.c#track_activities](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1400-L1410)) |
| How many workers were requested | `DEBUG1` message of `index_build()` ([index.c#index_build-request-message](../../../../raw/postgres-17/src/backend/catalog/index.c#L3007-L3017)) | "building index ... with request for N parallel workers", or "... serially" | `client_min_messages`: `user` context, boot value `notice` ([guc_tables.c#client_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4776-L4785)); a session-level `SET client_min_messages = debug1` shows the message |
| Whether each sort stayed in memory | `trace_sort` ([tuplesort.c#tuplesort_free-trace](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L930-L940), [tuplesort.c#performsort-trace](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1472-L1483)) | One `LOG` line when a sort ends: "internal sort" or "external sort" ("parallel external sort" for a participant), the worker number, and the kB or disk blocks used. A further line reports when each `performsort` is done | `trace_sort`: `user` context, boot value off; a session-level `SET trace_sort = on` enables it. It exists because `TRACE_SORT` is defined by default ([guc_tables.c#trace_sort](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1659-L1670), [pg_config_manual.h#TRACE_SORT](../../../../raw/postgres-17/src/include/pg_config_manual.h#L376-L380), [config.sgml#trace_sort](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L11747-L11761)) |
| How much was spilled | `log_temp_files` ([fd.c#ReportTemporaryFileUsage](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1524-L1539)) | One `LOG` line per temporary file when the file is deleted, with its final size, if the size is at least the setting | `log_temp_files`: `superuser` context, boot value -1, which disables it; 0 logs every file; N logs files of at least N kB ([guc_tables.c#log_temp_files](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3554-L3563), [config.sgml#log_temp_files](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L7957-L7979)) |
| Totals per database | `pg_stat_database.temp_files` and `temp_bytes` ([pgstat_database.c#pgstat_report_tempfile](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_database.c#L171-L185), [monitoring.sgml#temp_files-and-temp_bytes](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L3360-L3382)) | Number and bytes of deleted temporary files from every cause, regardless of `log_temp_files` | `track_counts`: `superuser` context, boot value on ([guc_tables.c#track_counts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1411-L1419)) |
| GiST buffering decision | `DEBUG1` messages of the GiST build ([gistbuild.c#buffering-started](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L773-L776), [gistbuild.c#buffering-refused](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L749-L758)) | "switched to buffered GiST build; level step = ..., pagesPerBuffer = ...", or "failed to switch to buffered GiST build". The first message reports the chosen `levelStep` | `client_min_messages`, as above |
| GIN dumps of the first build | Nothing | No message and no counter reports how often the collection was dumped ([gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314)) | Only a comparison of elapsed times |

How these settings are applied:

- A `user`-context setting is changed with a session-level `SET` and needs no
  reload and no restart.
- A `superuser`-context setting (`track_activities`, `track_counts`,
  `log_temp_files`) is changed the same way, but only by a role that is allowed
  to change it.
- `LOG` messages go to the server log under the boot value `warning` of
  `log_min_messages`, because the server log ranks `LOG` above `ERROR`
  ([elog.c#is_log_level_output](../../../../raw/postgres-17/src/backend/utils/error/elog.c#L196-L228),
  [guc_tables.c#log_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4872-L4881),
  [config.sgml#log_min_messages](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L6992-L7015)).
  They are not sent to the client under the boot value `notice` of
  `client_min_messages`, because for the client `LOG` ranks below `NOTICE`
  ([elog.c#should_output_to_client](../../../../raw/postgres-17/src/backend/utils/error/elog.c#L244-L264),
  [config.sgml#client_min_messages](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8967-L8991)).
  `log_min_messages` has `superuser` context, so only a role that is allowed to
  change it can lower it for a session.

Four facts keep the readings honest:

- The progress view reports blocks, tuples and lockers. It does not report sort
  memory, temporary bytes or the number of participants
  ([monitoring.sgml#pg_stat_progress_create_index-view](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L5847-L6016)).
- With `trace_sort`, every line carries the worker number, so the lines show
  how many participants sorted. Each run that is written is reported with its
  number
  ([tuplesort.c#dumptuples-run-trace](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L2428-L2433)).
  In a parallel build every participant reports run 1, which it must write in
  any case. Only a run 2 or higher, and the merging that follows, indicate a
  share that was too small
  ([tuplesort.c#header-parallel-sorts](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L82-L88),
  [tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434)).
- A parallel sort keeps its tapes in a shared file set
  ([tuplesort.c#tuplesort_initialize_shared](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L2968-L2991)).
  Those files are reported like any other. When the last process detaches, the
  set's directories are removed, and that removal deletes each file through the
  same reporting function
  ([sharedfileset.c#SharedFileSetOnDetach](../../../../raw/postgres-17/src/backend/storage/file/sharedfileset.c#L88-L114),
  [fileset.c#FileSetDeleteAll](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L146-L165),
  [fd.c#PathNameDeleteTemporaryDir](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1687-L1707),
  [fd.c#unlink_if_exists_fname](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L3771-L3786),
  [fd.c#PathNameDeleteTemporaryFile](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1927-L1972)).
- The database counters add up all causes and record file volume, not merge
  passes or waiting time
  ([monitoring.sgml#temp_files-and-temp_bytes](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L3360-L3382)).

Stop raising the value when all of the following stay the same across
comparable runs:

- the number of workers requested and the number of participants that sorted;
- every serial sort is reported as internal;
- parallel participants write only their one required run;
- the temporary-file volume of the run no longer falls;
- the time spent in `index validation: sorting tuples` and the total elapsed
  time no longer improve.

Stop earlier if the host starts swapping
([ref/create_index.sgml#maintenance_work_mem-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L804)).
Even at the plateau the command still performs its scans, builds and writes its
pages, and waits three times
([indexcmds.c#wait-1-to-mark-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1773)).

#### Extension and build boundary

**Extensions.** Core passes no memory budget to an access method: the `ambuild`
callback has no such argument
([amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105)).
The shipped access methods read the exported global variable
([miscadmin.h:269](../../../../raw/postgres-17/src/include/miscadmin.h#L269)),
and a third-party access method can apply a different policy. The validation
sort, however, belongs to core, in `validate_index()`, whatever the access
method is
([index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399)).

The flag `amusemaintenanceworkmem` in the access-method struct sounds related
but does not control `CREATE INDEX`. Parallel
[VACUUM](../../../glossary.md#vacuum)
reads it to count the indexes that use the setting and to divide the workers'
memory by that count
([amapi.h#amusemaintenanceworkmem](../../../../raw/postgres-17/src/include/access/amapi.h#L255-L256),
[vacuumparallel.c#mwm-index-count](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L350-L351),
[vacuumparallel.c#mwm-worker-share](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L372-L375)).

**Generated files.** Two pieces of this path are plain hand-written source in
PostgreSQL 17. The GUC entry is a C initializer in `guc_tables.c`, which both
build systems compile like any other file
([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474),
[utils/misc/Makefile#OBJS](../../../../raw/postgres-17/src/backend/utils/misc/Makefile#L17-L34),
[utils/misc/meson.build#backend_sources](../../../../raw/postgres-17/src/backend/utils/misc/meson.build#L3-L20)).
The callback type is declared in `amapi.h`
([amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105)).
The step from an index to its `ambuild` function is different: it runs through
[catalog](../../../glossary.md#catalog)
data and a generated table. The
[relcache](../../../glossary.md#relcache)
looks up the `pg_am` row of the index's access method and keeps the OID of its
handler function
([relcache.c#access-method-handler-lookup](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L1461-L1471)).
It then fills the index's access-method struct by calling that function
([relcache.c#InitIndexAmRoutine](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L1398-L1422),
[amapi.c#GetIndexAmRoutine](../../../../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46)).
For the in-core access methods those rows come from `pg_am.dat`
([pg_am.dat#index-access-methods](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18-L35)),
and built-in functions such as the handlers are reached through the function
manager's table, which a script generates from `pg_proc.dat` at build time
([Gen_fmgrtab.pl#header](../../../../raw/postgres-17/src/backend/utils/Gen_fmgrtab.pl#L2-L15),
[utils/Makefile#fmgr-stamp](../../../../raw/postgres-17/src/backend/utils/Makefile#L48-L53)).
Bloom is not in `pg_am.dat`; its
[extension](../../../glossary.md#extension)
script registers the handler with `CREATE ACCESS METHOD`
([bloom--1.0.sql:12](../../../../raw/postgres-17/contrib/bloom/bloom--1.0.sql#L12)).

#### Test coverage of the memory paths

Two tests in the tree set `maintenance_work_mem` in order to steer an index
build, as their own comments say. Neither builds concurrently. Other tests set
it for other commands, for example for `CLUSTER`
([cluster.sql#external-tuplesort-test](../../../../raw/postgres-17/src/test/regress/sql/cluster.sql#L256-L273))
and for parallel `VACUUM`
([vacuum.sql#parallel-vacuum-minimum-memory](../../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L137-L143)).

| Test | Settings it changes (boot or default value) | What it exercises |
|---|---|---|
| [Regression test](../../../glossary.md#regression-test) `create_index.sql`, hash block ([create_index.sql#hash-tuplesort-test](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L372-L380)) | `maintenance_work_mem = '1MB'` (64 MB) and the index's [fillfactor](../../../glossary.md#fillfactor) set to 10 (hash default 75: [reloptions.c#hash-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L195-L204), [hash.h:296](../../../../raw/postgres-17/src/include/access/hash.h#L296)) | The sorted hash build. The test's own comment says it forces the sort with both settings. It then checks that a query through the index returns the expected count |
| `pageinspect` test `brin.sql`, parallel block ([pageinspect/sql/brin.sql#parallel-build](../../../../raw/postgres-17/contrib/pageinspect/sql/brin.sql#L87-L115)) | `min_parallel_table_scan_size = 0` (8 MB: [guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529)), `max_parallel_maintenance_workers = 4` (2), `maintenance_work_mem = '128MB'` (64 MB) | A parallel BRIN build of a small table. By step W5, 128 MB allows three workers plus the leader, that is four participants ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)). The test compares the result with a serially built index |
| `pageinspect` test `brin.sql`, fallback block ([pageinspect/sql/brin.sql#serial-fallback](../../../../raw/postgres-17/contrib/pageinspect/sql/brin.sql#L119-L126)) | In addition `max_parallel_workers = 0` (8: [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439)) | No worker can start, so the build falls back to a serial one ([brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525)) |

The comment of the BRIN test says that it sets the memory "for 4 workers". The
code allows three workers at 128 MB; the wording is recorded under
[Open Questions](#open-questions).

The tests that run a CIC do not change the setting: the block of concurrent
builds in `create_index.sql`
([create_index.sql#concurrent-builds](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L481-L582)),
the two
[isolation tests](../../../glossary.md#isolation-test)
([multiple-cic.spec](../../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43),
[prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37)),
and the two
[TAP tests](../../../glossary.md#tap-test)
of `amcheck`, of which the second runs CIC against the prepared transactions
of [two-phase commit](../../../glossary.md#two-phase-commit)
([amcheck/t/002_cic.pl](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L4-L88),
[amcheck/t/003_cic_2pc.pl](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L4-L170)).
They exercise the life cycle, concurrency and correctness of the command. No
test in the tree exercises a spilling sort, the worker-memory thresholds or a
performance plateau under CIC. The complete list of CIC tests is under
[Test coverage](#test-coverage).

#### Causal summary of the memory paths

1. **Initial state.** The session's `maintenance_work_mem`, M. A session-level
   `SET` is the only form that lasts through the command.
2. **Transaction 2, worker request.** Steps W1 to W5 turn M into a number of
   workers, at 32 MB per participant, unless the table's `parallel_workers`
   replaces that calculation.
3. **Transaction 2, build.** The access method turns M into sort budgets
   (B-tree, sorted GiST, parallel BRIN), into a size threshold (hash), into a
   limit on `levelStep` (buffered GiST) or into a dump size (GIN). SP-GiST and
   Bloom ignore it.
4. **Interaction.** Workers divide M by the planned count and the leader by
   the launched count. Therefore missing workers leave part of M unused.
5. **Commit 2 and wait 2.** The memory of the first build is released before
   validation starts, so the two uses never add up.
6. **Transaction 3.** GIN first empties its pending list with M. Then one
   serial sort takes one slot per reported TID. It is a single in-memory
   quicksort when the slots fit M, and an external sort otherwise.
7. **Result.** More memory helps only until the worker count, the
   access-method threshold and both kinds of sort stop changing. The scans, the
   page writes and the three waits remain.
8. **Next change of state.** Nothing in this chain is stored except the table's
   `parallel_workers`. A new session value takes effect with the next CIC.

### GUCs that affect CIC performance

No [GUC](../../../glossary.md#guc) changes what
[CIC](../../../glossary.md#concurrently) does. The four transactions, the two
[heap](../../../glossary.md#heap) scans and the three waits are fixed by the
algorithm (see [Logic map](#logic-map)). A setting can only change how long one
of those steps takes, or whether the statement gives up while it waits.

The settings fall into three groups, named for the step they act on:

1. **Build, scan and storage settings** act on the first build in transaction 2
   and on validation in transaction 3. They decide the memory budget, the number
   of parallel workers, how the heap is read, and where files go.
2. **WAL and commit settings** act on the [WAL](../../../glossary.md#wal) that
   the build writes and on the commits that end the transactions.
3. **Wait and timeout settings** act on the first
   [table lock](../../../glossary.md#heavyweight-lock) and on the three waits.
   They cannot make a wait shorter. They can only cancel the statement, or
   cancel one kind of blocker.

This first half of the section explains where every value comes from, maps the
settings onto the steps, and details group 1. The second half details groups 2
and 3, the settings that look relevant but are not, the observation settings,
and a priority order.

Two results surprise most readers. First, one familiar setting is read and then
does nothing. Both heap scans read
[`effective_io_concurrency`](../../../glossary.md#effective_io_concurrency), but
in PostgreSQL 17 that value cannot change their I/O, because the scans tell
their [read stream](../../../glossary.md#read-stream) that the access is
sequential
([heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259),
[read_stream.c#advice-condition](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L508-L520)).
Second, no memory, worker or I/O setting can shorten a wait. A wait ends only
when the transactions that CIC waits for have ended
([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973),
[indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497)).

For a normal PostgreSQL 17 [B-tree](../../../glossary.md#b-tree) CIC, the first
settings to examine are
[`maintenance_work_mem`](../../../glossary.md#maintenance_work_mem),
`max_parallel_maintenance_workers`, `min_parallel_table_scan_size`, and the two
limits on starting workers. Those two limits are `max_parallel_workers`, which
the CIC session compares with a cluster-wide count of active parallel workers,
and `max_worker_processes`, which sizes the server's pool of
[background worker](../../../glossary.md#background-worker) slots. The first
setting controls the build and validation memory paths. The others decide
whether the B-tree first build can use workers and whether the requested workers
actually start. [BRIN](../../../glossary.md#brin) shares the parallel-worker
controls. [GiST](../../../glossary.md#gist) adds
[`effective_cache_size`](../../../glossary.md#effective_cache_size), and
[GIN](../../../glossary.md#gin) adds two pending-list settings
([planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019),
[bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014),
[ref/create_index.sgml#memory-and-parallelism](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L845),
[gistbuild.c#gistInitBuffering](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L617-L777),
[ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471)).

This section uses a bounded meaning of "affect CIC performance". A setting
qualifies only if it changes a core CIC step, the build of a shipped
[access method](../../../glossary.md#access-method), worker availability,
storage placement or I/O, WAL or commit latency, or one of the command's waits.
Two things fall outside any list drawn from the core source. First, an index
expression or predicate can call arbitrary user code. Second, core calls the
access method's `ambuild` callback with the heap, the index and an `IndexInfo`
only. The callback interface carries no settings, so a third-party access method
can read whatever settings it likes
([amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105),
[index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3053)).

#### Setting values and where they come from

Every server process holds its own current value of every setting. The setting's
[context](../../../glossary.md#guc-context) says who may change that value and
when. The contexts that appear below are `user`, `superuser`, `sighup` and
`postmaster`
([guc.h#GucContext](../../../../raw/postgres-17/src/include/utils/guc.h#L35-L76),
[guc_tables.c#GucContext_Names](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L641-L650)).

For CIC it matters which process reads a setting, and when:

- **The CIC [backend](../../../glossary.md#backend)** reads most of them. It
  runs all four transactions, so one session value serves the whole statement.
- **Its parallel workers** start with a copy of the leader's values. The leader
  serializes its GUC state, and each worker restores that state before it does
  any work
  ([parallel.c#serialize-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L382-L385),
  [parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1455-L1457)).
- **Other processes** read their own values. A session that inserts into the
  table while CIC runs applies its own GIN settings, and the
  [checkpointer](../../../glossary.md#checkpointer) applies its own copy of the
  configuration
  ([gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-17/src/include/access/gin_private.h#L39-L45),
  [checkpointer.c#reload](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L566-L583)).

Six mechanisms appear in the tables of this section. Each is defined here first.

- **[Parallel index build](../../../glossary.md#parallel-index-build).** A
  B-tree or BRIN build can split its heap scan among worker processes. The
  backend that runs CIC is the leader. It first plans how many workers to
  request, and later tries to start them. The participants are the workers that
  started plus, in a standard build, the leader
  ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
  [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646),
  [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594)).
- **Bulk-read strategy.** When a heap scan covers more
  [blocks](../../../glossary.md#block) than a quarter of `NBuffers`, it reads
  through a small [ring of buffers](../../../glossary.md#ring-buffer) and reuses
  them, instead of spreading the table over all shared buffers. The ring is 256
  kB, and never more than one eighth of `NBuffers`
  ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452),
  [heapam.c#strategy-allocation](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L454-L465),
  [freelist.c#ring-sizes](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L545-L571),
  [freelist.c#ring-size-cap](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L598-L599)).
- **[Synchronized scan](../../../glossary.md#synchronized-scan).** A scan that
  starts at the block where another scan of the same table currently is, runs to
  the end, and then wraps around to the blocks it skipped. Each scan still
  visits every block. The gain is that the scans can share pages while those
  pages are cached
  ([syncscan.c#design](../../../../raw/postgres-17/src/backend/access/common/syncscan.c#L6-L19)).
- **Read stream.** In PostgreSQL 17 a heap
  [sequential scan](../../../glossary.md#sequential-scan) gets its blocks from a
  read stream. The stream looks ahead, merges adjacent blocks into one read, and
  keeps the buffers pinned until the scan takes them. "Advice" is a hint to the
  kernel that a block will be needed soon, which is how the server
  [prefetches](../../../glossary.md#prefetch)
  ([heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259),
  [read_stream.c#read_stream_look_ahead](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L301-L377),
  [bufmgr.c#advice-in-StartReadBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1326-L1343)).
- **[Pending list](../../../glossary.md#pending-list).** With GIN's `fastupdate`
  [storage parameter](../../../glossary.md#storage-parameter) on, which is its
  default, an insert appends to a pending list instead of updating the main
  structure. A later cleanup moves the entries
  ([reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131),
  [gin_private.h#fastupdate-default](../../../../raw/postgres-17/src/include/access/gin_private.h#L33-L38),
  [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530)).
- **[Temporary file](../../../glossary.md#temporary-file).** A
  [sort](../../../glossary.md#tuplesort) that spills writes its tapes to
  temporary files. When a sort spills is explained in
  [How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping)
  ([tuplesort.c#tape-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1965)).

The table below uses four words in its "Live/current or stored" column:

- **Live**: the process reads its current setting at the moment of use.
- **Captured**: the value is copied once into a working structure. That scan or
  build does not read the setting again.
- **Derived**: the value is computed once from other values.
- **Stored**: the value is kept in a [catalog](../../../glossary.md#catalog) row
  or in the index, and later steps read it from there.

A captured or stored value is not stale merely because it is a copy. While one
CIC statement runs, the CIC backend's own settings cannot change: the backend
handles no other client command, and it re-reads the configuration file only
between client commands
([postgres.c#main-loop-reload](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4752-L4760)).
The copies matter for two other reasons. First, they fix the moment at which a
value is taken, for example the heap size when a scan starts. Second, some of
the state they are compared with is live cluster state, such as the number of
active parallel workers, and that state does change while CIC runs.

| Value | Meaning | Source | Live/current or stored | Later use |
|---|---|---|---|---|
| `maintenance_work_mem` | Memory budget for maintenance work, in kB. | Setting of the CIC backend; boot value 65536 kB, which is 64 MB ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)). | Live in each process that uses it. A worker holds the leader's value. | The worker-request cap, the first build's memory, the validation sort and GIN's forced cleanup ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)). Details: [How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping). |
| `max_parallel_maintenance_workers` | Most workers that one maintenance command may request. | Setting of the CIC backend; boot value 2 ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)). | Live; read once per build, when the request is planned. | Zero stops the request. Any other value caps the storage-parameter path and the size model ([planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985), [allpaths.c#caller-cap](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4275-L4276)). |
| `min_parallel_table_scan_size` | Smallest heap, in blocks, for which the size model requests a worker. | Setting of the CIC backend; boot value `(8 * 1024 * 1024) / BLCKSZ` blocks, which is 8 MB ([guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529)). | Live; read once per build. It is compared with the heap's size in blocks as estimated at that moment ([planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998)). | Size model: no worker below it, then one more worker each time the heap is three times larger ([allpaths.c#minimum-size-test](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227), [allpaths.c#heap-size-steps](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4229-L4251)). |
| `parallel_workers` storage parameter of the table | A worker count set on the table itself. | The table's stored options; unset is -1 ([reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382), [rel.h#RelationGetParallelWorkers](../../../../raw/postgres-17/src/include/utils/rel.h#L392-L399), [plancat.c:205](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L205)). | Stored. It is set with `ALTER TABLE` and read from the table's [relcache](../../../glossary.md#relcache) entry when the request is planned ([ref/create_index.sgml#memory-and-parallelism](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L845)). | When set, it replaces the size model and the memory cap ([planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985)). |
| Requested workers (`ii_ParallelWorkers`) | Number of worker processes the build asks for. The leader is not counted. | Computed by `plan_create_index_workers()` when `index_build()` starts in transaction 2 ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005), [planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019)). | Derived once per build. | The access method registers this many workers and plans its participants from it ([nbtsort.c#begin-parallel-call](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L391), [brin.c#begin-parallel-call](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1153-L1163)). |
| `max_parallel_workers` and the active-worker count | The CIC session's limit on how many parallel workers may be active in the whole cluster. | The limit is a setting of the CIC backend, boot value 8. The count lives in shared memory: workers registered minus workers terminated ([guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439), [bgworker.c#parallel-worker-count](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L83-L100)). | Limit: live in the CIC backend. Count: live cluster state that the parallel workers of every session change. | Tested for each worker the leader registers ([bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014)). |
| `max_worker_processes` | Number of background-worker slots in the server. | Setting with `postmaster` context, boot value 8. It sizes the shared slot array ([guc_tables.c#max_worker_processes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3163-L3173), [bgworker.c#BackgroundWorkerShmemSize](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L142-L156), [bgworker.c:174](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L174)). | Stored at server start. Which slots are free is live cluster state. | Each registration needs a free slot ([bgworker.c#slot-search](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L1016-L1043)). |
| Launched workers (`nworkers_launched`) | Workers that were actually registered. | Counted by `LaunchParallelWorkers()` ([parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646)). | Derived once per build. The request is not planned again when fewer workers start. | Zero means a serial build. Otherwise the build runs with the launched workers plus the leader ([nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594), [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525)). |
| [`shared_buffers`](../../../glossary.md#shared_buffers) (`NBuffers`) | Number of shared buffers. | Setting with `postmaster` context. The compiled-in boot value is 16384 blocks; `initdb` writes its own probed value into a new cluster's configuration file ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270), [initdb.c#shared_buffers-write](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1283-L1290)). | Stored at server start. It cannot change while the server runs. | Three inputs: the `NBuffers / 4` test of each heap scan, the cap on the sort threshold of a [hash index](../../../glossary.md#hash-index) build, and the ring size and pin budget of each read stream ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452), [hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166), [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478)). |
| Heap size at scan start (`rs_nblocks`, `phs_nblocks`) | Number of heap blocks one scan will read. | The relation's size, read when a serial scan starts or when the leader sets up the parallel scan ([heapam.c#scan-size](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L414-L431), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L388-L404)). | Captured once per scan from live relation state. | Compared with `NBuffers / 4` ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452)). |
| `synchronize_seqscans` | Whether a large sequential scan may start where another scan of the same table currently is. | Setting, boot value on ([guc_tables.c#synchronize_seqscans](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1766-L1774)). | Live; read when a serial scan starts, or by the leader when it sets up the parallel scan ([heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L388-L404)). | The first heap scan, for the access methods that allow it. |
| [`io_combine_limit`](../../../glossary.md#io_combine_limit) | Most adjacent blocks that one read may cover. | Setting; boot value `DEFAULT_IO_COMBINE_LIMIT` ([guc_tables.c#io_combine_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3138-L3150), [bufmgr.h#io_combine_limit-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L168-L169)). | Captured into each read stream when its scan starts ([read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L530-L536)). | Upper bound on the size of each read in both heap scans ([read_stream.c#read_stream_look_ahead](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L301-L377)). |
| `effective_io_concurrency`, as `max_ios` | Number of reads a stream may have prepared but not yet performed. | The setting, or the option of the same name on the heap's [tablespace](../../../glossary.md#tablespace) when that is set ([read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439), [spccache.c#get_tablespace_io_concurrency](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L207-L223)). | Captured into each read stream when its scan starts ([read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L530-L536)). | Nothing that changes the I/O of these scans in PostgreSQL 17; see its row under [Build, scan, and storage settings](#build-scan-and-storage-settings). |
| Pin budget (`max_pinned_buffers`) | Most buffers a read stream may keep pinned. It bounds how far the stream looks ahead. | Computed when the stream is created, from `io_combine_limit`, `max_ios`, the scan's ring size and this backend's share of `NBuffers` ([read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478), [bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2107-L2144)). | Derived once per stream. | Together with `io_combine_limit` it caps the read size; see [Narrow build settings](#narrow-build-settings). |
| `effective_cache_size` | The size, in blocks, that the server assumes for shared buffers plus the [operating system's cache](../../../glossary.md#os-page-cache). | Setting, boot value 524288 blocks ([guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518), [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34)). | Live. | Only GiST's [non-sorted build](../../../glossary.md#gist-build-method) reads it ([gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900), [gistbuild.c#levelStep-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L741)). |
| `temp_tablespaces` | Where a process creates its temporary files. | Setting; the boot value is empty, which means the database's default tablespace ([guc_tables.c#temp_tablespaces](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4148-L4157), [fd.c#temp-tablespace-choice](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1737-L1763)). | Live when a process first needs a temporary file. A parallel sort captures the leader's list in its file set, so that all participants agree ([fileset.c#FileSetInit](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L36-L86)). | Sort tapes and GiST's buffering file. |
| `temp_file_limit` and `temporary_files_size` | The cap on one process's temporary-file bytes, and that process's running total. | The cap is a setting with `superuser` context, boot value -1, which means no limit. The total is a variable private to each process ([guc_tables.c#temp_file_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2504-L2513), [fd.c#temporary_files_size](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L233-L239)). | Both live, per process. | A temporary-file write that would pass the cap raises an error ([fd.c#temp_file_limit-check](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L2211-L2237)). |
| `default_tablespace`, then the index's tablespace | Where the index's files are created. | Setting, boot value empty. `DefineIndex` reads it in transaction 1 unless the statement has a `TABLESPACE` clause ([guc_tables.c#default_tablespace](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4137-L4146), [indexcmds.c#tablespace-selection](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L767-L784)). | Stored: the chosen tablespace is passed to `index_create()`, which creates the index relation in transaction 1 ([indexcmds.c#index_create-call](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1218-L1227)). | The index is created in that tablespace, so its pages are written there ([ref/create_index.sgml#tablespace](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L362-L372)). |
| `gin_pending_list_limit` | Size of a GIN index's pending list, in kB, above which an insert starts a cleanup. | The inserting backend's setting, boot value 4096 kB, unless the index has the storage parameter of the same name, whose default -1 means unset ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585), [gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-17/src/include/access/gin_private.h#L39-L45), [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)). | Live in each inserting backend. The list's length is state kept in the index's metapage. | Tested after every insert that used the pending list ([ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471)). |
| [`work_mem`](../../../glossary.md#work_mem) | Memory budget of a GIN pending-list cleanup that an insert started. | The inserting backend's setting, boot value 4096 kB ([guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2458)). | Live in the inserting backend. | Chosen as the working memory of that cleanup ([ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)). |
| `default_toast_compression` and the index column's `attcompression` | Compression method for an index value that is too wide. | The setting of the process that forms the index [tuple](../../../glossary.md#tuple), boot value `pglz`. The column's own method is copied from the table column, or left unset for an expression column, when the index is created ([guc_tables.c#default_toast_compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4809-L4818), [index.c:359](../../../../raw/postgres-17/src/backend/catalog/index.c#L359), [index.c#expression-attcompression](../../../../raw/postgres-17/src/backend/catalog/index.c#L390-L397)). | Setting: live at each compression. Column method: stored in the index's tuple descriptor. | See [Narrow build settings](#narrow-build-settings). |
| Commit settings: `synchronous_commit`, `synchronous_standby_names`, [`commit_delay`](../../../glossary.md#commit_delay), `commit_siblings` | How a commit flushes WAL, and whether it waits for a [synchronous standby](../../../glossary.md#synchronous-replication). | Settings; the second half of this section gives each context and default. | Live at each commit. | The commits of transactions 1, 2 and 4, which have a [transaction ID](../../../glossary.md#transaction-id). Transaction 3 normally has none, so its commit writes no commit record, does not wait for a WAL flush and does not wait for a standby. The exception is an index expression or predicate that assigns one ([xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546), [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452), [create_index.sql#xid-assigning-predicate](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L509-L520)). |
| WAL and checkpoint settings, for example `wal_compression`, `wal_buffers`, `max_wal_size`, `checkpoint_timeout` | The cost of the WAL that the build writes, and when a [checkpoint](../../../glossary.md#checkpoint) runs. | Settings; the second half gives each context and default. | Live in the process that uses each one; the second half says which. | For an index that needs WAL, the first build logs its pages through a path that depends on the access method. For example, B-tree hands its pages to the [bulk writer](../../../glossary.md#bulk-writer), which logs them in batches, and GIN logs the whole built index at the end of the build ([nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149), [bulk_write.c#smgr_bulk_flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L313), [gininsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L408-L417)). |
| Wait and timeout settings: [`lock_timeout`](../../../glossary.md#statement_timeout-and-lock_timeout), `deadlock_timeout`, `log_lock_waits`, `statement_timeout`, [`transaction_timeout`](../../../glossary.md#transaction_timeout-and-idle_in_transaction_session_timeout) | Limits on waiting, and reports about it. | Settings; the second half gives each context and default. | Live when a lock wait begins, when a transaction starts, or when the statement starts. | Each lock wait arms `deadlock_timeout` and, if it is set, `lock_timeout`. After its [deadlock](../../../glossary.md#deadlock) check, a waiter cancels a blocking [autovacuum](../../../glossary.md#autovacuum) worker that is not protecting against wraparound. `transaction_timeout` restarts in each internal transaction. `statement_timeout` is armed only when it is positive and either smaller than `transaction_timeout` or `transaction_timeout` is zero ([proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333), [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493), [xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2174-L2176), [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2315-L2317), [postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268)). |
| Observation settings | Reports about the command. | See "Observation settings" in the second half. | Not applicable. | Not applicable. |

#### Logic map: which setting acts on which step

The map follows the statement from start to end. Each line names the settings
that act on that step. A key in brackets points to the sources listed under the
map.

```text
One client statement: CREATE INDEX CONCURRENTLY
  statement_timeout, if armed, runs across all four transactions       [T1]
  transaction_timeout restarts in each transaction                     [T2]

Transaction 1: catalog entry
  first table lock ......... lock_timeout, deadlock_timeout            [L1]
  choose index tablespace .. default_tablespace                        [S1]
  commit 1 (has an XID) .... commit settings                           [C1]

Transaction 2: wait 1, then the first build
  wait 1 ................... lock_timeout, deadlock_timeout            [L2]
  worker request ........... max_parallel_maintenance_workers,
                             min_parallel_table_scan_size,
                             maintenance_work_mem,
                             parallel_workers storage parameter        [P1]
  worker launch ............ max_parallel_workers,
                             max_worker_processes                      [P2]
  heap scan 1 .............. shared_buffers, synchronize_seqscans,
                             io_combine_limit
                             (effective_io_concurrency: read, no effect) [H1]
  access method build ...... maintenance_work_mem;
                             hash: shared_buffers;
                             GiST: effective_cache_size;
                             wide values: default_toast_compression    [B1]
  temporary files .......... temp_tablespaces, temp_file_limit         [F1]
  index pages and WAL ...... WAL and checkpoint settings               [W1]
  commit 2 (has an XID) .... commit settings                           [C1]

Transaction 3: wait 2, then validation
  wait 2 ................... lock_timeout, deadlock_timeout            [L2]
  index scan ............... GIN only: maintenance_work_mem
                             (forced pending-list cleanup)             [V1]
  TID sort ................. maintenance_work_mem, temp_tablespaces,
                             temp_file_limit                           [V2]
  heap scan 2 .............. shared_buffers, io_combine_limit;
                             never synchronized                        [H2]
  inserts of missing tuples  GIN only: gin_pending_list_limit and
                             work_mem of the CIC backend               [V3]
  commit 3 (normally no XID) no commit record, no flush wait           [C2]

Transaction 4: wait 3, then mark valid
  wait 3 ................... lock_timeout, deadlock_timeout            [L3]
  commit 4 (has an XID) .... commit settings                           [C1]

Other sessions, from commit 2 on
  inserts into a GIN index . their own gin_pending_list_limit
                             and work_mem                              [X1]
```

- `[T1]` The statement timer is started when the client statement begins and
  cancelled when it ends. It is started only if `statement_timeout` is positive
  and either smaller than `transaction_timeout` or `transaction_timeout` is
  zero. CIC's internal commits call `CommitTransactionCommand()` and
  `StartTransactionCommand()` directly, so they do not pass through the two
  routines that start and cancel the timer
  ([postgres.c#start_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2796),
  [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821),
  [postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268),
  [indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)).
- `[T2]` Every transaction start schedules the transaction timer, and every
  commit disables it
  ([xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2174-L2176),
  [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2315-L2317)).
- `[L1]` `ProcessUtilitySlow` takes the table lock before `DefineIndex` runs.
  Any lock wait arms the deadlock timer and, if `lock_timeout` is set, the lock
  timer. After the deadlock check, the waiter cancels a blocking autovacuum
  worker that is not protecting against wraparound
  ([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480),
  [proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333),
  [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493)).
- `[S1]`
  [indexcmds.c#tablespace-selection](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L767-L784).
- `[C1]` Transaction 1 commits the new catalog rows. Transaction 2 commits the
  `indisready` update. Transaction 4 updates `indisvalid` and is committed by
  the caller when the statement ends. Each has an XID, so its commit follows the
  flush and standby rules
  ([indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619),
  [index.c#set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1551-L1556),
  [indexcmds.c#commit-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1687-L1691),
  [indexcmds.c#set-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1770-L1773),
  [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821),
  [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521),
  [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546)).
- `[L2]` Waits 1 and 2 collect the transactions that hold a conflicting lock on
  the table. They then wait for each one by acquiring a lock on its
  [virtual transaction ID](../../../glossary.md#virtual-transaction-id). Each of
  those acquisitions is an ordinary lock wait, so the timers of `[L1]` apply to
  each one separately
  ([indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658),
  [indexcmds.c#wait-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1697-L1705),
  [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973),
  [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663)).
- `[P1]`
  [index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
  [planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019).
  The decision tree below expands this step.
- `[P2]`
  [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646),
  [bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014),
  [bgworker.c#slot-search](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L1016-L1043).
- `[H1]`
  [heapam_handler.c#build-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1248-L1283),
  [heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452),
  [heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496),
  [heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259).
- `[B1]` Core calls the access method's `ambuild`. The memory use of each access
  method is in
  [How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping)
  ([index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3053),
  [hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166),
  [gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900),
  [indextuple.c#compress-wide-value](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138)).
- `[F1]`
  [tuplesort.c#tape-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1965),
  [fileset.c#FileSetInit](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L36-L86),
  [fd.c#temp_file_limit-check](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L2211-L2237).
- `[W1]` The path depends on the access method; the second half compares them.
  Two examples:
  [bulk_write.c#smgr_bulk_flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L313),
  [gininsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L408-L417).
- `[V1]`
  [index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404),
  [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602),
  [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828).
- `[V2]`
  [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399),
  [tuplesort.c#tape-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1965).
- `[H2]`
  [heapam_handler.c#validation-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1804).
- `[V3]`
  [heapam_handler.c#validation-insert](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1963-L1971),
  [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530),
  [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471).
- `[C2]` Validation updates no catalog row, so transaction 3 normally has no
  XID. Its commit then writes no commit record and takes the
  [asynchronous commit](../../../glossary.md#asynchronous-commit) branch, which
  does not wait for the WAL flush. The exception is an index expression or
  predicate that assigns an XID, which the
  [regression tests](../../../glossary.md#regression-test) exercise
  ([indexcmds.c#commit-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1742-L1751),
  [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452),
  [xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395),
  [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521),
  [create_index.sql#xid-assigning-predicate](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L509-L520)).
- `[L3]` Wait 3 lists the transactions whose
  [snapshots](../../../glossary.md#snapshot) are too old and waits for each one
  through the same virtual-transaction lock, so the timers of `[L1]` apply here
  as well
  ([indexcmds.c#wait-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1768),
  [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497),
  [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663)).
- `[X1]`
  [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530),
  [gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-17/src/include/access/gin_private.h#L39-L45).

**The worker request and the worker launch.** Steps `[P1]` and `[P2]` have the
most branches, so they get their own map. It applies to a parallel index build,
which only B-tree and BRIN support. The request is planned first. The launch
happens later and can deliver fewer workers than were requested.

```text
index_build() in transaction 2                                        [D0]
│
├─ access method is not B-tree or BRIN ........ request stays 0: serial
│
├─ standalone backend, or
│  max_parallel_maintenance_workers = 0 ....... request 0: serial     [D1]
│
├─ heap is a temporary table, or an index
│  expression or predicate is not
│  parallel safe .............................. request 0: serial     [D2]
│
├─ table has the parallel_workers storage parameter                   [D3]
│     request = min(parallel_workers,
│                   max_parallel_maintenance_workers)
│     no size test and no memory test: go to "request"
│
├─ size model                                                         [D4]
│     heap blocks < min_parallel_table_scan_size .. request 0: serial
│     otherwise 1 worker, plus 1 each time the heap is 3 times larger,
│     then at most max_parallel_maintenance_workers
│
├─ memory cap                                                         [D5]
│     while maintenance_work_mem / (workers + 1) < 32768 kB:
│         drop one worker
▼
request (ii_ParallelWorkers)
│
├─ request = 0 ................................ serial build          [D6]
│
├─ create the parallel context and its shared memory segment          [D7]
│     no segment available .................... serial build
│
├─ register each requested worker                                     [D8]
│     active parallel workers in the cluster
│       >= max_parallel_workers ............... refused
│     no free slot among the
│       max_worker_processes slots ............ refused
│     after the first refusal the rest are skipped
│
├─ launched = 0 ............................... serial build          [D9]
│
└─ launched > 0 ......... parallel build: launched workers
                          plus the leader                             [D10]
```

- `[D0]` `index_concurrently_build()` always asks for a parallel build.
  `index_build()` plans a request only when the access method sets
  `amcanbuildparallel`, which B-tree and BRIN do. The other shipped access
  methods set it to false, so their request keeps its initial value 0
  ([index.c#index_concurrently_build-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L1538-L1539),
  [index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005),
  [nbtree.c:120](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L120),
  [brin.c:266](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L266),
  [hash.c:76](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L76),
  [gist.c:78](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L78),
  [spgutils.c:63](../../../../raw/postgres-17/src/backend/access/spgist/spgutils.c#L63),
  [ginutil.c:56](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L56),
  [blutils.c:125](../../../../raw/postgres-17/contrib/bloom/blutils.c#L125),
  [makefuncs.c:849](../../../../raw/postgres-17/src/backend/nodes/makefuncs.c#L849)).
- `[D1]`
  [planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913).
- `[D2]`
  [planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971).
- `[D3]`
  [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985).
- `[D4]`
  [planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998),
  [allpaths.c#minimum-size-test](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227),
  [allpaths.c#heap-size-steps](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4229-L4251),
  [allpaths.c#caller-cap](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4275-L4276).
- `[D5]`
  [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012).
- `[D6]`
  [nbtsort.c#begin-parallel-call](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L388-L391),
  [brin.c#begin-parallel-call](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1153-L1163).
- `[D7]`
  [parallel.c#non-interruptible](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L235-L242),
  [parallel.c#session-dsm](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L244-L261),
  [parallel.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L310-L334),
  [nbtsort.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1486-L1497),
  [brin.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2432-L2443).
- `[D8]`
  [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646),
  [bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014),
  [bgworker.c#slot-search](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L1016-L1043).
- `[D9]`
  [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594),
  [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525).
- `[D10]`
  [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594),
  [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525).

The steps in order:

1. **Is a request planned at all?** *Input:* the access method. *Decision:* only
   an access method that can build in parallel gets a request. *Output:* for
   hash, GiST, [SP-GiST](../../../glossary.md#sp-gist), GIN and Bloom the
   request is 0. *Consequence:* the worker settings do nothing for those access
   methods (`[D0]`).
2. **Two early exits.** *Input:* the process type, the setting
   `max_parallel_maintenance_workers`, and the index definition. *Decision:* a
   standalone backend or a setting of zero ends the planning with 0. So does a
   temporary heap, or an index expression or predicate that is not parallel
   safe. *Consequence:* both exits come before the storage parameter is read.
   Therefore `parallel_workers` cannot force workers for a parallel-unsafe index
   (`[D1]`, `[D2]`).
3. **The storage parameter.** *Input:* the table's `parallel_workers`.
   *Decision:* if it is set, the planning stops here. *Calculation:* the request
   is the smaller of the parameter and `max_parallel_maintenance_workers`.
   *Consequence:* table size and `maintenance_work_mem` are not consulted, as
   the source comment states. A parameter of 0 gives a serial build (`[D3]`).
4. **The size model.** *Input:* the heap's size in blocks, estimated now, and
   `min_parallel_table_scan_size`. *Decision:* a heap below the setting gets no
   worker. The test applies because the planning uses a base relation; the code
   skips it only for inheritance children
   ([relnode.c:207](../../../../raw/postgres-17/src/backend/optimizer/util/relnode.c#L207)).
   *Calculation:* start with one worker and a threshold equal to the setting.
   While the heap is at least three times the threshold, add a worker and triple
   the threshold. *Output:* the smaller of that count and
   `max_parallel_maintenance_workers` (`[D4]`).
5. **The memory cap.** *Input:* the count from step 4 and
   `maintenance_work_mem`. *Calculation:* while `maintenance_work_mem` divided
   by the number of participants, which is the workers plus the leader, is below
   32768 kB, drop one worker. *Output:* the request. *Consequence:* a request of
   `w` workers needs at least `(w + 1) * 32` MB (`[D5]`).
6. **The parallel context.** *Input:* a request above 0. *Decision:* the access
   method creates a parallel context and a
   [dynamic shared memory](../../../glossary.md#dynamic-shared-memory) segment.
   If no segment can be created, the build backs out. *Consequence:* a serial
   build with the request ignored (`[D6]`, `[D7]`).
7. **Registration.** *Input:* the request, the cluster-wide count of active
   parallel workers, and the free slots. *Decision:* each registration is
   refused when the count has reached the CIC session's `max_parallel_workers`,
   or when no slot is free. After the first refusal the leader skips the
   remaining ones. *Output:* the number launched (`[D8]`).
8. **The result.** *Decision:* with no worker launched, the access method ends
   parallel mode and builds serially. Otherwise it builds with the workers it
   got, and in a standard build the leader takes part in the scan as well.
   *Consequence:* the request is a ceiling, not a promise (`[D9]`, `[D10]`).

The request as a formula. The terms are:

- `H`: the heap's size in blocks, as estimated when the request is planned.
- `T`: `min_parallel_table_scan_size`, in blocks. The tripling starts from at
  least 1 block.
- `Wmax`: `max_parallel_maintenance_workers`.
- `M`: `maintenance_work_mem`, in kB.

```text
size count   S = 0        if H < T
             S = 1 + k    otherwise; k is the largest number for which
                          T * 3^k <= H   (k = 0 when H < 3 * T)
capped count C = min(S, Wmax)
request      R = the largest w <= C with M / (w + 1) >= 32768;  0 if none
```

In plain language: the table's size proposes a count, the worker setting caps
it, and memory removes workers until each participant has 32 MB. The size count
grows by one each time the table is three times larger. With the boot values,
any heap of at least three times `T`, which is 24 MB, already proposes two
workers, the value of `Wmax`. So for larger heaps the two caps decide. The
storage parameter replaces all three lines with
`R = min(parallel_workers, Wmax)`
([allpaths.c#heap-size-steps](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4229-L4251),
[planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012),
[planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985)).

A worked example, derived from the source, not measured. The inputs are the boot
values and an assumed block size of 8 kB:

| Input | Value | Where it comes from |
|---|---|---|
| `T` | 1024 blocks | Boot value `(8 * 1024 * 1024) / BLCKSZ` with [`BLCKSZ`](../../../glossary.md#blcksz) = 8192 ([guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529)). |
| `Wmax` | 2 | Boot value ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)). |
| `M` | 65536 kB | Boot value ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)). |
| `H` | 131072 blocks | An assumed 1 GB heap. |
| `parallel_workers` | unset | Default -1 ([reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382)). |

The calculation:

1. `S`: 1024 * 3 = 3072, 9216, 27648 and 82944 are all at most 131072; the next
   step, 248832, is not. So `k = 4` and `S = 5`.
2. `C = min(5, 2) = 2`.
3. `R`: for `w = 2`, 65536 / 3 = 21845, which is below 32768, so one worker is
   dropped. For `w = 1`, 65536 / 2 = 32768, which is not below 32768. So
   `R = 1`.
4. Launch: one registration. It succeeds if fewer than 8 parallel workers are
   active in the cluster, 8 being the boot value of `max_parallel_workers`, and
   a slot is free
   ([guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439)).

The result is one worker plus the leader. It shows that with every boot value in
place the memory cap, not `max_parallel_maintenance_workers`, is the binding
limit. With `M` = 98304 kB (96 MB) step 3 keeps `w = 2`, and from there on
`Wmax` is the binding limit.

The cases side by side:

| Condition / state | Behavior | Consequence |
|---|---|---|
| Access method other than B-tree or BRIN | No request is planned ([index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005)). | Serial first build. The worker settings have no effect. |
| `max_parallel_maintenance_workers` = 0; the boot value is 2 | The request is 0 before any other test ([planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913)). | Serial first build. This is the session-level way to turn the parallel build off. |
| Index expression or predicate that is not parallel safe | The request is 0 ([planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971)). | Serial first build, even with the storage parameter set. The same exit covers a temporary heap, but a CIC on a temporary table has already become a plain `CREATE INDEX` ([Preconditions and restrictions](#preconditions-and-restrictions)). |
| `parallel_workers` set on the table; the default is unset | The request is the smaller of the parameter and `max_parallel_maintenance_workers` ([planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985)). | Table size and memory are ignored. A value of 0 disables the parallel build for this table. |
| Heap smaller than `min_parallel_table_scan_size` | The size model returns 0 ([allpaths.c#minimum-size-test](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227)). | Serial first build. |
| `maintenance_work_mem` below 65536 kB, with no `parallel_workers` set on the table | The memory cap removes every worker, because even one worker needs two shares of 32768 kB ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)). | Serial first build, whatever the table size. |
| No shared memory segment for the parallel context | The access method backs out ([nbtsort.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1486-L1497), [brin.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2432-L2443)). | Serial first build. |
| Worker limit reached or no free slot during registration | That registration and all later ones are skipped ([parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646)). | Fewer workers than requested, possibly none. |
| No worker launched | The access method ends parallel mode ([nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594), [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525)). | Serial first build by the leader alone. |

The key interaction is between the two halves of the map. The request uses the
CIC session's `max_parallel_maintenance_workers` and `maintenance_work_mem`. The
launch uses the cluster-wide count of active parallel workers and the slot pool
that was sized at server start. Because the request is planned before the launch
and is not planned again, a larger request does nothing when the pool is the
binding limit. Therefore check the launch first when a build runs with fewer
workers than the settings suggest
([planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019),
[parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646)).
How the memory budget is then divided among the participants is explained in
[How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping).

#### Application scope used in the tables

The tables give each setting's default and say how a change applies. They use
four phrases, one for each GUC context:

| Phrase in the tables | Context | Who can change it, and how | When CIC sees the change |
|---|---|---|---|
| session | `user` | Any role, at any time, with `SET` in the session ([guc.h#GucContext](../../../../raw/postgres-17/src/include/utils/guc.h#L35-L76)). | In the next statement of that session. |
| session, for a role allowed to change it | `superuser` | A superuser, or a role that was granted `SET` on the setting, with `SET` in the session ([guc.h#GucContext](../../../../raw/postgres-17/src/include/utils/guc.h#L35-L76), [ref/set.sgml#session-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L32-L52)). | In the next statement of that session. |
| reload | `sighup` | Change the configuration file and reload the server ([guc.h#GucContext](../../../../raw/postgres-17/src/include/utils/guc.h#L35-L76)). | Each process applies the reload in its own main loop. A backend does that between client commands ([postgres.c#main-loop-reload](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4752-L4760)). |
| restart | `postmaster` | Only when the server starts ([guc.h#GucContext](../../../../raw/postgres-17/src/include/utils/guc.h#L35-L76)). | After the restart. |

The lower-case context names are the ones the server displays
([guc_tables.c#GucContext_Names](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L641-L650)).

Only the session form of `SET` can cover one CIC run, for three reasons that
build on each other:

1. A `SET LOCAL` value lasts only to the end of the current transaction
   ([ref/set.sgml#SET-LOCAL-lifetime](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L54-L61)).
2. CIC is rejected inside a transaction block, and outside a transaction block
   `SET LOCAL` has no effect
   ([utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463),
   [ref/set.sgml#LOCAL](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L109-L120)).
3. A plain `SET` issued before the statement is already committed when CIC
   starts. Therefore it stays in force across CIC's internal commits
   ([ref/set.sgml#session-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L32-L52)).

The events that change which value a step sees:

| Event | State before | State after | Effect on later calculation |
|---|---|---|---|
| `SET` in the session, before the statement | The session uses its previous value. | The session uses the new value until it is set again or the session ends. | All four transactions read the new value ([ref/set.sgml#session-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L32-L52)). |
| `index_build()` or `validate_index()` starts | The settings are as the session left them. | A new GUC nesting level is open, and `search_path` is restricted for the step. | A setting that an index function changes during the step is rolled back when the step ends, so it does not reach a later step ([index.c#index_build-guc-nesting](../../../../raw/postgres-17/src/backend/catalog/index.c#L3019-L3028), [index.c#index_build-guc-rollback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3149-L3150), [index.c#validate_index-guc-rollback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3443-L3444), [ref/create_index.sgml#search_path](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L792-L796)). |
| The leader sets up a parallel build | There are no workers. | The leader's GUC state is serialized, and each worker restores it when it starts. | Workers compute with the leader's values ([parallel.c#serialize-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L382-L385), [parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1455-L1457)). |
| A heap scan starts | The scan has no read stream. | The stream holds its own copies of `io_combine_limit` and `max_ios`. | The scan uses those copies to its end ([read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L530-L536)). |
| The server reloads its configuration while CIC runs | Every process has the old values. | Processes such as the checkpointer apply the new file in their own loops. The CIC backend still has the old values. | A reload setting that the CIC backend reads itself changes only for its next statement. A reload setting that another process reads can change while CIC runs ([postgres.c#main-loop-reload](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4752-L4760), [checkpointer.c#reload](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L566-L583)). |

#### Build, scan, and storage settings

The table uses the mechanisms that are defined under
[Setting values and where they come from](#setting-values-and-where-they-come-from):
the parallel index build, the bulk-read strategy, synchronized scans, read
streams, the GIN pending list and temporary files.

| GUC | Exact CIC effect and boundary | Default and how a change applies |
|---|---|---|
| `maintenance_work_mem` | The primary direct control. It sizes the access method's first build in transaction 2, for the access methods that read it. It caps the automatic B-tree or BRIN worker request so that every participant, the leader included, has at least 32 MB. It sizes the serial sort of [TIDs](../../../glossary.md#tid) in validation. For GIN it also sizes the forced pending-list cleanup that starts validation ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)). See [How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping). | Boot value 65536 kB (64 MB). `user` context: session; no reload or restart ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)). |
| `max_parallel_maintenance_workers` | Caps the number of workers that a B-tree or BRIN first build requests. Zero ends the planning before any other test, so the build is serial. It is a cap on the request, not a promise that workers start ([planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913), [allpaths.c#caller-cap](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4275-L4276), [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646)). | Boot value 2. `user` context: session; no reload or restart ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)). |
| `min_parallel_table_scan_size` | The size model requests no worker for a heap smaller than this, one worker from this size on, and one more each time the heap is three times larger. A `parallel_workers` storage parameter on the table bypasses this size model and the 32 MB test, but stays capped by `max_parallel_maintenance_workers` ([allpaths.c#minimum-size-test](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227), [allpaths.c#heap-size-steps](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4229-L4251), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985)). | Boot value 8 MB, kept in blocks. `user` context: session; no reload or restart ([guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529)). |
| `max_parallel_workers`; `max_worker_processes` | Two different limits act when the leader registers its workers. First, a registration is refused when the number of parallel workers active in the whole cluster has reached the CIC session's `max_parallel_workers`. The count is cluster-wide, but the limit is the registering session's own value, so sessions with different values apply different limits. Second, a registration needs a free slot in the pool that `max_worker_processes` sized at server start. After the first refusal the leader skips the remaining registrations. The build then runs with the workers it got; with none, B-tree and BRIN both fall back to a serial build ([bgworker.c#parallel-worker-count](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L83-L100), [bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014), [bgworker.c#slot-search](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L1016-L1043), [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646), [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594), [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525)). The documentation words the first limit differently; see [Open Questions](#open-questions). | Both boot values are 8. `max_parallel_workers`: `user` context, session, no reload or restart. `max_worker_processes`: `postmaster` context, restart ([guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439), [guc_tables.c#max_worker_processes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3163-L3173)). |
| `effective_cache_size` | Only GiST's non-sorted build reads it. That build runs when some key column's [operator class](../../../glossary.md#operator-class) has no sort support function, or when the index has `buffering = on`; otherwise GiST sorts and never reads the setting. In `auto` mode, the default of the `buffering` storage parameter, the build switches to buffering once the index has more blocks than `effective_cache_size`. When buffering starts, the build picks the largest `levelStep` whose subtree fits in a quarter of `effective_cache_size`; the same loop also stops at a `maintenance_work_mem` limit. With `buffering = off` the build never buffers, so it never reads the setting. The setting reserves no memory ([gistbuild.c#buffering-mode-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L226), [reloptions.c#buffering](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L521-L531), [gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248), [gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900), [gistbuild.c#levelStep-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L741)). Its only other reader in the pinned tree is the [planner](../../../glossary.md#planner)'s cost model ([costsize.c#effective_cache_size-use](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L914-L915)). | Boot value 524288 blocks, which is 4 GB with 8 kB blocks. `user` context: session; no reload or restart ([guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518), [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34)). |
| `shared_buffers` | Its value, `NBuffers`, feeds three CIC decisions. First, a heap scan uses the bulk-read strategy, and may be synchronized, only when the heap has more than `NBuffers / 4` blocks. Second, the build of a non-temporary hash index caps its sort threshold at `NBuffers`. Third, each read stream's pin budget is limited by the ring, which is at most `NBuffers / 8`, and by this backend's share of `NBuffers`; a budget below `io_combine_limit` caps the read size ([Narrow build settings](#narrow-build-settings)). So the effect depends on the access method and the table size. A larger value is not always faster ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L388-L404), [hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166), [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478), [freelist.c#ring-size-cap](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L598-L599), [bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2107-L2144)). | The compiled-in boot value is 16384 blocks, which is 128 MB with 8 kB blocks. A new cluster that `initdb` created runs on a second default: `initdb` tries a list of sizes from that same 128 MB downward, keeps the first one with which a test run of the server program succeeds, and writes it into `postgresql.conf`. The two values differ only when the probe had to go lower. `postmaster` context: restart ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270), [initdb.c#trial_bufs](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1124-L1128), [initdb.c#shared_buffers-probe](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1173-L1186), [initdb.c#test_specific_config_settings](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1199-L1241), [initdb.c#shared_buffers-write](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1283-L1290)). |
| `io_combine_limit` | Both heap scans read through a sequential read stream. The stream merges adjacent blocks into one pending read and starts that read when it holds `io_combine_limit` blocks; it never builds a larger one. So the setting is an upper bound on the read size. The actual size can be smaller for two reasons. The stream starts with a look-ahead distance of one block, doubles it only after a read that needed I/O, and shrinks it again while blocks are found in shared buffers. And the distance cannot exceed the pin budget, which can be smaller than `io_combine_limit` ([Narrow build settings](#narrow-build-settings)). Each stream copies the setting when its scan starts, and in a parallel build every participant opens its own scan ([heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259), [read_stream.c#read_stream_look_ahead](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L301-L377), [read_stream.c#initial-distance](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L545-L553), [read_stream.c#distance-after-wait](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L707-L728), [read_stream.c#read_stream_start_pending_read](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L211-L299), [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478), [read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L530-L536), [nbtsort.c#worker-scan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1920-L1927)). | Boot value `DEFAULT_IO_COMBINE_LIMIT`: 128 kB worth of blocks, or the platform's largest I/O vector if that is smaller, and never more than 32 blocks. That is 16 blocks with 8 kB blocks. `user` context: session; no reload or restart ([guc_tables.c#io_combine_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3138-L3150), [bufmgr.h#io_combine_limit-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L168-L169), [pg_iovec.h#PG_IOV_MAX](../../../../raw/postgres-17/src/include/port/pg_iovec.h#L42-L43)). |
| `effective_io_concurrency` | Read, but without effect on the I/O of these scans in PostgreSQL 17. When a heap scan creates its read stream, the stream takes this setting, or the option of the same name on the heap's tablespace, as its `max_ios`. It does not take `maintenance_io_concurrency`, because the heap access method passes `READ_STREAM_SEQUENTIAL` and not `READ_STREAM_MAINTENANCE`. **Default:** in a build with prefetch support, a read stream may issue advice when `max_ios` is above zero and direct I/O is off. **Cause:** the heap scan's `READ_STREAM_SEQUENTIAL` flag. **Mechanism:** the flag turns advice off for the stream on every platform. Without advice, no read starts ahead of the scan: starting a read only pins its buffers, and the read itself happens when the scan reaches the block. Without advice the look-ahead distance also only moves toward `io_combine_limit`, so the longer queue that a larger `max_ios` permits is never used. A value of zero is treated as one. What remains is bookkeeping: `max_ios` sizes the stream's array of prepared reads and bounds how many reads may be prepared but not yet performed ([read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439), [spccache.c#get_tablespace_io_concurrency](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L207-L223), [heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259), [read_stream.h#READ_STREAM_SEQUENTIAL](../../../../raw/postgres-17/src/include/storage/read_stream.h#L29-L35), [read_stream.c#advice-condition](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L508-L520), [bufmgr.c#advice-in-StartReadBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1326-L1343), [bufmgr.c:1519](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1519), [read_stream.c#distance-after-wait](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L707-L728), [read_stream.c#behavior-B](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L27-L32), [read_stream.c#zero-max_ios](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L522-L528), [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478)). The documentation describes this setting differently; see [Open Questions](#open-questions). | Boot value 1 in a build with prefetch support. Otherwise 0, and then any other value is rejected. `user` context: session; no reload or restart. The tablespace option of the same name, when set, replaces the setting for tables in that tablespace ([guc_tables.c#effective_io_concurrency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3109-L3121), [bufmgr.h#I/O-concurrency-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L164), [variable.c#check_effective_io_concurrency](../../../../raw/postgres-17/src/backend/commands/variable.c#L1222-L1233), [reloptions.c#effective_io_concurrency](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L348-L360)). |
| `synchronize_seqscans` | For a heap of more than `NBuffers / 4` blocks, it lets the first heap scan start at the block where another synchronized scan of the same table currently is, and wrap around. The scan still reads every block; the possible gain is I/O shared with the other scan. Whether a first build allows this depends on the access method, as the next table shows. The second heap scan, in validation, never allows it ([heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452), [heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496), [syncscan.c#design](../../../../raw/postgres-17/src/backend/access/common/syncscan.c#L6-L19), [heapam_handler.c#validation-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1804)). | Boot value on. `user` context: session; no reload or restart ([guc_tables.c#synchronize_seqscans](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1766-L1774)). |
| `temp_tablespaces`; `temp_file_limit` | `temp_tablespaces` chooses where temporary files are created: the tapes of a serial sort that spills, the tapes of a parallel sort's participants, and the file that GiST's buffering build uses for buffer pages that do not fit in memory. With the boot value, an empty list, they go to the database's default tablespace. `temp_file_limit` does not slow the work down; it stops it. Each process counts its own temporary-file bytes, and a write that would take the count past the limit raises an error. In CIC that error comes after transaction 1 has committed the index's catalog rows, so it leaves an [invalid index](../../../glossary.md#invalid-index) behind ([Failure handling](#failure-handling)) ([tuplesort.c#tape-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1965), [tuplesort.c:2985](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L2985), [fileset.c#FileSetInit](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L36-L86), [gistbuildbuffers.c#temporary-file](../../../../raw/postgres-17/src/backend/access/gist/gistbuildbuffers.c#L53-L57), [buffile.c#BufFileCreateTemp](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L180-L216), [fd.c#temp-tablespace-choice](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1737-L1763), [fd.c#temporary_files_size](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L233-L239), [fd.c#temp_file_limit-check](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L2211-L2237), [indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619)). | `temp_tablespaces`: boot value empty; `user` context: session. `temp_file_limit`: boot value -1, which means no limit; the unit is kB; `superuser` context: session, only for a role allowed to change it. Neither needs a reload or restart ([guc_tables.c#temp_tablespaces](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4148-L4157), [guc_tables.c#temp_file_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2504-L2513)). |
| `default_tablespace` | When the statement has no `TABLESPACE` clause, this setting names the tablespace in which the index is created. With the boot value, an empty string, or with a name that does not exist, the index goes to the database's default tablespace. The index does not follow the table's tablespace. An explicit `TABLESPACE` clause overrides the setting. The choice changes where index pages are written, and so their I/O cost. It does not change the CIC algorithm ([indexcmds.c#tablespace-selection](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L767-L784), [tablespace.c#GetDefaultTablespace](../../../../raw/postgres-17/src/backend/commands/tablespace.c#L1126-L1182), [ref/create_index.sgml#tablespace](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L362-L372)). | Boot value empty. `user` context: session; no reload or restart ([guc_tables.c#default_tablespace](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4137-L4146)). |
| `gin_pending_list_limit`; `work_mem` | Neither sizes GIN's first build, which collects entries in memory up to `maintenance_work_mem` and writes them straight into the index. Both act on the pending list. After an insert has appended to the list, the inserting backend compares the list's size with its `gin_pending_list_limit`, or with the index's storage parameter of the same name when that is set. Above the limit it runs a regular cleanup, whose working memory is that backend's `work_mem`. Two kinds of backend insert during CIC. Concurrent writers do, from the commit of `indisready` on. The CIC backend does too, when validation inserts the tuples that the first build missed. So the CIC session's own two values act in transaction 3, not only the writers' values. Separately, validation begins with a forced, complete cleanup that uses the CIC backend's `maintenance_work_mem`; the writers' limits change how long a list that cleanup finds ([gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314), [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530), [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-17/src/include/access/gin_private.h#L39-L45), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828), [index.c#set-ready](../../../../raw/postgres-17/src/backend/catalog/index.c#L1551-L1556), [heapam_handler.c#validation-insert](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1963-L1971), [index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404), [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)). | Both boot values are 4096 kB (4 MB). Both have `user` context: session, in the session that inserts; no reload or restart. With `fastupdate = off` on the index, inserts bypass the pending list, and neither setting acts ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585), [guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2458), [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530)). |

Whether the first heap scan may be synchronized is decided by the caller. A
serial build passes its `allow_sync` argument on to the scan. A parallel build
follows the shared scan that the leader set up
([heapam_handler.c#build-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1248-L1283)).
"Yes" in the table below means the scan is allowed to synchronize. It then does
so only if `synchronize_seqscans` is on and the heap has more than
`NBuffers / 4` blocks.

| Heap scan | May it be synchronized? | Why |
|---|---|---|
| First build, B-tree, serial | Yes | The build passes `allow_sync = true` ([nbtsort.c#serial-or-parallel-scan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L473-L480)). |
| First build, B-tree or BRIN, parallel | Yes | The leader marks the shared scan as synchronized when the setting is on and the heap is large enough, and each participant joins that shared scan ([tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L388-L404), [heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496), [nbtsort.c#worker-scan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1920-L1927), [brin.c#worker-scan](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2816-L2824)). |
| First build, hash | Yes | The build passes `allow_sync = true` ([hash.c#build-scan](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L172-L175)). |
| First build, GiST, sorted or not | Yes | Both build paths pass `allow_sync = true` ([gistbuild.c#sorted-build-scan](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L273-L276), [gistbuild.c#insert-build-scan](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L312-L315)). |
| First build, SP-GiST | Yes | The build passes `allow_sync = true` ([spginsert.c#build-scan](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L124-L126)). |
| First build, Bloom ([contrib](../../../glossary.md#contrib)) | Yes | The build passes `allow_sync = true` ([blinsert.c#build-scan](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L142-L145)). |
| First build, GIN | No | The build passes `false`, because GIN's page-filling code prefers to receive tuples in TID order ([gininsert.c#build-scan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L378-L384)). |
| First build, BRIN, serial | No | The build passes `false`, because it must see the heap blocks in physical order, starting at block 0 ([brin.c#serial-scan-order](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1217-L1224)). |
| Validation, every access method | No | The scan passes `false`, because it must read from block zero forward to match the sorted TIDs ([heapam_handler.c#validation-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1804)). |

The three "No" rows depart from the default, so here is why. **Default:** a
large first heap scan may be synchronized, because `synchronize_seqscans` is on
by default. **Cause:** the caller of the scan passes `allow_sync = false`.
**Mechanism:** without that flag the scan clears its synchronization flag and
starts at block 0
([guc_tables.c#synchronize_seqscans](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1766-L1774),
[heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496)).

#### Narrow build settings

Two more groups of settings reach the build only under a narrow condition.

| Setting | Condition under which it acts | Effect | Default and how a change applies |
|---|---|---|---|
| `default_toast_compression` | An index value that is wider than `TOAST_INDEX_TARGET`, in an index column that has no compression method of its own. | It selects the method that compresses the value when the index tuple is formed. | Boot value `pglz`; `lz4` is offered only in a build with LZ4 support. `user` context: session; no reload or restart ([guc_tables.c#default_toast_compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4809-L4818), [guc_tables.c#default_toast_compression_options](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L461-L467)). |
| `max_connections`, `autovacuum_max_workers`, `max_wal_senders`, `max_worker_processes` | A read-stream pin budget that is smaller than `io_combine_limit`. | Together they form `MaxBackends`, which divides `NBuffers` into each backend's share of pins. A small share shrinks the largest read of both heap scans. | Boot values 100, 3, 10 and 8. All have `postmaster` context: restart. `initdb` also probes `max_connections` and writes the result into a new cluster's configuration file ([guc_tables.c#max_connections](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2214-L2222), [guc_tables.c#autovacuum_max_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3398-L3407), [guc_tables.c#max_wal_senders](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2935-L2943), [guc_tables.c#max_worker_processes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3163-L3173), [initdb.c#max_connections-probe](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1153-L1166), [initdb.c#max_connections-write](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1279-L1281)). |

**Wide index values.** An index tuple cannot move a value out of line the way a
heap tuple can with [TOAST](../../../glossary.md#toast). It can only compress
the value in place
([heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68)).
The three parts of the non-default case:

| Part | What the source states |
|---|---|
| Default | When an index tuple is formed, a variable-length value is compressed in place if it is not yet compressed, is larger than `TOAST_INDEX_TARGET`, which is one sixteenth of the largest heap tuple, and belongs to a column whose storage type allows compression. With every setting at its default the method is `pglz` ([indextuple.c#compress-wide-value](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138), [heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68), [guc_tables.c#default_toast_compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4809-L4818)). |
| Cause | The session that runs CIC has set `default_toast_compression` to `lz4`, or the table column has its own compression method, which a plain index column copies ([guc_tables.c#default_toast_compression_options](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L461-L467), [index.c:359](../../../../raw/postgres-17/src/backend/catalog/index.c#L359)). |
| Mechanism | The tuple-forming code compresses with the index column's `attcompression`. When that method is unset, the compression routine substitutes the current `default_toast_compression`. An [expression index](../../../glossary.md#expression-index) column is always unset, because it has no table column to copy from ([indextuple.c#compress-wide-value](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138), [toast_internals.c#toast_compress_datum](../../../../raw/postgres-17/src/backend/access/common/toast_internals.c#L32-L104), [index.c#expression-attcompression](../../../../raw/postgres-17/src/backend/catalog/index.c#L390-L397)). |

The B-tree first build forms its tuples this way when it feeds its sort, in the
leader and in every worker, and the inserts that validation makes form them the
same way
([nbtsort.c#_bt_spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L521-L529),
[tuplesortvariants.c#tuplesort_putindextuplevalues](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L747-L782),
[nbtree.c:192](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L192)).
The GiST and GIN code forms its index tuples through the same function
([gistutil.c#index_form_tuple-call](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L582-L584),
[ginentrypage.c:68](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L68)).
BRIN applies the same rule to a wide summary value. It uses the column's method
only when the summary has the column's data type, and the default method
otherwise
([brin_tuple.c#compress-wide-summary](../../../../raw/postgres-17/src/backend/access/brin/brin_tuple.c#L215-L251)).
The setting therefore changes the CPU time spent on each wide value and the size
of the stored value. It changes nothing for an index whose values stay below the
threshold.

**The pin budget.** A read stream keeps the buffers it has looked ahead to
pinned, and the [buffer manager](../../../glossary.md#buffer-manager) limits how
many buffers one backend may pin for such a batch. The terms of the limit are:

- `NBuffers`: `shared_buffers`, in blocks
  ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270)).
- `MaxBackends`: `max_connections` + `autovacuum_max_workers` +
  `max_worker_processes` + `max_wal_senders` + 2. The 2 stands for the two
  special worker processes
  ([postinit.c#InitializeMaxBackends](../../../../raw/postgres-17/src/backend/utils/init/postinit.c#L565-L588),
  [proc.h#NUM_SPECIAL_WORKER_PROCS](../../../../raw/postgres-17/src/include/storage/proc.h#L424-L431)).
- `NUM_AUXILIARY_PROCS`: 6
  ([proc.h#NUM_AUXILIARY_PROCS](../../../../raw/postgres-17/src/include/storage/proc.h#L433-L443)).
- `REFCOUNT_ARRAY_ENTRIES`: 8
  ([bufmgr.c#REFCOUNT_ARRAY_ENTRIES](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L97-L98)).
- `max_ios`: `effective_io_concurrency` as the stream took it. At this point a
  value of 0 is still 0; the stream replaces it by 1 only afterwards
  ([read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439),
  [read_stream.c#zero-max_ios](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L522-L528)).

```text
want    = max(max_ios * 4, io_combine_limit)
ring    = NBuffers                              scan without a strategy
        = min(NBuffers / 8, 256 kB / BLCKSZ)    scan with the bulk-read strategy
share   = NBuffers / (MaxBackends + NUM_AUXILIARY_PROCS)
          - (pins already in the overflow table + REFCOUNT_ARRAY_ENTRIES)
          and at least 1
budget  = max(1, min(want, ring, share))
largest read, in blocks = min(io_combine_limit, budget)
```

In plain language: the stream wants room for a full read, the ring and the
backend's share of the buffer pool can each give it less, and the read size
follows the smallest of them. All divisions are integer divisions
([read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478),
[freelist.c#GetAccessStrategyPinLimit](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L632-L672),
[freelist.c#ring-sizes](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L545-L571),
[freelist.c#ring-size-cap](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L598-L599),
[bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2107-L2144),
[read_stream.c#distance-after-wait](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L707-L728)).

Two worked examples, derived from the source, not measured. Both assume 8 kB
blocks, a heap larger than `NBuffers / 4`, `max_ios` = 1, and no pins in the
overflow table when the stream is created.

| Input | Example 1 | Example 2 | Where it comes from |
|---|---|---|---|
| `NBuffers` | 16384 | 16384 | Boot value of `shared_buffers` ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270)). |
| `max_connections` | 100 | 1000 | Boot value in example 1 ([guc_tables.c#max_connections](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2214-L2222)); an assumed setting in example 2. |
| Other `MaxBackends` terms | 3 + 8 + 10 + 2 | 3 + 8 + 10 + 2 | Boot values and the constant ([guc_tables.c#autovacuum_max_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3398-L3407), [guc_tables.c#max_worker_processes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3163-L3173), [guc_tables.c#max_wal_senders](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2935-L2943), [proc.h#NUM_SPECIAL_WORKER_PROCS](../../../../raw/postgres-17/src/include/storage/proc.h#L424-L431)). |
| `io_combine_limit` | 16 | 16 | Boot value with 8 kB blocks ([bufmgr.h#io_combine_limit-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L168-L169)). |

| Step | Example 1 | Example 2 |
|---|---|---|
| `MaxBackends` | 100 + 23 = 123 | 1000 + 23 = 1023 |
| `want` | max(4, 16) = 16 | max(4, 16) = 16 |
| `ring` | min(16384 / 8, 32) = 32 | 32 |
| `share` | 16384 / 129 = 127; 127 - 8 = 119 | 16384 / 1029 = 15; 15 - 8 = 7 |
| `budget` | min(16, 32, 119) = 16 | min(16, 32, 7) = 7 |
| Largest read | min(16, 16) = 16 blocks, which is 128 kB | min(16, 7) = 7 blocks, which is 56 kB |

Example 1 shows that with the boot values `io_combine_limit` is the binding
limit. Example 2 shows the non-default case. **Default:** the read size is
bounded by `io_combine_limit`. **Cause:** a small `shared_buffers` relative to
`MaxBackends`, here through `max_connections` = 1000. **Mechanism:** the
backend's share of pins falls below `io_combine_limit`, the stream's look-ahead
distance cannot pass the budget, and a pending read cannot grow past the
distance
([bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2107-L2144),
[read_stream.c#distance-after-wait](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L707-L728),
[read_stream.c#read_stream_look_ahead](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L301-L377)).

The key interaction: the read stream uses `io_combine_limit`, while the buffer
manager's limit uses `NBuffers` and `MaxBackends`. Because the stream takes the
smaller of the two as its ceiling, raising `io_combine_limit` does nothing once
the pin budget is the smaller one. Therefore, in that state, only a larger
`shared_buffers` or a smaller `MaxBackends` can enlarge the reads, and both need
a restart
([read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478),
[guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270),
[guc_tables.c#max_connections](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2214-L2222)).

#### WAL, checkpoints, and the four commits

A CIC on a permanent table writes [WAL](../../../glossary.md#wal) for the
[catalog](../../../glossary.md#catalog) rows of transactions 1, 2 and 4, for the
index pages of the first build, and for whatever validation adds in transaction
3. Its two heap scans can add WAL of their own: page pruning, and hint-bit
records when those are logged
([heapam.c#heap_prepare_pagescan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L601-L690)).
The settings in this subsection never change the CIC state machine. They change
how many WAL bytes those steps produce, when the new index file is forced to
disk, and how long the commits take.

Two facts decide which setting can matter, so they come before the settings
table:

- The first build's WAL path depends on the
  [access method](../../../glossary.md#access-method) (AM). Some builds log
  every page once as a forced image. Others write one ordinary record per
  insertion.
- Only three of the four commits behave like the commit of a writing
  transaction. Transaction 3 normally has no
  [transaction ID](../../../glossary.md#transaction-id) (XID), so its commit
  neither flushes WAL nor waits for a standby.

An index on an unlogged table inherits that persistence, so `RelationNeedsWAL()`
is false for it and its pages are not WAL-logged. The rest of this subsection is
about permanent tables
([index.c#inherited-properties](../../../../raw/postgres-17/src/backend/catalog/index.c#L783-L792),
[rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L620-L631)).

**How the first build writes WAL, by access method.** Three mechanisms appear in
the table below.

- A [full-page image](../../../glossary.md#full-page-image) is a copy of a whole
  page inside a WAL record. For an ordinary record, `XLogRecordAssemble()` adds
  one only when page writes are enabled and the page has not changed since the
  redo pointer of the latest [checkpoint](../../../glossary.md#checkpoint)
  (`page_lsn <= RedoRecPtr`). Page writes are enabled when `full_page_writes` is
  on or a base backup is running. A caller can instead force the image with
  `REGBUF_FORCE_IMAGE`. That flag is tested first, so a forced image depends
  neither on `full_page_writes` nor on checkpoints
  ([xloginsert.c#image-decision](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L626),
  [xlog.c:847](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L847)).
- The [bulk writer](../../../glossary.md#bulk-writer) queues finished pages in
  private memory, logs each batch with `log_newpages()`, which forces the image
  of every page, and then writes the pages to the file itself
  ([bulk_write.c#smgr_bulk_flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L313),
  [xloginsert.c#log_newpages](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1169-L1224)).
- `log_newpage_range()` is the after-the-fact variant. The build fills pages in
  [shared buffers](../../../glossary.md#buffer-manager) without writing WAL, and
  one pass at the end of `ambuild` logs every initialized page as a forced image
  ([xloginsert.c#log_newpage_range](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1252-L1342)).

| Index AM | How transaction 2 logs index pages | Effect of `full_page_writes` | Effect of checkpoints during the build | Effect of `wal_compression` | Checkpoint-crossing sync |
|---|---|---|---|---|---|
| [B-tree](../../../glossary.md#b-tree) | Bulk writer, started and finished by `_bt_load()` ([nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149), [nbtsort.c:1376](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1376)) | None: every image is forced | No change in image count, because a bulk write writes and logs each block once ([bulk_write.c#smgr_bulk_write](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L315-L335)) | Every page image is a candidate | Yes |
| [GiST](../../../glossary.md#gist), [sorted build](../../../glossary.md#gist-build-method) | Bulk writer ([gistbuild.c#gist_indexsortbuild](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L396-L454)) | None: every image is forced | No change in image count | Every page image is a candidate | Yes |
| GiST, buffered or plain insertion build | No WAL while building, then `log_newpage_range()` over the whole index ([gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338), [gistbuild.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L328-L337)) | None: every image is forced | No change in image count, because one pass logs each page once | Every page image is a candidate | No |
| [GIN](../../../glossary.md#gin) | No WAL while building, then `log_newpage_range()` over the whole index ([gininsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L408-L417)) | None: every image is forced | No change in image count | Every page image is a candidate | No |
| [SP-GiST](../../../glossary.md#sp-gist) | No WAL while building, then `log_newpage_range()` over the whole index ([spginsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L132-L141)) | None: every image is forced | No change in image count | Every page image is a candidate | No |
| [Hash](../../../glossary.md#hash-index) | `_hash_init()` logs each initial bucket page once as a forced image. After that, every insertion writes an ordinary `XLOG_HASH_INSERT` record, with or without the build's pre-sort ([hashpage.c#initial-bucket-loop](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L415-L437), [xloginsert.c#log_newpage](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1130-L1167), [hashsort.c#_h_indexbuild](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L115-L157), [hash.c#hashbuildCallback](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L206-L242), [hashinsert.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/hash/hashinsert.c#L215-L235)) | Decides whether an insertion record can carry a first-change image | Each checkpoint makes the next change to every page carry an image again | Only the images that are written | No |
| [BRIN](../../../glossary.md#brin) | Every summary tuple insertion writes an ordinary `XLOG_BRIN_INSERT` record, in a serial build and in the leader's merge of a parallel build ([brin.c#form_and_insert_tuple](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1971-L1989), [brin.c#parallel-merge-insert](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2693-L2702), [brin_pageops.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L425-L449)) | Decides whether an insertion record can carry a first-change image | Each checkpoint makes the next change to every page carry an image again | Only the images that are written | No |
| [contrib](../../../glossary.md#contrib) Bloom | Each filled build page is written through a generic WAL record that forces the full image ([blinsert.c#flushCachedPage](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L43-L58), [generic_xlog.c#GenericXLogFinish](../../../../raw/postgres-17/src/backend/access/transam/generic_xlog.c#L332-L436)) | None: every image is forced | No change in image count | Every page image is a candidate | No |

The table has three consequences.

First, turning `full_page_writes` off does not shrink the WAL of a forced-image
build, because the force flag is tested before the page-writes test. It only
removes the first-change images of hash and BRIN builds
([xloginsert.c#image-decision](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L626)).

Second, checkpoint frequency changes the volume of index-page WAL only for hash
and BRIN. Their insertion records register the page without the force flag, so
each checkpoint that starts during the build makes the next change to every page
carry a full image again
([hashinsert.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/hash/hashinsert.c#L215-L235),
[brin_pageops.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L425-L449),
[xloginsert.c#image-decision](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L626)).

Third, the bulk writer has a checkpoint rule of its own. It is a linear chain:

1. `smgr_bulk_start_rel()` decides `use_wal` from `RelationNeedsWAL()`, and
   `smgr_bulk_start_smgr()` saves the redo pointer that is current when the bulk
   write starts (`start_RedoRecPtr`)
   ([bulk_write.c#smgr_bulk_start_rel](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L84-L93),
   [bulk_write.c#smgr_bulk_start_smgr](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L95-L122)).
2. Each `smgr_bulk_write()` queues one page. When `MAX_PENDING_WRITES` pages are
   queued, `smgr_bulk_flush()` logs the batch and writes the pages with
   `skipFsync = true`, so these writes are not registered with the
   [checkpointer](../../../glossary.md#checkpointer) one by one
   ([bulk_write.c#smgr_bulk_write](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L315-L335),
   [bulk_write.c:48](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L48),
   [bulk_write.c#smgr_bulk_flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L313)).
3. `smgr_bulk_finish()` flushes the remaining pages and compares the saved redo
   pointer with the current one. If they are equal, it registers the whole file,
   so that the next checkpoint fsyncs it. If they differ, a checkpoint started
   during the bulk write and "already missed fsyncing the pages we had written
   before the checkpoint started", so the
   [backend](../../../glossary.md#backend) fsyncs the index file itself with
   `smgrimmedsync()`
   ([bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223)).

Therefore the settings that control checkpoint frequency decide whether a B-tree
or sorted-GiST CIC ends its first build with a synchronous fsync of the whole
new index file. `_bt_load()` and `gist_indexsortbuild()` are the only shipped
builds that start a bulk write on the index's main fork, so the rule does not
apply to the other rows
([nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149),
[gistbuild.c#gist_indexsortbuild](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L396-L454)).

**The four commits.** CIC commits three times inside `DefineIndex()`, after
transactions 1, 2 and 3. The fourth commit is the ordinary end-of-statement
commit of transaction 4
([indexcmds.c:1618](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1618),
[indexcmds.c:1690](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1690),
[indexcmds.c:1750](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1750),
[postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821)).
Every one of them runs `RecordTransactionCommit()`. What that function does
depends on two values that are current at commit time: whether the transaction
has an XID, and whether it wrote WAL.

```text
RecordTransactionCommit()
  |
  [A] Does the transaction have an XID?
  |
  +-- no ---- [B] Did it write any WAL?
  |             +-- no  -> nothing to write, flush or wait for
  |             +-- yes -> no commit record; report the LSN to the
  |                        WAL writer (asynchronous branch);
  |                        no standby wait
  |
  +-- yes --- [C] write a commit record
                |
                [D] Is synchronous_commit above "off"?
                |     +-- yes -> XLogFlush() through the commit record
                |     +-- no  -> report the LSN to the WAL writer
                |                (asynchronous branch)
                |
                [E] SyncRepWaitForLSN(): returns at once unless
                    synchronous_commit is above "local",
                    max_wal_senders > 0 and
                    synchronous_standby_names defines a standby
```

- `[A]` The XID test:
  [xact.c#markXidCommitted](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1306-L1307).
- `[B]` Without an XID there is no commit record, and the function leaves early
  if no WAL was written. If WAL was written, the flush test below fails on the
  missing XID, so the commit takes the asynchronous branch, and the standby wait
  is skipped for the same reason
  ([xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395),
  [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521),
  [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546)).
- `[C]` The commit record:
  [xact.c#commit-record](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1428-L1437).
- `[D]` The flush test:
  [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521).
  The same test also flushes after `ForceSyncCommit()` or when files must be
  deleted at commit. Neither applies to CIC. The only callers of
  `ForceSyncCommit()` are tablespace and database commands, and the new index
  file is registered for deletion on abort only
  ([xact.c#ForceSyncCommit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1140-L1152),
  [tablespace.c:379](../../../../raw/postgres-17/src/backend/commands/tablespace.c#L379),
  [tablespace.c:553](../../../../raw/postgres-17/src/backend/commands/tablespace.c#L553),
  [dbcommands.c:1526](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L1526),
  [dbcommands.c:1855](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L1855),
  [dbcommands.c:2224](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L2224),
  [storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L107-L180)).
- `[E]` The standby wait and its fast exit:
  [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546),
  [syncrep.c#fast-exit](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L159-L181),
  [syncrep.h#SyncRepRequested](../../../../raw/postgres-17/src/include/replication/syncrep.h#L18-L19).

| Transaction | WAL it writes | Has an XID? | Commit with `synchronous_commit = on` (the boot value) | Commit with `synchronous_commit = off` | Can wait for a synchronous standby? |
|---|---|---|---|---|---|
| 1 | The new index's catalog rows and its file-creation record ([storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L107-L180)) | Yes. The first catalog insert assigns one ([heapam.c:2111](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2111)) | Commit record, then `XLogFlush()` | Commit record; the flush is left to the [WAL writer](../../../glossary.md#wal-writer) | Yes |
| 2 | The first build's index pages (table above) and the [pg_index](../../../glossary.md#pg_index) update that sets `indisready` | Yes. `index_set_state_flags()` updates the row through `CatalogTupleUpdate()`, and a heap update assigns an XID ([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557), [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550), [heapam.c:3352](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3352)) | Commit record, then `XLogFlush()`. The flush covers all WAL up to the commit record, so it includes any part of the first build's WAL that is not on disk yet ([xlog.c#XLogFlush-contract](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2771-L2776)) | Commit record; the flush is left to the WAL writer | Yes. The standby has to confirm the position of the commit record, which lies after the first build's WAL ([syncrep.c#SyncRepWaitForLSN](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L133-L363)) |
| 3 | Whatever validation produces, for example insertions of missing tuples, a GIN pending-list cleanup, and page [pruning](../../../glossary.md#pruning) during the heap scan ([heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974), [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602), [heapam.c#heap_prepare_pagescan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L601-L690)) | Normally no. Transaction 3 only waits, takes a snapshot and runs `validate_index()`, which updates no catalog row ([indexcmds.c#transaction-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1687-L1751), [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)) | No commit record and no flush. If WAL was written, the commit only reports its [LSN](../../../glossary.md#lsn) to the WAL writer ([xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521)) | The same. The missing XID decides, not the setting | No ([xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546)) |
| 4 | The `pg_index` update that sets `indisvalid` | Yes, through the same `index_set_state_flags()` path ([indexcmds.c:1773](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1773)) | Commit record, then `XLogFlush()`. That flush also covers any WAL of transaction 3 that is still not on disk | Commit record; the flush is left to the WAL writer | Yes |

Two things keep transaction 3 without an XID. It updates no catalog row, and
inserting into an index does not assign one. A search of the shipped access
methods, `index.c` and the heap validation scan for `GetCurrentTransactionId()`
and `GetTopTransactionId()` finds only a comment, the one that explains why
B-tree page deletion works without an XID
([nbtpage.c#no-xid-comment](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2637)).

The key interaction is between the XID and the setting. Commits 1, 2 and 4 read
`synchronous_commit`, because each of those transactions changed a catalog row
and therefore has an XID. Commit 3 never reaches that question, because
`RecordTransactionCommit()` requires an XID before it considers a synchronous
flush or a standby wait. Therefore `synchronous_commit` and
`synchronous_standby_names` can lengthen three commits, not four, and commit 2
is the one that has the first build's WAL in front of its commit record.

"Normally" has one exception. Validation evaluates an
[index expression](../../../glossary.md#expression-index) or a
[partial-index](../../../glossary.md#partial-index) predicate only for a tuple
that is missing from the index. If that happens and the function assigns an XID,
transaction 3 has an XID after all, and its commit then behaves like the other
three. The regression suite contains such a predicate function, which runs
`txid_current()` during a concurrent build
([heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974),
[create_index.sql#xid-assigning-predicate](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L509-L520)).

**WAL, commit and checkpoint settings.** With those two facts, each setting has
a precise scope:

| GUC | Exact CIC effect and boundary | Default and application |
|---|---|---|
| `synchronous_commit` | Read by the commits of transactions 1, 2 and 4 only. With `on`, each of them flushes WAL through its commit record before it returns. With `off`, they skip that flush and the standby wait ([asynchronous commit](../../../glossary.md#asynchronous-commit)). `local` keeps the local flush and never waits for a standby. `remote_write`, `on` and `remote_apply` also ask a synchronous standby to confirm write, flush or apply. The setting changes commit latency and durability or confirmation semantics, not scan or sort work ([xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.h#SyncCommitLevel](../../../../raw/postgres-17/src/include/access/xact.h#L68-L80), [syncrep.c#assign_synchronous_commit](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L1120-L1138), [guc_tables.c#synchronous_commit_options](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L340-L357)) | `on`; `user` context; session `SET`; no reload or restart ([guc_tables.c#synchronous_commit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4925-L4933)) |
| `synchronous_standby_names` | Decides whether those three commits wait for a standby at all ([synchronous replication](../../../glossary.md#synchronous-replication)). The wait returns at once unless `synchronous_commit` is above `local`, `max_wal_senders` is above zero and this list defines a synchronous standby. With the boot value, an empty list, no commit waits ([syncrep.c#fast-exit](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L159-L181), [syncrep.h#SyncRepRequested](../../../../raw/postgres-17/src/include/replication/syncrep.h#L18-L19)) | Empty string; `sighup` context; reload ([guc_tables.c#synchronous_standby_names](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4571-L4580)) |
| `wal_compression` | Applies to every WAL record that carries a page image: the forced images of the bulk-writer, `log_newpage_range()` and Bloom builds, and the first-change images of hash and BRIN builds. When it is not `off`, `XLogRecordAssemble()` tries to compress each image and keeps the result only if it is smaller. It trades CPU in the process that writes the record for fewer WAL bytes; the gain depends on the data and on the method (`pglz`, or `lz4` or `zstd` in a build that has the library) ([xloginsert.c#image-compression](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L685-L695), [xloginsert.c#XLogCompressBackupBlock](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L936-L1017)) | `off`; `superuser` context; session `SET` only for a role allowed to change it; no reload or restart ([guc_tables.c#wal_compression](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4976-L4984)) |
| `wal_buffers`; `wal_sync_method` | Generic WAL-path controls. `wal_buffers` sizes the shared buffer that the command's WAL passes through. `wal_sync_method` selects how WAL is forced to disk, which is what the flush of a synchronous commit does. Neither determines worker count or scan shape ([guc_tables.c#wal_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2891-L2900), [guc_tables.c#wal_sync_method](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5026-L5034)) | `wal_buffers`: `-1`, which derives the size from `shared_buffers`; `postmaster` context; restart. `wal_sync_method`: the platform default; `sighup` context; reload ([xlogdefs.h#DEFAULT_WAL_SYNC_METHOD](../../../../raw/postgres-17/src/include/access/xlogdefs.h#L68-L81)) |
| `max_wal_size`; `checkpoint_timeout` | Decide how often automatic checkpoints start: `checkpoint_timeout` is the maximum time between them, and `max_wal_size` is the WAL size that triggers one earlier. For CIC that frequency has the two build-specific effects shown above. A checkpoint that starts during a bulk-writer build makes `smgr_bulk_finish()` fsync the index file itself, and every checkpoint during a hash or BRIN build brings back first-change images. Both settings are cluster-wide. Raising them can also lengthen crash recovery, and whether one particular build is hit depends on when it runs relative to the checkpoint schedule, so the effect on a CIC is not monotonic ([config.sgml#max_wal_size](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3702-L3723), [config.sgml#checkpoint_timeout](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3605-L3623), [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223)) | `max_wal_size`: boot value 1 GB (64 segments of 16 MB); `initdb` also writes it into the new `postgresql.conf` as 64 segments of the cluster's WAL segment size, which is the same value unless another segment size was chosen. `checkpoint_timeout`: 300 s. Both `sighup` context; reload ([guc_tables.c#max_wal_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2842-L2852), [xlog_internal.h:92](../../../../raw/postgres-17/src/include/access/xlog_internal.h#L92), [pg_config_manual.h:20](../../../../raw/postgres-17/src/include/pg_config_manual.h#L20), [initdb.c#wal-size-settings](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1336-L1341), [initdb.c#pretty_wal_size](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1243-L1258), [guc_tables.c#checkpoint_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2854-L2863)) |
| `checkpoint_completion_target`; `checkpoint_flush_after`; `backend_flush_after` | Pace data-file writes that run beside the build. The first spreads a checkpoint's writes over that fraction of the checkpoint interval. The second makes the checkpointer ask the operating system to start writeback after that much written data. The third is the same request for buffers that the CIC backend has to write itself, which happens when it must evict a [dirty buffer](../../../glossary.md#dirty-buffer). That can include the build's own index pages for every AM that fills pages in shared buffers, which is every row above except the two bulk-writer builds. With the boot value 0 the backend issues no writeback request ([config.sgml#checkpoint_completion_target](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3625-L3645), [config.sgml#checkpoint_flush_after](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3647-L3677), [bufmgr.c#dirty-victim-write](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1988-L2058), [bufmgr.c#ScheduleBufferTagForWriteback](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5975-L6007)) | `checkpoint_completion_target`: 0.9. `checkpoint_flush_after`: 32 blocks where `sync_file_range()` exists, which the source notes is only Linux, otherwise 0. Both `sighup` context; reload. `backend_flush_after`: 0 on every platform; `user` context; session `SET`; no reload or restart ([guc_tables.c#checkpoint_completion_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3916-L3924), [guc_tables.c#checkpoint_flush_after](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2880-L2889), [guc_tables.c#backend_flush_after](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3152-L3161), [pg_config_manual.h#flush-after-defaults](../../../../raw/postgres-17/src/include/pg_config_manual.h#L165-L179)) |
| `commit_delay`; `commit_siblings` | The [group-commit delay](../../../glossary.md#commit_delay). When `commit_delay` is above zero, `XLogFlush()` sleeps that many microseconds before it flushes, but only if `fsync` is on and at least `commit_siblings` other backends have active transactions. The sleep lets more commits share one flush. It can raise aggregate commit throughput while it adds latency to each CIC commit that flushes. The boot values add no delay ([xlog.c#group-commit-delay](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2870-L2895)) | `commit_delay`: 0; `superuser` context; session `SET` only for a role allowed to change it. `commit_siblings`: 5; `user` context; session `SET`. No reload or restart ([guc_tables.c#commit_delay](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2980-L2990), [guc_tables.c#commit_siblings](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2992-L3001)) |
| `wal_log_hints`; data checksums | Make the two heap scans write WAL of their own. Both scans test tuple visibility page by page and can set [hint bits](../../../glossary.md#hint-bits) on tuples that lack them. With either feature on, the first such change to a page after a checkpoint writes an `XLOG_FPI_FOR_HINT` record, which carries the page image while `full_page_writes` is on. With both off, which is the default, setting a hint bit writes no WAL. A page whose all-visible flag is set skips the per-tuple test and therefore sets no hint bits ([heapam.c#heap_prepare_pagescan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L601-L690), [heapam.c#page_collect_tuples](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L551-L599), [heapam_visibility.c#SetHintBits](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L82-L132), [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5036-L5182), [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1043-L1128), [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L109-L118)) | `wal_log_hints`: off; `postmaster` context; restart. [Data checksums](../../../glossary.md#data-checksums) are a cluster property chosen when `initdb` creates the cluster (`-k`, off unless requested in this version). The read-only `data_checksums` parameter reports them; its context is `internal`, so it cannot be changed ([guc_tables.c#wal_log_hints](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1170-L1178), [initdb.c:167](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L167), [initdb.c#data-checksums-option](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L3293-L3295), [guc_tables.c#data_checksums](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1882-L1891)) |

The [fsync](../../../glossary.md#fsync) and `full_page_writes` settings both
default to on. Both have the `sighup` context, so a change needs a reload.
Turning either off changes generic durability I/O, but neither is a production
CIC tuning method. In particular, turning `full_page_writes` off does not remove
the forced images that B-tree, GiST, GIN, SP-GiST and Bloom builds write. It
only removes the first-change images of hash and BRIN builds and of the hint-bit
records. PostgreSQL's documentation warns that turning either setting off can
cause unrecoverable or silent corruption after a system failure
([guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1096-L1107),
[guc_tables.c#full_page_writes](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1156-L1168),
[xloginsert.c#image-decision](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L626),
[config.sgml#fsync](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3022-L3082),
[config.sgml#full_page_writes](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3294-L3339)).

#### Wait and cancellation settings

These settings do not make the build faster. They bound how long the command may
run or wait, decide when a lock wait is examined for a
[deadlock](../../../glossary.md#deadlock) and when a blocking
[autovacuum](../../../glossary.md#autovacuum) worker is cancelled, and log long
waits.

**Where CIC waits for a lock.** Every wait in the next table is a
[heavyweight lock](../../../glossary.md#heavyweight-lock) wait inside
`ProcSleep()`, so the same timers apply to each of them.

| Lock wait | What CIC waits for | Source |
|---|---|---|
| Table lock at the start | `ShareUpdateExclusiveLock` on the table, requested by `RangeVarGetRelidExtended()` before `DefineIndex()` runs. It queues behind any conflicting holder, for example `VACUUM`, `ANALYZE`, an autovacuum worker, another CIC or DDL; see [All steps and locks required on the table](#all-steps-and-locks-required-on-the-table) | [utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480) |
| Wait 1 and wait 2 | One [virtual transaction ID](../../../glossary.md#virtual-transaction-id) (VXID) lock for each transaction that holds a conflicting table lock, one after the other. For a prepared transaction ([two-phase commit](../../../glossary.md#two-phase-commit)) the wait ends up on its XID lock | [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973), [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663), [lock.c#XactLockForVirtualXact](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4497-L4548) |
| Validation of a unique index | The XID lock of another transaction. It happens when a tuple that validation inserts conflicts with an index entry whose inserting or deleting transaction is still in progress | [heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974), [nbtinsert.c#unique-conflict-wait](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L213-L233), [lmgr.c#XactLockTableWait](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L642-L724) |
| Wait 3 | One VXID lock for each transaction whose [snapshot](../../../glossary.md#snapshot) is older than the reference snapshot, one after the other | [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497) |

The table locks that `index_concurrently_build()` and `validate_index()` take
later are not on this list. The session already holds the same lock mode, and a
lock that the backend already holds is granted locally without a wait
([index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557),
[index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452),
[lock.c#already-held](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L876-L889)).

**What happens during one lock wait.**

```text
CIC must wait for a heavyweight lock -> ProcSleep()
  |
  [A] arm the deadlock_timeout timer;
      also arm the lock_timeout timer if lock_timeout > 0
  |
  +-- the lock is granted first ...... disable both timers; continue    [B]
  +-- lock_timeout fires first ....... ERROR: canceling statement
  |                                    due to lock timeout             [C]
  +-- deadlock_timeout fires first ... run the deadlock check once     [D]
        +-- hard deadlock ............ ERROR: deadlock detected        [E]
        +-- blocked directly by an autovacuum worker that is not
        |   protecting against wraparound
        |   .......................... cancel that worker; keep waiting [F]
        +-- log_lock_waits is on ..... LOG "still waiting" now and
        |                              "acquired" when the lock arrives [G]
        +-- otherwise ................ keep waiting
```

- `[A]`
  [proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333)
- `[B]`
  [proc.c#timer-disable](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1646-L1666)
- `[C]`
  [postgres.c#query-cancel-errors](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3372-L3430)
- `[D]`
  [proc.c#deadlock-check-wakeup](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1391-L1403),
  [proc.c#CheckDeadLock](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1782-L1870)
- `[E]`
  [deadlock.c#DeadLockReport](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L1068-L1136)
- `[F]`
  [deadlock.c#blocking-autovacuum](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L594-L618),
  [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493)
- `[G]`
  [proc.c#lock-wait-logging](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1495-L1643)

Both timers belong to one lock acquisition. They start again for the next one,
so neither of them measures the whole of wait 1, 2 or 3 when that wait has
several blockers.

For CIC, branch `[F]` arises at the table lock at the start. Lazy
[VACUUM](../../../glossary.md#vacuum) and `ANALYZE`, including the ones
autovacuum runs, hold `ShareUpdateExclusiveLock` on the table, which conflicts
with CIC's request
([vacuum.c#vacuum_rel-lock-mode](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2059),
[analyze.c#analyze_rel-lock](../../../../raw/postgres-17/src/backend/commands/analyze.c#L134-L145)).
During waits 1 and 2 CIC already holds that lock at
[session level](../../../glossary.md#session-level-lock), so no autovacuum
worker can hold a conflicting lock on the table, and wait 3 skips autovacuum
workers
([indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497)).
The documentation describes the interruption but not its timing; the source
gives the timing, which is one `deadlock_timeout`
([maintenance.sgml#autovacuum-interruption](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L995-L1005),
[deadlock.c#blocking-autovacuum](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L594-L618)).

**Which timer bounds the whole command.** `statement_timeout` and
`transaction_timeout` interact, and CIC makes the interaction visible because it
is one statement but four transactions.

- The statement timer is armed once, by `start_xact_command()` when the
  statement arrives, and disarmed by `finish_xact_command()` when it ends. CIC's
  internal commits call `CommitTransactionCommand()` directly, so they do not
  touch it
  ([postgres.c#start_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2796),
  [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821),
  [indexcmds.c:1618](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1618)).
- `enable_statement_timeout()` arms it only when `statement_timeout` is above
  zero and either `transaction_timeout` is zero or `statement_timeout` is the
  smaller of the two. Otherwise it disables the timer
  ([postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268)).
- The transaction timer is armed by every `StartTransaction()` and disarmed by
  every commit, so CIC gets a new one four times
  ([xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2174-L2176),
  [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2315-L2317)).

| `statement_timeout` | `transaction_timeout` | Statement timer | Transaction timer | What bounds the whole CIC |
|---|---|---|---|---|
| 0 (boot value) | 0 (boot value) | Not armed | Not armed | Nothing |
| Above 0 | 0 | Armed once; survives the three internal commits | Not armed | `statement_timeout`, end to end |
| Above 0 and smaller than `transaction_timeout` | Above 0 | Armed once; survives the three internal commits | Armed again for each of the four transactions | `statement_timeout` end to end, and each transaction also has its own limit |
| Above 0 and not smaller than `transaction_timeout` | Above 0 | Not armed | Armed again for each of the four transactions | Only the per-transaction limit. The command can use up to four full budgets, and nothing caps it end to end |
| 0 | Above 0 | Not armed | Armed again for each of the four transactions | Only the per-transaction limit |

The fourth row is the surprising one. The session has a `statement_timeout`, but
`transaction_timeout` is not larger, so the statement timer is never armed. For
an ordinary single-transaction statement that loses nothing, because the
transaction timer fires first. For CIC it removes the only end-to-end cap,
because the transaction timer restarts three times. The documentation states the
rule ("the longer timeout is ignored") and describes the limit as applying to
"an implicitly started transaction corresponding to a single statement", which
does not describe a command that commits internally
([config.sgml#transaction_timeout](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9498-L9532),
[postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268)).

| GUC | CIC behavior | Default and application |
|---|---|---|
| `statement_timeout` | When armed (table above), it covers the client statement across the internal commits, both scans and every wait. Expiry cancels the statement with `ERROR: canceling statement due to statement timeout`. It is an end-to-end cap, not a per-transaction budget; see [statement and lock timeouts](../../../glossary.md#statement_timeout-and-lock_timeout) ([postgres.c#start_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2796), [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821), [postgres.c#query-cancel-errors](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3372-L3430)) | 0 (disabled); `user` context; session `SET`; no reload or restart ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)) |
| `transaction_timeout` | A fresh timer for each of the four transactions. Each wait runs inside the transaction that follows it (wait 1 in transaction 2, wait 2 in transaction 3, wait 3 in transaction 4), so a wait and the work after it share one budget. Expiry terminates the connection with `FATAL: terminating connection due to transaction timeout`; see [transaction and idle-in-transaction timeouts](../../../glossary.md#transaction_timeout-and-idle_in_transaction_session_timeout) ([xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2174-L2176), [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2315-L2317), [indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658), [indexcmds.c:1705](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1705), [indexcmds.c:1768](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1768), [postgres.c#transaction-timeout-fatal](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3464)) | 0 (disabled); `user` context; session `SET`; no reload or restart ([guc_tables.c#transaction_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653)) |
| `lock_timeout` | Applies separately to every lock wait listed above: the table lock at the start, each VXID or XID lock of waits 1 to 3, and each unique-validation wait. The timer restarts for every lock acquisition, so it is not an aggregate cap across blockers or across the three waits. Expiry cancels the statement with `ERROR: canceling statement due to lock timeout`. An in-tree test shows it: with `lock_timeout = 10`, a CIC that is blocked in wait 1 behind a prepared transaction is cancelled ([proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333), [postgres.c#query-cancel-errors](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3372-L3430), [prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37), [prepared-transactions-cic.out#expected-output](../../../../raw/postgres-17/src/test/isolation/expected/prepared-transactions-cic.out#L1-L19)) | 0 (disabled); `user` context; session `SET`; no reload or restart ([guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)) |
| `deadlock_timeout` | The time each lock wait lasts before the deadlock check runs. A shorter value reports a real deadlock sooner and runs the check for more waits. A real deadlock is possible in the waits: the source uses real lock acquisition there because a transaction CIC waits for may itself be queued for a lock on the table. The value can also end one kind of non-deadlocked wait: it is the time CIC queues behind an autovacuum worker at the start before it cancels that worker (branch `[F]`). A manual `VACUUM` or `ANALYZE`, and an autovacuum that protects against wraparound, are not cancelled ([proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333), [indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658), [deadlock.c#blocking-autovacuum](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L594-L618), [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493), [config.sgml#deadlock_timeout](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L10564-L10608)) | 1000 ms; `superuser` context; session `SET` only for a role allowed to change it; no reload or restart ([guc_tables.c#deadlock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2147-L2157)) |
| `log_lock_waits` | Logs a lock wait that is still pending when the deadlock check has run, with the processes that hold the lock and the wait queue, and logs again when the lock is acquired. It changes what is reported, not how long CIC waits ([proc.c#lock-wait-logging](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1495-L1643), [config.sgml#log_lock_waits](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L7792-L7808)) | off; `superuser` context; session `SET` only for a role allowed to change it; no reload or restart ([guc_tables.c#log_lock_waits](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1513-L1521)) |
| `idle_in_transaction_session_timeout`; a blocker's own `transaction_timeout` | Blocker-side policy. The idle timer never fires in the CIC backend: it is armed only when a backend goes idle inside a transaction block, and CIC cannot run in one. In another session it terminates a blocker that sits idle in a transaction, which ends a CIC wait on it. It is armed only if it is smaller than that session's `transaction_timeout` or that setting is zero. The blocker's own `transaction_timeout` terminates it whether it is idle or active. Neither is a CIC speed setting ([postgres.c#idle-in-transaction-timer](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4620-L4647), [utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463), [postgres.c#idle-in-transaction-fatal](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3435-L3451), [postgres.c#transaction-timeout-fatal](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3464)) | Both 0 (disabled); `user` context; session `SET` in the blocker's session; no reload or restart ([guc_tables.c#idle_in_transaction_session_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2633-L2642), [guc_tables.c#transaction_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653)) |

A timeout that fires after transaction 1 has committed leaves the index behind
as an [invalid index](../../../glossary.md#invalid-index), like any other error
at that point; see [Failure handling](#failure-handling). `statement_timeout`
and `lock_timeout` raise an `ERROR`, which aborts the current internal
transaction and returns control to the session. `transaction_timeout` raises
`FATAL`, which also ends the session. Before the first commit, any of them rolls
the whole command back and leaves no index. A timeout is therefore a completion
policy, not free acceleration
([indexcmds.c:1618](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1618),
[postgres.c#query-cancel-errors](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3372-L3430),
[postgres.c#transaction-timeout-fatal](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3464)).

#### Settings that are easy to confuse with CIC controls

- `min_parallel_index_scan_size` does not affect the worker request.
  `plan_create_index_workers()` passes `index_pages = -1` to
  `compute_parallel_worker()`, and that function consults the index threshold
  only when `index_pages` is zero or more. Only `min_parallel_table_scan_size`
  takes part
  ([planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998),
  [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4187-L4279)).
- `max_parallel_workers_per_gather` and `parallel_leader_participation` control
  the `Gather` and `Gather Merge` nodes of a
  [parallel query](../../../glossary.md#parallel-query), not a
  [parallel index build](../../../glossary.md#parallel-index-build). Whether the
  leader takes part in a B-tree or BRIN build is fixed at compile time:
  `leaderparticipates` starts true, and only a build with
  `DISABLE_LEADER_PARTICIPATION` defined turns it off
  ([config.sgml#max_parallel_workers_per_gather](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2822-L2862),
  [config.sgml#parallel_leader_participation](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2923-L2944),
  [nbtsort.c#leader-participation-switch](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1413-L1415),
  [brin.c#leader-participation-switch](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2370-L2372)).
- `maintenance_io_concurrency` is not selected by CIC's heap
  [read streams](../../../glossary.md#read-stream). The heap scan opens its
  stream with `READ_STREAM_SEQUENTIAL` only, and the stream reads the
  maintenance setting only for a caller that passes `READ_STREAM_MAINTENANCE`.
  The stream therefore takes the branch described in the
  `effective_io_concurrency` row above. In this checkout the maintenance setting
  has three readers: `ANALYZE`'s block-sampling stream, the heap
  [prefetch](../../../glossary.md#prefetch) that accompanies index-tuple
  deletion, and WAL prefetching during recovery. None of them is one of CIC's
  two heap scans
  ([heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439),
  [analyze.c#block-sampling-read-stream](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1199-L1205),
  [heapam.c#index-delete-prefetch-distance](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L8547-L8558),
  [xlogprefetcher.c#recovery-prefetch-limit](../../../../raw/postgres-17/src/backend/access/transam/xlogprefetcher.c#L1000-L1005)).
- [work_mem](../../../glossary.md#work_mem) does not size the validation sort;
  that sort is given
  [maintenance_work_mem](../../../glossary.md#maintenance_work_mem). A unique
  B-tree build does create a secondary spool sized by `work_mem` for dead
  tuples. In a concurrent build that spool stays empty: the scan uses an
  [MVCC](../../../glossary.md#mvcc) snapshot, every tuple it returns is flagged
  alive, the callback sends alive tuples to the primary spool, and the unused
  secondary spool is destroyed
  ([index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452),
  [nbtsort.c#secondary-spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L471),
  [heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629),
  [nbtsort.c#_bt_build_callback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L573-L600),
  [nbtsort.c#spool2-discard](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L500-L506)).

  `work_mem` does matter for a GIN index with `fastupdate` on, which is the
  default, and in two kinds of backend. Concurrent writers append to the
  [pending list](../../../glossary.md#pending-list) once `indisready` is
  committed. The CIC backend does the same for every tuple that validation
  inserts, because those insertions go through the ordinary `gininsert()`.
  Whichever backend pushes the list past `gin_pending_list_limit`, or past the
  index's [storage parameter](../../../glossary.md#storage-parameter) of that
  name, starts a regular cleanup sized by its own `work_mem`, unless another
  cleanup of that index is already running. Only the forced cleanup at the start
  of validation uses `maintenance_work_mem`; see the GIN row in the build table
  above
  ([gininsert.c#gininsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L482-L536),
  [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471),
  [gin_private.h#GIN-options](../../../../raw/postgres-17/src/include/access/gin_private.h#L23-L45),
  [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828),
  [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)).
- `autovacuum_work_mem`, the `vacuum_cost_*` settings and `temp_buffers` neither
  size nor throttle CIC.
  - `autovacuum_work_mem` is chosen for a forced GIN cleanup only inside an
    autovacuum worker. The CIC backend gets `maintenance_work_mem`
    ([ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)).
  - The [vacuum cost delay](../../../glossary.md#vacuum-cost-delay) code is
    reachable, because validation calls each AM's `ambulkdelete` and those loops
    call `vacuum_delay_point()`; B-tree's does, for example. It never sleeps
    there. The function returns before any delay unless `VacuumCostActive` is
    true. That flag starts false. Only `VacuumUpdateCosts()` sets it, and that
    function serves autovacuum workers and backends that run `VACUUM` or
    `ANALYZE`. `vacuum()` clears the flag again when its command ends
    ([nbtree.c:1096](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1096),
    [vacuum.c#vacuum_delay_point](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2383-L2464),
    [globals.c:159](../../../../raw/postgres-17/src/backend/utils/init/globals.c#L159),
    [autovacuum.c#VacuumUpdateCosts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1630-L1695),
    [vacuum.c#cost-accounting-scope](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L599-L684)).
  - `temp_buffers` sizes the buffers of temporary tables, and a CIC on a
    temporary table has already fallen back to a non-concurrent build
    ([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615)).
- `wal_level = minimal` does not give CIC the WAL skip that a plain
  `CREATE INDEX` gets at that [WAL level](../../../glossary.md#wal-level).
  - Default: the boot value is `replica` and the context is `postmaster`, so a
    change needs a restart. At `replica` or above, `XLogIsNeeded()` is true, and
    `RelationNeedsWAL()` is then true for every permanent relation
    ([guc_tables.c#wal_level](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4986-L4994),
    [xlog.h#XLogIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L103-L107),
    [rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L620-L631)).
  - Cause and mechanism at `minimal`: `XLogIsNeeded()` is false, so
    `RelationNeedsWAL()` is true only for a relation that was neither created
    nor given a new file in the current transaction. A plain `CREATE INDEX`
    creates and builds the index in one transaction, so its build skips WAL. CIC
    creates the index in transaction 1 without building it, and builds it in
    transaction 2. The commit in between resets the
    [relcache](../../../glossary.md#relcache) fields that mark the relation as
    new. In transaction 2 `RelationNeedsWAL()` is therefore true again, and the
    first build is logged exactly as at `replica`
    ([rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L620-L631),
    [indexcmds.c#index_create-flags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1181-L1199),
    [indexcmds.c:1618](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1618),
    [relcache.c#subid-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3367-L3374)).
  - `wal_skip_threshold` matters only for a relation whose WAL was skipped: at
    commit, a file smaller than the threshold is WAL-logged instead of fsynced.
    In CIC that can only be the still-empty index file at the commit of
    transaction 1, so the setting is not a CIC WAL-volume or memory lever
    ([storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L107-L180),
    [storage.c#small-relation-WAL-rule](../../../../raw/postgres-17/src/backend/catalog/storage.c#L771-L778)).

None of these settings needs a change for CIC. For reference, this is how each
would be applied:

| Setting | Context | Boot value | How a change is applied |
|---|---|---|---|
| `min_parallel_index_scan_size` | `user` | 512 kB, stored in blocks ([guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540)) | Session `SET`; no reload or restart |
| `max_parallel_workers_per_gather` | `user` | 2 ([guc_tables.c#max_parallel_workers_per_gather](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3419-L3428)) | Session `SET`; no reload or restart |
| `parallel_leader_participation` | `user` | on ([guc_tables.c#parallel_leader_participation](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1913-L1922)) | Session `SET`; no reload or restart |
| `maintenance_io_concurrency` | `user` | 10 in a build with prefetch support, otherwise 0 ([guc_tables.c#maintenance_io_concurrency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3123-L3136), [bufmgr.h#I/O-concurrency-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L164)) | Session `SET`; no reload or restart |
| `work_mem` | `user` | 4096 kB ([guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2458)) | Session `SET`; no reload or restart |
| `autovacuum_work_mem` | `sighup` | -1 ([guc_tables.c#autovacuum_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3441-L3450)) | Reload |
| `vacuum_cost_delay` | `user` | 0 ms ([guc_tables.c#vacuum_cost_delay](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3864-L3873)) | Session `SET`; no reload or restart |
| `temp_buffers` | `user` | 1024 blocks ([guc_tables.c#temp_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2382-L2391)) | Session `SET`; no reload or restart |
| `wal_level` | `postmaster` | `replica` ([guc_tables.c#wal_level](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4986-L4994)) | Restart |
| `wal_skip_threshold` | `user` | 2048 kB ([guc_tables.c#wal_skip_threshold](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2924-L2933)) | Session `SET`; no reload or restart |

#### Observation settings

The observation settings of the memory section (`track_activities`,
`trace_sort`, `log_temp_files` and `track_counts`) expose progress and spills
without changing what CIC does; see
[How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping).
Three more help with the settings of this section.

- `track_io_timing` and `track_wal_io_timing` add timing to the statistics of
  data-file and WAL I/O. Both are off by default, because they query the clock
  repeatedly, which "may cause significant overhead on some platforms". Both
  have the `superuser` context, so a role allowed to change them can use a
  session `SET`, with no reload or restart
  ([guc_tables.c#track_io_timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1420-L1428),
  [guc_tables.c#track_wal_io_timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1429-L1437),
  [config.sgml#track_io_timing](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8451-L8479),
  [config.sgml#track_wal_io_timing](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8481-L8501)).
- `log_lock_waits`, described above, names the processes that hold the lock CIC
  is waiting for. That identifies the blocker of the table-lock wait and of each
  step of waits 1 to 3
  ([proc.c#lock-wait-logging](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1495-L1643)).
- `log_checkpoints` shows whether a checkpoint started during a first build. Its
  boot value is on and its context is `sighup`, so a change needs a reload. Each
  checkpoint start is logged as `checkpoint starting: ...`. The bulk writer
  reports a crossing itself, but only at `DEBUG1`:
  `flushed relation because a checkpoint occurred concurrently`
  ([guc_tables.c#log_checkpoints](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1200-L1208),
  [xlog.c#LogCheckpointStart](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6623-L6653),
  [bulk_write.c:215](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L215)).

#### Practical priority

Tune and measure in this order:

1. `maintenance_work_mem`, checking the first build and validation separately
   and avoiding swapping
   ([ref/create_index.sgml#maintenance_work_mem-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L804)).
2. For B-tree or BRIN, the request and launch chain:
   `min_parallel_table_scan_size`, `max_parallel_maintenance_workers`,
   `max_parallel_workers` and `max_worker_processes`. Stop adding workers when
   CPU or I/O is already saturated, because a parallel
   [utility command](../../../glossary.md#utility-command) can use substantially
   more of both
   ([planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019),
   [config.sgml#max_parallel_maintenance_workers](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2864-L2900)).
3. Apply the AM-specific boundary. Keep `effective_cache_size` representative of
   usable cache for a buffered GiST build, because it limits the `levelStep` the
   build can choose. For a write-heavy GIN CIC, measure `gin_pending_list_limit`
   and `work_mem` in the writers and in the CIC session
   ([gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741),
   [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471),
   [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828)).
4. Put index output and sort spills on suitable storage. Then measure
   `io_combine_limit`, checkpoint crossings, WAL volume and the time spent in
   commits 1, 2 and 4, rather than weakening durability settings
   ([ref/create_index.sgml#tablespace](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L362-L372),
   [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223),
   [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521)).
5. Diagnose blockers separately. Memory and workers cannot shorten the
   table-lock wait, either `WaitForLockers()` call or the old-snapshot wait
   ([utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480),
   [indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658),
   [indexcmds.c:1705](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1705),
   [indexcmds.c:1768](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1768)).

#### Test coverage of the settings

The in-tree tests establish lifecycle and correctness, not the comparative
performance of settings. Five test files are dedicated to CIC or contain its
main block:

| Test | What it runs | Settings of this section that it changes |
|---|---|---|
| `create_index.sql`, concurrent block ([regression test](../../../glossary.md#regression-test)) | Concurrent builds on a small table, including a unique violation, a build inside a transaction block that must fail, and temporary tables | None ([create_index.sql#concurrent-builds](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L481-L582)) |
| `multiple-cic.spec` ([isolation test](../../../glossary.md#isolation-test)) | Two CIC commands on two tables at the same time | None ([multiple-cic.spec](../../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43)) |
| `prepared-transactions-cic.spec` (isolation test) | A CIC while a prepared transaction that inserted into the table is pending | `SET lock_timeout = 10` in the CIC session. The expected output is CIC failing with `canceling statement due to lock timeout` ([prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37), [prepared-transactions-cic.spec:25](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L25), [prepared-transactions-cic.out#expected-output](../../../../raw/postgres-17/src/test/isolation/expected/prepared-transactions-cic.out#L1-L19)) |
| `002_cic.pl` (amcheck [TAP test](../../../glossary.md#tap-test)) | CIC under concurrent pgbench inserts, checked with amcheck | `lock_timeout` in `postgresql.conf`, set to the test framework's default timeout. The test expects no error, so the value only bounds a hang ([amcheck/t/002_cic.pl](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L4-L88), [002_cic.pl#lock_timeout](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L20-L21)) |
| `003_cic_2pc.pl` (amcheck TAP test) | CIC against overlapping prepared transactions, across a restart, and under pgbench | The same `lock_timeout` guard ([amcheck/t/003_cic_2pc.pl](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L4-L170), [003_cic_2pc.pl#lock_timeout](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L24-L25)) |

`prepared-transactions-cic.spec` is not part of the default isolation schedule.
It runs only through the `check-prepared-txns` and `installcheck-prepared-txns`
make targets, because it needs prepared transactions to be enabled
([isolation/Makefile#prepared-txns-targets](../../../../raw/postgres-17/src/test/isolation/Makefile#L67-L74)).
A comment in the spec also notes that its CIC could time out while it waits for
an autovacuum-held lock, and that this would not change the expected output. The
table-lock wait described above is one way that can happen; the test does not
arrange it
([prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37)).

No in-tree test that runs CIC changes a WAL, checkpoint or commit setting,
`statement_timeout`, `transaction_timeout`, `deadlock_timeout` or
`idle_in_transaction_session_timeout`. That holds for the five files above and
for the four single CIC statements listed under [Test coverage](#test-coverage).
`lock_timeout` is the only setting of this section with direct CIC test
evidence. The commit behavior of transaction 3, the timer interaction and the
autovacuum cancel, as they apply to CIC, therefore rest on the source reading
alone, and comparing settings needs workload-specific measurements.

#### Causal summary of the settings

The settings act on a fixed sequence, so their effects can be read in order:

1. **Start.** The statement timer is armed once, and only if `statement_timeout`
   is above zero and smaller than a nonzero `transaction_timeout`, or
   `transaction_timeout` is zero. The table lock is requested; that wait runs
   under `lock_timeout` and `deadlock_timeout`, and after one `deadlock_timeout`
   CIC cancels an ordinary autovacuum worker that blocks it
   ([postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268),
   [utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480),
   [proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333),
   [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493)).
2. **Transaction 1 commits with an XID.** Its commit flushes unless
   `synchronous_commit` is `off`, and can wait for a synchronous standby
   ([heapam.c:2111](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2111),
   [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521),
   [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546)).
3. **Transaction 2 waits, then builds.** Wait 1 takes one VXID lock per blocker,
   each under its own lock timers and all under one transaction timer. The
   worker request comes from `plan_create_index_workers()`. The access method
   then fixes the WAL path, which decides whether `full_page_writes` and
   checkpoint spacing change the WAL volume and whether a checkpoint during the
   build costs an fsync of the index at the end. Commit 2 has an XID, so with a
   synchronous commit it pays for the unflushed tail of that WAL
   ([lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973),
   [planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019),
   [xloginsert.c#image-decision](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L626),
   [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223),
   [xlog.c#XLogFlush-contract](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2771-L2776)).
4. **Transaction 3 waits, then validates.** Wait 2 works like wait 1. Validation
   sorts with `maintenance_work_mem`, can wait on another transaction's XID for
   a unique index, and for GIN can run a pending-list cleanup with the CIC
   backend's `work_mem`. The transaction normally has no XID, so its commit
   never flushes and never waits for a standby, whatever `synchronous_commit`
   says
   ([index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452),
   [nbtinsert.c#unique-conflict-wait](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L213-L233),
   [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828),
   [xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395)).
5. **Transaction 4 waits, then marks the index valid.** Wait 3 takes one VXID
   lock per old snapshot. The `pg_index` update gives the transaction an XID, so
   the final commit flushes and can wait like commits 1 and 2, and its flush
   also covers the WAL of transaction 3
   ([indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497),
   [indexcmds.c:1773](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1773),
   [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521)).
6. **Outcome.** Memory and worker settings change the first build and the
   validation sort. WAL settings change the bytes written and the latency of
   commits 1, 2 and 4. Checkpoint settings change image volume for hash and BRIN
   and the end-of-build fsync for bulk-writer builds. Timeouts change only
   whether the command completes: after the first commit, any of them leaves an
   invalid index
   ([indexcmds.c:1618](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1618),
   [postgres.c#query-cancel-errors](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3372-L3430),
   [postgres.c#transaction-timeout-fatal](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3464)).

### What changed from PostgreSQL 12

The algorithm did not change. What changed is who CIC waits for in wait 3, how
the flag updates are stored, the security context of the build, a privilege
check, fixes for prepared transactions, and the machinery the first build
runs on.

**How these statements are supported.** Every comparison below comes from the
PostgreSQL 17 checkout's own git history, not from the PostgreSQL 12 checkout
and not from the wiki's v12 page. The state "as of PostgreSQL 12" is the tree
at commit `9e1c9f9594` ("pgindent run prior to branching v12.", 2019-07-01),
the parent of `615cebc94b` ("Stamp HEAD as 13devel."). The checkout has no
tags, so a commit's first release is read from the stamp commits it lies
between: `d10b19e224` (14devel, 2020-06-07), `596b5af1d3` (15devel,
2021-06-28), `d31d30973a` (16devel, 2022-06-30), `5bcc7e6dc8` (17devel,
2023-06-29) and `86a2d2a321` (17beta1, 2024-05-20). Where a commit message
says the change was back-patched, the table says so; this page does not
establish which PostgreSQL 12 minor releases contain a back-patched change.

**What is the same.** In the `9e1c9f9594` tree, `DefineIndex` already takes the
session lock, commits, waits with `WaitForLockers(heaplocktag, ShareLock,
true)`, calls `index_concurrently_build`, commits, waits the same way again,
calls `validate_index`, commits, and calls `WaitForOlderSnapshots(limitXmin,
true)`. That is the sequence the current code has
([indexcmds.c#DefineIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L539-L1793)).
The lock mode, the three flags and the rejection of system catalogs are also
in that tree.

**What differs.**

| # | Change | Commits in the v17 history | First release | Current code |
|---|---|---|---|---|
| 1 | Wait 3 skips backends that build a plain index concurrently. In the `9e1c9f9594` tree the exclusion mask was only `PROC_IS_AUTOVACUUM` and `PROC_IN_VACUUM`, so two concurrent builds on different tables waited for each other's snapshots | `c98763bf51` (2020-11-25, "Avoid spurious waits in concurrent indexing") added the flag, set it in `DefineIndex` and added it to the mask. `f9900df5f9` (2021-01-15, "Avoid spurious wait in concurrent reindex") made `REINDEX CONCURRENTLY` set it too. `cd6b2ae3e7` (2024-09-09, "Fix waits of REINDEX CONCURRENTLY for indexes with predicates or expressions", back-patched through 14) stopped `REINDEX CONCURRENTLY` from setting it for unsafe indexes | 14 | [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497), [indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4505) |
| 2 | **No net change:** `VACUUM` still honors a concurrent build's `xmin` | `d9d076222f` (2021-02-23, "VACUUM: ignore indexing operations with CONCURRENTLY") let the horizon computation ignore safe builds. `e28bb88519` (2022-05-31, "Revert changes to CONCURRENTLY that \"sped up\" Xmin advance", back-patched to 14) reverted it, because rows that were HOT-updated and pruned during the build could go missing from the new index | 14, reverted | The horizon computation skips only backends in `VACUUM` or logical decoding ([procarray.c#ComputeXidHorizons-skip](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1826-L1832)) |
| 3 | The flag updates are transactional. In the `9e1c9f9594` tree `index_set_state_flags` wrote the row in place with `heap_inplace_update` | `83158f74d3` (2020-09-14, "Make index_set_state_flags() transactional") replaced the in-place update with `CatalogTupleUpdate` | 14 | [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550) |
| 4 | Process state moved from `PGXACT` to `PGPROC`. This renames what the code reads; it does not change behavior | `5788e258bb` (committed 2020-08-14, "snapshot scalability: Move PGXACT->vacuumFlags to ProcGlobal->vacuumFlags.") moved the flags. `1f51c17c68` (2020-08-13, "snapshot scalability: Move PGXACT->xmin back to PGPROC.") changed `DefineIndex`'s assertion from `MyPgXact->xmin` to `MyProc->xmin`. `cd9c1b3e19` (2020-11-16, "Rename PGPROC->vacuumFlags to statusFlags") gave the field its current name | 14 | [indexcmds.c#wait-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1768), [proc.h#statusFlags](../../../../raw/postgres-17/src/include/storage/proc.h#L54-L78) |
| 5 | The switch to the table owner happens earlier. In the `9e1c9f9594` tree only `index_build`, and `validate_index` after it had opened the index, switched identity | `a117cebd63` (2022-05-09, "Make relation-enumerating operations be security-restricted operations.", back-patched to v10) moved the switch into `index_concurrently_build` and to the top of `validate_index`, before either opens the index | 15, back-patched | [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557), [index.c#validate_index-owner-switch](../../../../raw/postgres-17/src/backend/catalog/index.c#L3355-L3364) |
| 6 | Index expressions and predicates run with a fixed `search_path` | `2af07e2f74` (2024-03-04, "Fix search_path to a safe value during maintenance operations.") set it; `59825d1639` (same day, "Fix buildfarm failures from 2af07e2f74.") wrapped it in `RestrictSearchPath()` | 17 | The value is `pg_catalog, pg_temp` ([guc.c#GUC_SAFE_SEARCH_PATH](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L70-L74), [guc.c#RestrictSearchPath](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L2242-L2253)). `DefineIndex`, `index_concurrently_build`, `index_build` and `validate_index` each call it ([indexcmds.c#restrict-search-path](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L591-L593), [index.c#index_build-guc-nesting](../../../../raw/postgres-17/src/backend/catalog/index.c#L3019-L3028)) |
| 7 | The invoking role needs `USAGE` on the types an index expression or predicate depends on | `d1c8aa0b09` (2026-08-10, "Check for USAGE privilege on types used by stored expressions.", back-patched through 14) | 17.11 | [indexcmds.c#type-usage-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L931-L945) |
| 8 | Waits 1 and 2 also wait for prepared transactions | `8a54e12a38` (2021-01-30, "Fix CREATE INDEX CONCURRENTLY for simultaneous prepared transactions.", back-patched to 9.5) and `3cd9c3b921` (2021-10-23, "Fix CREATE INDEX CONCURRENTLY for the newest prepared transactions.", back-patched to 9.6) | 14 and 15, back-patched | [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973), [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663) |
| 9 | A backend that rebuilds its cache entry for the index retries if a relevant invalidation arrives meanwhile | `fdd965d074` (2021-10-23, "Avoid race in RelationBuildDesc() affecting CREATE INDEX CONCURRENTLY.", back-patched to 9.6) | 15, back-patched | [relcache.c#in_progress_list](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L156-L172) |
| 10 | GiST can build by sorting | `16fa9b2b30` (2020-09-17, "Add support for building GiST index by sorting.") | 14 | [gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248) |
| 11 | BRIN can build in parallel | `b437571714` (2023-12-08, "Allow parallel CREATE INDEX for BRIN indexes") | 17 | [brin.c:266](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L266) |
| 12 | B-tree and sorted GiST builds write their pages through the bulk writer | `8af2565248` (2024-02-23, "Introduce a new smgr bulk loading facility.") | 17 | [nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149), [gistbuild.c:409](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L409) |
| 13 | Both heap scans read through a sequential read stream, and `io_combine_limit` caps its read size | `b7b0f3f272` (2024-04-08, "Use streaming I/O in sequential scans.") and `210622c60e` (2024-04-03, "Provide vectored variant of ReadBuffer().") | 17 | [heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259), [guc_tables.c#io_combine_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3138-L3150) |
| 14 | `transaction_timeout` exists and can end one of CIC's internal transactions | `51efe38cb9` (2024-02-15, "Introduce transaction_timeout") | 17 | [guc_tables.c#transaction_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653) |
| 15 | CIC on a temporary table runs a plain build. The `9e1c9f9594` tree has no such fallback | `a904abe2e2` (2020-01-22, "Fix concurrent indexing operations with temporary tables"); its message says the multi-transaction path caused confusing errors with `ON COMMIT` actions | 13, back-patched | [indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615) |

Three of these deserve a sentence more.

**Row 1 is the only change to the waiting itself.** It shortens wait 3 only,
and only with respect to backends that build a plain index. It does not touch
waits 1 and 2, and row 2 shows that it does not let `VACUUM` remove rows
earlier. The commit message for `f9900df5f9` names the second benefit: two
such builds no longer deadlock on each other's snapshots.

**Row 6 can change what an index function sees.** A function called from an
index expression or predicate now runs with `search_path` set to
`pg_catalog, pg_temp`. The commit message for `2af07e2f74` states the
consequence: a function that relies on a different search path must be
declared with `CREATE FUNCTION ... SET search_path`.

**Rows 10 to 14 change the cost of the build, not its steps.** They are
described where they act: in
[How maintenance_work_mem is used and where increases stop helping](#how-maintenance_work_mem-is-used-and-where-increases-stop-helping)
and
[GUCs that affect CIC performance](#gucs-that-affect-cic-performance).

### Test coverage

The in-tree tests check that CIC completes, fails cleanly and stays correct
under concurrency. None of them asserts the internal steps directly.

| Test | What it exercises | What it shows |
|---|---|---|
| [Regression](../../../glossary.md#regression-test) block in `create_index.sql` ([create_index.sql#concurrent-builds](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L481-L582)) | CIC on an empty table, `IF NOT EXISTS`, unique, partial and expression indexes, a default index name, CIC inside a transaction block, a predicate function that assigns an XID, a failed unique build and its repair, temporary tables | The block's own comment says it covers "about half the code paths", because nothing updates the table concurrently |
| Unique violation at build time ([create_index.out#failed-unique-build](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1418-L1421), [create_index.out#invalid-index-repaired](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1448-L1484)) | A failure in transaction 2 | The index stays, marked `INVALID`; `REINDEX TABLE` repairs it once the duplicates are gone |
| Temporary tables ([create_index.sql#temp-tables](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L536-L559)) | The fallback to a plain build | CIC and `DROP INDEX CONCURRENTLY` succeed on a temporary table; CIC inside a transaction block still fails |
| [Isolation](../../../glossary.md#isolation-test) spec `multiple-cic` ([multiple-cic.spec](../../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43), [multiple-cic.out](../../../../raw/postgres-17/src/test/isolation/expected/multiple-cic.out#L1-L23)) | Two builds on two tables at once. The index predicates take and release an advisory lock, which is how the spec makes the first build block until the second one runs | Both builds complete. The spec was added by commit `54eff5311d` (2018-01-02, "Fix deadlock hazard in CREATE INDEX CONCURRENTLY"), so it guards against a deadlock between two builds. Its `(*)` marker makes the second build be reported as waiting "even if it completes very quickly", so the output does not show which wait, if any, it sat in |
| Isolation spec `prepared-transactions-cic` ([prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37), [prepared-transactions-cic.out#expected-output](../../../../raw/postgres-17/src/test/isolation/expected/prepared-transactions-cic.out#L1-L19)) | CIC while a prepared transaction that inserted into the table is pending, with `lock_timeout = 10` | CIC is cancelled with `canceling statement due to lock timeout`, which shows that it waited for the prepared transaction. A later query still finds the row |
| [TAP](../../../glossary.md#tap-test) test `002_cic.pl` in `contrib/amcheck` ([amcheck/t/002_cic.pl](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L4-L88)) | CIC under a `pgbench` write load, checked with `bt_index_check`; then CIC while an older transaction is open and a row was deleted | The resulting B-tree passes the heap-versus-index checks |
| TAP test `003_cic_2pc.pl` in `contrib/amcheck` ([amcheck/t/003_cic_2pc.pl](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L4-L170)) | CIC overlapping three prepared transactions, a prepared transaction that spans a server restart, and a `pgbench` load that mixes two-phase commits with CIC and `REINDEX CONCURRENTLY` | The index passes `bt_index_check` in each case |
| Single statements elsewhere | CIC on a unique index with `INCLUDE` columns, on a GiST index with `INCLUDE` columns, on an expression index with a predicate whose function asserts that it runs as the table's owner, and once only as a way to wait for other transactions ([index_including.sql#cic](../../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L161-L168), [index_including_gist.sql#cic](../../../../raw/postgres-17/src/test/regress/sql/index_including_gist.sql#L37-L44), [privileges.sql#index-functions-run-as-owner](../../../../raw/postgres-17/src/test/regress/sql/privileges.sql#L1227-L1248), [brin.sql#cic-as-wait](../../../../raw/postgres-17/src/test/regress/sql/brin.sql#L486-L490)) | The statement succeeds |

Two absences are worth stating:

- **No test checks `PROC_IN_SAFE_IC` for CIC.** `DefineIndex` has no injection
  point. The only test of the "safe" decision is for `REINDEX CONCURRENTLY`,
  which has two injection points that report whether each index was classified
  as safe
  ([indexcmds.c#reindex-safe-injection-points](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3808-L3817),
  [reindex_conc.sql](../../../../raw/postgres-17/src/test/modules/injection_points/sql/reindex_conc.sql#L1-L28)).
- **No test varies a memory or performance setting for CIC.** The two sections
  on settings say which tests set which setting.

### Final causal summary

1. **Initial state.** A table that other sessions read and write. The command
   may not block those writes, so it takes only `ShareUpdateExclusiveLock`.
2. **Announce.** Transaction 1 commits an index that is live but not ready and
   not valid. Because every session now counts it for HOT-safety, no new row
   chain can mix different key values. Wait 1 outlasts the transactions that
   opened the table before the announcement.
3. **Build.** Transaction 2 indexes every row its snapshot sees, then commits
   `indisready`. Because every new transaction now inserts into the index, the
   only rows that can be missing are those written by transactions that
   started before that commit. Wait 2 outlasts them.
4. **Validate.** Transaction 3 takes the reference snapshot, compares the heap
   with the index's sorted TIDs, and inserts every row the snapshot sees and
   the index lacks. The index is now complete for that snapshot and every
   newer one.
5. **Outwait old snapshots.** Older snapshots can still see rows that were
   deleted before the reference snapshot and are therefore not in the index.
   Wait 3 outlasts every such snapshot, except those of autovacuum, `VACUUM`
   and other builds of plain indexes, which cannot be hurt by missing entries.
6. **Publish.** Transaction 4 commits `indisvalid` and invalidates the table's
   cache entry, so new plans can use the index.
7. **If anything fails after the announcement,** the index stays behind,
   invalid. It limits HOT updates in every case, and it also costs inserts and
   enforces uniqueness if the failure came after commit 2.

Memory settings change how fast steps 3 and 4 run, and timeouts and blockers
change how long the three waits last; neither changes the order of the steps.

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md` and `wiki/log.md`,
  for navigation only.
- The shared [Wiki Glossary (unverified)](../../../glossary.md): the entries
  this page links, all checked on PostgreSQL 17. Eight entries were added for
  this page on 2026-10-07 and checked on PostgreSQL 17 only: Bulk writer,
  commit_delay, Parallel index build, PROC_IN_SAFE_IC, Session-level lock,
  Synchronized scan, Temporary file and Virtual transaction ID.
- `wiki/v17/common-concepts/`: its three pages cover bloat tests and none of
  the concepts this page uses, so nothing is linked from there.
- Pinned checkout `raw/postgres-17/` at commit
  `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11, seven commits
  after the "Stamp 17.11." commit `083ac03341`).
- `ProcessUtilitySlow`'s `IndexStmt` case in `src/backend/tcop/utility.c`, and
  the statement-end commit in `src/backend/tcop/postgres.c`.
- `DefineIndex`, `WaitForOlderSnapshots` and `set_indexsafe_procflags` in
  `src/backend/commands/indexcmds.c`.
- `index_create`, `UpdateIndexRelation`, `index_concurrently_build`,
  `BuildIndexInfo`, `index_build`, `validate_index`, its callback and
  `index_set_state_flags` in `src/backend/catalog/index.c`.
- The heap build scan and validation scan in
  `src/backend/access/heap/heapam_handler.c`, and the sections "CREATE INDEX"
  and "CREATE INDEX CONCURRENTLY" of `src/backend/access/heap/README.HOT`.
- `WaitForLockersMultiple` in `lmgr.c`; `LockConflicts`, `GetLockConflicts`,
  `LockAcquireExtended` and `VirtualXactLock` in `lock.c`; `ProcSleep` and
  `ProcReleaseLocks` in `proc.c`; `GetCurrentVirtualXIDs`,
  `ComputeXidHorizons` and `ProcArrayInstallRestoredXmin` in `procarray.c`;
  the status flags in `proc.h`.
- The readers of the three flags: `RelationGetIndexList` and
  `RelationGetIndexAttrBitmap` in `relcache.c`, `ExecInsertIndexTuples` in
  `execIndexing.c`, `get_relation_info` in `plancat.c`; `pg_index.h` and
  `catalogs.sgml`.
- For the two follow-up sections: the build and bulk-delete routines of
  B-tree, hash, GiST, GIN, SP-GiST, BRIN and contrib Bloom; `tuplesort.c` and
  `tuplesortvariants.c`; `plan_create_index_workers`; the parallel and
  background-worker launch paths; heap scan setup and `read_stream.c`; the
  bulk writer, `xloginsert.c`, `xlog.c`, `xact.c` and `syncrep.c`; the timeout
  code in `postgres.c` and `proc.c`; and each setting's `guc_tables.c` entry.
- Docs: `ref/create_index.sgml`, `catalogs.sgml`, `config.sgml`,
  `ref/set.sgml`, `monitoring.sgml`, `gist.sgml`, `gin.sgml`.
- Tests: every in-tree test that runs CIC; see
  [Test coverage](#test-coverage).
- History: the commits named under
  [What changed from PostgreSQL 12](#what-changed-from-postgresql-12), read
  with `git log` and `git show` in the v17 checkout, placed by the stamp
  commits, and compared with the tree at `9e1c9f9594`.
- 2026-10-07 review: all 36 findings of that day's review were fixed in this
  revision, from the pinned source. No server was started and the page reports
  no measured numbers.

## Evidence Map

| Claim | Source |
|---|---|
| CIC takes `ShareUpdateExclusiveLock`; the lock is first acquired in `ProcessUtilitySlow` | [utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480), [indexcmds.c#lock-mode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L663-L679) |
| CIC cannot run in a transaction block; a temporary table gets a plain build | [utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463), [indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L615) |
| Partitioned tables and system catalogs are rejected | [indexcmds.c#partitioned-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L709-L731), [index.c#concurrent-restrictions](../../../../raw/postgres-17/src/backend/catalog/index.c#L852-L869) |
| Since 17.11 an expression or predicate type without `USAGE` fails before `index_create()`, for `check_rights = true` callers only | [indexcmds.c#type-usage-check](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L931-L945), [index.c#index_create-contract](../../../../raw/postgres-17/src/backend/catalog/index.c#L727-L728), [utility.c#IndexStmt-DefineIndex](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1540-L1553), [tablecmds.c#attach-partition-clone-index](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18999-L19016) |
| The index is created live, not ready, not valid, without a build | [indexcmds.c#index_create-flags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1181-L1199), [index.c#index_create-state](../../../../raw/postgres-17/src/backend/catalog/index.c#L1042-L1057), [index.c#index_create-skip-build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1256-L1281) |
| What each flag means and who reads it | [pg_index.h#state-flags](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45), [relcache.c#RelationGetIndexList-live](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L4864-L4871), [execIndexing.c#skip-not-ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L357-L359), [plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267) |
| A live but not-ready index still counts for HOT-safety | [relcache.c#RelationGetIndexAttrBitmap-all-indexes](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5325-L5334), [README.HOT#CIC-not-ready-entry](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L368-L376) |
| Flag changes are transactional and reach other sessions at commit | [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550) |
| Four transactions, a session lock across them, three waits | [indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619), [indexcmds.c#wait-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1642-L1658), [indexcmds.c#wait-2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1697-L1705), [indexcmds.c#wait-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1757-L1768), [indexcmds.c#release-session-lock](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1785-L1792) |
| The first scan uses an MVCC snapshot and indexes only live rows, under the root TID of a HOT chain | [heapam_handler.c#build-scan-snapshot](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1270), [heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629), [heapam_handler.c#build-scan-root-tid](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1663-L1702) |
| The build sets `indisready` | [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557) |
| Validation collects the index's TIDs, sorts them, and inserts rows the reference snapshot sees and the index lacks | [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452), [index.c#validate_index_callback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3454-L3466), [heapam_handler.c#heapam_index_validate_scan](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1747-L1986) |
| `limitXmin` is the reference snapshot's `xmin`; the snapshot is dropped before the wait | [indexcmds.c#drop-reference-snapshot](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1730-L1751) |
| Waits 1 and 2 wait on the VXIDs of current lock holders and take no table lock | [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973), [lock.c#GetLockConflicts-contract](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L2884-L2902), [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663) |
| Lock conflicts of `ShareUpdateExclusiveLock` and `ShareLock` | [lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L59-L104), [lockdefs.h#lock-modes](../../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46) |
| Wait 3 covers this database's backends with `xmin` at or before `limitXmin`, except autovacuum, `VACUUM` and safe concurrent builds | [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497), [procarray.c#GetCurrentVirtualXIDs](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L3296-L3378) |
| `PROC_IN_SAFE_IC` is set for plain indexes at the start of transactions 2 to 4, is reset at transaction end, and is inherited by parallel workers | [indexcmds.c#safe_index](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1144-L1146), [indexcmds.c#set_indexsafe_procflags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4473-L4505), [proc.h#statusFlags](../../../../raw/postgres-17/src/include/storage/proc.h#L54-L78), [procarray.c#restored-xmin-flags](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2644-L2651) |
| Marking valid, the table's relcache invalidation, and the statement-end commit | [indexcmds.c#mark-valid](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1770-L1783), [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821) |
| A concurrent build never sets `indcheckxmin` | [index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3071-L3124), [README.HOT#CIC-no-indcheckxmin](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L410-L414) |
| The new index is locked `AccessExclusiveLock` in transaction 1 and `RowExclusiveLock` in transactions 2 and 3; parallel workers use the leader's modes | [index.c#new-index-lock](../../../../raw/postgres-17/src/backend/catalog/index.c#L1001-L1006), [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557), [nbtsort.c#worker-lock-modes](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1778-L1792) |
| What a failure leaves behind, by transaction; abort releases the session lock | [execIndexing.c#skip-not-ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L357-L359), [proc.c#ProcReleaseLocks](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L794-L821), [ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667), [create_index.out#invalid-index-repaired](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1448-L1484) |
| `PROC_IN_SAFE_IC` arrived in 14 | commits `c98763bf51` (2020-11-25), `f9900df5f9` (2021-01-15), `cd6b2ae3e7` (2024-09-09) |
| The `VACUUM` change was added and reverted | commits `d9d076222f` (2021-02-23), `e28bb88519` (2022-05-31); [procarray.c#ComputeXidHorizons-skip](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1826-L1832) |
| Transactional flag updates arrived in 14 | commit `83158f74d3` (2020-09-14) |
| `PGXACT` to `PGPROC` moves and the `statusFlags` rename | commits `5788e258bb` (2020-08-14), `1f51c17c68` (2020-08-13), `cd9c1b3e19` (2020-11-16) |
| Earlier owner switch; fixed `search_path` | commits `a117cebd63` (2022-05-09), `2af07e2f74` and `59825d1639` (2024-03-04); [guc.c#RestrictSearchPath](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L2242-L2253) |
| Prepared-transaction and relcache fixes | commits `8a54e12a38` (2021-01-30), `3cd9c3b921` and `fdd965d074` (2021-10-23) |
| Build machinery new since 12 | commits `16fa9b2b30`, `b437571714`, `8af2565248`, `b7b0f3f272`, `210622c60e`, `51efe38cb9` |
| The PostgreSQL 12 shape | tree at commit `9e1c9f9594` (2019-07-01), parent of `615cebc94b` |
| Tests | [create_index.sql#concurrent-builds](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L481-L582), [multiple-cic.spec](../../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43), [prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37), [amcheck/t/002_cic.pl](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L4-L88), [amcheck/t/003_cic_2pc.pl](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L4-L170) |
| `maintenance_work_mem` is a `user`-context GUC in kB with boot value 64 MB and minimum 64 kB; only a session-level `SET` can cover a CIC, and parallel workers use the leader's value | [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474), [ref/set.sgml#SET-LOCAL-lifetime](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L54-L61), [ref/set.sgml#LOCAL](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L109-L120), [utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463), [ref/set.sgml#SET-persists-after-commit](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L45-L52), [parallel.c#serialize-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L382-L385), [parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1455-L1457) |
| The first build and validation use memory in separate transactions: the validation sort is created after commit 2 and wait 2, and it is ended before `validate_index()` returns | [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1489-L1557), [indexcmds.c#build-to-validation](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1678-L1728), [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399), [index.c#validate_index-sort-and-merge](../../../../raw/postgres-17/src/backend/catalog/index.c#L3406-L3434) |
| Validation uses one serial, full-budget sort of TIDs encoded as `int8`, fed by the access method's `ambulkdelete` | [index.c#validation-sort](../../../../raw/postgres-17/src/backend/catalog/index.c#L3390-L3399), [index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404), [index.c#validate_index_callback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3454-L3466), [tuplesort.c#coordinate-roles](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L717-L742) |
| The validation sort charges only its slot array, so it fits when N times the `SortTuple` size fits the budget (derived from the source, not measured); the array holds at most `INT_MAX` slots | [tuplesortvariants.c#tuplesort_begin_datum](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L583-L661), [tuplesortvariants.c#tuplesort_putdatum](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L820-L867), [tuplesort.h#SortTuple](../../../../raw/postgres-17/src/include/utils/tuplesort.h#L117-L153), [tuplesort.c#memtuples-charge](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L808-L812), [tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183) |
| Worker request in the order W1 to W5; the memory cap keeps 32 MB per participant, leader included, so 64 MB allows one worker, 96 MB two and 128 MB three | [planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913), [planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985), [planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998), [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012) |
| The table's `parallel_workers` skips the size model and the memory cap, but not the parallel-safety test | [planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985), [reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382), [ref/create_index.sgml#parallel_workers](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L835-L845) |
| B-tree and BRIN shares: workers divide M by the planned count, the leader by the launched count; without a DSM segment or without a launched worker the build is serial | [nbtsort.c:1426](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1426), [nbtsort.c#worker-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1826-L1829), [nbtsort.c#leader-share](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1715-L1720), [nbtsort.c#no-DSM-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1489-L1497), [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594), [brin.c#worker-share](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2911-L2919), [brin.c#_brin_leader_participate_as_worker](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2764-L2783), [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525) |
| The leader takes part in the scan unless the server is compiled with `DISABLE_LEADER_PARTICIPATION` | [nbtsort.c#DISABLE_LEADER_PARTICIPATION](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L69-L73), [nbtsort.c#leaderparticipates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1410-L1415) |
| A unique B-tree build adds a second sort (spool2) sized by `work_mem`; in a CIC it stays empty and is discarded | [nbtsort.c#secondary-spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L471), [nbtsort.c#worker-spool2](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1886-L1908), [heapam_handler.c#build-scan-snapshot-serial-and-parallel](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1235-L1283), [heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629), [nbtsort.c#_bt_build_callback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L573-L600), [nbtsort.c#spool2-discard](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L500-L506) |
| Tuplesort: 64 kB floor; one in-memory quicksort when the input fits, runs and a merge otherwise; a parallel participant always writes one run | [tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700), [tuplesort.c#performsort-TSS_INITIAL](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1397-L1434), [tuplesort.c#performsort-TSS_BUILDRUNS](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1451-L1465), [tuplesort.c#worker_nomergeruns](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L3078-L3093), [tuplesort.c#leader_takeover_tapes](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L3095-L3160) |
| Hash uses M first as a size threshold capped by `NBuffers`, then as the budget of a serial sort | [hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166), [hashsort.c#_h_spoolinit](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L56-L93) |
| GiST chooses a sorted, plain or buffered build; `buffering = on` skips the sorted build; M and `effective_cache_size` limit `levelStep`; a `DEBUG1` line reports the chosen `levelStep` | [gistbuild.c#buffering-mode-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L226), [gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248), [gistbuild.c#build-strategy](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L256-L338), [gistbuild.c#levelStep-loop](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L717-L741), [gistbuild.c#buffering-refused](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L749-L758), [gistbuild.c#buffering-started](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L773-L776), [reloptions.c#buffering](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L521-L531) |
| GIN's first build dumps its in-memory collection when it reaches M, checked after each heap tuple | [gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314), [gininsert.c#final-dump](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L386-L397) |
| GIN raises "posting list is too long", with a hint to reduce `maintenance_work_mem`, when one key's TID list must grow beyond `INT_MAX` slots | [ginbulk.c#posting-list-growth](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L39-L52) |
| GIN validation forces a full pending-list cleanup with M; validation's own inserts use fast update and can trigger a regular cleanup with `work_mem` | [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828), [ginfast.c#empty-list-return](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L835-L841), [ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000), [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530), [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471) |
| BRIN: a serial build creates no sort; a parallel build sorts summaries with shares of M; validation gets no TID, so every visible heap tuple is passed to `index_insert()` | [brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245), [brin.c#_brin_parallel_scan_and_build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2785-L2847), [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301), [heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974) |
| SP-GiST and Bloom do not read the setting in the first build but report TIDs to the validation sort | [spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L148), [spgvacuum.c#live-tuple-callback](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L153-L181), [blinsert.c#blbuild](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L117-L158), [blvacuum.c#tuple-loop](../../../../raw/postgres-17/contrib/bloom/blvacuum.c#L88-L107) |
| `amusemaintenanceworkmem` is read by parallel VACUUM, not by `CREATE INDEX` | [amapi.h#amusemaintenanceworkmem](../../../../raw/postgres-17/src/include/access/amapi.h#L255-L256), [vacuumparallel.c#mwm-index-count](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L350-L351), [vacuumparallel.c#mwm-worker-share](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L372-L375) |
| The GUC entry and the `ambuild` type are hand-written source; the access-method handler is resolved through `pg_am` data and the generated function-manager table | [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474), [utils/misc/Makefile#OBJS](../../../../raw/postgres-17/src/backend/utils/misc/Makefile#L17-L34), [utils/misc/meson.build#backend_sources](../../../../raw/postgres-17/src/backend/utils/misc/meson.build#L3-L20), [amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105), [relcache.c#access-method-handler-lookup](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L1461-L1471), [relcache.c#InitIndexAmRoutine](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L1398-L1422), [amapi.c#GetIndexAmRoutine](../../../../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46), [pg_am.dat#index-access-methods](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18-L35), [utils/Makefile#fmgr-stamp](../../../../raw/postgres-17/src/backend/utils/Makefile#L48-L53) |
| Observation: progress phases with B-tree sub-phases, the `DEBUG1` worker-request message, `trace_sort` `LOG` lines, `log_temp_files` (boot value -1) and the `pg_stat_database` temp counters | [system_views.sql#pg_stat_progress_create_index](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1256-L1289), [nbtutils.c#btbuildphasename](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4604-L4625), [index.c#index_build-request-message](../../../../raw/postgres-17/src/backend/catalog/index.c#L3007-L3017), [tuplesort.c#tuplesort_free-trace](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L930-L940), [guc_tables.c#trace_sort](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1659-L1670), [guc_tables.c#log_temp_files](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3554-L3563), [fd.c#ReportTemporaryFileUsage](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1524-L1539), [pgstat_database.c#pgstat_report_tempfile](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_database.c#L171-L185) |
| Tests: the hash sort is forced with 1 MB and `fillfactor = 10`; the `pageinspect` BRIN test runs with 128 MB, which allows three workers plus the leader, and also tests the serial fallback; no test that runs CIC changes the setting | [create_index.sql#hash-tuplesort-test](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L372-L380), [pageinspect/sql/brin.sql#parallel-build](../../../../raw/postgres-17/contrib/pageinspect/sql/brin.sql#L87-L115), [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012), [pageinspect/sql/brin.sql#serial-fallback](../../../../raw/postgres-17/contrib/pageinspect/sql/brin.sql#L119-L126), [create_index.sql#concurrent-builds](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L481-L582), [multiple-cic.spec](../../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43), [prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37), [amcheck/t/002_cic.pl](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L4-L88), [amcheck/t/003_cic_2pc.pl](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L4-L170) |
| A third-party access method's `ambuild` receives the heap, the index and an `IndexInfo` but no settings, so no core-source list of GUCs can cover it | [amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105), [index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3049-L3053) |
| Parallel workers start with the leader's GUC values; a backend re-reads the configuration file only between client commands; an index function's GUC change is rolled back at the end of the build or validation step | [parallel.c#serialize-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L382-L385), [parallel.c#restore-GUC-state](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L1455-L1457), [postgres.c#main-loop-reload](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4752-L4760), [index.c#index_build-guc-nesting](../../../../raw/postgres-17/src/backend/catalog/index.c#L3019-L3028), [index.c#index_build-guc-rollback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3149-L3150), [index.c#validate_index-guc-rollback](../../../../raw/postgres-17/src/backend/catalog/index.c#L3443-L3444) |
| Only the session form of `SET` can cover a CIC run | [ref/set.sgml#session-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L32-L52), [ref/set.sgml#SET-LOCAL-lifetime](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L54-L61), [ref/set.sgml#LOCAL](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L109-L120), [utility.c#IndexStmt-prevent-in-xact](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1461-L1463) |
| The worker request is planned only for B-tree and BRIN, in this order: no-parallelism test, parallel-safety test, `parallel_workers` storage parameter, heap-size model, 32 MB-per-participant memory cap | [index.c#index_build-worker-request](../../../../raw/postgres-17/src/backend/catalog/index.c#L2995-L3005), [planner.c#no-parallelism-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6908-L6913), [planner.c#parallel-safety-test](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6958-L6971), [planner.c#parallel_workers-override](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6973-L6985), [planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998), [allpaths.c#minimum-size-test](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227), [allpaths.c#heap-size-steps](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4229-L4251), [allpaths.c#caller-cap](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4275-L4276), [planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012) |
| The launch compares a cluster-wide count of active parallel workers with the CIC session's `max_parallel_workers` and needs a free `max_worker_processes` slot; with no shared memory segment or no launched worker, B-tree and BRIN build serially | [bgworker.c#parallel-worker-count](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L83-L100), [bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014), [bgworker.c#slot-search](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L1016-L1043), [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646), [parallel.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L310-L334), [nbtsort.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1486-L1497), [nbtsort.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1569-L1594), [brin.c#dsm-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2432-L2443), [brin.c#launch-and-fallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2501-L2525) |
| `shared_buffers` feeds three CIC inputs: the `NBuffers / 4` test for the bulk-read strategy and synchronized scans, the cap on hash's sort threshold, and the read-stream ring and pin budget; its boot value is 16384 blocks, while a new cluster runs on the value `initdb` probed and wrote | [heapam.c#bulk-and-sync-threshold](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L452), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L388-L404), [hash.c#sort-threshold](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L139-L166), [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478), [freelist.c#ring-size-cap](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L598-L599), [bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2107-L2144), [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270), [initdb.c#shared_buffers-probe](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1173-L1186), [initdb.c#shared_buffers-write](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1283-L1290) |
| Both CIC heap scans read through a sequential read stream; `io_combine_limit` is an upper bound on the read size, which the look-ahead distance and the pin budget can lower | [heapam_handler.c#build-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1248-L1283), [heapam_handler.c#validation-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1804), [heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259), [read_stream.c#read_stream_look_ahead](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L301-L377), [read_stream.c#initial-distance](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L545-L553), [read_stream.c#distance-after-wait](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L707-L728), [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478), [read_stream.c#captured-GUCs](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L530-L536) |
| The pin budget is the smallest of the stream's own wish, the scan's ring and the backend's share `NBuffers / (MaxBackends + NUM_AUXILIARY_PROCS)` minus a constant | [read_stream.c#pin-budget](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478), [freelist.c#GetAccessStrategyPinLimit](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L632-L672), [bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2107-L2144), [postinit.c#InitializeMaxBackends](../../../../raw/postgres-17/src/backend/utils/init/postinit.c#L565-L588), [proc.h#NUM_AUXILIARY_PROCS](../../../../raw/postgres-17/src/include/storage/proc.h#L433-L443) |
| `effective_io_concurrency` (or the tablespace option) becomes the stream's `max_ios`, but `READ_STREAM_SEQUENTIAL` turns advice off, reads happen only when the scan reaches the block, and the distance only moves toward `io_combine_limit`; so it cannot change these scans' I/O in PostgreSQL 17 | [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439), [spccache.c#get_tablespace_io_concurrency](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L207-L223), [read_stream.c#advice-condition](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L508-L520), [bufmgr.c#advice-in-StartReadBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1326-L1343), [bufmgr.c:1519](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1519), [read_stream.c#distance-after-wait](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L707-L728), [read_stream.c#behavior-B](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L27-L32) |
| Synchronized first heap scans are allowed for B-tree, hash, GiST, SP-GiST, Bloom and parallel BRIN builds, and refused for GIN, serial BRIN and the validation scan | [nbtsort.c#serial-or-parallel-scan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L473-L480), [nbtsort.c#worker-scan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1920-L1927), [hash.c#build-scan](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L172-L175), [gistbuild.c#sorted-build-scan](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L273-L276), [gistbuild.c#insert-build-scan](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L312-L315), [spginsert.c#build-scan](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L124-L126), [blinsert.c#build-scan](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L142-L145), [brin.c#worker-scan](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2816-L2824), [gininsert.c#build-scan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L378-L384), [brin.c#serial-scan-order](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1217-L1224), [heapam_handler.c#validation-scan-start](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1793-L1804), [heapam.c#syncscan-choice](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L467-L496) |
| `effective_cache_size` is read only by GiST's non-sorted build: the `auto` switch to buffering and the `levelStep` choice | [gistbuild.c#sorted-build-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L228-L248), [gistbuild.c#buffering-mode-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L226), [gistbuild.c#auto-switch](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L878-L900), [gistbuild.c#levelStep-choice](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L670-L741), [costsize.c#effective_cache_size-use](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L914-L915) |
| `temp_tablespaces` places sort tapes and GiST's buffering file; `temp_file_limit` raises an error per process, after transaction 1 has committed | [tuplesort.c#tape-tablespaces](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1960-L1965), [fileset.c#FileSetInit](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L36-L86), [gistbuildbuffers.c#temporary-file](../../../../raw/postgres-17/src/backend/access/gist/gistbuildbuffers.c#L53-L57), [buffile.c#BufFileCreateTemp](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L180-L216), [fd.c#temp-tablespace-choice](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1737-L1763), [fd.c#temp_file_limit-check](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L2211-L2237), [indexcmds.c#commit-1](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1594-L1619) |
| `default_tablespace` is read once, in transaction 1, when the statement has no `TABLESPACE` clause | [indexcmds.c#tablespace-selection](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L767-L784), [tablespace.c#GetDefaultTablespace](../../../../raw/postgres-17/src/backend/commands/tablespace.c#L1126-L1182), [indexcmds.c#index_create-call](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1218-L1227) |
| GIN pending list: validation starts with a forced cleanup under `maintenance_work_mem`; every later insert, by a writer or by validation itself, can start a regular cleanup under the inserting backend's `gin_pending_list_limit` and `work_mem` | [index.c#validation-bulkdelete](../../../../raw/postgres-17/src/backend/catalog/index.c#L3402-L3404), [ginvacuum.c#forced-cleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828), [heapam_handler.c#validation-insert](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1963-L1971), [gininsert.c#fast-update-branch](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L510-L530), [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-17/src/include/access/gin_private.h#L39-L45) |
| `default_toast_compression` selects the method for an index value wider than `TOAST_INDEX_TARGET` whose index column has no method of its own | [indextuple.c#compress-wide-value](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138), [toast_internals.c#toast_compress_datum](../../../../raw/postgres-17/src/backend/access/common/toast_internals.c#L32-L104), [index.c:359](../../../../raw/postgres-17/src/backend/catalog/index.c#L359), [index.c#expression-attcompression](../../../../raw/postgres-17/src/backend/catalog/index.c#L390-L397), [heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68) |
| Commit settings act at the commits of transactions 1, 2 and 4; transaction 3 normally has no XID; lock-wait timers and the autovacuum cancel act at every lock wait; `statement_timeout` is armed conditionally | [xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546), [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452), [proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333), [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493), [postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268) |
| The first build's WAL path depends on the access method: bulk writer for B-tree and sorted GiST, `log_newpage_range()` at the end of `ambuild` for the other GiST builds, GIN and SP-GiST, ordinary per-insertion records for hash and BRIN, a forced generic image per flushed page for Bloom. No other shipped build starts a bulk write on the index's main fork (checked by searching the checkout for `smgr_bulk_start_rel`) | [nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149), [gistbuild.c#gist_indexsortbuild](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L396-L454), [gistbuild.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L328-L337), [gininsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L408-L417), [spginsert.c#build-WAL](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L132-L141), [hashpage.c#initial-bucket-loop](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L415-L437), [hashinsert.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/hash/hashinsert.c#L215-L235), [brin_pageops.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L425-L449), [blinsert.c#flushCachedPage](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L43-L58), [generic_xlog.c#GenericXLogFinish](../../../../raw/postgres-17/src/backend/access/transam/generic_xlog.c#L332-L436) |
| A forced image ignores `full_page_writes` and checkpoints; an ordinary record carries an image only on the first change after the redo pointer while page writes are enabled | [xloginsert.c#image-decision](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L626), [xlog.c:847](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L847), [xloginsert.c#log_newpages](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1169-L1224), [xloginsert.c#log_newpage_range](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1252-L1342) |
| A bulk-writer build fsyncs the index file itself when a checkpoint started during the bulk write; otherwise it registers the file for the next checkpoint | [bulk_write.c#smgr_bulk_start_smgr](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L95-L122), [bulk_write.c#smgr_bulk_flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L239-L313), [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223) |
| Transactions 1, 2 and 4 have an XID; their commits write a commit record, flush unless `synchronous_commit = off`, and can wait for a synchronous standby. CIC neither forces a synchronous commit nor deletes files at commit | [heapam.c:2111](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2111), [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3468-L3550), [heapam.c:3352](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3352), [indexcmds.c:1773](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1773), [xact.c#commit-record](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1428-L1437), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546), [xact.c#ForceSyncCommit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1140-L1152), [storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L107-L180) |
| Transaction 3 normally has no XID: no commit record, no commit-time flush and no standby wait, even when validation wrote WAL. It updates no catalog row, and no index insertion path assigns an XID (checked by searching the access methods, `index.c` and `heapam_handler.c`; the only match is a comment). An expression or predicate function that assigns an XID while validation inserts a tuple is the exception | [indexcmds.c#transaction-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1687-L1751), [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452), [nbtpage.c#no-xid-comment](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2637), [xact.c#markXidCommitted](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1306-L1307), [xact.c#no-xid-branch](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1339-L1395), [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521), [xact.c#syncrep-wait](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1536-L1546), [heapam_handler.c#validation-missing-tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1911-L1974), [create_index.sql#xid-assigning-predicate](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L509-L520) |
| The standby wait returns at once unless `synchronous_commit` is above `local`, `max_wal_senders` is above zero and a synchronous standby is defined | [syncrep.c#fast-exit](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L159-L181), [syncrep.h#SyncRepRequested](../../../../raw/postgres-17/src/include/replication/syncrep.h#L18-L19), [syncrep.c#assign_synchronous_commit](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L1120-L1138) |
| `wal_compression` is tried for every page image and kept only when the result is smaller | [xloginsert.c#image-compression](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L685-L695), [xloginsert.c#XLogCompressBackupBlock](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L936-L1017) |
| `commit_delay` sleeps before a WAL flush only with `fsync` on and at least `commit_siblings` other active backends | [xlog.c#group-commit-delay](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L2870-L2895) |
| With `wal_log_hints` or data checksums, setting a hint bit during a heap scan can write an `XLOG_FPI_FOR_HINT` record | [heapam.c#heap_prepare_pagescan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L601-L690), [heapam_visibility.c#SetHintBits](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L82-L132), [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5036-L5182), [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L109-L118), [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1043-L1128) |
| `backend_flush_after` applies to buffers the backend writes itself when it evicts a dirty buffer; the boot value 0 issues no writeback request | [bufmgr.c#dirty-victim-write](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1988-L2058), [bufmgr.c#ScheduleBufferTagForWriteback](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5975-L6007), [pg_config_manual.h#flush-after-defaults](../../../../raw/postgres-17/src/include/pg_config_manual.h#L165-L179) |
| CIC's lock waits are the table lock at the start, one VXID or prepared-transaction XID lock per blocker in waits 1 to 3, and the XID lock of a conflicting in-progress transaction during validation of a unique index; later table re-locks cannot block | [utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480), [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973), [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663), [lock.c#XactLockForVirtualXact](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4497-L4548), [nbtinsert.c#unique-conflict-wait](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L213-L233), [indexcmds.c#WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L396-L497), [lock.c#already-held](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L876-L889) |
| `lock_timeout` and `deadlock_timeout` are armed for each lock acquisition and disabled when it ends | [proc.c#lock-wait-timers](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1277-L1333), [proc.c#timer-disable](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1646-L1666) |
| After the deadlock check, a waiter that is blocked directly by an autovacuum worker cancels it unless it protects against wraparound; for CIC that is the table lock at the start | [deadlock.c#blocking-autovacuum](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L594-L618), [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493), [vacuum.c#vacuum_rel-lock-mode](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2059), [analyze.c#analyze_rel-lock](../../../../raw/postgres-17/src/backend/commands/analyze.c#L134-L145) |
| `statement_timeout` is armed once per statement, and only when it is above zero and smaller than a nonzero `transaction_timeout` or `transaction_timeout` is zero; CIC's internal commits do not touch it | [postgres.c#start_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2767-L2796), [postgres.c#enable_statement_timeout](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L5244-L5268), [postgres.c#finish_xact_command](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L2798-L2821), [indexcmds.c:1618](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1618) |
| `transaction_timeout` is armed again for each of CIC's four transactions and ends the session with `FATAL` | [xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2174-L2176), [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2315-L2317), [postgres.c#transaction-timeout-fatal](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3453-L3464) |
| In-tree evidence for `lock_timeout`: a CIC blocked behind a prepared transaction is cancelled with `lock_timeout = 10`; the spec runs only through the prepared-transaction make targets | [prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37), [prepared-transactions-cic.out#expected-output](../../../../raw/postgres-17/src/test/isolation/expected/prepared-transactions-cic.out#L1-L19), [isolation/Makefile#prepared-txns-targets](../../../../raw/postgres-17/src/test/isolation/Makefile#L67-L74) |
| `min_parallel_index_scan_size`, the `Gather` settings and `maintenance_io_concurrency` do not act on a CIC build or its heap scans | [planner.c#size-model-call](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6987-L6998), [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4187-L4279), [nbtsort.c#leader-participation-switch](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1413-L1415), [brin.c#leader-participation-switch](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2370-L2372), [heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259), [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439) |
| `work_mem`: the unique build's secondary spool stays empty in a concurrent build; with GIN `fastupdate`, the backend that pushes the pending list past its limit, the CIC backend included, cleans up with its own `work_mem` | [nbtsort.c#secondary-spool](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L433-L471), [heapam_handler.c#build-scan-mvcc-alive](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1624-L1629), [nbtsort.c#spool2-discard](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L500-L506), [gininsert.c#gininsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L482-L536), [ginfast.c#pending-list-threshold](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [ginfast.c#cleanup-memory-selection](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L828) |
| `vacuum_delay_point()` never sleeps in a CIC backend, because `VacuumCostActive` is false outside `VACUUM` and `ANALYZE` | [vacuum.c#vacuum_delay_point](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2383-L2464), [globals.c:159](../../../../raw/postgres-17/src/backend/utils/init/globals.c#L159), [autovacuum.c#VacuumUpdateCosts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1630-L1695), [vacuum.c#cost-accounting-scope](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L599-L684) |
| `wal_level = minimal` does not skip WAL for CIC's first build, because the index was created in an earlier, committed transaction | [guc_tables.c#wal_level](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4986-L4994), [xlog.h#XLogIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L103-L107), [rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L620-L631), [relcache.c#subid-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3367-L3374), [indexcmds.c#index_create-flags](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1181-L1199) |

## Open Questions

- **Docs say the index "may not be immediately usable"; the code has no
  mechanism for that on the concurrent path.** The reference page says that
  even after the command ends, "in the worst case" the index "cannot be used
  as long as transactions exist that predate the start of the index build"
  ([ref/create_index.sgml#concurrent-phases](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L627-L643)).
  In the source, the flag that delays index use, `indcheckxmin`, is set only
  by a non-concurrent build, and both `index_build` and README.HOT say a
  concurrent build does not need it because wait 3 has already outlasted those
  transactions
  ([index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3071-L3124),
  [README.HOT#CIC-no-indcheckxmin](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L410-L414)).
  This page follows the source. The pinned checkout does not say whether the
  sentence in the docs is a leftover or refers to something else.
- **Docs say an invalid index "will still consume update overhead" without a
  condition.** In the source, writes insert only into an index that is ready,
  so an index left behind by a failure before commit 2 receives no inserts; it
  only limits HOT updates
  ([ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667),
  [execIndexing.c#skip-not-ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L357-L359),
  [relcache.c#RelationGetIndexAttrBitmap-all-indexes](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5325-L5334)).
  [Failure handling](#failure-handling) follows the source.
- **Which PostgreSQL 12 minor releases contain the back-patched changes is not
  established here.** The comparison under
  [What changed from PostgreSQL 12](#what-changed-from-postgresql-12) rests on
  the PostgreSQL 17 checkout's history: the tree at `9e1c9f9594` for the state
  PostgreSQL 12 was branched from, and each commit's own message for
  back-patching. Rows 5, 8, 9 and 15 of that table were back-patched, so a
  12.x release can already contain them. The
  [v12 page](../../../v12/questions/indexing/create-index-concurrently.md)
  describes its own pin and is linked for navigation, not as evidence.
- **`PROC_IN_SAFE_IC` has no in-tree test for CIC.** The behavior described
  for wait 3 comes from reading `WaitForOlderSnapshots` and
  `set_indexsafe_procflags`; see [Test coverage](#test-coverage).
- **No common concept page covers what this page leans on.** PostgreSQL 17 has
  no `common-concepts` page for lock modes and conflicts, HOT chains,
  snapshots and `xmin`, or tuplesort, so this page explains what it needs
  inline and links the glossary.
- The value at which a particular CIC stops improving cannot be derived from
  the source alone. The source gives thresholds: the 32 MB rule of the worker
  request
  ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012))
  and the fit condition of the validation sort
  ([tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)).
  How much time each threshold saves for a given table, index definition,
  concurrent workload, storage device and worker pool needs controlled runs
  that time both the first build and validation. This page reports no such
  run.
- The documentation and a source comment describe `maintenance_work_mem` as a
  limit for the whole index build: "the maximum amount of memory that can be
  used by each index build operation as a whole"
  ([ref/create_index.sgml#parallel-index-build](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L806-L817)),
  "a limit to be applied to the entire utility command"
  ([config.sgml#parallel-utility-memory-note](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2887-L2898)),
  and "an absolute high watermark"
  ([nbtsort.c#leader-or-serial-sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L406-L431)).
  The code allows more in two places: each participant's share is raised to at
  least 64 kB
  ([tuplesort.c#allowedMem-floor](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L694-L700)),
  and a unique B-tree build gives every participant a further sort of up to
  `work_mem`
  ([nbtsort.c#worker-spool2](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1886-L1908)).
  The page follows the code. The pinned tree does not say whether the stronger
  wording is meant as an approximation.
- The `pageinspect` BRIN test sets `maintenance_work_mem = '128MB'` under the
  comment "so we set maintenance_work_mem for 4 workers"
  ([pageinspect/sql/brin.sql#parallel-build](../../../../raw/postgres-17/contrib/pageinspect/sql/brin.sql#L87-L115)).
  The memory cap in `plan_create_index_workers()` keeps three workers at
  128 MB, which gives four participants with the leader
  ([planner.c#memory-worker-cap](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L7000-L7012)).
  The page follows the code. Whether the comment counts the leader as one of
  the four is not stated in the pinned tree.
- The 24-byte size of a `SortTuple` slot in the validation-sort calculation is
  derived from the struct's fields on a 64-bit build
  ([tuplesort.h#SortTuple](../../../../raw/postgres-17/src/include/utils/tuplesort.h#L117-L153));
  the source does not state the number, and it was not confirmed by compiling
  or by running a server. A run with `trace_sort` on, which reports "internal
  sort" or "external sort" when the validation sort ends
  ([tuplesort.c#tuplesort_free-trace](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L930-L940)),
  would confirm the fit point.
- `read_stream.h` names `CREATE INDEX` as an example user of
  `READ_STREAM_MAINTENANCE`, but no index build in the pinned tree passes that
  flag. The heap scan passes only `READ_STREAM_SEQUENTIAL`, so the stream of
  each CIC heap scan selects `effective_io_concurrency`. The only caller in the
  tree that passes `READ_STREAM_MAINTENANCE` is the block-sampling stream of
  `ANALYZE`; the only other stream creator besides the heap scan is
  `pg_prewarm`. The pinned source does not say whether the comment's example
  describes an intention or a flag that was left out
  ([read_stream.h#READ_STREAM_MAINTENANCE](../../../../raw/postgres-17/src/include/storage/read_stream.h#L22-L27),
  [heapam.c#read-stream-setup](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L1237-L1259),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439),
  [analyze.c#block-sampling-read-stream](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1199-L1205),
  [pg_prewarm.c#prewarm-stream](../../../../raw/postgres-17/contrib/pg_prewarm/pg_prewarm.c#L263-L269)).
- The documentation says of `effective_io_concurrency`: "Currently, this setting
  only affects bitmap heap scans." The source also reads the setting when a heap
  sequential scan creates its read stream, which includes both CIC heap scans.
  There the value becomes `max_ios`, which cannot change the I/O of a stream
  that passes `READ_STREAM_SEQUENTIAL`. So the sentence agrees with the I/O
  these scans perform, but not with the list of code paths that read the
  setting. This page follows the source
  ([config.sgml#effective_io_concurrency-description](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2718-L2726),
  [read_stream.c#I/O-setting-selection](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L419-L439),
  [read_stream.c#advice-condition](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L508-L520)).
- The documentation describes `max_parallel_workers` as "the maximum number of
  workers that the cluster can support for parallel operations". In the source
  it is a `user`-context setting, and each registration compares the
  cluster-wide count of active parallel workers with the registering backend's
  own value. Two sessions with different values therefore apply different
  limits. This page states what the code does
  ([config.sgml#max_parallel_workers](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L2902-L2921),
  [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439),
  [bgworker.c#max_parallel_workers-test](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L996-L1014)).
- The source shows which code path each build, scan and storage setting selects.
  It does not establish how much elapsed time any of them saves on a given
  table, for example whether `lz4` is cheaper than `pglz` for wide index values,
  or how much a synchronized scan or a larger read helps. Those amounts need a
  measurement on the workload in question.
- The documentation describes `transaction_timeout` as applying to explicit
  transactions and to "an implicitly started transaction corresponding to a
  single statement". CIC is a single statement that runs four transactions, and
  the source arms the timer in every `StartTransaction()` and disarms it at
  every commit, so the limit applies to each of the four separately. The page
  follows the source. No in-tree test that runs CIC sets `transaction_timeout`
  or `statement_timeout`, so the per-transaction behavior and the interaction of
  the two timers during CIC are untested
  ([config.sgml#transaction_timeout](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9498-L9532),
  [xact.c#transaction-timeout-start](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2174-L2176),
  [xact.c#transaction-timeout-stop](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L2315-L2317),
  [indexcmds.c#transaction-3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1687-L1751)).
- The documentation of `deadlock_timeout` covers only the deadlock check and
  lock-wait logging, and the autovacuum documentation says that a conflicting
  lock request interrupts autovacuum without saying when. The source makes
  `deadlock_timeout` that delay: the cancel is sent only after the deadlock
  check has run. The page follows the source. No in-tree test that runs CIC
  arranges a conflict with autovacuum; one isolation spec only notes in a
  comment that its CIC could time out behind an autovacuum-held lock
  ([config.sgml#deadlock_timeout](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L10564-L10608),
  [maintenance.sgml#autovacuum-interruption](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L995-L1005),
  [deadlock.c#blocking-autovacuum](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L594-L618),
  [proc.c#autovacuum-cancel](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1493),
  [prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37)).

## Source References

One representative citation per cited source file, sorted by path:

- [amcheck/t/002_cic.pl](../../../../raw/postgres-17/contrib/amcheck/t/002_cic.pl#L4-L88)
- [amcheck/t/003_cic_2pc.pl](../../../../raw/postgres-17/contrib/amcheck/t/003_cic_2pc.pl#L4-L170)
- [blinsert.c#bloomBuildCallback](../../../../raw/postgres-17/contrib/bloom/blinsert.c#L70-L115)
- [bloom--1.0.sql:12](../../../../raw/postgres-17/contrib/bloom/bloom--1.0.sql#L12)
- [blutils.c:125](../../../../raw/postgres-17/contrib/bloom/blutils.c#L125)
- [blvacuum.c#tuple-loop](../../../../raw/postgres-17/contrib/bloom/blvacuum.c#L88-L107)
- [pageinspect/sql/brin.sql#parallel-build](../../../../raw/postgres-17/contrib/pageinspect/sql/brin.sql#L87-L115)
- [pg_prewarm.c#prewarm-stream](../../../../raw/postgres-17/contrib/pg_prewarm/pg_prewarm.c#L263-L269)
- [catalogs.sgml#indisvalid](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L4443-L4454)
- [config.sgml#fsync](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L3022-L3082)
- [gin.sgml#maintenance_work_mem](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L584-L593)
- [gist.sgml#default-build-choice](../../../../raw/postgres-17/doc/src/sgml/gist.sgml#L1226-L1234)
- [maintenance.sgml#autovacuum-interruption](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L995-L1005)
- [monitoring.sgml#pg_stat_progress_create_index-view](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L5847-L6016)
- [ref/create_index.sgml#memory-and-parallelism](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L798-L845)
- [ref/set.sgml#session-scope](../../../../raw/postgres-17/doc/src/sgml/ref/set.sgml#L32-L52)
- [brin.c#parallel-or-serial-build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1165-L1245)
- [brin_pageops.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L425-L449)
- [brin_tuple.c#compress-wide-summary](../../../../raw/postgres-17/src/backend/access/brin/brin_tuple.c#L215-L251)
- [indextuple.c#compress-wide-value](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138)
- [reloptions.c#effective_io_concurrency](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L348-L360)
- [syncscan.c#design](../../../../raw/postgres-17/src/backend/access/common/syncscan.c#L6-L19)
- [toast_internals.c#toast_compress_datum](../../../../raw/postgres-17/src/backend/access/common/toast_internals.c#L32-L104)
- [gin/README#inverted-index-structure](../../../../raw/postgres-17/src/backend/access/gin/README#L17-L26)
- [ginbulk.c#ginAllocEntryAccumulator](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L83-L106)
- [gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L734-L864)
- [ginentrypage.c:68](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L68)
- [ginfast.c#flush-or-advance](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L897-L1000)
- [gininsert.c#gininsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L482-L536)
- [ginutil.c:56](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L56)
- [ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L457-L569)
- [gist.c:78](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L78)
- [gistbuild.c#gistInitBuffering](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L617-L777)
- [gistbuildbuffers.c#temporary-file](../../../../raw/postgres-17/src/backend/access/gist/gistbuildbuffers.c#L53-L57)
- [gistutil.c#index_form_tuple-call](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L582-L584)
- [gistvacuum.c#callback-loop](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L336-L354)
- [hash.c#hashbuildCallback](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L206-L242)
- [hashinsert.c#insert-WAL](../../../../raw/postgres-17/src/backend/access/hash/hashinsert.c#L215-L235)
- [hashpage.c#initial-bucket-loop](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L415-L437)
- [hashsort.c#_h_indexbuild](../../../../raw/postgres-17/src/backend/access/hash/hashsort.c#L115-L157)
- [README.HOT#HOT-update-requirement](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L136-L147)
- [heapam.c#heap_prepare_pagescan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L601-L690)
- [heapam_handler.c#heapam_index_validate_scan](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1747-L1986)
- [heapam_visibility.c#SetHintBits](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L82-L132)
- [amapi.c#GetIndexAmRoutine](../../../../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46)
- [nbtinsert.c#unique-conflict-wait](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L213-L233)
- [nbtpage.c#no-xid-comment](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2637)
- [nbtree.c#btreevacuumposting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1396-L1449)
- [nbtsort.c#_bt_parallel_scan_and_sort](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1849-L1963)
- [nbtutils.c#btbuildphasename](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4604-L4625)
- [spginsert.c#spgbuild](../../../../raw/postgres-17/src/backend/access/spgist/spginsert.c#L69-L148)
- [spgutils.c:63](../../../../raw/postgres-17/src/backend/access/spgist/spgutils.c#L63)
- [spgvacuum.c#live-tuple-callback](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L153-L181)
- [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L388-L404)
- [generic_xlog.c#GenericXLogFinish](../../../../raw/postgres-17/src/backend/access/transam/generic_xlog.c#L332-L436)
- [parallel.c#worker-registration-loop](../../../../raw/postgres-17/src/backend/access/transam/parallel.c#L611-L646)
- [twophase.c:462](../../../../raw/postgres-17/src/backend/access/transam/twophase.c#L462)
- [xact.c#sync-versus-async-commit](../../../../raw/postgres-17/src/backend/access/transam/xact.c#L1462-L1521)
- [xlog.c#LogCheckpointStart](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6623-L6653)
- [xloginsert.c#log_newpage_range](../../../../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1252-L1342)
- [xlogprefetcher.c#recovery-prefetch-limit](../../../../raw/postgres-17/src/backend/access/transam/xlogprefetcher.c#L1000-L1005)
- [dependency.c#CheckUsageOnTypesInSingleRelExpr](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1740-L1766)
- [index.c#validate_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3324-L3452)
- [storage.c#RelationCreateStorage](../../../../raw/postgres-17/src/backend/catalog/storage.c#L107-L180)
- [system_views.sql#pg_stat_progress_create_index](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1256-L1289)
- [analyze.c#analyze_rel-lock](../../../../raw/postgres-17/src/backend/commands/analyze.c#L134-L145)
- [dbcommands.c:1526](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L1526)
- [indexcmds.c#DefineIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L539-L1793)
- [tablecmds.c#attach-partition-clone-index](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18999-L19016)
- [tablespace.c#GetDefaultTablespace](../../../../raw/postgres-17/src/backend/commands/tablespace.c#L1126-L1182)
- [vacuum.c#cost-accounting-scope](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L599-L684)
- [vacuumparallel.c#mwm-worker-share](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L372-L375)
- [variable.c#check_effective_io_concurrency](../../../../raw/postgres-17/src/backend/commands/variable.c#L1222-L1233)
- [execIndexing.c#skip-not-ready](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L357-L359)
- [makefuncs.c:849](../../../../raw/postgres-17/src/backend/nodes/makefuncs.c#L849)
- [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4187-L4279)
- [costsize.c#effective_cache_size-use](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L914-L915)
- [planner.c#plan_create_index_workers](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6876-L7019)
- [plancat.c#skip-invalid-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267)
- [relnode.c:207](../../../../raw/postgres-17/src/backend/optimizer/util/relnode.c#L207)
- [autovacuum.c#VacuumUpdateCosts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1630-L1695)
- [bgworker.c#slot-search](../../../../raw/postgres-17/src/backend/postmaster/bgworker.c#L1016-L1043)
- [checkpointer.c#reload](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L566-L583)
- [syncrep.c#SyncRepWaitForLSN](../../../../raw/postgres-17/src/backend/replication/syncrep.c#L133-L363)
- [read_stream.c#read_stream_start_pending_read](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L211-L299)
- [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5036-L5182)
- [freelist.c#GetAccessStrategyPinLimit](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L632-L672)
- [buffile.c#BufFileCreateTemp](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L180-L216)
- [fd.c#PathNameDeleteTemporaryFile](../../../../raw/postgres-17/src/backend/storage/file/fd.c#L1927-L1972)
- [fileset.c#FileSetInit](../../../../raw/postgres-17/src/backend/storage/file/fileset.c#L36-L86)
- [sharedfileset.c#SharedFileSetOnDetach](../../../../raw/postgres-17/src/backend/storage/file/sharedfileset.c#L88-L114)
- [procarray.c#GetCurrentVirtualXIDs](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L3296-L3378)
- [deadlock.c#DeadLockReport](../../../../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L1068-L1136)
- [lmgr.c#WaitForLockersMultiple](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L889-L973)
- [lock.c#VirtualXactLock](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4550-L4663)
- [proc.c#lock-wait-logging](../../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1495-L1643)
- [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L124-L223)
- [postgres.c#query-cancel-errors](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L3372-L3430)
- [utility.c#IndexStmt-lock](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1465-L1480)
- [Gen_fmgrtab.pl#header](../../../../raw/postgres-17/src/backend/utils/Gen_fmgrtab.pl#L2-L15)
- [utils/Makefile#fmgr-stamp](../../../../raw/postgres-17/src/backend/utils/Makefile#L48-L53)
- [backend_progress.c#pgstat_progress_start_command](../../../../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L20-L40)
- [pgstat_database.c#pgstat_report_tempfile](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_database.c#L171-L185)
- [relcache.c#InitIndexAmRoutine](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L1398-L1422)
- [spccache.c#get_tablespace_io_concurrency](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L207-L223)
- [elog.c#is_log_level_output](../../../../raw/postgres-17/src/backend/utils/error/elog.c#L196-L228)
- [globals.c:159](../../../../raw/postgres-17/src/backend/utils/init/globals.c#L159)
- [postinit.c#InitializeMaxBackends](../../../../raw/postgres-17/src/backend/utils/init/postinit.c#L565-L588)
- [utils/misc/Makefile#OBJS](../../../../raw/postgres-17/src/backend/utils/misc/Makefile#L17-L34)
- [guc.c#RestrictSearchPath](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L2242-L2253)
- [guc_tables.c#synchronous_commit_options](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L340-L357)
- [utils/misc/meson.build#backend_sources](../../../../raw/postgres-17/src/backend/utils/misc/meson.build#L3-L20)
- [tuplesort.c#grow_memtuples](../../../../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L1056-L1183)
- [tuplesortvariants.c#tuplesort_begin_datum](../../../../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L583-L661)
- [initdb.c#test_specific_config_settings](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1199-L1241)
- [amapi.h#ambuild_function](../../../../raw/postgres-17/src/include/access/amapi.h#L102-L105)
- [gin_private.h#GIN-options](../../../../raw/postgres-17/src/include/access/gin_private.h#L23-L45)
- [hash.h:296](../../../../raw/postgres-17/src/include/access/hash.h#L296)
- [heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68)
- [xact.h#SyncCommitLevel](../../../../raw/postgres-17/src/include/access/xact.h#L68-L80)
- [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L109-L118)
- [xlog_internal.h:92](../../../../raw/postgres-17/src/include/access/xlog_internal.h#L92)
- [xlogdefs.h#DEFAULT_WAL_SYNC_METHOD](../../../../raw/postgres-17/src/include/access/xlogdefs.h#L68-L81)
- [pg_am.dat#index-access-methods](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18-L35)
- [pg_index.h#state-flags](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)
- [miscadmin.h:269](../../../../raw/postgres-17/src/include/miscadmin.h#L269)
- [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34)
- [pg_config_manual.h#flush-after-defaults](../../../../raw/postgres-17/src/include/pg_config_manual.h#L165-L179)
- [pg_iovec.h#PG_IOV_MAX](../../../../raw/postgres-17/src/include/port/pg_iovec.h#L42-L43)
- [syncrep.h#SyncRepRequested](../../../../raw/postgres-17/src/include/replication/syncrep.h#L18-L19)
- [bufmgr.h#I/O-concurrency-defaults](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L164)
- [lockdefs.h#lock-modes](../../../../raw/postgres-17/src/include/storage/lockdefs.h#L36-L46)
- [proc.h#statusFlags](../../../../raw/postgres-17/src/include/storage/proc.h#L54-L78)
- [read_stream.h#READ_STREAM_SEQUENTIAL](../../../../raw/postgres-17/src/include/storage/read_stream.h#L29-L35)
- [guc.h#GucContext](../../../../raw/postgres-17/src/include/utils/guc.h#L35-L76)
- [rel.h#RelationNeedsWAL](../../../../raw/postgres-17/src/include/utils/rel.h#L620-L631)
- [tuplesort.h#SortTuple](../../../../raw/postgres-17/src/include/utils/tuplesort.h#L117-L153)
- [isolation/Makefile#prepared-txns-targets](../../../../raw/postgres-17/src/test/isolation/Makefile#L67-L74)
- [multiple-cic.out](../../../../raw/postgres-17/src/test/isolation/expected/multiple-cic.out#L1-L23)
- [prepared-transactions-cic.out#expected-output](../../../../raw/postgres-17/src/test/isolation/expected/prepared-transactions-cic.out#L1-L19)
- [multiple-cic.spec](../../../../raw/postgres-17/src/test/isolation/specs/multiple-cic.spec#L1-L43)
- [prepared-transactions-cic.spec#spec](../../../../raw/postgres-17/src/test/isolation/specs/prepared-transactions-cic.spec#L1-L37)
- [reindex_conc.sql](../../../../raw/postgres-17/src/test/modules/injection_points/sql/reindex_conc.sql#L1-L28)
- [create_index.out#invalid-index-repaired](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L1448-L1484)
- [privileges.out#rebuilds-need-no-usage](../../../../raw/postgres-17/src/test/regress/expected/privileges.out#L1417-L1428)
- [brin.sql#cic-as-wait](../../../../raw/postgres-17/src/test/regress/sql/brin.sql#L486-L490)
- [cluster.sql#external-tuplesort-test](../../../../raw/postgres-17/src/test/regress/sql/cluster.sql#L256-L273)
- [create_index.sql#concurrent-builds](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L481-L582)
- [index_including.sql#cic](../../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L161-L168)
- [index_including_gist.sql#cic](../../../../raw/postgres-17/src/test/regress/sql/index_including_gist.sql#L37-L44)
- [privileges.sql#index-functions-run-as-owner](../../../../raw/postgres-17/src/test/regress/sql/privileges.sql#L1227-L1248)
- [vacuum.sql#parallel-vacuum-minimum-memory](../../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L137-L143)

## Navigation

- [v17 index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [v12: How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](../../../v12/questions/indexing/create-index-concurrently.md)
