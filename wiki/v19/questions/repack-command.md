---
type: question
version: 19
pinned_commit: 4b0bf0788b066a4ca1d4f959566678e44ec93422
verified: false
verified_by_agent: not yet
---

# How the REPACK Command Works in PostgreSQL 19, and Its 40 Feature-Scope Commits (unverified)

## Question

Follow AGENTS.md.
In PostgreSQL 19, do a comprehensive explanation of how `pg_repack` works, explain all commits to the feature.

> Scope note (agreed with the asker): the third-party `pg_repack` extension is **not** in the pinned `raw/postgres-19/` checkout, so it cannot be cited under the wiki evidence rules. PostgreSQL 19 instead adds an **in-core `REPACK` command** (with a `CONCURRENTLY` option) that brings pg_repack/pg_squeeze-style online table rewriting into core. This page documents that in-core feature.

## Short Answer

`REPACK` is a new top-level SQL command in PostgreSQL 19 that rewrites a table into a fresh relfilenode to reclaim dead-tuple space, optionally clustering rows by an index. It **absorbs the functionality of `VACUUM FULL` and `CLUSTER`** into one command: physical-order rewrite (like `VACUUM FULL`) and index-order rewrite (like `CLUSTER`) are just two modes of `REPACK`, and the two legacy commands are kept but de-emphasized [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L22) [ac58465e commit message](#feature-scope-commits). The command is documented as "rewrite a table to reclaim disk space" [ref/repack.sgml#refpurpose](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L17-L20).

`REPACK` has two execution modes:

- **Blocking (default).** Takes `AccessExclusiveLock`, builds a new heap, copies live tuples in the chosen order, rebuilds indexes via `REINDEX`, then swaps relfilenodes. This is the classic `CLUSTER`/`VACUUM FULL` path, now reached through `repack.c` [repack.c#rebuild_relation](../../../raw/postgres-19/src/backend/commands/repack.c#L1114-L1139).
- **`CONCURRENTLY`.** Takes only `ShareUpdateExclusiveLock` for almost all of the work, so readers and writers keep using the table. It captures concurrent DML with **logical decoding** through a dedicated background worker and a temporary replication slot, replays those changes onto the new heap, and only upgrades to `AccessExclusiveLock` for the brief final relfilenode swap [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L7-L21) [ref/repack.sgml#CONCURRENTLY](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L221-L234).

This page tracks **40 feature-scope commits (2026-03-10 to 2026-05-30)**: commits whose subject/body explicitly references REPACK, adjacent prerequisite/support commits used by that history, and tree-wide fixes that changed REPACK-specific code or text. The bulk were authored by Antonin Houska and Álvaro Herrera, building on Houska's `pg_squeeze` extension. They are listed and explained under [Feature-Scope Commits](#feature-scope-commits).

## Syntax and Modes

The grammar produces a single `RepackStmt` node for `REPACK` and `CLUSTER`, distinguished by a `RepackCommand` tag [parsenodes.h#RepackStmt](../../../raw/postgres-19/src/include/nodes/parsenodes.h#L4097-L4112) [gram.y#RepackStmt](../../../raw/postgres-19/src/backend/parser/gram.y#L12587-L12680). `VACUUM (FULL)` still parses as `VacuumStmt`; when the `FULL` option is present, `vacuum.c` routes execution into the same rewrite engine with `REPACK_COMMAND_VACUUMFULL` [gram.y#VacuumStmt](../../../raw/postgres-19/src/backend/parser/gram.y#L12700-L12732) [vacuum.c#VACUUMFULL-repack](../../../raw/postgres-19/src/backend/commands/vacuum.c#L2302-L2305):

```
REPACK [ ( option [, ...] ) ] [ table [ ( column [, ...] ) ] [ USING INDEX [ index_name ] ] ]
REPACK [ ( option [, ...] ) ] USING INDEX
```

where `option` is `VERBOSE`, `ANALYZE`, or `CONCURRENTLY` [ref/repack.sgml#synopsis](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L22-L36). Key forms:

| Form | Effect |
|---|---|
| `REPACK t` | Rewrite `t` in physical order (like `VACUUM FULL`) [ref/repack.sgml#Description](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L42-L50). |
| `REPACK t USING INDEX idx` | Rewrite `t` ordered by `idx`, and mark `idx` as the clustered index (like `CLUSTER`) [ref/repack.sgml#clustering](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L79-L93). |
| `REPACK t USING INDEX` | Use the table's already-clustered index; error if none [repack.c#determine_clustered_index](../../../raw/postgres-19/src/backend/commands/repack.c#L2466-L2507). |
| `REPACK` (no table) | Process every table/matview the caller has `MAINTAIN` on; not allowed in a transaction block or with `CONCURRENTLY` [ref/repack.sgml#no-table](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L52-L60). |
| `REPACK USING INDEX` (no table) | Process all tables that have a clustering index configured [ref/repack.sgml#no-table-using-index](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L95-L99). |
| `REPACK (CONCURRENTLY) t` | Online rewrite; requires an explicit single non-partitioned table [repack.c#multi-table-guard](../../../raw/postgres-19/src/backend/commands/repack.c#L334-L350). |

`utility.c` dispatches `T_RepackStmt` to `ExecRepack()`, and reports the command tag as `REPACK` unless the statement is a legacy `CLUSTER` [utility.c#ExecRepack](../../../raw/postgres-19/src/backend/tcop/utility.c#L866-L867) [utility.c#CMDTAG](../../../raw/postgres-19/src/backend/tcop/utility.c#L2894-L2898). `VACUUM (FULL)` reaches the same engine by calling `cluster_rel(REPACK_COMMAND_VACUUMFULL, …)` from `vacuum.c` [vacuum.c#cluster_rel](../../../raw/postgres-19/src/backend/commands/vacuum.c#L2303-L2305). To run `REPACK`, the caller needs the `MAINTAIN` privilege on the table [repack.c#repack_is_permitted](../../../raw/postgres-19/src/backend/commands/repack.c#L2347-L2361) [ref/repack.sgml#Notes](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L355-L358).

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

The output plugin is built as `pgrepack` and linked into the backend (`backend_targets += pgrepack`), not shipped as a `contrib` extension; the worker loads it by the fixed name `"pgrepack"` [pgrepack/meson.build](../../../raw/postgres-19/src/backend/replication/pgrepack/meson.build#L13-L20) [repack_worker.c#REPL_PLUGIN_NAME](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L31). Despite the file/dir name `pgrepack`, the user-facing command is `REPACK`; the third-party extension is unrelated.

## Blocking REPACK (the simple rewrite)

`ExecRepack()` parses the option list (`VERBOSE`/`ANALYZE`/`CONCURRENTLY`), sets the `CLUOPT_*` bitmask in `ClusterParams`, picks the lock level, and either processes one relation or builds a work list for the multi-table forms [repack.c#ExecRepack](../../../raw/postgres-19/src/backend/commands/repack.c#L246-L478) [repack.h#CLUOPT](../../../raw/postgres-19/src/include/commands/repack.h#L24-L36). Multiple tables are each processed in their own transaction (like `VACUUM`) to avoid holding many strong locks at once, which is why the multi-table and `CONCURRENTLY` forms cannot run inside a transaction block [repack.c#per-table-txn](../../../raw/postgres-19/src/backend/commands/repack.c#L223-L245).

`cluster_rel()` is the per-relation driver. It starts progress reporting, switches to the table owner's userid under a security-restricted context, sets `search_path` to `pg_catalog, pg_temp`, rechecks the relation in the multi-table case, validates the clustering index, and then calls `rebuild_relation()` [repack.c#cluster_rel](../../../raw/postgres-19/src/backend/commands/repack.c#L520-L701) [ref/repack.sgml#search_path](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L360-L364). The lock level is `AccessExclusiveLock` for blocking mode and `ShareUpdateExclusiveLock` for concurrent mode [repack.c#RepackLockLevel](../../../raw/postgres-19/src/backend/commands/repack.c#L487-L494).

`rebuild_relation()` (blocking path):

1. If an index was given, mark it `indisclustered` [repack.c#mark_index_clustered](../../../raw/postgres-19/src/backend/commands/repack.c#L1066-L1068).
2. `make_new_heap()` creates a transient `pg_temp_<oid>` heap with the old heap's tuple descriptor, reloptions, access method, and (if needed) a TOAST table [repack.c#make_new_heap](../../../raw/postgres-19/src/backend/commands/repack.c#L1153-L1227).
3. `copy_table_data()` copies live tuples into the new heap. It chooses **sequential-scan-and-sort** vs **index scan** vs **plain seqscan** via `plan_cluster_use_sort()` for a btree clustering index, computes freeze/multixact cutoffs, and hands the actual copy to the table AM's `table_relation_copy_for_cluster()` [repack.c#copy_table_data](../../../raw/postgres-19/src/backend/commands/repack.c#L1413-L1448).
4. `finish_heap_swap()` swaps the relfilenodes of old and new heaps, rebuilds the indexes (`reindex = true`), applies the new `relfrozenxid`/`relminmxid`, and drops the transient heap [repack.c#finish_heap_swap-call](../../../raw/postgres-19/src/backend/commands/repack.c#L1130-L1138).

Predicate (SSI) locks on tuples/pages are promoted to a relation lock before the rewrite, because tuples move [repack.c#TransferPredicateLocks](../../../raw/postgres-19/src/backend/commands/repack.c#L659-L670).

## REPACK CONCURRENTLY (online rewrite via logical decoding)

The concurrent path keeps the table online by combining a low-strength lock with logical decoding. The cast of characters:

| Component | Role |
|---|---|
| **Leader backend** | Runs `REPACK`; holds `ShareUpdateExclusiveLock`, copies data, applies captured changes, does the final swap [repack.c#rebuild_relation](../../../raw/postgres-19/src/backend/commands/repack.c#L1030-L1113). |
| **Decoding worker** | `bgworker` running `RepackWorkerMain`; owns a temporary logical slot and decodes WAL into spill files [repack_worker.c#RepackWorkerMain](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L59-L164). |
| **`pgrepack` output plugin** | Decoding callbacks; writes each insert/update/delete tuple of the target relation to the current spill file [pgrepack.c#change_cb](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L92-L161). |
| **Temporary replication slot** | `repack_<pid>`, `RS_TEMPORARY` so it is dropped on error/exit; pins the WAL/snapshot start point [repack_worker.c#slot](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L215-L225). |
| **DSM + spill fileset** | `DecodingWorkerShared` in DSM coordinates the two processes; a `SharedFileSet` holds the exported snapshot and change files [repack_internal.h#DecodingWorkerShared](../../../raw/postgres-19/src/include/commands/repack_internal.h#L63-L119). |

### Step by step

1. **Precondition checks.** `check_concurrent_repack_requirements()` enforces `wal_level >= replica`, rejects catalogs, TOAST tables, non-permanent (temp/unlogged) tables, and `REPLICA IDENTITY NOTHING`/`FULL`, and obtains the **identity index** (replica-identity index or a non-deferrable primary key). Without a usable identity index, concurrent mode is refused [repack.c#check_concurrent_repack_requirements](../../../raw/postgres-19/src/backend/commands/repack.c#L894-L985).
2. **Start the worker before any XID is assigned.** The leader becomes a lock-group leader, then `start_repack_decoding_worker()` sets up the DSM, registers the `bgworker`, and blocks until the worker reports `initialized` [repack.c#start_repack_decoding_worker](../../../raw/postgres-19/src/backend/commands/repack.c#L1030-L1056) [repack.c#start_repack_decoding_worker-impl](../../../raw/postgres-19/src/backend/commands/repack.c#L3439-L3521). The worker creates the temporary slot, enables logical decoding, restricts decoding to the target relation (and its TOAST relation), builds an initial historic snapshot, and exports it to a file [repack_worker.c#setup](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L194-L309).
3. **Initial copy under the exported snapshot.** The leader reads the exported snapshot with `get_initial_snapshot()`, pushes it active, and runs `copy_table_data()` against that MVCC snapshot, so the initial copy is a consistent point-in-time image [repack.c#get_initial_snapshot](../../../raw/postgres-19/src/backend/commands/repack.c#L1058-L1095) [repack.c#get_initial_snapshot-impl](../../../raw/postgres-19/src/backend/commands/repack.c#L3583-L3632). Meanwhile the worker keeps decoding WAL in a loop, and later advances the slot's restart/confirmed LSN at WAL-segment boundaries so processed WAL can be recycled [repack_worker.c#decode_loop](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L152-L164) [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L392-L421).
4. **Build new indexes** on the new heap (instead of reindexing the old one), since this is slow and is done before the exclusive lock [repack.c#build_new_indexes](../../../raw/postgres-19/src/backend/commands/repack.c#L3327-L3358).
5. **Catch-up #1 (still under `ShareUpdateExclusiveLock`).** `process_concurrent_changes()` tells the worker an `lsn_upto`, waits for the next spill file, and `apply_concurrent_changes()` replays it onto the new heap. Doing this first minimizes how long the exclusive lock is later held [repack.c#catch-up-1](../../../raw/postgres-19/src/backend/commands/repack.c#L3180-L3193).
6. **Lock upgrade + catch-up #2.** The leader takes `AccessExclusiveLock` on the table, all its indexes, and its TOAST relation, transfers predicate locks, flushes WAL, and applies the remaining changes with `done = true` so the worker exits [repack.c#lock-upgrade](../../../raw/postgres-19/src/backend/commands/repack.c#L3196-L3250).
7. **Swap.** `swap_relation_files()` swaps each index's storage, then `finish_heap_swap()` swaps the heap (and TOAST), dropping the transient relation. The old OID, grants, and dependencies are preserved [repack.c#swap](../../../raw/postgres-19/src/backend/commands/repack.c#L3263-L3314).

### How concurrent changes are captured and replayed

The `pgrepack` plugin's `change_cb` only ever sees the target relation (other relations are filtered out during decoding by `change_useless_for_repack()`), and writes a compact record per change: a one-byte kind (`i`/`u`/`U`/`d`), the heap tuple bytes, and any out-of-line external attributes spilled separately to stay under `MaxAllocSize` [pgrepack.c#repack_store_change](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L172-L287) [repack_worker.c#change_useless_for_repack](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L517-L554) [repack_internal.h#ConcurrentChangeKind](../../../raw/postgres-19/src/include/commands/repack_internal.h#L24-L32).

`apply_concurrent_changes()` reads the spill file and replays each change onto the new heap, bumping the command counter between dependent changes [repack.c#apply_concurrent_changes](../../../raw/postgres-19/src/backend/commands/repack.c#L2527-L2647):

- **INSERT** → `table_tuple_insert(..., TABLE_INSERT_NO_LOGICAL)` plus index maintenance. The `NO_LOGICAL` flag stops these replayed rows from being decoded again [repack.c#apply_concurrent_insert](../../../raw/postgres-19/src/backend/commands/repack.c#L2653-L2668).
- **UPDATE/DELETE** → locate the existing tuple in the new heap via the identity index (`find_target_tuple()`), then `table_tuple_update`/`table_tuple_delete` with `NO_LOGICAL`. For updates, unchanged TOAST pointers are re-pointed at the new relation's TOAST table via `adjust_toast_pointers()` [repack.c#apply_concurrent_update](../../../raw/postgres-19/src/backend/commands/repack.c#L2674-L2738) [repack.c#toast-pointers](../../../raw/postgres-19/src/backend/commands/repack.c#L2620-L2630).

The identity-index scan key is precomputed in `initialize_change_context()`, which resolves the equality strategy from the index's operator family (so it works for non-btree identity indexes too — see commit `36f52a59`) [repack.c#initialize_change_context](../../../raw/postgres-19/src/backend/commands/repack.c#L3001-L3088).

### Error handling and worker teardown

The leader runs `rebuild_relation()` inside `PG_ENSURE_ERROR_CLEANUP(stop_repack_decoding_worker_cb)` so the worker and its slot are cleaned up even on `FATAL` exit (e.g. `pg_terminate_backend()`), and `stop_repack_decoding_worker()` terminates the worker, detaches the error queue, cancels CV sleeps, and frees the DSM [repack.c#ensure-cleanup](../../../raw/postgres-19/src/backend/commands/repack.c#L681-L689) [repack.c#stop_repack_decoding_worker](../../../raw/postgres-19/src/backend/commands/repack.c#L3529-L3578). Worker errors are forwarded to the leader through a shared-memory message queue and re-thrown with a "REPACK decoding worker" context line, using the `PROCSIG_REPACK_MESSAGE` interrupt and `ProcessRepackMessages()` [repack.c#ProcessRepackMessages](../../../raw/postgres-19/src/backend/commands/repack.c#L3654-L3792) [repack_worker.c#RepackWorkerShutdown](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L169-L179).

### WAL retention and `effective_wal_level`

Normally a replication slot pins `restart_lsn` so WAL can't be recycled. `REPACK` does not need crash-safe replay (a crash just restarts the whole operation), so the worker advances the slot's restart/confirmed LSN as it crosses each WAL-segment boundary, letting old WAL be recycled while `REPACK` runs (commit `45b02984`) [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L392-L421). Because the temporary slot is logical, the worker calls `EnsureLogicalDecodingEnabled()` at setup and `ReplicationSlotDropAcquired(true)` at teardown so `effective_wal_level` does not stay stuck at `logical` afterward (commit `2af1dc89`) [repack_worker.c#enable-decoding](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L211-L225) [repack_worker.c#cleanup](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L311-L322).

## Restrictions on CONCURRENTLY

`REPACK (CONCURRENTLY)` is rejected for any of: an `UNLOGGED` or temporary table; a partitioned table; a table without a primary key or index-based replica identity (including deferrable PKs); a system catalog or TOAST table; execution inside a transaction block; or when `max_repack_replication_slots` leaves no slot free [ref/repack.sgml#cannot-use](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L259-L302) [repack.c#check_concurrent_repack_requirements](../../../raw/postgres-19/src/backend/commands/repack.c#L894-L985). It is documented as **not MVCC-safe** [ref/repack.sgml#warning](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L304-L310). After the `01a80f06` revert (see commits), only **one** `REPACK (CONCURRENTLY)` may run at a time across the whole instance.

## GUC: `max_repack_replication_slots`

Concurrent `REPACK` consumes a logical replication slot for its lifetime. Rather than forcing users to inflate `max_replication_slots`, v19 adds a dedicated pool sized by `max_repack_replication_slots` (default `5`, min `0`) [guc_parameters.dat#max_repack_replication_slots](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2109-L2115) [slot.c#decl](../../../raw/postgres-19/src/backend/replication/slot.c#L163-L164). `ReplicationSlotCreate(..., repack = true, ...)` allocates from the extra `[max_replication_slots, max_replication_slots + max_repack_replication_slots)` band of the slot array, so REPACK slots never compete with ordinary replication slots [slot.c#ReplicationSlotCreate](../../../raw/postgres-19/src/backend/replication/slot.c#L372-L463).

**Scope:** `max_repack_replication_slots` is `PGC_POSTMASTER` — it **requires a server restart** [guc_parameters.dat#context](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2109-L2109). The decoding worker also needs a free `max_worker_processes` slot, or `REPACK (CONCURRENTLY)` errors out [repack.c#worker-slots](../../../raw/postgres-19/src/backend/commands/repack.c#L3494-L3498).

## I/O Impact and Throttling

**There is no built-in way to throttle `REPACK`.** Unlike lazy (non-`FULL`) `VACUUM`, the rewrite path has **no cost-based delay**: `cluster_rel()` and the heap copy it drives contain no `vacuum_delay_point()` calls [repack.c#cluster_rel](../../../raw/postgres-19/src/backend/commands/repack.c#L520-L701) [heapam_handler.c#heapam_relation_copy_for_cluster](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L594-L602). Consequently `vacuum_cost_delay` / `vacuum_cost_limit` (and the autovacuum cost knobs) **have no effect** on `REPACK`, `CLUSTER`, or `VACUUM FULL` — there is no GUC or option that tells the command to run slower.

What does exist:

| Lever | Effect on I/O | Notes |
|---|---|---|
| **BULKREAD ring buffer** (automatic) | Limits shared-buffer pollution while reading the old heap | Only on the **sequential-scan** path, and only once the relation exceeds `NBuffers/4` [heapam.c#initscan](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L396-L409). |
| **BULKWRITE ring buffer** (automatic, CONCURRENTLY) | Limits cache pollution while inserting into the new heap | `GetBulkInsertState()` uses `BAS_BULKWRITE` [heapam.c#GetBulkInsertState](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L1934-L1942). |
| `maintenance_work_mem` | Bounds sort / index-build memory; does **not** throttle | Sizes `tuplesort_begin_cluster` [heapam_handler.c#tuplesort](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L651-L655). It is `PGC_USERSET`, so settable per session (no restart/reload) [guc_parameters.dat#maintenance_work_mem](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1952-L1961). High values trade RAM for fewer temp-file write bursts. |
| `CONCURRENTLY` | Reduces **lock** contention, not I/O | Avoids the long `AccessExclusiveLock` [repack.c#overview](../../../raw/postgres-19/src/backend/commands/repack.c#L7-L21) but does *more* total I/O (decode + spill + replay). It does let old WAL recycle as it runs [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L392-L421). |

The read-side ring buffer is the only automatic "niceness," and it has a sharp edge: `REPACK t USING INDEX` on a btree, when the planner chooses an **index scan** rather than scan-and-sort, fetches heap pages through the normal buffer manager with **no** ring buffer [heapam_handler.c#index-scan-branch](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L681-L686). The plain sequential-scan path keeps the `BAS_BULKREAD` ring because `table_beginscan()` always sets `SO_ALLOW_STRAT` [heapam_handler.c#seq-scan-branch](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L694-L697) [tableam.h#table_beginscan](../../../raw/postgres-19/src/include/access/tableam.h#L943-L951).

Practical recipe for going easy on the disk, given no in-server throttle:

- Prefer physical-order (`REPACK t`) or scan-and-sort over an index scan, so the read ring buffer stays in play. To force scan-and-sort for an ordered rewrite, set `enable_indexscan = off` for the session; `enable_indexscan` is `PGC_USERSET`, so the change is session/transaction-scoped and needs no reload or restart [guc_parameters.dat#enable_indexscan](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L937-L942).
- Repack **one table or partition at a time** rather than the bare `REPACK;` form that sweeps the whole database, to bound each burst and let you space them out.
- If changing `maintenance_work_mem`, use a moderate value; it is `PGC_USERSET`, so the change is session/transaction-scoped and needs no reload or restart [guc_parameters.dat#maintenance_work_mem](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1952-L1961).
- Use `CONCURRENTLY` to protect query availability — but treat it as adding I/O, not reducing it.

## Progress reporting

Each backend running `REPACK` reports through `pg_stat_progress_repack`, exposing the command, phase, clustering-index OID, heap tuples scanned/inserted/updated/deleted, block counts, and index-rebuild count [system_views.sql#pg_stat_progress_repack](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L1349-L1378) [progress.h#PROGRESS_REPACK](../../../raw/postgres-19/src/include/commands/progress.h#L77-L106). The phases include the concurrent-only `catch-up` and `swapping relation files` steps [progress.h#phases](../../../raw/postgres-19/src/include/commands/progress.h#L97-L106). The old `pg_stat_progress_cluster` view is preserved as a compatibility wrapper over the same data [system_views.sql#pg_stat_progress_cluster](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L1380-L1398). While catching up, the leader waits on the `REPACK_WORKER_EXPORT` wait event [wait_event_names.txt#REPACK_WORKER_EXPORT](../../../raw/postgres-19/src/backend/utils/activity/wait_event_names.txt#L159).

## Tests

Because `REPACK (CONCURRENTLY)` needs `wal_level >= replica`, its functional tests live in `contrib/test_decoding` rather than the core regression suite (commit `4b2aa4b3`). The SQL test covers partition ownership checks, `attmissingval` preservation, and the full matrix of rejection cases (partitioned, catalog, TOAST, temp, unlogged, `REPLICA IDENTITY NOTHING`, no PK/RI, deferrable PK) [test_decoding/sql/repack.sql](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql#L1-L77). Timing-sensitive concurrency (a transaction modifying the table during the rewrite) is driven by injection points in isolation specs `repack`, `repack_toast`, `repack_temporal`, and `repack_temporal_multirange`, hooked at `repack-concurrently-before-lock` [repack.c#injection-point](../../../raw/postgres-19/src/backend/commands/repack.c#L3174-L3178).

## Feature-Scope Commits

These are the 40 REPACK feature-scope commits in the pinned `master` checkout (`4b0bf078`), grouped by topic. Scope includes commits whose subject/body explicitly references REPACK, prerequisite/support commits cited by that history, and tree-wide fixes that changed REPACK-specific code or documentation. Broad commits that only touch shared infrastructure files are excluded unless their REPACK-specific effect is listed here. The 2026-06-02 repin from `21298c2c` added only release-note/docs, translation, and version-stamp commits outside this REPACK scope; no cited REPACK file changed.

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
| `2af1dc89` | 2026-05-27 | Á. Herrera | Disable logical decoding after REPACK so `effective_wal_level` doesn't stay at `logical`; adds a flag to `ReplicationSlotDropAcquired()` [repack_worker.c#cleanup](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L311-L322). |
| `38470c2c` | 2026-05-29 | Á. Herrera | Advance `restart_lsn` more eagerly in `LogicalConfirmReceivedLocation` (supports the WAL-recycling change). |
| `45b02984` | 2026-05-30 | Á. Herrera | **Allow old WAL recycling during REPACK CONCURRENTLY** by moving the slot forward each WAL segment [repack_worker.c#WAL-recycling](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L392-L421). |

### Correctness fixes

| Commit | Date | Author | What it does |
|---|---|---|---|
| `5dbb63fc` | 2026-04-20 | Á. Herrera | REPACK no longer requires the `REPLICATION` or `LOGIN` role attributes; the worker connects with `BGWORKER_BYPASS_ROLELOGINCHECK` [repack_worker.c#connect](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L104-L106). |
| `832e220d` | 2026-04-27 | Á. Herrera | Don't allow **deferrable primary keys** as the identity index (a modified tuple may not yet be indexed) [repack.c#deferrable-pk](../../../raw/postgres-19/src/backend/commands/repack.c#L955-L982). |
| `6ca631b9` | 2026-04-28 | Á. Herrera | Fix processing of **toasted tuples** in the change spill/replay path. |
| `b5f92b8e` | 2026-05-04 | Á. Herrera | Fix an off-by-one in the repack index loop (from `28d534e2`). |
| `eb2e2eb4` | 2026-05-05 | Á. Herrera | **Don't lose `attmissingval` columns**: the tuple-reform fast path discarded fast-default values during any VACUUM FULL/CLUSTER/REPACK. |
| `a0a0c0c2` | 2026-05-05 | Á. Herrera | Skip **other sessions' temp tables** in the no-table forms of REPACK/CLUSTER/VACUUM FULL to avoid blocking on them [repack.c#skip-temp](../../../raw/postgres-19/src/backend/commands/repack.c#L2180-L2187). |
| `36f52a59` | 2026-05-11 | Á. Herrera | Fix REPACK with **`WITHOUT OVERLAPS`** (GiST) replica-identity indexes: derive the equality strategy from the opfamily instead of hard-coding btree [repack.c#eq-strategy](../../../raw/postgres-19/src/backend/commands/repack.c#L3047-L3074). |
| `a120ecf5` | 2026-05-18 | Fujii Masao | Fix REPACK **option parsing**: `CONCURRENTLY OFF` was rejected, and repeated options weren't last-wins. |
| `0160143a` | 2026-05-19 | Á. Herrera | Fix the decoding worker not being cleaned up on **`FATAL` exit** of the leader. |
| `1588d89a` | 2026-05-26 | Á. Herrera | Restructure worker teardown so the DSM segment is always freed even if init fails partway [repack.c#stop_repack_decoding_worker](../../../raw/postgres-19/src/backend/commands/repack.c#L3529-L3571). |
| `5bcc3fbd` | 2026-04-07 | Á. Herrera | Fix a valgrind failure. |
| `a3b069ef` | 2026-04-07 | Á. Herrera | Avoid a different-size pointer-to-integer cast. |
| `2cff3637` | 2026-04-08 | Á. Herrera | Simplify a `memcpy` target declaration. |
| `2fd84e22` | 2026-04-16 | Fujii Masao | Use `XLogRecPtrIsValid()` consistently for WAL-position checks. |
| `05c401d5` | 2026-04-16 | Á. Herrera | Add a missing initialization. |
| `5d48d3b1` | 2026-05-29 | Á. Herrera | Remove an unnecessary signal-handler change (bgworkers already use `die()`). |

### Error messages, docs, style

| Commit | Date | Author | What it does |
|---|---|---|---|
| `8fb95a8a` | 2026-04-07 | Á. Herrera | doc: add a `REPACK (CONCURRENTLY)` example [ref/repack.sgml#example](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L400-L405). |
| `d3bba041` | 2026-04-21 | M. Paquier | Tree-wide typo/grammar fixes touching REPACK text. |
| `d14f69a3` | 2026-04-22 | P. Geoghegan | Harmonize function parameter names across v19, including repack. |
| `3bf63730` | 2026-05-13 | Á. Herrera | Fix style in a few REPACK `ereport`s. |
| `43649b6a` | 2026-05-28 | Á. Herrera | Add an upfront, REPACK-specific error when `wal_level < replica` instead of the cryptic slot-requirement message [repack.c#wal_level-check](../../../raw/postgres-19/src/backend/commands/repack.c#L901-L907). |
| `497e92dc` | 2026-05-28 | Á. Herrera | Fix minor issues in repack `ereport()`s. |
| `378dffaf` | 2026-05-28 | Á. Herrera | Improve REPACK (CONCURRENTLY) error messages further. |

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
- `repack.c` — `ExecRepack`, `cluster_rel`, `rebuild_relation`, `copy_table_data`, concurrent finish/apply, worker control [repack.c](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L32).
- `repack_worker.c` — worker main loop, decoding setup, WAL recycling, change filtering [repack_worker.c](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L1-L555).
- `pgrepack.c` — output-plugin callbacks and tuple spill format [pgrepack.c](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L1-L288).
- `repack.h` / `repack_internal.h` — API, `CLUOPT_*`, `DecodingWorkerShared`, change kinds [repack_internal.h](../../../raw/postgres-19/src/include/commands/repack_internal.h#L1-L125).
- Grammar/parse/dispatch — `parsenodes.h`, `gram.y`, `utility.c`, `vacuum.c` [gram.y#RepackStmt](../../../raw/postgres-19/src/backend/parser/gram.y#L12587-L12680).
- GUC/slot pool — `guc_parameters.dat`, `slot.c` [slot.c#ReplicationSlotCreate](../../../raw/postgres-19/src/backend/replication/slot.c#L372-L463).
- Observability — `system_views.sql`, `progress.h`, `wait_event_names.txt`.
- Tests — `contrib/test_decoding/sql/repack.sql`, `src/test/modules/injection_points` repack specs.
- `git log --regexp-ignore-case --grep=repack` plus source-path history for the REPACK-specific files, checked against the pinned checkout for the 40 feature-scope commits. Re-checked 2026-06-02 after repinning from `21298c2c` to `4b0bf078`: no file cited by this page changed.

## Evidence Map

| Claim | Source |
|---|---|
| REPACK absorbs VACUUM FULL + CLUSTER | [repack.c#L1-L22](../../../raw/postgres-19/src/backend/commands/repack.c#L1-L22), `ac58465e` |
| Two lock levels: AEL vs SUEL | [repack.c#L487-L494](../../../raw/postgres-19/src/backend/commands/repack.c#L487-L494) |
| CONCURRENTLY preconditions / identity index | [repack.c#L894-L985](../../../raw/postgres-19/src/backend/commands/repack.c#L894-L985) |
| Worker starts before XID; one initial snapshot | [repack.c#L1030-L1095](../../../raw/postgres-19/src/backend/commands/repack.c#L1030-L1095) |
| Decoding setup, temp slot, target-only filter | [repack_worker.c#L194-L309](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L194-L309) |
| Change spill format | [pgrepack.c#L172-L287](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L172-L287) |
| Replay with NO_LOGICAL + identity-index lookup | [repack.c#L2527-L2738](../../../raw/postgres-19/src/backend/commands/repack.c#L2527-L2738) |
| Lock upgrade + double catch-up + swap | [repack.c#L3174-L3314](../../../raw/postgres-19/src/backend/commands/repack.c#L3174-L3314) |
| WAL recycling during REPACK | [repack_worker.c#L392-L421](../../../raw/postgres-19/src/backend/commands/repack_worker.c#L392-L421), `45b02984` |
| `max_repack_replication_slots` = PGC_POSTMASTER | [guc_parameters.dat#L2109-L2115](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2109-L2115) |
| Progress view + phases | [system_views.sql#L1349-L1398](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L1349-L1398), [progress.h#L77-L106](../../../raw/postgres-19/src/include/commands/progress.h#L77-L106) |
| Tests live in test_decoding (wal_level) | [repack.sql#L1-L6](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql#L1-L6), `4b2aa4b3` |
| No cost-delay throttle in the rewrite path | [repack.c#L520-L701](../../../raw/postgres-19/src/backend/commands/repack.c#L520-L701), [heapam_handler.c#L594-L602](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L594-L602) |
| Large seqscan reads via BULKREAD ring | [heapam.c#L396-L409](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L396-L409), [tableam.h#L943-L951](../../../raw/postgres-19/src/include/access/tableam.h#L943-L951) |

## Source References

- [repack.c](../../../raw/postgres-19/src/backend/commands/repack.c) — `ExecRepack`, `cluster_rel`, `rebuild_relation`, `copy_table_data`, concurrent finish/apply, worker control (formerly `cluster.c`).
- [repack_worker.c](../../../raw/postgres-19/src/backend/commands/repack_worker.c) — decoding background worker: setup, decode loop, WAL recycling, change filtering.
- [pgrepack.c](../../../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c) — `pgrepack` logical-decoding output plugin and tuple spill format.
- [repack.h](../../../raw/postgres-19/src/include/commands/repack.h) / [repack_internal.h](../../../raw/postgres-19/src/include/commands/repack_internal.h) — public API, `CLUOPT_*`, `ConcurrentChangeKind`, `RepackDecodingState`, `DecodingWorkerShared`.
- [ref/repack.sgml](../../../raw/postgres-19/doc/src/sgml/ref/repack.sgml) — `REPACK` reference page.
- [parsenodes.h](../../../raw/postgres-19/src/include/nodes/parsenodes.h) / [gram.y](../../../raw/postgres-19/src/backend/parser/gram.y) / [utility.c](../../../raw/postgres-19/src/backend/tcop/utility.c) / [vacuum.c](../../../raw/postgres-19/src/backend/commands/vacuum.c) — `RepackStmt`/`RepackCommand`, grammar, dispatch, and `VACUUM FULL` routing.
- [slot.c](../../../raw/postgres-19/src/backend/replication/slot.c) / [guc_parameters.dat](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat) — `max_repack_replication_slots`, `maintenance_work_mem`, `enable_indexscan`, and the slot pool.
- [heapam_handler.c](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c) / [heapam.c](../../../raw/postgres-19/src/backend/access/heap/heapam.c) / [tableam.h](../../../raw/postgres-19/src/include/access/tableam.h) — the table-AM heap copy, sort sizing, and the BULKREAD/BULKWRITE ring-buffer behavior relevant to I/O impact.
- [system_views.sql](../../../raw/postgres-19/src/backend/catalog/system_views.sql) / [progress.h](../../../raw/postgres-19/src/include/commands/progress.h) / [wait_event_names.txt](../../../raw/postgres-19/src/backend/utils/activity/wait_event_names.txt) — `pg_stat_progress_repack`, phases, wait event.
- [test_decoding/sql/repack.sql](../../../raw/postgres-19/contrib/test_decoding/sql/repack.sql) — functional and rejection-case tests.

## Open Questions

- **Exact WAL/lock deadlock window.** The code comments warn that a concurrent `CREATE INDEX` (which conflicts with `ShareUpdateExclusiveLock`) can deadlock the worker against the leader's transaction; the precise reproduction and whether `deadlock.c` always detects it via the lock group is asserted in comments but not proven by a test here [repack.c#deadlock-note](../../../raw/postgres-19/src/backend/commands/repack.c#L1044-L1056).
- **`REPLICA IDENTITY FULL` support.** The code explicitly rejects `FULL` as "not implemented yet"; whether a later commit on `master` past this pin adds it is out of scope for this checkout [repack.c#replident](../../../raw/postgres-19/src/backend/commands/repack.c#L939-L953).
- This page is drafted from a close source read but is **not yet agent-verified claim-by-claim**; `verified_by_agent` remains `not yet`.

## Related Pages

- [v19/index](../index.md) — PostgreSQL 19 landing page.
- [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](pg-plan-advice.md) — the other deep v19 feature walkthrough.
- [versions](../../versions.md) — source pin manifest.
