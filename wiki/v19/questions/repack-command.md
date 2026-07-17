---
type: question
version: 19
pinned_commit: 3aa54433b0cdce48facb610a5b720208cc760654
verified: false
verified_by_agent: not yet
---

# How the REPACK Command Works in PostgreSQL 19, and Its 47 Feature-Scope Commits (unverified)

## Contents

- [Question](#question)
- [Short Answer](#short-answer)
- [Syntax and Modes](#syntax-and-modes)
- [Source Layout](#source-layout)
- [Blocking REPACK (the simple rewrite)](#blocking-repack-the-simple-rewrite)
  - [Multi-table discovery and recheck](#multi-table-discovery-and-recheck)
- [REPACK CONCURRENTLY (online rewrite via logical decoding)](#repack-concurrently-online-rewrite-via-logical-decoding)
  - [Step by step](#step-by-step)
  - [How concurrent changes are captured and replayed](#how-concurrent-changes-are-captured-and-replayed)
  - [Error handling and worker teardown](#error-handling-and-worker-teardown)
  - [WAL retention and `effective_wal_level`](#wal-retention-and-effective_wal_level)
- [Restrictions on CONCURRENTLY](#restrictions-on-concurrently)
- [GUC: `max_repack_replication_slots`](#guc-max_repack_replication_slots)
- [I/O Impact and Throttling](#io-impact-and-throttling)
- [Progress reporting](#progress-reporting)
- [Tests](#tests)
- [Feature-Scope Commits](#feature-scope-commits)
  - [Foundational](#foundational)
  - [CONCURRENTLY infrastructure](#concurrently-infrastructure)
  - [Correctness fixes](#correctness-fixes)
  - [Error messages, docs, style](#error-messages-docs-style)
  - [Tests](#tests-1)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Source References](#source-references)
- [Open Questions](#open-questions)
- [Related Pages](#related-pages)

## Question

Follow AGENTS.md.
In PostgreSQL 19, do a comprehensive explanation of how `pg_repack` works, explain all commits to the feature.

> Scope note (agreed with the asker): the third-party `pg_repack` extension is **not** in the pinned `raw/postgres-19/` checkout, so it cannot be cited under the wiki evidence rules. PostgreSQL 19 instead adds an **in-core `REPACK` command** (with a `CONCURRENTLY` option) that brings pg_repack/pg_squeeze-style online table rewriting into core. This page documents that in-core feature.

## Short Answer

`REPACK` is a new top-level SQL command in PostgreSQL 19 that rewrites a table into a fresh relfilenode to reclaim dead-tuple space, optionally clustering rows by an index. It **absorbs the functionality of `VACUUM FULL` and `CLUSTER`** into one command: physical-order rewrite (like `VACUUM FULL`) and index-order rewrite (like `CLUSTER`) are just two modes of `REPACK`, and the two legacy commands are kept but de-emphasized [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L22) [ac58465e commit message](#feature-scope-commits). The command is documented as "rewrite a table to reclaim disk space" [ref/repack.sgml#refpurpose](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L17-L20).

`REPACK` has two execution modes:

- **Blocking (default).** Takes `AccessExclusiveLock`, builds a new heap, copies live tuples in the chosen order, rebuilds indexes via `REINDEX`, then swaps relfilenodes. This is the classic `CLUSTER`/`VACUUM FULL` path, now reached through `repack.c` [repack.c#rebuild_relation](../../../raw/postgres-19/src/backend/commands/repack.c#L1134-L1159).
- **`CONCURRENTLY`.** Takes only `ShareUpdateExclusiveLock` for almost all of the work, so readers and writers keep using the table. It captures concurrent DML with **logical decoding** through a dedicated background worker and a temporary replication slot, replays those changes onto the new heap, and only upgrades to `AccessExclusiveLock` for the brief final relfilenode swap [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L7-L21) [ref/repack.sgml#CONCURRENTLY](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L221-L234).

This page tracks **47 feature-scope commits (2026-03-10 to 2026-07-16)**: commits whose subject/body explicitly references REPACK, adjacent prerequisite/support commits used by that history, and tree-wide fixes that changed REPACK-specific code or text. The bulk were authored by Antonin Houska and Álvaro Herrera, building on Houska's `pg_squeeze` extension. They are listed and explained under [Feature-Scope Commits](#feature-scope-commits).

## Syntax and Modes

The grammar produces a single `RepackStmt` node for `REPACK` and `CLUSTER`, distinguished by a `RepackCommand` tag [parsenodes.h#RepackStmt](../../../raw/postgres-19/src/include/nodes/parsenodes.h#L4092-L4107) [gram.y#RepackStmt](../../../raw/postgres-19/src/backend/parser/gram.y#L12587-L12680). `VACUUM (FULL)` still parses as `VacuumStmt`; when the `FULL` option is present, `vacuum.c` routes execution into the same rewrite engine with `REPACK_COMMAND_VACUUMFULL` [gram.y#VacuumStmt](../../../raw/postgres-19/src/backend/parser/gram.y#L12700-L12732) [vacuum.c#VACUUMFULL-repack](../../../raw/postgres-19/src/backend/commands/vacuum.c#L2302-L2305):

```
REPACK [ ( option [, ...] ) ] [ table [ ( column [, ...] ) ] [ USING INDEX [ index_name ] ] ]
REPACK [ ( option [, ...] ) ] USING INDEX
```

where `option` is `VERBOSE`, `ANALYZE`, or `CONCURRENTLY` [ref/repack.sgml#synopsis](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L22-L36). Key forms:

| Form | Effect |
|---|---|
| `REPACK t` | Rewrite `t` in physical order (like `VACUUM FULL`) [ref/repack.sgml#Description](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L42-L50). |
| `REPACK t USING INDEX idx` | Rewrite `t` ordered by `idx`, and mark `idx` as the clustered index (like `CLUSTER`) [ref/repack.sgml#clustering](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L79-L93). |
| `REPACK t USING INDEX` | Use the table's already-clustered index; error if none [repack.c#determine_clustered_index](../../../raw/postgres-19/src/backend/commands/repack.c#L2467-L2508). |
| `REPACK` (no table) | Process every table/matview the caller has `MAINTAIN` on; not allowed in a transaction block or with `CONCURRENTLY` [ref/repack.sgml#no-table](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L52-L60). |
| `REPACK USING INDEX` (no table) | Process all tables that have a clustering index configured [ref/repack.sgml#no-table-using-index](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L95-L99). |
| `REPACK (CONCURRENTLY) t` | Online rewrite; requires an explicit single non-partitioned table [repack.c#multi-table-guard](../../../raw/postgres-19/src/backend/commands/repack.c#L337-L353). |

`utility.c` dispatches `T_RepackStmt` to `ExecRepack()`, and reports the command tag as `REPACK` unless the statement is a legacy `CLUSTER` [utility.c#ExecRepack](../../../raw/postgres-19/src/backend/tcop/utility.c#L866-L867) [utility.c#CMDTAG](../../../raw/postgres-19/src/backend/tcop/utility.c#L2894-L2898). `VACUUM (FULL)` reaches the same engine by calling `cluster_rel(REPACK_COMMAND_VACUUMFULL, …)` from `vacuum.c` [vacuum.c#cluster_rel](../../../raw/postgres-19/src/backend/commands/vacuum.c#L2303-L2305). To run `REPACK`, the caller needs the `MAINTAIN` privilege on the table [repack.c#repack_is_permitted](../../../raw/postgres-19/src/backend/commands/repack.c#L2323-L2362) [ref/repack.sgml#Notes](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L355-L358).

The generic privilege reference now lists `REPACK` among the commands authorized by `MAINTAIN`, and the predefined `pg_maintain` role grants that right on all relations [ddl.sgml#MAINTAIN](../../../raw/postgres-19/doc/src/sgml/ddl.sgml#L2536-L2548) [user-manag.sgml#pg_maintain](../../../raw/postgres-19/doc/src/sgml/user-manag.sgml#L655-L670). At runtime, membership in `pg_maintain` adds `ACL_MAINTAIN` when the relation ACL is checked [aclchk.c#pg_class_aclmask_ext](../../../raw/postgres-19/src/backend/catalog/aclchk.c#L3419-L3428).

## Source Layout

```text
src/backend/commands/repack.c            ExecRepack, cluster_rel, the rewrite engine
                                         (formerly cluster.c; renamed in c0b53ec0)
src/backend/commands/repack_worker.c     background worker that does the logical decoding
src/backend/replication/pgrepack/        "pgrepack" logical-decoding output plugin
  pgrepack.c                             change_cb spills tuples to per-relation files
src/include/commands/repack.h            ExecRepack/cluster_rel API, CLUOPT_* flags
src/include/commands/repack_internal.h   ConcurrentChangeKind, RepackDecodingState,
                                         DecodingWorkerShared
doc/src/sgml/ref/repack.sgml             REPACK reference page
```

The output plugin is built as `pgrepack` and linked into the backend (`backend_targets += pgrepack`), not shipped as a `contrib` extension; the worker loads it by the fixed name `"pgrepack"` [pgrepack/meson.build](../../../raw/postgres-19/src/backend/replication/pgrepack/meson.build#L13-L18) [repack_worker.c#PGREPACK_PLUGIN](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L31). The plugin refuses any use outside the REPACK worker: `repack_startup()` raises `unsupported use of logical decoding plugin` unless `AmRepackWorker()` is true, so it cannot be driven from the SQL logical-decoding interface (commit `cd7b204b`) [pgrepack.c#repack_startup](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L50-L82). Despite the file/dir name `pgrepack`, the user-facing command is `REPACK`; the third-party extension is unrelated.

## Blocking REPACK (the simple rewrite)

`ExecRepack()` parses the option list (`VERBOSE`/`ANALYZE`/`CONCURRENTLY`), sets the `CLUOPT_*` bitmask in `ClusterParams`, picks the lock level, and either processes one relation or builds a work list for the multi-table forms [repack.c#ExecRepack](../../../raw/postgres-19/src/backend/commands/repack.c#L249-L489) [repack.h#CLUOPT](../../../raw/postgres-19/src/include/commands/repack.h#L24-L36). Multiple tables are each processed in their own transaction (like `VACUUM`) to avoid holding many strong locks at once, which is why the multi-table and `CONCURRENTLY` forms cannot run inside a transaction block [repack.c#per-table-txn](../../../raw/postgres-19/src/backend/commands/repack.c#L226-L248).

`cluster_rel()` is the per-relation driver. It starts progress reporting, switches to the table owner's userid under a security-restricted context, sets `search_path` to `pg_catalog, pg_temp`, rechecks the relation in the multi-table case, validates the clustering index, and then calls `rebuild_relation()` [repack.c#cluster_rel](../../../raw/postgres-19/src/backend/commands/repack.c#L531-L712) [ref/repack.sgml#search_path](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L360-L364). The lock level is `AccessExclusiveLock` for blocking mode and `ShareUpdateExclusiveLock` for concurrent mode [repack.c#RepackLockLevel](../../../raw/postgres-19/src/backend/commands/repack.c#L498-L505).

`rebuild_relation()` (blocking path):

1. If an index was given, mark it `indisclustered` [repack.c#mark_index_clustered](../../../raw/postgres-19/src/backend/commands/repack.c#L1079-L1081).
2. `make_new_heap()` creates a transient `pg_temp_<oid>` heap with the old heap's tuple descriptor, reloptions, access method, and (if needed) a TOAST table [repack.c#make_new_heap](../../../raw/postgres-19/src/backend/commands/repack.c#L1173-L1247).
3. `copy_table_data()` copies live tuples into the new heap. It chooses **sequential-scan-and-sort** vs **index scan** vs **plain seqscan** via `plan_cluster_use_sort()` for a btree clustering index, computes freeze/multixact cutoffs, and hands the actual copy to the table AM's `table_relation_copy_for_cluster()` [repack.c#copy_table_data](../../../raw/postgres-19/src/backend/commands/repack.c#L1433-L1468).
4. `finish_heap_swap()` swaps the relfilenodes of old and new heaps, rebuilds the indexes (`reindex = true`), applies the new `relfrozenxid`/`relminmxid`, and drops the transient heap [repack.c#finish_heap_swap-call](../../../raw/postgres-19/src/backend/commands/repack.c#L1150-L1158).

Predicate (SSI) locks on tuples/pages are promoted to a relation lock before the rewrite, because tuples move [repack.c#TransferPredicateLocks](../../../raw/postgres-19/src/backend/commands/repack.c#L670-L681).

### Multi-table discovery and recheck

The no-table forms and partition fan-out first collect lightweight `RelToCluster { tableOid, indexOid }` records. They do **not** retain a lock on each target while building that list, so each OID pair is only a candidate and may become stale before its turn [repack.c#RelToCluster](../../../raw/postgres-19/src/backend/commands/repack.c#L86-L96) [repack.c#get_tables_to_repack](../../../raw/postgres-19/src/backend/commands/repack.c#L2139-L2255) [repack.c#get_tables_to_repack_partitioned](../../../raw/postgres-19/src/backend/commands/repack.c#L2257-L2319).

For each candidate, `ExecRepack()` starts a new transaction and obtains the final mode from `RepackLockLevel()` with `try_relation_open()`. It silently skips an OID that was dropped or now names something other than a plain table or materialized view. With that target locked, `cluster_rel_recheck()` checks the caller's `MAINTAIN` privilege, remote-temporary-table status, index existence, and — where required — whether the index is still marked `indisclustered` [repack.c#per-table-open](../../../raw/postgres-19/src/backend/commands/repack.c#L448-L481) [repack.c#cluster_rel_recheck](../../../raw/postgres-19/src/backend/commands/repack.c#L714-L768). The unlocked discovery-time permission probe also treats a concurrently dropped relation as absent and returns without warning [repack.c#repack_is_permitted_for_relation](../../../raw/postgres-19/src/backend/commands/repack.c#L2323-L2362).

Commit `133eba078f7` made these rules consistent. The earlier whole-database list path briefly acquired transaction-scoped `AccessShareLock`s, but the database transaction committed immediately after list construction and therefore released them before processing. The commit removed those ineffective locks and hardened the locked per-table path against drops and OIDs that now identify a non-table relkind. It added no direct test or documentation change.

## REPACK CONCURRENTLY (online rewrite via logical decoding)

The concurrent path keeps the table online by combining a low-strength lock with logical decoding. The cast of characters:

| Component | Role |
|---|---|
| **Leader backend** | Runs `REPACK`; holds `ShareUpdateExclusiveLock`, copies data, applies captured changes, does the final swap [repack.c#rebuild_relation](../../../raw/postgres-19/src/backend/commands/repack.c#L1043-L1133). |
| **Decoding worker** | `bgworker` running `RepackWorkerMain`; owns a temporary logical slot and decodes WAL into spill files [repack_worker.c#RepackWorkerMain](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L59-L164). |
| **`pgrepack` output plugin** | Decoding callbacks; writes each insert/update/delete tuple of the target relation to the current spill file [pgrepack.c#change_cb](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L113-L182). |
| **Temporary replication slot** | `pg_repack_<pid>`, `RS_TEMPORARY` so it is dropped on error/exit; pins the WAL/snapshot start point [repack_worker.c#slot](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L217-L223). |
| **DSM + spill fileset** | `DecodingWorkerShared` in DSM coordinates the two processes; a `SharedFileSet` holds the exported snapshot and change files [repack_internal.h#DecodingWorkerShared](../../../raw/postgres-19/src/include/commands/repack_internal.h#L61-L117). |

### Step by step

1. **Precondition checks.** `check_concurrent_repack_requirements()` enforces `wal_level >= replica`, rejects catalogs, TOAST tables, non-permanent (temp/unlogged) tables, and `REPLICA IDENTITY NOTHING`/`FULL`, and obtains the **identity index** (replica-identity index or a non-deferrable primary key). Without a usable identity index, concurrent mode is refused [repack.c#check_concurrent_repack_requirements](../../../raw/postgres-19/src/backend/commands/repack.c#L907-L998).
2. **Start the worker before any XID is assigned.** The leader becomes a lock-group leader, then `start_repack_decoding_worker()` sets up the DSM, registers the `bgworker`, and blocks until the worker reports `initialized` [repack.c#start_repack_decoding_worker](../../../raw/postgres-19/src/backend/commands/repack.c#L1043-L1069) [repack.c#start_repack_decoding_worker-impl](../../../raw/postgres-19/src/backend/commands/repack.c#L3587-L3669). The worker creates the temporary slot, enables logical decoding, restricts decoding to the target relation (and its TOAST relation), builds an initial historic snapshot, and exports it to a file [repack_worker.c#setup](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L194-L291).
3. **Initial copy under the exported snapshot.** The leader reads the exported snapshot with `get_initial_snapshot()`, pushes it active, and runs `copy_table_data()` against that MVCC snapshot, so the initial copy is a consistent point-in-time image [repack.c#get_initial_snapshot](../../../raw/postgres-19/src/backend/commands/repack.c#L1071-L1115) [repack.c#get_initial_snapshot-impl](../../../raw/postgres-19/src/backend/commands/repack.c#L3731-L3780). Meanwhile the worker keeps decoding WAL in a loop, and later advances the slot's restart/confirmed LSN at WAL-segment boundaries so processed WAL can be recycled [repack_worker.c#decode_loop](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L152-L164) [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L374-L403).
4. **Build new indexes** on the new heap (instead of reindexing the old one), since this is slow and is done before the exclusive lock [repack.c#build_new_indexes](../../../raw/postgres-19/src/backend/commands/repack.c#L3383-L3414).
5. **Catch-up #1 (still under `ShareUpdateExclusiveLock`).** `process_concurrent_changes()` tells the worker an `lsn_upto`, waits for the next spill file, and `apply_concurrent_changes()` replays it onto the new heap. Doing this first minimizes how long the exclusive lock is later held [repack.c#catch-up-1](../../../raw/postgres-19/src/backend/commands/repack.c#L3236-L3249).
6. **Lock upgrade + catch-up #2.** The leader takes `AccessExclusiveLock` on the table, all its indexes, and its TOAST relation, transfers predicate locks, flushes WAL, and applies the remaining changes with `done = true` so the worker exits [repack.c#lock-upgrade](../../../raw/postgres-19/src/backend/commands/repack.c#L3252-L3306).
7. **Swap.** `swap_relation_files()` swaps each index's storage, then `finish_heap_swap()` swaps the heap (and TOAST), dropping the transient relation. The old OID, grants, and dependencies are preserved [repack.c#swap](../../../raw/postgres-19/src/backend/commands/repack.c#L3319-L3370).

### How concurrent changes are captured and replayed

The `pgrepack` plugin's `change_cb` only ever sees the target relation (other relations are filtered out during decoding by `change_useless_for_repack()`), and writes a compact record per change: a one-byte kind (`i`/`u`/`U`/`d`), the heap tuple bytes, and any out-of-line external attributes spilled separately to stay under `MaxAllocSize` [pgrepack.c#repack_store_change](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L193-L308) [repack_worker.c#change_useless_for_repack](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L499-L536) [repack_internal.h#ConcurrentChangeKind](../../../raw/postgres-19/src/include/commands/repack_internal.h#L24-L32).

`apply_concurrent_changes()` reads the spill file and replays each change onto the new heap, bumping the command counter between dependent changes [repack.c#apply_concurrent_changes](../../../raw/postgres-19/src/backend/commands/repack.c#L2528-L2648):

- **INSERT** → `table_tuple_insert(..., TABLE_INSERT_NO_LOGICAL)` plus index maintenance. The `NO_LOGICAL` flag stops these replayed rows from being decoded again [repack.c#apply_concurrent_insert](../../../raw/postgres-19/src/backend/commands/repack.c#L2654-L2669).
- **UPDATE/DELETE** → locate the existing tuple in the new heap via the identity index (`find_target_tuple()`), then `table_tuple_update`/`table_tuple_delete` with `NO_LOGICAL`. For updates, unchanged TOAST pointers are re-pointed at the new relation's TOAST table via `adjust_toast_pointers()` [repack.c#apply_concurrent_update](../../../raw/postgres-19/src/backend/commands/repack.c#L2675-L2739) [repack.c#toast-pointers](../../../raw/postgres-19/src/backend/commands/repack.c#L2621-L2631).

The identity-index scan key is precomputed in `initialize_change_context()`, which resolves the equality strategy from the index's operator family (so it works for non-btree identity indexes too — see commit `36f52a59`) [repack.c#initialize_change_context](../../../raw/postgres-19/src/backend/commands/repack.c#L3005-L3144). When replay reconstructs a spilled tuple, `restore_tuple()` checks that every separately stored varlena attribute is consumed and errors if the spill file contains too few or too many separately stored attributes; commit `e2a8cabc` added the leftover-attribute check [repack.c#restore_tuple](../../../raw/postgres-19/src/backend/commands/repack.c#L2752-L2818).

### Error handling and worker teardown

The leader runs `rebuild_relation()` inside `PG_ENSURE_ERROR_CLEANUP(stop_repack_decoding_worker_cb)` so the worker and its slot are cleaned up even on `FATAL` exit (e.g. `pg_terminate_backend()`), and `stop_repack_decoding_worker()` terminates the worker, detaches the error queue, cancels CV sleeps, and frees the DSM [repack.c#ensure-cleanup](../../../raw/postgres-19/src/backend/commands/repack.c#L692-L700) [repack.c#stop_repack_decoding_worker](../../../raw/postgres-19/src/backend/commands/repack.c#L3677-L3726). Worker errors are forwarded to the leader through a shared-memory message queue and re-thrown with a "REPACK decoding worker" context line, using the `PROCSIG_REPACK_MESSAGE` interrupt and `ProcessRepackMessages()` [repack.c#ProcessRepackMessages](../../../raw/postgres-19/src/backend/commands/repack.c#L3802-L3940) [repack_worker.c#RepackWorkerShutdown](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L169-L179).

### WAL retention and `effective_wal_level`

Normally a replication slot pins `restart_lsn` so WAL can't be recycled. `REPACK` does not need crash-safe replay (a crash just restarts the whole operation), so the worker advances the slot's restart/confirmed LSN as it crosses each WAL-segment boundary, letting old WAL be recycled while `REPACK` runs (commit `45b02984`) [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L374-L403). Because the temporary slot is logical, the worker calls `EnsureLogicalDecodingEnabled()` at setup and `ReplicationSlotDropAcquired(true)` at teardown so `effective_wal_level` does not stay stuck at `logical` afterward (commit `2af1dc89`) [repack_worker.c#enable-decoding](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L211-L224) [repack_worker.c#cleanup](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L293-L304).

## Restrictions on CONCURRENTLY

`REPACK (CONCURRENTLY)` is rejected for any of: an `UNLOGGED` or temporary table; a partitioned table; a table without a primary key or index-based replica identity (including deferrable PKs); a system catalog or TOAST table; execution inside a transaction block; or when `max_repack_replication_slots` leaves no slot free [ref/repack.sgml#cannot-use](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L259-L302) [repack.c#check_concurrent_repack_requirements](../../../raw/postgres-19/src/backend/commands/repack.c#L907-L998). It is documented as **not MVCC-safe** [ref/repack.sgml#warning](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L304-L310). After the `01a80f06` revert (see commits), only **one** `REPACK (CONCURRENTLY)` may run at a time across the whole instance.

## GUC: `max_repack_replication_slots`

Concurrent `REPACK` consumes a logical replication slot for its lifetime. Rather than forcing users to inflate `max_replication_slots`, v19 adds a dedicated pool sized by `max_repack_replication_slots` (default `5`, min `0`) [guc_parameters.dat#max_repack_replication_slots](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2108-L2114) [slot.c#decl](../../../raw/postgres-19/src/backend/replication/slot.c#L163-L164). `ReplicationSlotCreate(..., repack = true, ...)` allocates from the extra `[max_replication_slots, max_replication_slots + max_repack_replication_slots)` band of the slot array, so REPACK slots never compete with ordinary replication slots [slot.c#ReplicationSlotCreate](../../../raw/postgres-19/src/backend/replication/slot.c#L372-L463).

**Scope:** `max_repack_replication_slots` is `PGC_POSTMASTER` — it **requires a server restart** [guc_parameters.dat#context](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2108-L2108). The decoding worker also needs a free `max_worker_processes` slot, or `REPACK (CONCURRENTLY)` errors out [repack.c#worker-slots](../../../raw/postgres-19/src/backend/commands/repack.c#L3642-L3646).

## I/O Impact and Throttling

**There is no built-in way to throttle `REPACK`.** Unlike lazy (non-`FULL`) `VACUUM`, the rewrite path has **no cost-based delay**: `cluster_rel()` and the heap copy it drives contain no `vacuum_delay_point()` calls [repack.c#cluster_rel](../../../raw/postgres-19/src/backend/commands/repack.c#L531-L712) [heapam_handler.c#heapam_relation_copy_for_cluster](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L594-L602). Consequently `vacuum_cost_delay` / `vacuum_cost_limit` (and the autovacuum cost knobs) **have no effect** on `REPACK`, `CLUSTER`, or `VACUUM FULL` — there is no GUC or option that tells the command to run slower.

What does exist:

| Lever | Effect on I/O | Notes |
|---|---|---|
| **BULKREAD ring buffer** (automatic) | Limits shared-buffer pollution while reading the old heap | Only on the **sequential-scan** path, and only once the relation exceeds `NBuffers/4` [heapam.c#initscan](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L397-L410). |
| **BULKWRITE ring buffer** (automatic, CONCURRENTLY) | Limits cache pollution while inserting into the new heap | `GetBulkInsertState()` uses `BAS_BULKWRITE` [heapam.c#GetBulkInsertState](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L1935-L1943). |
| `maintenance_work_mem` | Bounds sort / index-build memory; does **not** throttle | Sizes `tuplesort_begin_cluster` [heapam_handler.c#tuplesort](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L651-L655). It is `PGC_USERSET`, so settable per session (no restart/reload) [guc_parameters.dat#maintenance_work_mem](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1951-L1959). High values trade RAM for fewer temp-file write bursts. |
| `CONCURRENTLY` | Reduces **lock** contention, not I/O | Avoids the long `AccessExclusiveLock` [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L7-L21) but does *more* total I/O (decode + spill + replay). It does let old WAL recycle as it runs [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L374-L403). |

The read-side ring buffer is the only automatic "niceness," and it has a sharp edge: `REPACK t USING INDEX` on a btree, when the planner chooses an **index scan** rather than scan-and-sort, fetches heap pages through the normal buffer manager with **no** ring buffer [heapam_handler.c#index-scan-branch](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L681-L686). The plain sequential-scan path keeps the `BAS_BULKREAD` ring because `table_beginscan()` always sets `SO_ALLOW_STRAT` [heapam_handler.c#seq-scan-branch](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L694-L697) [tableam.h#table_beginscan](../../../raw/postgres-19/src/include/access/tableam.h#L943-L951).

Practical recipe for going easy on the disk, given no in-server throttle:

- Prefer physical-order (`REPACK t`) or scan-and-sort over an index scan, so the read ring buffer stays in play. To force scan-and-sort for an ordered rewrite, set `enable_indexscan = off` for the session; `enable_indexscan` is `PGC_USERSET`, so the change is session/transaction-scoped and needs no reload or restart [guc_parameters.dat#enable_indexscan](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L936-L941).
- Repack **one table or partition at a time** rather than the bare `REPACK;` form that sweeps the whole database, to bound each burst and let you space them out.
- If changing `maintenance_work_mem`, use a moderate value; it is `PGC_USERSET`, so the change is session/transaction-scoped and needs no reload or restart [guc_parameters.dat#maintenance_work_mem](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1951-L1959).
- Use `CONCURRENTLY` to protect query availability — but treat it as adding I/O, not reducing it.

## Progress reporting

Each backend running `REPACK` reports through `pg_stat_progress_repack`, exposing the command, phase, clustering-index OID, heap tuples scanned/inserted/updated/deleted, block counts, and index-rebuild count [system_views.sql#pg_stat_progress_repack](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L1349-L1378) [progress.h#PROGRESS_REPACK](../../../raw/postgres-19/src/include/commands/progress.h#L77-L106). The phases include the concurrent-only `catch-up` and `swapping relation files` steps [progress.h#phases](../../../raw/postgres-19/src/include/commands/progress.h#L97-L106). The old `pg_stat_progress_cluster` view is preserved as a compatibility wrapper over the same data [system_views.sql#pg_stat_progress_cluster](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L1380-L1398). While catching up, the leader waits on the `REPACK_WORKER_EXPORT` wait event [wait_event_names.txt#REPACK_WORKER_EXPORT](../../../raw/postgres-19/src/backend/utils/activity/wait_event_names.txt#L156).

## Tests

Because `REPACK (CONCURRENTLY)` needs `wal_level >= replica`, its functional tests live in `contrib/test_decoding` rather than the core regression suite (commit `4b2aa4b3`). The SQL test covers partition ownership checks, `attmissingval` preservation, the full matrix of rejection cases (partitioned, catalog, TOAST, temp, unlogged, `REPLICA IDENTITY NOTHING`, no PK/RI, deferrable PK), and a check that the `pgrepack` plugin cannot be driven directly through the SQL logical-decoding interface (added by commit `cd7b204b`) [test_decoding/sql/repack.sql](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql#L1-L84). Timing-sensitive concurrency (a transaction modifying the table during the rewrite) is driven by injection points in isolation specs `repack`, `repack_toast`, `repack_temporal`, and `repack_temporal_multirange`, hooked at `repack-concurrently-before-lock` [repack.c#injection-point](../../../raw/postgres-19/src/backend/commands/repack.c#L3230-L3234).

## Feature-Scope Commits

These are the 47 REPACK feature-scope commits in the pinned `REL_19_STABLE` checkout (`3aa54433`), grouped by topic. Scope includes commits whose subject/body explicitly references REPACK, prerequisite/support commits cited by that history, and tree-wide fixes that changed REPACK-specific code or documentation. Broad commits that only touch shared infrastructure files are excluded unless their REPACK-specific effect is listed here.

### Foundational

| Commit | Date | Author | What it does |
|---|---|---|---|
| `ac58465e` | 2026-03-10 | Á. Herrera | **Introduce the REPACK command.** Adds the `REPACK` statement that absorbs `VACUUM FULL` and `CLUSTER` as physical-order vs index-order modes; keeps but de-emphasizes the legacy commands [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L22). |
| `a630ac5c` | 2026-03-12 | Á. Herrera | Document the `command` column of `pg_stat_progress_repack`. |
| `c0b53ec0` | 2026-04-06 | Á. Herrera | **Rename `cluster.c` to `repack.c`** (and the header), since the code now centers on `REPACK`. |
| `28d534e2` | 2026-04-06 | Á. Herrera | **Add the `CONCURRENTLY` option.** Initial copy under `ShareUpdateExclusiveLock` + MVCC snapshot, a temporary slot, a decoding worker that stashes changes, and replay before the final swap. Based on Houska's `pg_squeeze` [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L7-L21). |

### CONCURRENTLY infrastructure

| Commit | Date | Author | What it does |
|---|---|---|---|
| `33bf7318` | 2026-04-04 | Á. Herrera | Make `index_concurrently_create_copy` more general, so REPACK can reuse index-copy machinery. |
| `caec9d9f` | 2026-04-05 | Á. Herrera | Let `index_create` suppress `index_build` progress reporting (used when rebuilding indexes on the new heap). |
| `e76d8c74` | 2026-04-07 | Á. Herrera | **Add `max_repack_replication_slots`** so REPACK reserves slots from its own pool instead of `max_replication_slots` [slot.c#ReplicationSlotCreate](../../../raw/postgres-19/src/backend/replication/slot.c#L372-L463). |
| `0d3dba38` | 2026-04-07 | Á. Herrera | Allow logical-replication snapshots to be database-specific (attempt to let multiple REPACKs coexist). |
| `01a80f06` | 2026-05-23 | Á. Herrera | **Revert `0d3dba38`** as fundamentally flawed; restricts REPACK (CONCURRENTLY) to one process at a time instance-wide. |
| `2af1dc89` | 2026-05-27 | Á. Herrera | Disable logical decoding after REPACK so `effective_wal_level` doesn't stay at `logical`; adds a flag to `ReplicationSlotDropAcquired()` [repack_worker.c#cleanup](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L293-L304). |
| `38470c2c` | 2026-05-29 | Á. Herrera | Advance `restart_lsn` more eagerly in `LogicalConfirmReceivedLocation` (supports the WAL-recycling change). |
| `45b02984` | 2026-05-30 | Á. Herrera | **Allow old WAL recycling during REPACK CONCURRENTLY** by moving the slot forward each WAL segment [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L374-L403). |

### Correctness fixes

| Commit | Date | Author | What it does |
|---|---|---|---|
| `5dbb63fc` | 2026-04-20 | Á. Herrera | REPACK no longer requires the `REPLICATION` or `LOGIN` role attributes; the worker connects with `BGWORKER_BYPASS_ROLELOGINCHECK` [repack_worker.c#connect](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L104-L106). |
| `832e220d` | 2026-04-27 | Á. Herrera | Don't allow **deferrable primary keys** as the identity index (a modified tuple may not yet be indexed) [repack.c#deferrable-pk](../../../raw/postgres-19/src/backend/commands/repack.c#L968-L995). |
| `6ca631b9` | 2026-04-28 | Á. Herrera | Fix processing of **toasted tuples** in the change spill/replay path. |
| `b5f92b8e` | 2026-05-04 | Á. Herrera | Fix an off-by-one in the repack index loop (from `28d534e2`). |
| `eb2e2eb4` | 2026-05-05 | Á. Herrera | **Don't lose `attmissingval` columns**: the tuple-reform fast path discarded fast-default values during any VACUUM FULL/CLUSTER/REPACK. |
| `a0a0c0c2` | 2026-05-05 | Á. Herrera | Skip **other sessions' temp tables** in the no-table forms of REPACK/CLUSTER/VACUUM FULL to avoid blocking on them [repack.c#skip-temp](../../../raw/postgres-19/src/backend/commands/repack.c#L2187-L2197). |
| `36f52a59` | 2026-05-11 | Á. Herrera | Fix REPACK with **`WITHOUT OVERLAPS`** (GiST) replica-identity indexes: derive the equality strategy from the opfamily instead of hard-coding btree [repack.c#eq-strategy](../../../raw/postgres-19/src/backend/commands/repack.c#L3103-L3130). |
| `a120ecf5` | 2026-05-18 | Fujii Masao | Fix REPACK **option parsing**: `CONCURRENTLY OFF` was rejected, and repeated options weren't last-wins. |
| `0160143a` | 2026-05-19 | Á. Herrera | Fix the decoding worker not being cleaned up on **`FATAL` exit** of the leader. |
| `1588d89a` | 2026-05-26 | Á. Herrera | Restructure worker teardown so the DSM segment is always freed even if init fails partway [repack.c#stop_repack_decoding_worker](../../../raw/postgres-19/src/backend/commands/repack.c#L3677-L3719). |
| `5bcc3fbd` | 2026-04-07 | Á. Herrera | Fix a valgrind failure. |
| `a3b069ef` | 2026-04-07 | Á. Herrera | Avoid a different-size pointer-to-integer cast. |
| `2cff3637` | 2026-04-08 | Á. Herrera | Simplify a `memcpy` target declaration. |
| `2fd84e22` | 2026-04-16 | Fujii Masao | Use `XLogRecPtrIsValid()` consistently for WAL-position checks. |
| `05c401d5` | 2026-04-16 | Á. Herrera | Add a missing initialization. |
| `5d48d3b1` | 2026-05-29 | Á. Herrera | Remove an unnecessary signal-handler change (bgworkers already use `die()`). |
| `cd7b204b` | 2026-06-09 | Á. Herrera | **Disallow direct use of the `pgrepack` plugin.** Reject driving the output plugin outside `REPACK (CONCURRENTLY)` (direct use caused assertion failures and production crashes from bogus memory lifetime); also moves `output_writer_private` allocation into `repack_startup()`, always sets `->relid`, and renames the worker's plugin-name macro `REPL_PLUGIN_NAME` to `PGREPACK_PLUGIN` [pgrepack.c#repack_startup](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L50-L82). |
| `e2a8cabc` | 2026-06-16 | Á. Herrera | **Check leftover separately stored TOAST attributes during concurrent REPACK replay.** `restore_tuple()` now errors if the spill file contains too few or too many separately stored varlena attributes after reconstructing a tuple [repack.c#restore_tuple](../../../raw/postgres-19/src/backend/commands/repack.c#L2752-L2818). |
| `fb284f2f9bd` | 2026-07-03 | Á. Herrera | **Fix REPACK CONCURRENTLY for stored generated columns** (backpatch-through 19). Concurrent replay must recompute a generated column when a tuple is updated/inserted during the rewrite, which needs the transient table's `pg_attrdef` rows; `rebuild_relation` now copies the attribute defaults onto the new heap in concurrent mode via the new `copy_attribute_defaults()` [repack.c#copy-attrdefs-call](../../../raw/postgres-19/src/backend/commands/repack.c#L1098-L1104) [repack.c#copy_attribute_defaults](../../../raw/postgres-19/src/backend/commands/repack.c#L3500-L3580). |
| `5e450df50dc` | 2026-07-03 | Á. Herrera | **Initialize the change-replay range table more honestly** (backpatch-through 19). `initialize_change_context` now builds a one-entry range table whose `updatedCols` bitmapset marks every non-dropped column updated, installed via `ExecInitRangeTable`, so `ExecInsertIndexTuples` errs toward flagging index updates; this only affects btree unchanged-column optimizations and does not compromise correctness [repack.c#initialize_change_context](../../../raw/postgres-19/src/backend/commands/repack.c#L3005-L3144). |
| `133eba078f7` | 2026-07-10 | Á. Herrera | **Do not lock candidate tables while building a whole-database REPACK list.** The old transaction-scoped discovery locks ended at the database commit immediately after list construction and did not protect later processing. The final per-table transaction now uses `try_relation_open()`, skips a dropped target or an OID that now identifies a non-table relkind, and rechecks it while holding the real operation lock; discovery-time permission checks also tolerate concurrent drops [repack.c#per-table-open](../../../raw/postgres-19/src/backend/commands/repack.c#L448-L481) [repack.c#get_tables_to_repack](../../../raw/postgres-19/src/backend/commands/repack.c#L2139-L2255) [repack.c#repack_is_permitted_for_relation](../../../raw/postgres-19/src/backend/commands/repack.c#L2323-L2362). |

### Error messages, docs, style

| Commit | Date | Author | What it does |
|---|---|---|---|
| `8fb95a8a` | 2026-04-07 | Á. Herrera | doc: add a `REPACK (CONCURRENTLY)` example [ref/repack.sgml#example](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L400-L405). |
| `d3bba041` | 2026-04-21 | M. Paquier | Tree-wide typo/grammar fixes touching REPACK text. |
| `d14f69a3` | 2026-04-22 | P. Geoghegan | Harmonize function parameter names across v19, including repack. |
| `3bf63730` | 2026-05-13 | Á. Herrera | Fix style in a few REPACK `ereport`s. |
| `43649b6a` | 2026-05-28 | Á. Herrera | Add an upfront, REPACK-specific error when `wal_level < replica` instead of the cryptic slot-requirement message [repack.c#wal_level-check](../../../raw/postgres-19/src/backend/commands/repack.c#L914-L920). |
| `497e92dc` | 2026-05-28 | Á. Herrera | Fix minor issues in repack `ereport()`s. |
| `378dffaf` | 2026-05-28 | Á. Herrera | Improve REPACK (CONCURRENTLY) error messages further. |
| `da8889ccd7e` | 2026-07-06 | R. Haas | Tree-wide (backpatch-through 19): use `PG_MODULE_MAGIC_EXT` (carrying the module name and version) instead of plain `PG_MODULE_MAGIC` in the newly added modules, including the `pgrepack` output plugin [pgrepack.c#module-magic](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L21-L24). |
| `8a84ddd8c63` | 2026-07-16 | Fujii Masao; patch by Shinya Kato | Add `REPACK` to the generic `MAINTAIN` privilege description and to the predefined `pg_maintain` role's command list; the runtime `ACL_MAINTAIN` membership path already covered the command [ddl.sgml#MAINTAIN](../../../raw/postgres-19/doc/src/sgml/ddl.sgml#L2536-L2548) [user-manag.sgml#pg_maintain](../../../raw/postgres-19/doc/src/sgml/user-manag.sgml#L655-L670) [aclchk.c#pg_class_aclmask_ext](../../../raw/postgres-19/src/backend/catalog/aclchk.c#L3419-L3428). |

### Tests

| Commit | Date | Author | What it does |
|---|---|---|---|
| `be142fa0` | 2026-04-07 | Á. Herrera | Fix tests under `wal_level=minimal`. |
| `4b2aa4b3` | 2026-04-23 | Á. Herrera | Move the REPACK (CONCURRENTLY) test out of the stock regression suite into `test_decoding` [test_decoding/sql/repack.sql](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql#L1-L6). |
| `e5035950` | 2026-05-15 | Fujii Masao | psql: fix tab completion for REPACK boolean options. |
| `47ad2233` | 2026-05-27 | M. Sawada | Fix `051_effective_wal_level.pl` on builds without injection points. |
| `2670cc29` | 2026-05-29 | Á. Herrera | Cover additional error cases and corner conditions in `repack.c`; move some cases to `test_decoding`. |

## Context Reviewed

- `ref/repack.sgml` — command reference, modes, CONCURRENTLY semantics, restrictions, resource notes [ref/repack.sgml](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L1-L452).
- `ddl.sgml`, `user-manag.sgml`, and `aclchk.c` — generic `MAINTAIN` documentation, predefined `pg_maintain`, and its runtime `ACL_MAINTAIN` mapping [ddl.sgml#MAINTAIN](../../../raw/postgres-19/doc/src/sgml/ddl.sgml#L2536-L2548) [user-manag.sgml#pg_maintain](../../../raw/postgres-19/doc/src/sgml/user-manag.sgml#L655-L670) [aclchk.c#pg_class_aclmask_ext](../../../raw/postgres-19/src/backend/catalog/aclchk.c#L3419-L3428).
- `repack.c` — `ExecRepack`, `cluster_rel`, `rebuild_relation`, `copy_table_data`, concurrent finish/apply, worker control [repack.c](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L32).
- `repack_worker.c` — worker main loop, decoding setup, WAL recycling, change filtering [repack_worker.c](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L1-L536).
- `pgrepack.c` — output-plugin callbacks and tuple spill format [pgrepack.c](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L1-L308).
- `repack.h` / `repack_internal.h` — API, `CLUOPT_*`, `DecodingWorkerShared`, change kinds [repack_internal.h](../../../raw/postgres-19/src/include/commands/repack_internal.h#L1-L122).
- Grammar/parse/dispatch — `parsenodes.h`, `gram.y`, `utility.c`, `vacuum.c` [gram.y#RepackStmt](../../../raw/postgres-19/src/backend/parser/gram.y#L12587-L12680).
- GUC/slot pool — `guc_parameters.dat`, `slot.c` [slot.c#ReplicationSlotCreate](../../../raw/postgres-19/src/backend/replication/slot.c#L372-L463).
- Observability — `system_views.sql`, `progress.h`, `wait_event_names.txt`.
- Tests — `contrib/test_decoding/sql/repack.sql`, `src/test/modules/injection_points` repack specs.
- `git log --regexp-ignore-case --grep=repack` plus source-path history for the REPACK-specific files, checked against the pinned `REL_19_STABLE` checkout `3aa54433b0cdce48facb610a5b720208cc760654` for the 47 feature-scope commits. The 2026-07-09 repin from `cdae794a` (the former `master` post-`REL_19_BETA1` pin, now the point where `REL_19_STABLE` diverged from `master`/20devel) added three REPACK feature-scope commits in `cdae794a..01c544e1`: `fb284f2f9bd` (stored generated columns) and `5e450df50dc` (change-replay range table), both backpatched through 19, plus the tree-wide `da8889ccd7e` (`PG_MODULE_MAGIC_EXT`). The 2026-07-13 repin reviewed all 12 commits in `01c544e1..8055e337`; only `133eba078f7` changed REPACK, raising the count from 45 to 46. The 31-commit `8055e337..3aa54433` range changed no REPACK implementation, worker, output-plugin, header, direct test, or command-reference file. Its one REPACK-scoped commit is `8a84ddd8c63`, which adds REPACK to generic `MAINTAIN` and `pg_maintain` documentation and raises the count from 46 to 47. Broad heap VM-WAL and logical-decoding-status fixes in the range are excluded because they change no REPACK-specific code or text. Three cited `heapam.c` ranges shifted by one line and were refreshed.

## Evidence Map

| Claim | Source |
|---|---|
| REPACK absorbs VACUUM FULL + CLUSTER | [repack.c#L1-L22](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L22), `ac58465e` |
| `MAINTAIN` authorizes REPACK; `pg_maintain` supplies it on all relations | [ddl.sgml#L2536-L2548](../../../raw/postgres-19/doc/src/sgml/ddl.sgml#L2536-L2548) [user-manag.sgml#L655-L670](../../../raw/postgres-19/doc/src/sgml/user-manag.sgml#L655-L670) [aclchk.c#L3419-L3428](../../../raw/postgres-19/src/backend/catalog/aclchk.c#L3419-L3428), `8a84ddd8c63` |
| Two lock levels: AEL vs SUEL | [repack.c#L498-L505](../../../raw/postgres-19/src/backend/commands/repack.c#L498-L505) |
| Multi-table candidates are collected unlocked, then opened and rechecked under the operation lock | [repack.c#L367-L481](../../../raw/postgres-19/src/backend/commands/repack.c#L367-L481) [repack.c#L714-L768](../../../raw/postgres-19/src/backend/commands/repack.c#L714-L768) [repack.c#L2139-L2362](../../../raw/postgres-19/src/backend/commands/repack.c#L2139-L2362), `133eba078f7` |
| CONCURRENTLY preconditions / identity index | [repack.c#L907-L998](../../../raw/postgres-19/src/backend/commands/repack.c#L907-L998) |
| Worker starts before XID; one initial snapshot | [repack.c#L1043-L1115](../../../raw/postgres-19/src/backend/commands/repack.c#L1043-L1115) |
| Decoding setup, temp slot, target-only filter | [repack_worker.c#L194-L291](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L194-L291) |
| Change spill format | [pgrepack.c#L193-L308](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L193-L308) |
| Replay with NO_LOGICAL + identity-index lookup | [repack.c#L2528-L2739](../../../raw/postgres-19/src/backend/commands/repack.c#L2528-L2739) |
| Spilled tuple reconstruction validates separately stored attributes | [repack.c#L2752-L2818](../../../raw/postgres-19/src/backend/commands/repack.c#L2752-L2818) |
| Lock upgrade + double catch-up + swap | [repack.c#L3230-L3370](../../../raw/postgres-19/src/backend/commands/repack.c#L3230-L3370) |
| WAL recycling during REPACK | [repack_worker.c#L374-L403](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L374-L403), `45b02984` |
| `max_repack_replication_slots` = PGC_POSTMASTER | [guc_parameters.dat#L2108-L2114](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2108-L2114) |
| Progress view + phases | [system_views.sql#L1349-L1398](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L1349-L1398), [progress.h#L77-L106](../../../raw/postgres-19/src/include/commands/progress.h#L77-L106) |
| Tests live in test_decoding (wal_level) | [repack.sql#L1-L6](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql#L1-L6), `4b2aa4b3` |
| No cost-delay throttle in the rewrite path | [repack.c#L531-L712](../../../raw/postgres-19/src/backend/commands/repack.c#L531-L712), [heapam_handler.c#L594-L602](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L594-L602) |
| Large seqscan reads via BULKREAD ring | [heapam.c#L397-L410](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L397-L410), [tableam.h#L943-L951](../../../raw/postgres-19/src/include/access/tableam.h#L943-L951) |

## Source References

- [repack.c](../../../raw/postgres-19/src/backend/commands/repack.c) — `ExecRepack`, `cluster_rel`, `rebuild_relation`, `copy_table_data`, concurrent finish/apply, worker control (formerly `cluster.c`).
- [repack_worker.c](../../../raw/postgres-19/src/backend/commands/repack_worker.c) — decoding background worker: setup, decode loop, WAL recycling, change filtering.
- [pgrepack.c](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c) — `pgrepack` logical-decoding output plugin and tuple spill format.
- [repack.h](../../../raw/postgres-19/src/include/commands/repack.h) / [repack_internal.h](../../../raw/postgres-19/src/include/commands/repack_internal.h) — public API, `CLUOPT_*`, `ConcurrentChangeKind`, `RepackDecodingState`, `DecodingWorkerShared`.
- [ref/repack.sgml](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml) — `REPACK` reference page.
- [ddl.sgml](../../../raw/postgres-19/doc/src/sgml/ddl.sgml) / [user-manag.sgml](../../../raw/postgres-19/doc/src/sgml/user-manag.sgml) / [aclchk.c](../../../raw/postgres-19/src/backend/catalog/aclchk.c) — generic `MAINTAIN` documentation, predefined `pg_maintain`, and runtime role-to-ACL mapping.
- [parsenodes.h](../../../raw/postgres-19/src/include/nodes/parsenodes.h) / [gram.y](../../../raw/postgres-19/src/backend/parser/gram.y) / [utility.c](../../../raw/postgres-19/src/backend/tcop/utility.c) / [vacuum.c](../../../raw/postgres-19/src/backend/commands/vacuum.c) — `RepackStmt`/`RepackCommand`, grammar, dispatch, and `VACUUM FULL` routing.
- [slot.c](../../../raw/postgres-19/src/backend/replication/slot.c) / [guc_parameters.dat](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat) — `max_repack_replication_slots`, `maintenance_work_mem`, `enable_indexscan`, and the slot pool.
- [heapam_handler.c](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c) / [heapam.c](../../../raw/postgres-19/src/backend/access/heap/heapam.c) / [tableam.h](../../../raw/postgres-19/src/include/access/tableam.h) — the table-AM heap copy, sort sizing, and the BULKREAD/BULKWRITE ring-buffer behavior relevant to I/O impact.
- [system_views.sql](../../../raw/postgres-19/src/backend/catalog/system_views.sql) / [progress.h](../../../raw/postgres-19/src/include/commands/progress.h) / [wait_event_names.txt](../../../raw/postgres-19/src/backend/utils/activity/wait_event_names.txt) — `pg_stat_progress_repack`, phases, wait event.
- [test_decoding/sql/repack.sql](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql) — functional and rejection-case tests.

## Open Questions

- **Exact WAL/lock deadlock window.** The code comments warn that a concurrent `CREATE INDEX` (which conflicts with `ShareUpdateExclusiveLock`) can deadlock the worker against the leader's transaction; the precise reproduction and whether `deadlock.c` always detects it via the lock group is asserted in comments but not proven by a test here [repack.c#deadlock-note](../../../raw/postgres-19/src/backend/commands/repack.c#L1057-L1069).
- **`REPLICA IDENTITY FULL` support.** The code explicitly rejects `FULL` as "not implemented yet"; whether a later commit on `master` past this pin adds it is out of scope for this checkout [repack.c#replident](../../../raw/postgres-19/src/backend/commands/repack.c#L952-L966).
- This page is drafted from a close source read but is **not yet agent-verified claim-by-claim**; `verified_by_agent` remains `not yet`.

## Related Pages

- [v19/index](../index.md) — PostgreSQL 19 landing page.
- [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](pg-plan-advice.md) — the other deep v19 feature walkthrough.
- [versions](../../versions.md) — source pin manifest.
