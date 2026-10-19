---
type: question
version: 18
pinned_commit: baa7b142aace6821ce085906f314a75bcc4d95c8
verified: false
verified_by_agent: not yet
---

# Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Answer Up Front](#answer-up-front)
  - [Two Code Paths](#two-code-paths)
  - [In-Process Hook Variables](#in-process-hook-variables)
  - [Access-Method Callbacks](#access-method-callbacks)
  - [Adjacent Extension Surfaces](#adjacent-extension-surfaces)
  - [Registration Notes](#registration-notes)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Source References](#source-references)
- [Open Questions](#open-questions)
- [Navigation](#navigation)

## Question

> Follow AGENTS.md.
> In PostgreSQL 18, provide comprehensive documentation of all hooks available
> to an extension in the vacuum and autovacuum paths.

## Answer

### Answer Up Front

PostgreSQL 18 has no dedicated `vacuum_hook` or `autovacuum_hook`. Vacuum-aware
[extensions](../../../glossary.md#extension) reach into vacuum and [autovacuum](../../../glossary.md#autovacuum) through four kinds of extension
points: (1) general in-process [hook](../../../glossary.md#hook) variables (function pointers in core C
code) that happen to fire on vacuum's path, (2) shared-memory init hooks that
let an extension stand up its own state alongside vacuum, (3) per-relation
access-method callbacks invoked by `vacuum()` and `do_analyze_rel()`, and (4)
adjacent extension surfaces such as background-worker registration and custom
[cumulative statistics](../../../glossary.md#cumulative-statistics).

Two paths matter and they differ. A user-issued [`VACUUM`](../../../glossary.md#vacuum)/`ANALYZE` goes through
the parser, [`ProcessUtility`](../../../glossary.md#utility-command), then `ExecVacuum -> vacuum()`; autovacuum workers
call `vacuum()` directly from `autovacuum_do_vac_analyze`, bypassing parser
and `ProcessUtility`
([utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499),
[vacuum.c#ExecVacuum](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L162),
[autovacuum.c#autovacuum_do_vac_analyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3173)).
Hooks that sit on the parser/utility path therefore see only manual VACUUM and
ANALYZE; hooks that sit deeper in `vacuum()`, in the table or index AM, in
`elog()`, or in pgstat reporting fire for autovacuum as well.

The summary table below maps each hook to the paths that fire it. Detailed
semantics and call sites follow.

| Extension point | Manual VACUUM/ANALYZE | Autovacuum worker | Where it fires |
|---|---|---|---|
| `ProcessUtility_hook` | yes | no | utility dispatch |
| `post_parse_analyze_hook` | yes | no | after parse-analyze |
| `emit_log_hook` | yes | yes | every server log line |
| `shmem_request_hook` / `shmem_startup_hook` | [postmaster](../../../glossary.md#postmaster) startup only | postmaster startup only | shared-memory sizing and init |
| `TableAmRoutine.relation_vacuum` | yes (non-FULL) | yes (non-FULL) | per-relation vacuum |
| `TableAmRoutine.scan_analyze_next_block` / `scan_analyze_next_tuple` | yes (ANALYZE) | yes (ANALYZE) | per-block / per-tuple ANALYZE |
| `IndexAmRoutine.ambulkdelete` / [`amvacuumcleanup`](../../../glossary.md#index-vacuuming) | yes | yes | per-index VACUUM |
| `FdwRoutine.AnalyzeForeignTable` | yes ([foreign table](../../../glossary.md#foreign-data-wrapper) ANALYZE) | yes (foreign table ANALYZE) | start of ANALYZE on a foreign table |
| `pg_type.typanalyze` (per-column) | yes | yes | column-stats setup in ANALYZE |
| Custom [background worker](../../../glossary.md#background-worker) | n/a | n/a | extension-defined process |
| [Custom cumulative statistics](../../../glossary.md#cumulative-statistics-kind) | yes | yes | extension-defined counters |
| `object_access_hook` (indirect) | `VACUUM FULL` only | only when dropping an orphaned temporary table | [catalog](../../../glossary.md#catalog) code reached from `cluster_rel()` and `performDeletion()` |

Apart from those indirect `object_access_hook` calls, which
[Hooks that do NOT fire on vacuum's path](#hooks-that-do-not-fire-on-vacuums-path)
explains, PostgreSQL 18 does not expose any other in-process hook on the
vacuum or autovacuum code paths. There is no autovacuum scheduling hook, no
relation-selection hook, no per-vacuum-pass callback in core C
([vacuum.c#vacuum](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L477-L726),
[autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1878-L2599),
[autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L2927-L3163),
[vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L603-L1161)).

### Two Code Paths

#### Manual VACUUM/ANALYZE

A user-issued `VACUUM` or `ANALYZE` statement reaches the [backend](../../../glossary.md#backend) as a raw
[parse tree](../../../glossary.md#parse-tree), runs through parse-analyze, and is dispatched as a utility
statement. `ProcessUtility` calls `ProcessUtility_hook` if set, otherwise
`standard_ProcessUtility`, which routes `T_VacuumStmt` to `ExecVacuum`
([utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499),
[utility.c#T_VacuumStmt](../../../../raw/postgres-18/src/backend/tcop/utility.c#L861-L863)).
`ExecVacuum` parses options, builds a `VacuumParams` value, and calls
`vacuum()`
([vacuum.c#ExecVacuum](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L162)).

#### Autovacuum worker

The autovacuum launcher and worker are PostgreSQL processes, not SQL
statements. The worker connects to the target database, scans [`pg_class`](../../../glossary.md#pg_class) for
candidates, and then calls `autovacuum_do_vac_analyze`, which sets up
`VacuumParams` and calls `vacuum()` directly with `VacuumRelation` entries
identified by [OID](../../../glossary.md#oid)
([autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1885),
[autovacuum.c#autovacuum_do_vac_analyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3173)).
No `RawStmt` is created and no `ProcessUtility` is invoked, so hooks that fire
during parser dispatch never fire for autovacuum.

The autovacuum launcher initializes itself through `InitPostgres(NULL,
InvalidOid, NULL, InvalidOid, 0, NULL)`, with the flags argument set to `0`;
the worker uses `INIT_PG_OVERRIDE_ALLOW_CONNS`. Neither call passes
`INIT_PG_LOAD_SESSION_LIBS`, so autovacuum processes do not load
`session_preload_libraries`; only [`shared_preload_libraries`](../../../glossary.md#shared_preload_libraries), which is
processed in the postmaster before fork, contributes hook installations to
autovacuum
([autovacuum.c#AutoVacLauncherMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L368),
[autovacuum.c:419](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L419),
[autovacuum.c#AutoVacWorkerMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1376),
[autovacuum.c:1576-1578](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1576-L1578),
[postinit.c#INIT_PG_LOAD_SESSION_LIBS](../../../../raw/postgres-18/src/backend/utils/init/postinit.c#L1227-L1228),
[postmaster.c:930-933](../../../../raw/postgres-18/src/backend/postmaster/postmaster.c#L930-L933),
[miscinit.c#process_shared_preload_libraries](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1903)).

### In-Process Hook Variables

#### `ProcessUtility_hook`

Type:

```c
typedef void (*ProcessUtility_hook_type) (PlannedStmt *pstmt,
                                          const char *queryString,
                                          bool readOnlyTree,
                                          ProcessUtilityContext context,
                                          ParamListInfo params,
                                          QueryEnvironment *queryEnv,
                                          DestReceiver *dest,
                                          QueryCompletion *qc);
```

([utility.h#ProcessUtility_hook_type](../../../../raw/postgres-18/src/include/tcop/utility.h#L70-L78)).

`ProcessUtility` calls the hook in place of `standard_ProcessUtility` when set,
and the comment block explicitly tells plugins to chain forward by calling
`standard_ProcessUtility` to keep core dispatch working
([utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499)).
For VACUUM/ANALYZE, `standard_ProcessUtility` dispatches `T_VacuumStmt` to
`ExecVacuum`
([utility.c#T_VacuumStmt](../../../../raw/postgres-18/src/backend/tcop/utility.c#L861-L863)).

Use this hook to observe or wrap user-issued VACUUM and ANALYZE statements.
This hook does not fire for autovacuum: autovacuum's
`autovacuum_do_vac_analyze` calls `vacuum()` directly, never going through
`ProcessUtility`
([autovacuum.c#autovacuum_do_vac_analyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3173)).

#### `post_parse_analyze_hook`

Type:

```c
typedef void (*post_parse_analyze_hook_type) (ParseState *pstate,
                                              Query *query,
                                              JumbleState *jstate);
```

([analyze.h#post_parse_analyze_hook_type](../../../../raw/postgres-18/src/include/parser/analyze.h#L21-L25)).

`parse_analyze_fixedparams`, `parse_analyze_varparams`, and
`parse_analyze_withcb` all call the hook after producing the `Query` and after
optional jumbling
([analyze.c#parse_analyze_fixedparams](../../../../raw/postgres-18/src/backend/parser/analyze.c#L105),
[analyze.c#parse_analyze_varparams](../../../../raw/postgres-18/src/backend/parser/analyze.c#L145),
[analyze.c#parse_analyze_withcb](../../../../raw/postgres-18/src/backend/parser/analyze.c#L186)).
VACUUM and ANALYZE are utility statements wrapped in a `CMD_UTILITY` Query,
so this hook sees the statement when a user issues it
([analyze.c#transformStmt](../../../../raw/postgres-18/src/backend/parser/analyze.c#L412-L421)).
Autovacuum has no `RawStmt` to analyze and never reaches these functions.

#### `emit_log_hook`

Type:

```c
typedef void (*emit_log_hook_type) (ErrorData *edata);
```

([elog.h#emit_log_hook_type](../../../../raw/postgres-18/src/include/utils/elog.h#L476-L478)).

`EmitErrorReport` calls the hook before the message is sent to the server log,
once `errstart` has decided the message is interesting. The header comment
warns that the hook may turn off `edata->output_to_server` but is not
supposed to mutate other fields; `EmitErrorReport` rechecks `output_to_server`
afterwards
([elog.c#EmitErrorReport](../../../../raw/postgres-18/src/backend/utils/error/elog.c#L1692)).
Vacuum and autovacuum emit log lines through `ereport()` like every other
subsystem, so this hook is the one general-purpose extension point that fires
on both paths. For example, `vacuum_error_callback` and per-phase progress
logging all flow through `EmitErrorReport`
([vacuumlazy.c#vacuum_error_callback](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3782)).

#### `shmem_request_hook` and `shmem_startup_hook`

Types:

```c
typedef void (*shmem_request_hook_type) (void);
typedef void (*shmem_startup_hook_type) (void);
```

([miscadmin.h#shmem_request_hook_type](../../../../raw/postgres-18/src/include/miscadmin.h#L533-L534),
[ipc.h:22](../../../../raw/postgres-18/src/include/storage/ipc.h#L22),
[ipc.h:78](../../../../raw/postgres-18/src/include/storage/ipc.h#L78)).

Extensions that need their own shared memory to track vacuum-related state
install both hooks from `_PG_init` while
`process_shared_preload_libraries_in_progress` is true.
`process_shmem_requests` sets `process_shmem_requests_in_progress = true` only
around its call to `shmem_request_hook`. It has two callers: `PostmasterMain`
in a normal server and `PostgresSingleUserMain` in single-user mode. An
`EXEC_BACKEND` child re-runs `process_shared_preload_libraries` but never
`process_shmem_requests`
([miscinit.c#process_shmem_requests](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1927-L1937),
[postmaster.c:959-962](../../../../raw/postgres-18/src/backend/postmaster/postmaster.c#L959-L962),
[postgres.c:4126-4129](../../../../raw/postgres-18/src/backend/tcop/postgres.c#L4126-L4129),
[launch_backend.c:698-704](../../../../raw/postgres-18/src/backend/postmaster/launch_backend.c#L698-L704)).
`RequestAddinShmemSpace` and `RequestNamedLWLockTranche` raise [`FATAL`](../../../glossary.md#ereport) unless
that flag is set, so an extension may only call them from inside
`shmem_request_hook`
([ipci.c#RequestAddinShmemSpace](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L64-L79),
[lwlock.c#RequestNamedLWLockTranche](../../../../raw/postgres-18/src/backend/storage/lmgr/lwlock.c#L669-L687)).
`CreateSharedMemoryAndSemaphores`, in the postmaster or a single-user backend,
and `AttachSharedMemoryStructs`, in an `EXEC_BACKEND` child, each call
`CreateOrAttachShmemStructs` to create or attach the core shared structures,
and then invoke `shmem_startup_hook`
([ipci.c#AttachSharedMemoryStructs](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L163-L193),
[ipci.c#CreateSharedMemoryAndSemaphores](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L195-L250)).

These hooks are not vacuum-specific. They are the documented way for a library
loaded through `shared_preload_libraries` to reserve shared memory at server
start, which is typically how an extension keeps cross-vacuum counters or
queues; the documentation's worked example follows this pattern
([xfunc.sgml#xfunc-shared-addin-at-startup](../../../../raw/postgres-18/doc/src/sgml/xfunc.sgml#L3625-L3688)).
They are not the only way. PostgreSQL 18 also documents `GetNamedDSMSegment()`,
which creates or attaches a named [dynamic shared memory](../../../glossary.md#dynamic-shared-memory) segment after startup
without either hook, and pins the segment it creates
([xfunc.sgml#xfunc-shared-addin-after-startup](../../../../raw/postgres-18/doc/src/sgml/xfunc.sgml#L3690-L3727),
[dsm_registry.c#header](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L6-L16),
[dsm_registry.c#GetNamedDSMSegment](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L125-L133),
[dsm_registry.c:180-181](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L180-L181)).

#### Hooks that do NOT fire on vacuum's path

For completeness: `ExecutorStart_hook`, `ExecutorRun_hook`,
`ExecutorFinish_hook`, `ExecutorEnd_hook`, `ExecutorCheckPerms_hook`, the
[planner](../../../glossary.md#planner) and path hooks, and the `explain_*` hooks do not fire from the
vacuum/autovacuum/analyze paths.
A grep across `raw/postgres-18/src/backend/commands/vacuum.c`,
`raw/postgres-18/src/backend/commands/vacuumparallel.c`,
`raw/postgres-18/src/backend/commands/analyze.c`,
`raw/postgres-18/src/backend/access/heap/vacuumlazy.c`, and
`raw/postgres-18/src/backend/postmaster/autovacuum.c` finds no
`Invoke*Hook` macro calls and no `ExecutorStart`/`CreateQueryDesc`
invocations, and the analyze path does not run [SPI](../../../glossary.md#spi)
([analyze.c](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1-L3090),
[vacuum.c](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L1-L2703)).

`object_access_hook` is the exception. Those five files never invoke it, but
two vacuum-adjacent paths reach it through catalog code:

- `VACUUM FULL` hands the [relation](../../../glossary.md#relation) to `cluster_rel()`. The rebuild ends in
  `finish_heap_swap()`, whose `swap_relation_files()` runs
  `InvokeObjectPostAlterHookArg` for both relations, and the transient [heap](../../../glossary.md#heap) is
  then dropped through `performDeletion()`
  ([vacuum.c#vacuum_rel](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2303-L2311),
  [cluster.c:687](../../../../raw/postgres-18/src/backend/commands/cluster.c#L687),
  [cluster.c:1471](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1471),
  [cluster.c#swap_relation_files](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1300-L1307),
  [cluster.c:1563](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1563)).
- An autovacuum worker that finds an orphaned temporary table drops it with
  `performDeletion()`, and `deleteOneObject()` invokes the drop hook for each
  object it removes
  ([autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L2230-L2249),
  [dependency.c:229](../../../../raw/postgres-18/src/backend/catalog/dependency.c#L229),
  [dependency.c#deleteOneObject](../../../../raw/postgres-18/src/backend/catalog/dependency.c#L1253-L1255)).

The `get_relation_stats_hook`, `get_index_stats_hook`, and
`get_attavgwidth_hook` are read hooks consulted by the planner when it reads
`pg_statistic` [rows](../../../glossary.md#tuple) that ANALYZE wrote; they do not fire during ANALYZE itself
([selfuncs.c#examine_variable](../../../../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L5487-L5489),
[selfuncs.c#examine_simple_variable](../../../../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L5709-L5717),
[lsyscache.c#get_attavgwidth](../../../../raw/postgres-18/src/backend/utils/cache/lsyscache.c#L3459-L3470)).

### Access-Method Callbacks

#### Table AM

Vacuum and analyze dispatch through the table access-method routine in three
places. The relevant callbacks live in `TableAmRoutine`
([tableam.h#relation_vacuum](../../../../raw/postgres-18/src/include/access/tableam.h#L637-L654),
[tableam.h#scan_analyze_next_block](../../../../raw/postgres-18/src/include/access/tableam.h#L656-L674),
[tableam.h#scan_analyze_next_tuple](../../../../raw/postgres-18/src/include/access/tableam.h#L676-L688)):

- `relation_vacuum(Relation rel, struct VacuumParams *params,
  BufferAccessStrategy bstrategy)`. Called by `table_relation_vacuum` from
  `vacuum_rel` on non-FULL VACUUM. The header comment explicitly notes that
  VACUUM may be triggered by a user or by autovacuum, that ShareUpdateExclusive
  is held on entry, and that [VACUUM FULL](../../../glossary.md#vacuum-full) and ANALYZE do not go through this
  callback
  ([tableam.h#relation_vacuum](../../../../raw/postgres-18/src/include/access/tableam.h#L637-L654),
  [tableam.h#table_relation_vacuum](../../../../raw/postgres-18/src/include/access/tableam.h#L1662-L1678),
  [vacuum.c#vacuum_rel](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2300-L2317)).
- `scan_analyze_next_block(TableScanDesc scan, ReadStream *stream)`. Called
  per block by `acquire_sample_rows` through `table_scan_analyze_next_block`,
  during `do_analyze_rel`
  ([analyze.c#acquire_sample_rows](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1253-L1258),
  [tableam.h#table_scan_analyze_next_block](../../../../raw/postgres-18/src/include/access/tableam.h#L1680-L1692)).
- `scan_analyze_next_tuple(TableScanDesc scan, TransactionId OldestXmin,
  double *liverows, double *deadrows, TupleTableSlot *slot)`. Called per tuple
  by `acquire_sample_rows` through `table_scan_analyze_next_tuple`
  ([analyze.c#acquire_sample_rows](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1253-L1258),
  [tableam.h#table_scan_analyze_next_tuple](../../../../raw/postgres-18/src/include/access/tableam.h#L1694-L1712)).

A custom table AM that wants to react to vacuum or change vacuum semantics for
its relations implements these three callbacks.

#### Index AM

Vacuum calls into the index AM through two callbacks in `IndexAmRoutine`
([amapi.h#ambulkdelete_function](../../../../raw/postgres-18/src/include/access/amapi.h#L132-L140),
[amapi.h#IndexAmRoutine](../../../../raw/postgres-18/src/include/access/amapi.h#L297-L298)):

- `ambulkdelete(IndexVacuumInfo *info, IndexBulkDeleteResult *istat,
  IndexBulkDeleteCallback callback, void *callback_state)`. Bulk-removes index
  tuples whose heap [TIDs](../../../glossary.md#tid) the callback declares dead. `vac_bulkdel_one_index`
  wraps `index_bulk_delete`, which dispatches to `ambulkdelete` and emits the
  per-index `scanned index ... to remove` log line
  ([vacuum.c#vac_bulkdel_one_index](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2651),
  [indexam.c#index_bulk_delete](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L795)).
- `amvacuumcleanup(IndexVacuumInfo *info, IndexBulkDeleteResult *istat)`.
  Performs post-VACUUM cleanup and returns optionally updated index
  statistics. `vac_cleanup_one_index` wraps `index_vacuum_cleanup`, which
  dispatches to `amvacuumcleanup`
  ([vacuum.c#vac_cleanup_one_index](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2672),
  [indexam.c#index_vacuum_cleanup](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L816)).

[Parallel VACUUM](../../../glossary.md#parallel-vacuum) is opt-in per index AM via the bitmask
`amparallelvacuumoptions` on `IndexAmRoutine`. The flags
`VACUUM_OPTION_PARALLEL_BULKDEL`, `VACUUM_OPTION_PARALLEL_COND_CLEANUP`, and
`VACUUM_OPTION_PARALLEL_CLEANUP` describe which phases an AM may run in
[parallel workers](../../../glossary.md#parallel-query)
([amapi.h#amparallelvacuumoptions](../../../../raw/postgres-18/src/include/access/amapi.h#L281-L282),
[vacuum.h#VACUUM_OPTION](../../../../raw/postgres-18/src/include/commands/vacuum.h#L29-L66),
[vacuumparallel.c#parallel_vacuum_index_is_parallel_safe](../../../../raw/postgres-18/src/backend/commands/vacuumparallel.c#L946-L980)).
Parallel vacuum runs the bulk-delete and cleanup callbacks in parallel
workers and copies each index's `IndexBulkDeleteResult` back through a DSM
segment at the end
([vacuumparallel.c#parallel_vacuum_main](../../../../raw/postgres-18/src/backend/commands/vacuumparallel.c#L982-L987),
[vacuumparallel.c#parallel_vacuum_process_one_index](../../../../raw/postgres-18/src/backend/commands/vacuumparallel.c#L892-L926),
[vacuumparallel.c#parallel_vacuum_end](../../../../raw/postgres-18/src/backend/commands/vacuumparallel.c#L426-L452)).

A custom index AM implements the two vacuum callbacks (and declares its
parallel-vacuum capability) to participate in VACUUM.

#### Foreign Data Wrapper

Foreign tables get one ANALYZE-specific FDW callback in `FdwRoutine`:

```c
typedef bool (*AnalyzeForeignTable_function) (Relation relation,
                                              AcquireSampleRowsFunc *func,
                                              BlockNumber *totalpages);
```

([fdwapi.h#AnalyzeForeignTable_function](../../../../raw/postgres-18/src/include/foreign/fdwapi.h#L156-L158),
[fdwapi.h#FdwRoutine](../../../../raw/postgres-18/src/include/foreign/fdwapi.h#L256-L257)).

`analyze_rel` recognizes `RELKIND_FOREIGN_TABLE`, calls
`GetFdwRoutineForRelation`, and invokes `AnalyzeForeignTable`; if the FDW does
not implement analysis it returns false and `analyze_rel` emits a
"skipping ... cannot analyze this foreign table" warning and returns
([analyze.c#analyze_rel](../../../../raw/postgres-18/src/backend/commands/analyze.c#L109),
[fdwhandler.sgml#fdw-callbacks-analyze](../../../../raw/postgres-18/doc/src/sgml/fdwhandler.sgml#L1367)).
There is no FDW callback for VACUUM proper; the table AM `relation_vacuum`
callback is not called for foreign tables.

#### Per-Type analyze function (`pg_type.typanalyze`)

A type can supply a `typanalyze` function in `pg_type`; ANALYZE calls it from
`examine_attribute` via `OidFunctionCall1` with a populated `VacAttrStats *`,
falling back to `std_typanalyze` when none is set. The function must fill in
`compute_stats`, `minrows`, and optionally `extra_data` to take part in
per-column statistics
([analyze.c#examine_attribute](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1036),
[analyze.c#std_typanalyze](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1891),
[vacuum.h#VacAttrStats](../../../../raw/postgres-18/src/include/commands/vacuum.h#L71-L82),
[vacuum.h#typanalyze-fields](../../../../raw/postgres-18/src/include/commands/vacuum.h#L132-L138)).
An extension that defines a new SQL data type can plug into ANALYZE by
shipping a `typanalyze` function in its CREATE TYPE definition.

### Adjacent Extension Surfaces

#### Read-stream strategy clearing (new extension-visible API in 18.6)

The 18.4/18.6 range adds one exported function to the read-stream API that
extensions and out-of-core table AMs can call:
`read_stream_clear_strategy(ReadStream *stream)`. It sets
`op.strategy = NULL` on every one of the stream's `ReadBuffersOperation`s,
including in-flight ones, so subsequent reads use the whole [shared buffer pool](../../../glossary.md#shared_buffers)
instead of a ring; the caller still owns freeing the strategy's memory, and the
function's comment notes that an IO whose strategy was cleared mid-flight may
have a small part of its read time attributed to `IOCONTEXT_NORMAL`
([read_stream.c#read_stream_clear_strategy](../../../../raw/postgres-18/src/backend/storage/aio/read_stream.c#L1046-L1066),
[read_stream.h:102](../../../../raw/postgres-18/src/include/storage/read_stream.h#L102)).

The only in-tree caller is VACUUM's heap scan. When [`VacuumFailsafeActive`](../../../glossary.md#vacuum-failsafe) is
set — the [wraparound](../../../glossary.md#wraparound) failsafe — `lazy_scan_heap()` calls it once and records
that it did, so the rest of the vacuum may use all of shared buffers
([vacuumlazy.c#failsafe-clear-strategy](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1311-L1323)).
Commits `585181e0774` and `7c25cdb1ebf` restored that behavior, which a v18
refactor had accidentally broken, and made the clearing happen only once
([release-18.sgml#failsafe-buffer-pool](../../../../raw/postgres-18/doc/src/sgml/release-18.sgml#L2036-L2062)).
This is a buffer-access-strategy change only: the failsafe's other effects,
including skipping further index vacuuming and cleanup, are unchanged, and no
new hook or callback is introduced.

Separately, `4cc49cb7039` fixed the autovacuum launcher's database ordering:
`db_comparator()` was sorting databases from lowest to highest score, when the
launcher should process the highest-scoring database first. Extensions that
infer launcher behavior from observed per-database ordering should note the
change
([autovacuum.c#db_comparator](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1070-L1076),
[release-18.sgml#autovacuum-db-order](../../../../raw/postgres-18/doc/src/sgml/release-18.sgml#L2016-L2034)).

#### Background workers

`shared_preload_libraries`'s `_PG_init` may call `RegisterBackgroundWorker`
to register a static background worker. Only the postmaster registers one,
and only before background-worker shared memory is initialized; a later call
there raises `ERROR`. In any other process, such as an `EXEC_BACKEND` child or
a single-user backend that reloads the library, the call registers nothing: it
returns silently while `process_shared_preload_libraries_in_progress` is true
and logs `must be registered in "shared_preload_libraries"` otherwise
([bgworker.c#RegisterBackgroundWorker](../../../../raw/postgres-18/src/backend/postmaster/bgworker.c#L932-L977),
[bgworker.h#RegisterBackgroundWorker](../../../../raw/postgres-18/src/include/postmaster/bgworker.h#L114-L115)).
Extensions that want to add behavior alongside autovacuum (for example, a
maintenance scheduler or a custom retention policy) typically run as their
own background worker rather than trying to hook into the launcher.

#### Custom cumulative statistics

An extension can register a `PgStat_KindInfo` to define new shared cumulative
counters that report on vacuum-related events. The flow, validation, and
shared-memory layout are documented separately and are not duplicated here;
see [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](../observability/custom-cumulative-statistics.md)
([pgstat.c#pgstat_register_kind](../../../../raw/postgres-18/src/backend/utils/activity/pgstat.c#L1465),
[pgstat_internal.h#PgStat_KindInfo](../../../../raw/postgres-18/src/include/utils/pgstat_internal.h#L202)).

The built-in vacuum/autovacuum reporters that an extension can read from
shared cumulative stats are `pgstat_report_vacuum`, `pgstat_report_analyze`,
and `pgstat_report_autovac`
([vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L938-L943),
[analyze.c#do_analyze_rel](../../../../raw/postgres-18/src/backend/commands/analyze.c#L696-L700),
[autovacuum.c:1566](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1566)).
The first two write per-table cumulative entries, whose timestamps
[`pg_stat_all_tables`](../../../glossary.md#pg_stat_all_tables) exposes as `last_vacuum`, `last_autovacuum`,
`last_analyze`, and `last_autoanalyze`
([pgstat_relation.c#pgstat_report_vacuum](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L250-L261),
[pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L350-L361),
[system_views.sql#pg_stat_all_tables](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L700-L703)).
`pgstat_report_autovac` writes the per-database entry's `last_autovac_time`,
which the autovacuum launcher reads when it picks a database
([pgstat_database.c#pgstat_report_autovac](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_database.c#L49-L75),
[autovacuum.c:1244](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1244)).

#### Progress reporting

Vacuum and analyze publish progress through the cumulative-stats progress
API; the field numbers and phase constants are exposed in
`commands/progress.h` and surfaced as `pg_stat_progress_vacuum` and
`pg_stat_progress_analyze`. Extensions that want to display vacuum progress
read these views instead of installing a hook
([progress.h#vacuum](../../../../raw/postgres-18/src/include/commands/progress.h#L20-L39),
[progress.h#analyze](../../../../raw/postgres-18/src/include/commands/progress.h#L41-L57),
[vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L653-L654),
[analyze.c#do_analyze_rel](../../../../raw/postgres-18/src/backend/commands/analyze.c#L241-L242),
[vacuum.c#vacuum_delay_point](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2539-L2544),
[system_views.sql#pg_stat_progress_analyze](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L1204-L1224),
[system_views.sql#pg_stat_progress_vacuum](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L1226-L1245)).

### Registration Notes

- A hook is installed by assigning to its global variable from `_PG_init` of a
  loadable library. `_PG_init` runs during `process_shared_preload_libraries`,
  `process_session_preload_libraries`, or on the first call into the library;
  an extension's static background worker is registered only when the
  postmaster loads the library while
  `process_shared_preload_libraries_in_progress` is true, and shared-memory
  requests are accepted only inside `shmem_request_hook`, which only the
  postmaster and a single-user backend run, after preloading. Vacuum-tracking
  extensions therefore belong in `shared_preload_libraries`
  ([miscinit.c#process_shared_preload_libraries](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1903),
  [bgworker.c#RegisterBackgroundWorker](../../../../raw/postgres-18/src/backend/postmaster/bgworker.c#L932-L977),
  [ipci.c#RequestAddinShmemSpace](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L64-L79),
  [miscinit.c#process_shmem_requests](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1927-L1937)).
- `shared_preload_libraries` has [GUC context](../../../glossary.md#guc-context) `PGC_POSTMASTER`, so changing it
  requires a server restart
  ([guc_tables.c#shared_preload_libraries](../../../../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4486-L4495),
  [config.sgml#guc-shared-preload-libraries](../../../../raw/postgres-18/doc/src/sgml/config.sgml#L11004)).
- Plugins that install in-process hooks must save and call the previous hook
  value so multiple extensions can chain; the `ProcessUtility_hook` block
  comment in `utility.c` documents the chaining pattern for that hook
  ([utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499)).

## Context Reviewed

- VACUUM/ANALYZE command path:
  [utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499),
  [vacuum.c#ExecVacuum](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L162),
  [vacuum.c](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L1-L2703),
  [analyze.c](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1-L3090),
  [vacuumlazy.c](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1-L3871),
  [vacuumparallel.c](../../../../raw/postgres-18/src/backend/commands/vacuumparallel.c#L1-L1142).
- Autovacuum process path:
  [autovacuum.c#AutoVacLauncherMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L368),
  [autovacuum.c#AutoVacWorkerMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1376),
  [autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1885),
  [autovacuum.c#autovacuum_do_vac_analyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3173).
- In-process hook variables:
  [utility.c](../../../../raw/postgres-18/src/backend/tcop/utility.c#L1-L3786),
  [analyze.c](../../../../raw/postgres-18/src/backend/parser/analyze.c#L1-L3761),
  [elog.c](../../../../raw/postgres-18/src/backend/utils/error/elog.c#L1-L3826),
  [miscinit.c](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1-L1952),
  [ipci.c](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L1-L390).
- AM and FDW callbacks:
  [tableam.h](../../../../raw/postgres-18/src/include/access/tableam.h#L1-L2060),
  [amapi.h](../../../../raw/postgres-18/src/include/access/amapi.h#L1-L332),
  [fdwapi.h](../../../../raw/postgres-18/src/include/foreign/fdwapi.h#L1-L294),
  [vacuum.h](../../../../raw/postgres-18/src/include/commands/vacuum.h#L1-L410),
  [indexam.c](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L1-L1086).
- Adjacent extension surfaces:
  [bgworker.h](../../../../raw/postgres-18/src/include/postmaster/bgworker.h#L1-L164),
  [bgworker.c#RegisterBackgroundWorker](../../../../raw/postgres-18/src/backend/postmaster/bgworker.c#L940),
  [progress.h](../../../../raw/postgres-18/src/include/commands/progress.h#L1-L160),
  [read_stream.h](../../../../raw/postgres-18/src/include/storage/read_stream.h#L1-L106),
  [read_stream.c](../../../../raw/postgres-18/src/backend/storage/aio/read_stream.c#L1-L1124),
  [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](../observability/custom-cumulative-statistics.md).
- Target evidence checkout: `raw/postgres-18/` at pinned commit
  `baa7b142aace6821ce085906f314a75bcc4d95c8` (`REL_18_6-10-gbaa7b142aac`),
  re-checked for the 18.4/18.6 range against the previous pin
  `6cb307251c5c6261286c1566496920976640108e`. The hook and AM-callback inventory
  above is unchanged by that range; the only extension-visible addition is
  `read_stream_clear_strategy()`
  ([read_stream.h:102](../../../../raw/postgres-18/src/include/storage/read_stream.h#L102)).
- Documentation cross-checks:
  [xfunc.sgml](../../../../raw/postgres-18/doc/src/sgml/xfunc.sgml#L1-L4206),
  [fdwhandler.sgml#fdw-callbacks-analyze](../../../../raw/postgres-18/doc/src/sgml/fdwhandler.sgml#L1367),
  [config.sgml#guc-shared-preload-libraries](../../../../raw/postgres-18/doc/src/sgml/config.sgml#L11004).
- Read on 2026-10-19 while replacing file-level citations with line ranges;
  this pass corrected the shared-memory startup, background-worker
  registration and `object_access_hook` statements:
  [cluster.c](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1-L1754),
  [dependency.c](../../../../raw/postgres-18/src/backend/catalog/dependency.c#L1-L2922),
  [dsm_registry.c](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L1-L203),
  [postgres.c](../../../../raw/postgres-18/src/backend/tcop/postgres.c#L1-L5234),
  [launch_backend.c](../../../../raw/postgres-18/src/backend/postmaster/launch_backend.c#L1-L1072),
  [postmaster.c](../../../../raw/postgres-18/src/backend/postmaster/postmaster.c#L1-L4607),
  [lwlock.c](../../../../raw/postgres-18/src/backend/storage/lmgr/lwlock.c#L1-L2043),
  [postinit.c](../../../../raw/postgres-18/src/backend/utils/init/postinit.c#L1-L1450),
  [pgstat_relation.c](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L1-L1006),
  [pgstat_database.c](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_database.c#L1-L496),
  [selfuncs.c](../../../../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L1-L8887),
  [lsyscache.c](../../../../raw/postgres-18/src/backend/utils/cache/lsyscache.c#L1-L4018),
  [system_views.sql](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L1-L1409).

## Evidence Map

| Claim | Source |
|---|---|
| `ProcessUtility_hook` is called in place of `standard_ProcessUtility` | [utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499) |
| `standard_ProcessUtility` routes `T_VacuumStmt` to `ExecVacuum` | [utility.c#T_VacuumStmt](../../../../raw/postgres-18/src/backend/tcop/utility.c#L861-L863) |
| Autovacuum worker calls `vacuum()` directly, bypassing `ProcessUtility` | [autovacuum.c#autovacuum_do_vac_analyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3173) |
| Autovacuum launcher/worker `InitPostgres` does not load session preload libraries | [autovacuum.c#AutoVacLauncherMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L368), [autovacuum.c#AutoVacWorkerMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1376), [autovacuum.c:419](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L419), [autovacuum.c:1576-1578](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1576-L1578), [postinit.c#INIT_PG_LOAD_SESSION_LIBS](../../../../raw/postgres-18/src/backend/utils/init/postinit.c#L1227-L1228) |
| `post_parse_analyze_hook` fires from the three `parse_analyze_*` entry points | [analyze.c#parse_analyze_fixedparams](../../../../raw/postgres-18/src/backend/parser/analyze.c#L105), [analyze.c#parse_analyze_varparams](../../../../raw/postgres-18/src/backend/parser/analyze.c#L145), [analyze.c#parse_analyze_withcb](../../../../raw/postgres-18/src/backend/parser/analyze.c#L186) |
| `emit_log_hook` fires from `EmitErrorReport` for every server log line | [elog.c#EmitErrorReport](../../../../raw/postgres-18/src/backend/utils/error/elog.c#L1692) |
| `shmem_request_hook` runs inside `process_shmem_requests` | [miscinit.c#process_shmem_requests](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1931) |
| `process_shmem_requests` runs only in `PostmasterMain` and `PostgresSingleUserMain`, and only inside it do shared-memory requests succeed | [miscinit.c#process_shmem_requests](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1927-L1937), [postmaster.c:959-962](../../../../raw/postgres-18/src/backend/postmaster/postmaster.c#L959-L962), [postgres.c:4126-4129](../../../../raw/postgres-18/src/backend/tcop/postgres.c#L4126-L4129), [ipci.c#RequestAddinShmemSpace](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L64-L79), [lwlock.c#RequestNamedLWLockTranche](../../../../raw/postgres-18/src/backend/storage/lmgr/lwlock.c#L669-L687) |
| `shmem_startup_hook` runs from `CreateSharedMemoryAndSemaphores` and `AttachSharedMemoryStructs`, each after `CreateOrAttachShmemStructs` | [ipci.c#AttachSharedMemoryStructs](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L163-L193), [ipci.c#CreateSharedMemoryAndSemaphores](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L195-L250) |
| `GetNamedDSMSegment()` provides shared memory after startup without the startup hooks | [xfunc.sgml#xfunc-shared-addin-after-startup](../../../../raw/postgres-18/doc/src/sgml/xfunc.sgml#L3690-L3727), [dsm_registry.c#header](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L6-L16) |
| `object_access_hook` fires indirectly on `VACUUM FULL` and on autovacuum's orphaned temporary-table drop | [cluster.c#swap_relation_files](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1300-L1307), [cluster.c:1563](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1563), [autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L2230-L2249), [dependency.c#deleteOneObject](../../../../raw/postgres-18/src/backend/catalog/dependency.c#L1253-L1255) |
| `relation_vacuum`, `scan_analyze_next_block`, `scan_analyze_next_tuple` are table AM callbacks invoked from VACUUM/ANALYZE | [tableam.h#relation_vacuum](../../../../raw/postgres-18/src/include/access/tableam.h#L637-L654), [tableam.h#scan_analyze_next_block](../../../../raw/postgres-18/src/include/access/tableam.h#L656-L674), [tableam.h#scan_analyze_next_tuple](../../../../raw/postgres-18/src/include/access/tableam.h#L676-L688), [vacuum.c#vacuum_rel](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2300-L2317), [analyze.c#acquire_sample_rows](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1253-L1258) |
| `ambulkdelete` and `amvacuumcleanup` are index AM callbacks invoked from `vac_bulkdel_one_index`/`vac_cleanup_one_index` | [vacuum.c#vac_bulkdel_one_index](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2651), [vacuum.c#vac_cleanup_one_index](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2672), [indexam.c#index_bulk_delete](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L795), [indexam.c#index_vacuum_cleanup](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L816) |
| `amparallelvacuumoptions` controls per-AM parallel VACUUM participation | [amapi.h#amparallelvacuumoptions](../../../../raw/postgres-18/src/include/access/amapi.h#L281-L282), [vacuum.h#VACUUM_OPTION](../../../../raw/postgres-18/src/include/commands/vacuum.h#L29-L66) |
| `AnalyzeForeignTable` is the FDW ANALYZE entry; no FDW hook for VACUUM | [fdwapi.h#AnalyzeForeignTable_function](../../../../raw/postgres-18/src/include/foreign/fdwapi.h#L156-L158), [analyze.c#analyze_rel](../../../../raw/postgres-18/src/backend/commands/analyze.c#L109) |
| `pg_type.typanalyze` is invoked per attribute during ANALYZE | [analyze.c#examine_attribute](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1036) |
| An extension can register a static background worker only from `_PG_init` while the postmaster processes `shared_preload_libraries` | [bgworker.c#RegisterBackgroundWorker](../../../../raw/postgres-18/src/backend/postmaster/bgworker.c#L932-L977) |
| Custom cumulative statistics already covered in another page | [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](../observability/custom-cumulative-statistics.md) |

## Source References

- [utility.h](../../../../raw/postgres-18/src/include/tcop/utility.h#L1-L112) - `ProcessUtility_hook_type`.
- [utility.c#ProcessUtility](../../../../raw/postgres-18/src/backend/tcop/utility.c#L499) - `ProcessUtility_hook` dispatch and `T_VacuumStmt -> ExecVacuum` route.
- [analyze.h](../../../../raw/postgres-18/src/include/parser/analyze.h#L1-L68) - `post_parse_analyze_hook_type`.
- [analyze.c](../../../../raw/postgres-18/src/backend/parser/analyze.c#L1-L3761) - hook fires from `parse_analyze_fixedparams`/`parse_analyze_varparams`/`parse_analyze_withcb`.
- [elog.h](../../../../raw/postgres-18/src/include/utils/elog.h#L1-L532) - `emit_log_hook_type`.
- [elog.c#EmitErrorReport](../../../../raw/postgres-18/src/backend/utils/error/elog.c#L1692) - hook fires for every log line.
- [miscadmin.h](../../../../raw/postgres-18/src/include/miscadmin.h#L1-L543), [ipc.h](../../../../raw/postgres-18/src/include/storage/ipc.h#L1-L87) - `shmem_request_hook_type` and `shmem_startup_hook_type`.
- [miscinit.c#process_shmem_requests](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1931), [ipci.c](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L1-L390) - hook invocation sites.
- [miscinit.c#process_shared_preload_libraries](../../../../raw/postgres-18/src/backend/utils/init/miscinit.c#L1903) - `_PG_init` is run from here when `process_shared_preload_libraries_in_progress` is true.
- [vacuum.c](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L1-L2703), [vacuum.c#ExecVacuum](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L162), [vacuum.c#vac_bulkdel_one_index](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2651), [vacuum.c#vac_cleanup_one_index](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L2672) - VACUUM dispatch, per-index AM calls.
- [vacuumparallel.c](../../../../raw/postgres-18/src/backend/commands/vacuumparallel.c#L1-L1142) - parallel VACUUM workers calling `ambulkdelete`/`amvacuumcleanup` and copying `IndexBulkDeleteResult` through DSM.
- [read_stream.c#read_stream_clear_strategy](../../../../raw/postgres-18/src/backend/storage/aio/read_stream.c#L1046-L1066), [read_stream.h:102](../../../../raw/postgres-18/src/include/storage/read_stream.h#L102) - the 18.6 exported strategy-clearing entry point.
- [vacuumlazy.c#failsafe-clear-strategy](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1311-L1323) - the only in-tree caller, in VACUUM's wraparound failsafe.
- [autovacuum.c#db_comparator](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1070-L1076) - launcher database-ordering comparator fixed by `4cc49cb7039`.
- [release-18.sgml](../../../../raw/postgres-18/doc/src/sgml/release-18.sgml#L2016-L2062) - 18.6 release notes for the autovacuum database ordering and failsafe buffer-pool fixes.
- [vacuumlazy.c](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1-L3871) - lazy vacuum implementation, including `vacuum_error_callback`.
- [analyze.c](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1-L3090), [analyze.c#analyze_rel](../../../../raw/postgres-18/src/backend/commands/analyze.c#L109), [analyze.c#examine_attribute](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1036), [analyze.c#std_typanalyze](../../../../raw/postgres-18/src/backend/commands/analyze.c#L1891) - ANALYZE driver and per-attribute `typanalyze`.
- [tableam.h](../../../../raw/postgres-18/src/include/access/tableam.h#L1-L2060) - `relation_vacuum`, `scan_analyze_next_block`, `scan_analyze_next_tuple`, `table_relation_vacuum` wrapper.
- [amapi.h](../../../../raw/postgres-18/src/include/access/amapi.h#L1-L332) - `IndexAmRoutine`, `ambulkdelete`, `amvacuumcleanup`, `amparallelvacuumoptions`.
- [indexam.c#index_bulk_delete](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L795), [indexam.c#index_vacuum_cleanup](../../../../raw/postgres-18/src/backend/access/index/indexam.c#L816) - generic dispatchers.
- [vacuum.h](../../../../raw/postgres-18/src/include/commands/vacuum.h#L1-L410) - parallel VACUUM option flags, `VacAttrStats`, type analyze callback types.
- [fdwapi.h](../../../../raw/postgres-18/src/include/foreign/fdwapi.h#L1-L294) - `AnalyzeForeignTable_function`.
- [fdwhandler.sgml#fdw-callbacks-analyze](../../../../raw/postgres-18/doc/src/sgml/fdwhandler.sgml#L1367) - FDW analyze callback documentation.
- [autovacuum.c](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1-L3475), [autovacuum.c#AutoVacLauncherMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L368), [autovacuum.c#AutoVacWorkerMain](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1376), [autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L1885), [autovacuum.c#autovacuum_do_vac_analyze](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3173) - autovacuum process model.
- [bgworker.h](../../../../raw/postgres-18/src/include/postmaster/bgworker.h#L1-L164), [bgworker.c#RegisterBackgroundWorker](../../../../raw/postgres-18/src/backend/postmaster/bgworker.c#L940) - extension background workers.
- [progress.h](../../../../raw/postgres-18/src/include/commands/progress.h#L1-L160) - vacuum/analyze progress field constants.
- [guc_tables.c](../../../../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1-L5457), [config.sgml#guc-shared-preload-libraries](../../../../raw/postgres-18/doc/src/sgml/config.sgml#L11004) - `shared_preload_libraries` GUC context.
- [xfunc.sgml](../../../../raw/postgres-18/doc/src/sgml/xfunc.sgml#L1-L4206) - documentation pattern for `shmem_request_hook` and `shmem_startup_hook`.
- [xfunc.sgml#xfunc-shared-addin-after-startup](../../../../raw/postgres-18/doc/src/sgml/xfunc.sgml#L3690-L3727), [dsm_registry.c#header](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L6-L16), [dsm_registry.c#GetNamedDSMSegment](../../../../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L125-L133) - shared memory after startup without the startup hooks.
- [utility.c#T_VacuumStmt](../../../../raw/postgres-18/src/backend/tcop/utility.c#L861-L863), [analyze.c#transformStmt](../../../../raw/postgres-18/src/backend/parser/analyze.c#L412-L421) - VACUUM dispatch in `standard_ProcessUtility` and the `CMD_UTILITY` wrapper.
- [postinit.c#INIT_PG_LOAD_SESSION_LIBS](../../../../raw/postgres-18/src/backend/utils/init/postinit.c#L1227-L1228), [postmaster.c:930-933](../../../../raw/postgres-18/src/backend/postmaster/postmaster.c#L930-L933) - session preload gate and postmaster preload.
- [postmaster.c:959-962](../../../../raw/postgres-18/src/backend/postmaster/postmaster.c#L959-L962), [postgres.c:4126-4129](../../../../raw/postgres-18/src/backend/tcop/postgres.c#L4126-L4129), [launch_backend.c:698-704](../../../../raw/postgres-18/src/backend/postmaster/launch_backend.c#L698-L704) - the two `process_shmem_requests` callers and the `EXEC_BACKEND` child's preload.
- [ipci.c#RequestAddinShmemSpace](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L64-L79), [lwlock.c#RequestNamedLWLockTranche](../../../../raw/postgres-18/src/backend/storage/lmgr/lwlock.c#L669-L687), [ipci.c#AttachSharedMemoryStructs](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L163-L193), [ipci.c#CreateSharedMemoryAndSemaphores](../../../../raw/postgres-18/src/backend/storage/ipc/ipci.c#L195-L250) - request checks and `shmem_startup_hook` call sites.
- [cluster.c#swap_relation_files](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1300-L1307), [cluster.c:1563](../../../../raw/postgres-18/src/backend/commands/cluster.c#L1563), [dependency.c#deleteOneObject](../../../../raw/postgres-18/src/backend/catalog/dependency.c#L1253-L1255), [autovacuum.c#do_autovacuum](../../../../raw/postgres-18/src/backend/postmaster/autovacuum.c#L2230-L2249) - indirect `object_access_hook` calls from `VACUUM FULL` and the orphaned temporary-table drop.
- [selfuncs.c#examine_variable](../../../../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L5487-L5489), [selfuncs.c#examine_simple_variable](../../../../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L5709-L5717), [lsyscache.c#get_attavgwidth](../../../../raw/postgres-18/src/backend/utils/cache/lsyscache.c#L3459-L3470) - planner-side statistics read hooks.
- [pgstat_relation.c#pgstat_report_vacuum](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L250-L261), [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L350-L361), [pgstat_database.c#pgstat_report_autovac](../../../../raw/postgres-18/src/backend/utils/activity/pgstat_database.c#L49-L75) - built-in vacuum, analyze and autovacuum reporters.
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L700-L703), [system_views.sql#pg_stat_progress_analyze](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L1204-L1224), [system_views.sql#pg_stat_progress_vacuum](../../../../raw/postgres-18/src/backend/catalog/system_views.sql#L1226-L1245) - statistics and progress views.
- [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](../observability/custom-cumulative-statistics.md) - companion page for the cumulative-stats extension surface.

## Open Questions

- This page enumerates the in-process hook variables and AM/FDW callbacks
  observable on the vacuum/autovacuum/analyze paths in pinned
  `raw/postgres-18/` source. It does not exhaustively enumerate every minor
  per-callsite extension surface that vacuum-adjacent code shares with the
  rest of the backend (for example, `MemoryContextCallback`, error context
  callbacks, or `ErrorContextCallback`-based instrumentation in
  `vacuumlazy.c`); those are not pluggable extension hooks in the same sense
  ([vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L675-L678),
  [vacuumlazy.c#vacuum_error_callback](../../../../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3775-L3839)).
- This page documents existing PostgreSQL 18 behavior. It does not propose
  new vacuum or autovacuum hooks, nor a design for the future "separate
  callback to integrate with autovacuum's scheduling" suggested in the
  `relation_vacuum` header comment
  ([tableam.h:649-650](../../../../raw/postgres-18/src/include/access/tableam.h#L649-L650)).

## Navigation

- [v18/index](../../index.md)
- [wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
