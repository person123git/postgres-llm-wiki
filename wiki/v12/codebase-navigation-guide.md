---
type: codebase-navigation-guide
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# PostgreSQL 12 Codebase Navigation Guide (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Fast Map](#fast-map)
  - [SQL Statement Path](#sql-statement-path)
  - [Where To Start By Task](#where-to-start-by-task)
  - [Generated And Catalog Files](#generated-and-catalog-files)
  - [Key Data Structures](#key-data-structures)
  - [Tests And Docs](#tests-and-docs)
  - [Navigation Checklist](#navigation-checklist)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Create a codebase navigation guide document.

Prompt note: the user approved correcting the original prompt capitalization and punctuation before the guide was first filed.

## Answer

Use the PostgreSQL 12 tree as a set of entry points, not as one flat C program. Start with the build maps, then follow a SQL statement through `tcop`, simple or extended protocol entry points, parser, analyzer, rewriter, planner, executor, storage, and command code. The top-level `src/Makefile` lists the main source families, including `common`, `port`, `timezone`, `backend`, generated conversion/snowball code, `include`, `interfaces`, `fe_utils`, `bin`, `pl`, and test directories [src/Makefile#SUBDIRS](../../raw/postgres-12/src/Makefile#L15-L32). The backend makefile then narrows the server into subsystems such as `access`, `catalog`, `parser`, `commands`, `executor`, `optimizer`, `postmaster`, `replication`, `rewrite`, `statistics`, `storage`, `tcop`, `utils`, and `jit` [backend/Makefile#SUBDIRS](../../raw/postgres-12/src/backend/Makefile#L20-L24).

### Fast Map

| Need | Start here | Why |
|---|---|---|
| Server internals | `src/backend/` | The backend makefile is the subsystem map for parser, commands, executor, optimizer, storage, replication, rewrite, statistics, `tcop`, and utilities [backend/Makefile#SUBDIRS](../../raw/postgres-12/src/backend/Makefile#L20-L24). |
| Access methods and heap/index code | `src/backend/access/` | Its subdirectories separate BRIN, GIN, GiST, hash, heap, generic index code, B-tree, resource-manager descriptions, SP-GiST, table AM, table sampling, and transaction access code [access/Makefile#SUBDIRS](../../raw/postgres-12/src/backend/access/Makefile#L1-L14). |
| Storage manager and locks | `src/backend/storage/` | Its subdirectories cover buffer, file, freespace, IPC, large object, lock manager, page, smgr, and sync code [storage/Makefile#SUBDIRS](../../raw/postgres-12/src/backend/storage/Makefile#L1-L13). |
| Client tools | `src/bin/` | Its makefile lists client programs such as `initdb`, `pg_basebackup`, `pg_checksums`, `pg_ctl`, `pg_dump`, `pg_rewind`, `pg_waldump`, `pgbench`, `psql`, and script tools [bin/Makefile#SUBDIRS](../../raw/postgres-12/src/bin/Makefile#L16-L33). |
| Frontend interfaces | `src/interfaces/` | PostgreSQL 12 builds `libpq` and `ecpg` under `src/interfaces` [interfaces/Makefile#SUBDIRS](../../raw/postgres-12/src/interfaces/Makefile#L11-L19). |
| Background processes | `src/backend/postmaster/` | The postmaster subtree builds autovacuum, background worker, background writer, checkpointer, archiver, stats collector, postmaster, startup, syslogger, and WAL writer objects [postmaster/Makefile#OBJS](../../raw/postgres-12/src/backend/postmaster/Makefile#L11-L18). |
| Replication | `src/backend/replication/` | This subtree builds WAL sender/receiver, base backup, replication grammar, replication slots, synchronous replication, and has a `logical` subdirectory [replication/Makefile#OBJS](../../raw/postgres-12/src/backend/replication/Makefile#L17-L22). |
| Extensions and contrib modules | `contrib/` | The contrib makefile lists packaged modules such as `amcheck`, `auto_explain`, `pageinspect`, `pg_stat_statements`, `pgstattuple`, `postgres_fdw`, `test_decoding`, and many others [contrib/Makefile#SUBDIRS](../../raw/postgres-12/contrib/Makefile#L7-L51). |
| Developer docs | `doc/src/sgml/` | The SGML file list includes developer chapters for BKI, catalogs, GEQO, B-tree, table AM, index AM, protocol, sources, and storage [filelist.sgml#developer-guide](../../raw/postgres-12/doc/src/sgml/filelist.sgml#L81-L104). |

### SQL Statement Path

1. A backend session enters `PostgresMain()`, which initializes standalone state when needed, sets processing mode, initializes GUC options outside the postmaster, processes command-line switches, selects config files for standalone backends, and installs backend signal handlers [postgres.c#PostgresMain-startup](../../raw/postgres-12/src/backend/tcop/postgres.c#L3707-L3750) [postgres.c#PostgresMain-signals](../../raw/postgres-12/src/backend/tcop/postgres.c#L3752-L3805).

2. A simple-query protocol message enters `exec_simple_query()`. That path starts a transaction command, drops any previous unnamed statement, switches to `MessageContext`, and calls `pg_parse_query()` before looping over each raw statement [postgres.c#exec_simple_query-parse](../../raw/postgres-12/src/backend/tcop/postgres.c#L980-L1038) [postgres.c#exec_simple_query-loop](../../raw/postgres-12/src/backend/tcop/postgres.c#L1064-L1077).

3. Extended-query protocol messages take the prepared-statement path. `PostgresMain()` dispatches Parse messages to `exec_parse_message()`, Bind messages to `exec_bind_message()`, and Execute messages to `exec_execute_message()` [postgres.c#PostgresMain-extended-protocol](../../raw/postgres-12/src/backend/tcop/postgres.c#L4268-L4309). `exec_parse_message()` allows only one user statement, calls `pg_parse_query()`, creates a `CachedPlanSource`, uses `parse_analyze_varparams()` for parameter-aware analysis, rewrites with `pg_rewrite_query()`, and stores the finished plan source [postgres.c#exec_parse_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1314-L1498). `exec_bind_message()` fetches the named or unnamed prepared statement, reads parameter formats and values, obtains a `CachedPlan`, defines a portal, and starts it [postgres.c#exec_bind_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1576-L1898). `exec_execute_message()` then retrieves the portal and runs it with `PortalRun()` [postgres.c#exec_execute_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1946-L2096).

4. Raw parsing is deliberately separate from analysis and rewrite because analysis and rewrite need catalog access, while the raw parser must still identify transaction-control commands in an aborted transaction [postgres.c#pg_parse_query](../../raw/postgres-12/src/backend/tcop/postgres.c#L620-L642). The parser driver says grammar output is a raw parse tree that still needs analysis by `analyze.c`, and `raw_parser()` initializes the scanner, initializes the Bison parser, calls `base_yyparse()`, and returns a list of `RawStmt` nodes [parser.c#raw_parser-design](../../raw/postgres-12/src/backend/parser/parser.c#L3-L10) [parser.c#raw_parser](../../raw/postgres-12/src/backend/parser/parser.c#L28-L62).

5. Parse analysis turns a `RawStmt` into a `Query`. `parse_analyze()` builds a `ParseState`, records source text, handles fixed parameters when supplied, calls `transformTopLevelStmt()`, invokes `post_parse_analyze_hook` if set, and returns a `Query` [analyze.c#parse_analyze](../../raw/postgres-12/src/backend/parser/analyze.c#L90-L123). `RawStmt` comments state that parse analysis converts raw trees to `Query` nodes, while utility statements usually move the raw tree into `Query.utilityStmt` and do useful work at execution time [parsenodes.h#RawStmt](../../raw/postgres-12/src/include/nodes/parsenodes.h#L1470-L1488).

6. Rewrite happens after analysis. `pg_analyze_and_rewrite()` calls `parse_analyze()` and then `pg_rewrite_query()` [postgres.c#pg_analyze_and_rewrite](../../raw/postgres-12/src/backend/tcop/postgres.c#L670-L709). `pg_rewrite_query()` sends regular queries to `QueryRewrite()` but leaves utility queries in a one-item list [postgres.c#pg_rewrite_query](../../raw/postgres-12/src/backend/tcop/postgres.c#L762-L789). `QueryRewrite()` applies non-SELECT rules, then retrieve-instead-rewrite rules, and preserves the input query ID on rewritten queries [rewriteHandler.c#QueryRewrite](../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L3923-L3963).

7. Planning starts after rewrite. `pg_plan_queries()` wraps utility queries in a `PlannedStmt` without optimizer planning, while normal statements call `pg_plan_query()` [postgres.c#pg_plan_queries](../../raw/postgres-12/src/backend/tcop/postgres.c#L937-L975). `pg_plan_query()` skips utility commands, asserts a snapshot is active, and calls `planner()` for optimizer work [postgres.c#pg_plan_query](../../raw/postgres-12/src/backend/tcop/postgres.c#L856-L879). `planner()` uses `planner_hook` when installed, otherwise it calls `standard_planner()` [planner.c#planner](../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L270-L280).

8. The planner then builds global and per-query state, computes relation sizes, builds base-relation paths, builds join paths, chooses a best path, and converts that path tree into a plan tree. `standard_planner()` creates `PlannerGlobal` state for the planner invocation [planner.c#standard_planner-global-state](../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L280-L317). `subquery_planner()` creates a `PlannerInfo`, handles CTEs, replaces empty jointrees, pulls up sublinks and subqueries, and flattens simple `UNION ALL` queries when possible [planner.c#subquery_planner-start](../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L596-L680). `make_one_rel()` computes base-relation sizes, builds base pathlists, and builds the join relation for the join tree [allpaths.c#make_one_rel](../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L179-L234). `create_plan_recurse()` dispatches path node types to scan, join, append, result, aggregate, and other plan builders [createplan.c#create_plan_recurse](../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L320-L430).

9. Execution is portal-driven. `exec_simple_query()` creates an unnamed portal, defines it with the planned statement list, starts it, creates a destination receiver, and calls `PortalRun()` to completion [postgres.c#exec_simple_query-portal](../../raw/postgres-12/src/backend/tcop/postgres.c#L1153-L1222). `PortalRun()` marks the portal active, switches context, chooses the portal strategy, runs selects through `PortalRunSelect()`, runs multi-query portals through `PortalRunMulti()`, and restores portal/resource-owner state on success or error [pquery.c#PortalRun](../../raw/postgres-12/src/backend/tcop/pquery.c#L686-L850). `PortalRunSelect()` pushes the query snapshot and calls `ExecutorRun()` for normal query execution [pquery.c#PortalRunSelect](../../raw/postgres-12/src/backend/tcop/pquery.c#L871-L933).

10. The executor has explicit start/run/finish phases. `ExecutorStart()` uses `ExecutorStart_hook` if set, otherwise `standard_ExecutorStart()` [execMain.c#ExecutorStart](../../raw/postgres-12/src/backend/executor/execMain.c#L130-L153). `standard_ExecutorStart()` creates `EState`, switches to the per-query context, copies parameters and snapshots, starts trigger context when needed, and initializes the plan state tree [execMain.c#standard_ExecutorStart](../../raw/postgres-12/src/backend/executor/execMain.c#L180-L267). `ExecutorRun()` uses `ExecutorRun_hook` if set, otherwise `standard_ExecutorRun()`, and the standard path calls `ExecutePlan()` for non-`NoMovementScanDirection` runs [execMain.c#ExecutorRun](../../raw/postgres-12/src/backend/executor/execMain.c#L271-L309) [execMain.c#standard_ExecutorRun](../../raw/postgres-12/src/backend/executor/execMain.c#L355-L385). `ExecutorFinish()` runs post-processing and queued AFTER triggers before marking the executor state finished [execMain.c#ExecutorFinish](../../raw/postgres-12/src/backend/executor/execMain.c#L387-L447).

11. Utility statements are not normal executor plans. `ProcessUtility()` asserts `CMD_UTILITY`, exposes `ProcessUtility_hook`, and otherwise calls `standard_ProcessUtility()` [utility.c#ProcessUtility](../../raw/postgres-12/src/backend/tcop/utility.c#L338-L363). `standard_ProcessUtility()` directly handles utility commands without event-trigger support and sends event-trigger-supported commands to `ProcessUtilitySlow()` [utility.c#standard_ProcessUtility-design](../../raw/postgres-12/src/backend/tcop/utility.c#L365-L375). Direct utility cases include `EXPLAIN` through `ExplainQuery()` and `SET` through `ExecSetVariableStmt()` [utility.c#standard_ProcessUtility-Explain-Set](../../raw/postgres-12/src/backend/tcop/utility.c#L674-L686). For common DDL, `ProcessUtilitySlow()` routes `CREATE TABLE` through `transformCreateStmt()` and `DefineRelation()`, `ALTER TABLE` through `transformAlterTableStmt()`, and `CREATE INDEX` through `transformIndexStmt()` and `DefineIndex()` [utility.c#ProcessUtilitySlow-CreateStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L993-L1017) [utility.c#ProcessUtilitySlow-AlterTableStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L1096-L1125) [utility.c#ProcessUtilitySlow-IndexStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L1301-L1392).

12. Error and edge paths often explain control flow. In aborted transaction state, `exec_simple_query()` rejects all commands except transaction-exit statements before analysis, rewrite, or planning [postgres.c#aborted-transaction-check](../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105). The extended path has matching abort-state checks during parse, bind, and execute [postgres.c#exec_parse_message-abort-check](../../raw/postgres-12/src/backend/tcop/postgres.c#L1414-L1428) [postgres.c#exec_bind_message-abort-check](../../raw/postgres-12/src/backend/tcop/postgres.c#L1670-L1686) [postgres.c#exec_execute_message-abort-check](../../raw/postgres-12/src/backend/tcop/postgres.c#L2069-L2079). `standard_ProcessUtility()` checks read-only state before its utility dispatch, while commands such as `CLUSTER`, `VACUUM`, and `ANALYZE` explicitly call recovery-prevention checks before their command-specific work [utility.c#standard_ProcessUtility-readonly](../../raw/postgres-12/src/backend/tcop/utility.c#L390-L401) [utility.c#VacuumStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L655-L671). Server messages should use `ereport()` or `elog()`, with `ereport()` carrying severity plus message elements such as SQLSTATE and `errmsg()` [sources.sgml#error-reporting](../../raw/postgres-12/doc/src/sgml/sources.sgml#L83-L120). SQLSTATE macros and documentation tables are generated from `src/backend/utils/errcodes.txt` [errcodes.txt#generated-files](../../raw/postgres-12/src/backend/utils/errcodes.txt#L7-L23).

### Where To Start By Task

| Task | First files | Follow-up files |
|---|---|---|
| Trace a `SELECT` plan | `src/backend/tcop/postgres.c`, `src/backend/optimizer/plan/planner.c`, `src/backend/optimizer/path/allpaths.c` | `src/backend/optimizer/plan/createplan.c`, `src/include/nodes/pathnodes.h`, `src/include/nodes/plannodes.h`; the cited path shows `pg_plan_query()` to `planner()`, `standard_planner()`, `make_one_rel()`, and plan creation [postgres.c#pg_plan_query](../../raw/postgres-12/src/backend/tcop/postgres.c#L856-L879) [planner.c#planner](../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L270-L280) [allpaths.c#make_one_rel](../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L179-L234) [createplan.c#create_plan_recurse](../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L320-L430). |
| Trace prepared statements or extended-query protocol | `src/backend/tcop/postgres.c`, `src/backend/utils/cache/plancache.c` | Start at the Parse/Bind/Execute dispatch, then follow `exec_parse_message()` to `CachedPlanSource`, `exec_bind_message()` to `GetCachedPlan()` and `PortalDefineQuery()`, and `exec_execute_message()` to `PortalRun()` [postgres.c#PostgresMain-extended-protocol](../../raw/postgres-12/src/backend/tcop/postgres.c#L4268-L4309) [postgres.c#exec_parse_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1314-L1498) [postgres.c#exec_bind_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1871-L1898) [postgres.c#exec_execute_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L2084-L2096). |
| Trace executor behavior | `src/backend/executor/execMain.c`, executor `node*.c` files, `src/include/nodes/execnodes.h` | `QueryDesc` says it encapsulates executor inputs and later stores `EState` plus the `PlanState` tree; `PlanState` links each runtime node to one query-wide `EState` and its `ExecProcNode` method [execdesc.h#QueryDesc](../../raw/postgres-12/src/include/executor/execdesc.h#L22-L56) [execnodes.h#PlanState](../../raw/postgres-12/src/include/nodes/execnodes.h#L936-L980). |
| Trace DDL | `src/backend/tcop/utility.c` | Then jump to `src/backend/commands/` for command bodies and `src/backend/catalog/` for catalog writes; `CREATE TABLE`, `ALTER TABLE`, and `CREATE INDEX` are routed from `ProcessUtilitySlow()` to `DefineRelation()`, transformed ALTER TABLE work, and `DefineIndex()` respectively [utility.c#ProcessUtilitySlow-CreateStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L993-L1017) [utility.c#ProcessUtilitySlow-AlterTableStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L1096-L1125) [utility.c#ProcessUtilitySlow-IndexStmt](../../raw/postgres-12/src/backend/tcop/utility.c#L1301-L1392). |
| Trace heap/table storage | `src/include/access/tableam.h`, `src/backend/access/heap/` | `TableAmRoutine` defines the table-AM callback surface, including scan begin/end/rescan/get-next-slot callbacks and parallel scan callbacks [tableam.h#TableAmRoutine-scan](../../raw/postgres-12/src/include/access/tableam.h#L162-L245). Heap scans advance blocks, handle parallel next-page selection, call `heapgetpage()`, and test snapshots while walking pages [heapam.c#heapgettup-page-walk](../../raw/postgres-12/src/backend/access/heap/heapam.c#L1006-L1076). |
| Trace index access | `src/include/access/amapi.h`, `src/backend/access/index/`, access-method subdirectory | `IndexAmRoutine` defines AM capabilities and callback slots such as build, insert, bulk delete, vacuum cleanup, costing, validation, begin/rescan/gettuple/getbitmap/end scan, mark/restore position, and parallel-scan callbacks [amapi.h#IndexAmRoutine](../../raw/postgres-12/src/include/access/amapi.h#L159-L233). B-tree's handler fills that routine with B-tree capabilities and callbacks [nbtree.c#bthandler](../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L102-L151). |
| Trace relation metadata | `src/include/utils/rel.h`, `src/backend/utils/cache/relcache.c`, catalog headers | `RelationData` stores relation identity, relcache validity, `pg_class` form data, tuple descriptor, rewrite rules, triggers, FKs, partition cache fields, index list fields, table AM routine, and index AM routine fields [rel.h#RelationData-core](../../raw/postgres-12/src/include/utils/rel.h#L53-L141) [rel.h#RelationData-index-am](../../raw/postgres-12/src/include/utils/rel.h#L143-L180). |
| Trace GUCs | `src/backend/utils/misc/guc.c`, `src/include/utils/guc.h` | `GucContext` defines internal, postmaster, SIGHUP, backend, superuser, and user contexts; `guc.c` exposes display names for those contexts and stores built-in settings in typed arrays such as `ConfigureNamesBool` [guc.h#GucContext](../../raw/postgres-12/src/include/utils/guc.h#L60-L77) [guc.c#GucContext_Names](../../raw/postgres-12/src/backend/utils/misc/guc.c#L589-L603) [guc.c#ConfigureNamesBool-example](../../raw/postgres-12/src/backend/utils/misc/guc.c#L922-L1015). |
| Trace contrib extension behavior | Extension subdirectory under `contrib/`, then `src/backend/commands/extension.c` | Individual extension makefiles show the C objects, SQL upgrade files, control name, and regression tests; `pgstattuple` builds `pgstattuple.o`, `pgstatindex.o`, and `pgstatapprox.o`, installs the `pgstattuple` extension SQL files, and runs the `pgstattuple` regression test [pgstattuple/Makefile](../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24). `pg_stat_statements` builds one module, installs its versioned SQL files, uses a temp regression config, and disables `installcheck` because the tests require `shared_preload_libraries=pg_stat_statements` [pg_stat_statements/Makefile](../../raw/postgres-12/contrib/pg_stat_statements/Makefile#L1-L30). The core extension command code explains the runtime boundary: extensions are SQL-object collections backed by a control file and install script, `parse_extension_control_file()` reads control files, `CreateExtensionInternal()` selects a version/update path and inserts the `pg_extension` tuple, and `execute_sql_string()` parses, rewrites, plans, and executes each script command [extension.c#extension-overview](../../raw/postgres-12/src/backend/commands/extension.c#L6-L13) [extension.c#parse_extension_control_file](../../raw/postgres-12/src/backend/commands/extension.c#L468-L624) [extension.c#CreateExtensionInternal](../../raw/postgres-12/src/backend/commands/extension.c#L1262-L1541) [extension.c#execute_sql_string](../../raw/postgres-12/src/backend/commands/extension.c#L698-L779). |

### Generated And Catalog Files

Treat generated files as source-driven artifacts. `src/backend/Makefile` has a `generated-headers` target for parser grammar headers, LWLock names, catalog headers, and utility headers, and its `distprep` target generates parser, bootstrap, catalog, replication grammar, LWLock, utility, JSONPath, GUC-file, and sort artifacts [backend/Makefile#generated-headers](../../raw/postgres-12/src/backend/Makefile#L123-L190).

System catalogs start in `src/include/catalog/*.h` and selected `.dat` files. `src/backend/catalog/Makefile` declares the ordered catalog header list, notes that bootstrap catalogs must appear first, creates generated `_d.h` headers plus `schemapg.h`, includes `toasting.h` and `indexing.h` when building `postgres.bki`, and invokes `genbki.pl` over catalog headers and `.dat` files [catalog/Makefile#catalog-generation](../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90). `genbki.h` defines `CATALOG()` and related macros for C compilation, and the same words are recognized by `genbki.pl` to build the BKI bootstrap file [genbki.h#CATALOG](../../raw/postgres-12/src/include/catalog/genbki.h#L1-L64). `genbki.pl` says it generates BKI files and symbol-definition headers from specially formatted headers and `.dat` files, and those BKI files initialize the `postgres` template database [genbki.pl#usage](../../raw/postgres-12/src/backend/catalog/genbki.pl#L960-L974).

Catalog indexes are not ordinary handwritten SQL files. `indexing.h` says `DECLARE_INDEX` and `DECLARE_UNIQUE_INDEX` lines are processed by `genbki.pl` into statements that the bootstrap parser turns into `DefineIndex()` calls, and C code should reference generated OID macros instead of literal index names or numbers [indexing.h#DECLARE_INDEX](../../raw/postgres-12/src/include/catalog/indexing.h#L47-L66).

Parser generated files come from Bison and Flex. The parser makefile builds `analyze.o`, `gram.o`, `scan.o`, `parser.o`, and related parse modules, makes `gram.h` depend on `gram.c`, uses Bison flags to generate `gram.c` and `gram.h`, and states that `gram.c`, `gram.h`, and `scan.c` are shipped in distribution tarballs [parser/Makefile#parser-generated-files](../../raw/postgres-12/src/backend/parser/Makefile#L13-L54).

Utility generated headers include function-manager lookup files, error codes, DTrace probes, and LWLock names. `src/backend/utils/Makefile` uses `Gen_fmgrtab.pl` to generate `fmgroids.h`, `fmgrprotos.h`, and `fmgrtab.c`, uses `generate-errcodes.pl` to derive `errcodes.h` from `errcodes.txt`, and generates or stubs `probes.h` depending on DTrace support [utils/Makefile#generated-files](../../raw/postgres-12/src/backend/utils/Makefile#L24-L76). `storage/lmgr/Makefile` generates `lwlocknames.h` and `lwlocknames.c` from `lwlocknames.txt` via `generate-lwlocknames.pl` [lmgr/Makefile#lwlocknames](../../raw/postgres-12/src/backend/storage/lmgr/Makefile#L24-L43). `src/include/Makefile` installs generated server headers such as `errcodes.h`, `fmgroids.h`, `fmgrprotos.h`, catalog `_d.h` headers, parser `gram.h`, `lwlocknames.h`, and `probes.h` [include/Makefile#generated-install](../../raw/postgres-12/src/include/Makefile#L43-L60).

### Key Data Structures

| Structure | File | Role |
|---|---|---|
| `RawStmt` | `src/include/nodes/parsenodes.h` | Container for one raw parse tree with source-location fields; comments state parse analysis converts it to `Query` and utility work usually happens later [parsenodes.h#RawStmt](../../raw/postgres-12/src/include/nodes/parsenodes.h#L1470-L1488). |
| `Query` | `src/include/nodes/parsenodes.h` | Parse analysis turns statements into `Query` trees; utility statements set `utilityStmt`; planning converts `Query` into `PlannedStmt`, and the executor does not use `Query` directly [parsenodes.h#Query](../../raw/postgres-12/src/include/nodes/parsenodes.h#L97-L150). |
| `PlannedStmt` | `src/include/nodes/plannodes.h` | Planner output containing command type, query ID, return/modifying flags, plan tree, range table, result relations, dependencies, parameter types, utility statement pointer, and source location [plannodes.h#PlannedStmt](../../raw/postgres-12/src/include/nodes/plannodes.h#L42-L95). |
| `PlannerGlobal` and `PlannerInfo` | `src/include/nodes/pathnodes.h` and `planner.c` | `PlannerGlobal` holds state shared across a planner invocation, while `subquery_planner()` creates a per-query `PlannerInfo` linked to it [pathnodes.h#PlannerGlobal](../../raw/postgres-12/src/include/nodes/pathnodes.h#L97-L149) [planner.c#subquery_planner-PlannerInfo](../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L596-L637). |
| `RelOptInfo` | `src/include/nodes/pathnodes.h` | Planner relation record containing relids, row estimates, path lists, cheapest paths, base relation metadata, index/statistics lists, pages/tuples/all-visible fraction, FDW state, restriction clauses, join info, and partitioning state [pathnodes.h#RelOptInfo](../../raw/postgres-12/src/include/nodes/pathnodes.h#L632-L723). |
| `Path` | `src/include/nodes/pathnodes.h` | Planner alternative with node type, parent relation, target, parameterization, parallel flags, row estimate, startup/total cost, and pathkeys [pathnodes.h#Path](../../raw/postgres-12/src/include/nodes/pathnodes.h#L1104-L1126). |
| `QueryDesc` | `src/include/executor/execdesc.h` | Executor descriptor containing planner output, source text, snapshots, destination receiver, params, query environment, instrumentation options, result tuple descriptor, `EState`, and `PlanState` [execdesc.h#QueryDesc](../../raw/postgres-12/src/include/executor/execdesc.h#L22-L56). |
| `EState` | `src/include/nodes/execnodes.h` | Query-wide executor state with scan direction, snapshots, range table arrays, relation pointers, planned statement, target relation state, trigger state, parameter state, per-query memory context, tuple table, processed row count, instrumentation flags, and subplan state [execnodes.h#EState](../../raw/postgres-12/src/include/nodes/execnodes.h#L496-L590). |
| `PlanState` | `src/include/nodes/execnodes.h` | Runtime state for a plan node, including the associated `Plan`, query-wide `EState`, `ExecProcNode` methods, instrumentation, child state links, quals, subplans, tuple descriptors, slots, expression context, and projection info [execnodes.h#PlanState](../../raw/postgres-12/src/include/nodes/execnodes.h#L936-L1018). |
| `RelationData` | `src/include/utils/rel.h` | Relcache entry with physical relation identity, cached smgr handle, validity flags, `pg_class` form, tuple descriptor, rewrite rules, triggers, FK/index/statistics/partition caches, table AM routine, and index AM routine state [rel.h#RelationData-core](../../raw/postgres-12/src/include/utils/rel.h#L53-L141) [rel.h#RelationData-index-am](../../raw/postgres-12/src/include/utils/rel.h#L143-L180). |
| `TableAmRoutine` | `src/include/access/tableam.h` | Table access-method API with slot, scan, rescan, next-tuple, and parallel scan callbacks [tableam.h#TableAmRoutine-scan](../../raw/postgres-12/src/include/access/tableam.h#L162-L245). |
| `IndexAmRoutine` | `src/include/access/amapi.h` | Index access-method API with capability flags and callbacks for build, insert, delete/vacuum, costing, validation, scan, tuple/bitmap retrieval, mark/restore position, and parallel scans [amapi.h#IndexAmRoutine](../../raw/postgres-12/src/include/access/amapi.h#L159-L233). |
| `MemoryContextData` | `src/include/nodes/memnodes.h` | Memory context header containing type, reset state, method table, parent/child links, debugging names, and reset callbacks [memnodes.h#MemoryContextData](../../raw/postgres-12/src/include/nodes/memnodes.h#L76-L90). |

Memory contexts are central to PostgreSQL navigation. The memory manager README says PostgreSQL usually allocates in memory contexts, `palloc()` allocates in `CurrentMemoryContext`, and resetting transaction-or-shorter contexts reclaims transient memory at transaction, query, or tuple boundaries [mmgr/README#overview](../../raw/postgres-12/src/backend/utils/mmgr/README#L9-L45). It also says executor top-level routines normally run in a context created by `ExecutorStart()` and destroyed by `ExecutorEnd()`, with per-plan-node expression contexts reset during tuple processing [mmgr/README#executor-contexts](../../raw/postgres-12/src/backend/utils/mmgr/README#L278-L310).

### Tests And Docs

Use SQL regression tests when behavior can be observed with single-session SQL. The regression makefile builds `pg_regress`, builds a dynamically loaded `regress` module, collects SQL/expected/data files, and runs `check` against `parallel_schedule` or `installcheck` against `serial_schedule` [regress/GNUmakefile#pg_regress](../../raw/postgres-12/src/test/regress/GNUmakefile#L31-L87) [regress/GNUmakefile#check-targets](../../raw/postgres-12/src/test/regress/GNUmakefile#L125-L147).

Use isolation tests for multi-session concurrency behavior. The isolation makefile identifies `pg_isolation_regress` and `isolationtester` as the multi-client test driver, builds the driver from parser/tester objects, and runs schedules through `pg_isolation_regress_check` or `pg_isolation_regress_installcheck` [isolation/Makefile#driver](../../raw/postgres-12/src/test/isolation/Makefile#L1-L35) [isolation/Makefile#check-targets](../../raw/postgres-12/src/test/isolation/Makefile#L52-L66).

Use TAP tests for process-level behavior. `PostgresNode.pm` describes a PostgreSQL server-instance helper with methods to initialize, start, change configuration, restart, run `psql`, poll queries, run backups, restore nodes, stop servers, and find free ports [PostgresNode.pm#synopsis](../../raw/postgres-12/src/test/perl/PostgresNode.pm#L4-L80).

Use `src/test/modules/` when the behavior is best expressed through a test-only extension. Its makefile lists modules such as `brin`, `commit_ts`, `dummy_seclabel`, `snapshot_too_old`, `test_extensions`, `test_parser`, `test_pg_dump`, `test_rls_hooks`, `unsafe_tests`, and `worker_spi` [test/modules/Makefile#SUBDIRS](../../raw/postgres-12/src/test/modules/Makefile#L1-L26).

Use SGML docs as same-checkout supporting evidence, not as a substitute for implementation source. The SGML file list shows user, administrator, programmer, developer, reference, and contrib documentation entities, including developer pages for BKI, catalogs, access methods, protocol, sources, and storage [filelist.sgml#developer-guide](../../raw/postgres-12/doc/src/sgml/filelist.sgml#L81-L104) [filelist.sgml#contrib](../../raw/postgres-12/doc/src/sgml/filelist.sgml#L106-L120). The coding-conventions chapter states the code uses four-column tab spacing with tabs preserved, BSD brace placement, readable line lengths around an 80-column window, no C++ `//` comments, and sample editor settings under `src/tools` [sources.sgml#formatting](../../raw/postgres-12/doc/src/sgml/sources.sgml#L6-L80).

### Navigation Checklist

1. Start from the target major-version checkout, here `raw/postgres-12/`, and keep source citations inside that checkout.
2. Use makefiles to identify owning directories before following calls; the top-level and backend makefiles are the shortest reliable map [src/Makefile#SUBDIRS](../../raw/postgres-12/src/Makefile#L15-L32) [backend/Makefile#SUBDIRS](../../raw/postgres-12/src/backend/Makefile#L20-L24).
3. For SQL behavior, enter through `exec_simple_query()` for simple-query protocol or Parse/Bind/Execute dispatch for extended-query protocol, then separate normal statements from utility statements before entering planner or executor code [postgres.c#exec_simple_query-loop](../../raw/postgres-12/src/backend/tcop/postgres.c#L1064-L1222) [postgres.c#PostgresMain-extended-protocol](../../raw/postgres-12/src/backend/tcop/postgres.c#L4268-L4309) [postgres.c#pg_plan_queries](../../raw/postgres-12/src/backend/tcop/postgres.c#L937-L975).
4. For planner behavior, keep `Query`, `PlannerInfo`, `RelOptInfo`, `Path`, and `PlannedStmt` open side by side; those structs define the nouns used by `planner.c`, `allpaths.c`, and `createplan.c` [parsenodes.h#Query](../../raw/postgres-12/src/include/nodes/parsenodes.h#L97-L150) [pathnodes.h#RelOptInfo](../../raw/postgres-12/src/include/nodes/pathnodes.h#L632-L723) [pathnodes.h#Path](../../raw/postgres-12/src/include/nodes/pathnodes.h#L1104-L1126) [plannodes.h#PlannedStmt](../../raw/postgres-12/src/include/nodes/plannodes.h#L42-L95).
5. For executor behavior, keep `QueryDesc`, `EState`, `PlanState`, and the relevant executor node file open; `ExecutorStart()` builds executor state and `ExecutorRun()` drives `ExecutePlan()` [execdesc.h#QueryDesc](../../raw/postgres-12/src/include/executor/execdesc.h#L22-L56) [execnodes.h#EState](../../raw/postgres-12/src/include/nodes/execnodes.h#L496-L590) [execnodes.h#PlanState](../../raw/postgres-12/src/include/nodes/execnodes.h#L936-L1018) [execMain.c#standard_ExecutorRun](../../raw/postgres-12/src/backend/executor/execMain.c#L355-L385).
6. For storage behavior, identify the AM boundary first: heap/table code goes through `TableAmRoutine`, index code goes through `IndexAmRoutine`, and relation metadata comes through `RelationData` [tableam.h#TableAmRoutine-scan](../../raw/postgres-12/src/include/access/tableam.h#L162-L245) [amapi.h#IndexAmRoutine](../../raw/postgres-12/src/include/access/amapi.h#L159-L233) [rel.h#RelationData-core](../../raw/postgres-12/src/include/utils/rel.h#L53-L141).
7. For catalog behavior, check both the catalog header and generated artifacts: `genbki.h`/catalog headers define the C-visible forms, `genbki.pl` builds BKI and symbol headers, and `indexing.h` feeds bootstrap index creation [genbki.h#CATALOG](../../raw/postgres-12/src/include/catalog/genbki.h#L1-L64) [genbki.pl#usage](../../raw/postgres-12/src/backend/catalog/genbki.pl#L960-L974) [indexing.h#DECLARE_INDEX](../../raw/postgres-12/src/include/catalog/indexing.h#L47-L66).
8. For failure behavior, read the nearest `ereport()`/`elog()` path, the SQLSTATE source in `errcodes.txt`, and the transaction or utility-state guard that reaches it [sources.sgml#error-reporting](../../raw/postgres-12/doc/src/sgml/sources.sgml#L83-L120) [errcodes.txt#format](../../raw/postgres-12/src/backend/utils/errcodes.txt#L24-L35) [postgres.c#aborted-transaction-check](../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105).
9. For tests, choose SQL regression, isolation, TAP, contrib, or `src/test/modules` according to the behavior surface [regress/GNUmakefile#check-targets](../../raw/postgres-12/src/test/regress/GNUmakefile#L125-L147) [isolation/Makefile#check-targets](../../raw/postgres-12/src/test/isolation/Makefile#L52-L66) [PostgresNode.pm#synopsis](../../raw/postgres-12/src/test/perl/PostgresNode.pm#L4-L80) [test/modules/Makefile#SUBDIRS](../../raw/postgres-12/src/test/modules/Makefile#L1-L26).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/log.md` recent entries, and `wiki/v12/index.md`.
- Source layout makefiles: `src/Makefile`, `src/backend/Makefile`, `src/backend/access/Makefile`, `src/backend/storage/Makefile`, `src/backend/optimizer/Makefile`, `src/backend/postmaster/Makefile`, `src/backend/replication/Makefile`, `src/backend/utils/misc/Makefile`, `src/bin/Makefile`, `src/interfaces/Makefile`, `contrib/Makefile`.
- Query path: `src/backend/tcop/postgres.c`, `src/backend/tcop/pquery.c`, `src/backend/tcop/utility.c`, `src/backend/commands/extension.c`, `src/backend/parser/parser.c`, `src/backend/parser/analyze.c`, `src/backend/rewrite/rewriteHandler.c`, `src/backend/optimizer/plan/planner.c`, `src/backend/optimizer/path/allpaths.c`, `src/backend/optimizer/plan/createplan.c`, `src/backend/executor/execMain.c`.
- Data structures and generated-file roots: `src/include/nodes/parsenodes.h`, `src/include/nodes/plannodes.h`, `src/include/nodes/pathnodes.h`, `src/include/nodes/execnodes.h`, `src/include/executor/execdesc.h`, `src/include/utils/rel.h`, `src/include/access/tableam.h`, `src/include/access/amapi.h`, `src/include/nodes/memnodes.h`, `src/backend/utils/mmgr/README`, `src/include/catalog/genbki.h`, `src/include/catalog/indexing.h`, `src/backend/catalog/Makefile`, `src/backend/catalog/genbki.pl`, `src/backend/utils/Makefile`, `src/backend/storage/lmgr/Makefile`, `src/include/Makefile`.
- Test and documentation surfaces: `src/test/regress/GNUmakefile`, `src/test/isolation/Makefile`, `src/test/perl/PostgresNode.pm`, `src/test/modules/Makefile`, `doc/src/sgml/filelist.sgml`, `doc/src/sgml/sources.sgml`, `src/backend/utils/errcodes.txt`, representative contrib makefiles.

## Evidence Map

| Claim area | Primary evidence |
|---|---|
| Codebase layout | `src/Makefile`, `src/backend/Makefile`, subsystem makefiles, `contrib/Makefile`, `src/bin/Makefile`, `src/interfaces/Makefile`. |
| SQL simple-query path | `exec_simple_query()`, `pg_parse_query()`, `pg_analyze_and_rewrite()`, `pg_plan_queries()`, portal setup, `PortalRun()`, `ExecutorStart()`, `ExecutorRun()`, `ExecutorFinish()`. |
| SQL extended-query path | `PostgresMain()` Parse/Bind/Execute dispatch, `exec_parse_message()`, `exec_bind_message()`, `exec_execute_message()`, `CachedPlanSource`, `GetCachedPlan()`, portal setup, and `PortalRun()`. |
| Utility/DDL path | `pg_plan_queries()` utility wrapper, `ProcessUtility()`, `standard_ProcessUtility()` direct cases for `VACUUM`, `EXPLAIN`, and `SET`, and `ProcessUtilitySlow()` event-trigger-supported DDL dispatch for CREATE/ALTER/INDEX. |
| Planner internals | `planner_hook`, `standard_planner()`, `subquery_planner()`, `make_one_rel()`, `create_plan_recurse()`, plus `Query`, `PlannerGlobal`, `RelOptInfo`, `Path`, and `PlannedStmt`. |
| Storage and AM boundaries | `RelationData`, `TableAmRoutine`, `IndexAmRoutine`, heap scan page-walk code, B-tree handler callback registration. |
| Contrib and extension boundary | Contrib makefiles plus `extension.c` control-file parsing, `CREATE EXTENSION` version/script selection, `pg_extension` insertion, and extension script execution. |
| Generated files | backend generated-header rules, parser makefile, catalog generation makefile, `genbki.h`, `genbki.pl`, `indexing.h`, utils generated-header rules, LWLock name generation, include install rules. |
| Tests and docs | `pg_regress` makefile targets, isolation makefile targets, `PostgresNode.pm`, `src/test/modules`, SGML file list, coding-conventions and error-reporting docs. |

## Open Questions

None for this navigation-scope document. The guide is intentionally broad; subsystem-specific pages should still perform a fresh caller/callee, struct, macro, test, doc, error-path, and generated-header review for the exact behavior under investigation.

## Source References

- [src/Makefile#SUBDIRS](../../raw/postgres-12/src/Makefile#L15-L32)
- [backend/Makefile#SUBDIRS](../../raw/postgres-12/src/backend/Makefile#L20-L24)
- [backend/Makefile#generated-headers](../../raw/postgres-12/src/backend/Makefile#L123-L190)
- [postgres.c#exec_simple_query](../../raw/postgres-12/src/backend/tcop/postgres.c#L980-L1222)
- [postgres.c#PostgresMain-extended-protocol](../../raw/postgres-12/src/backend/tcop/postgres.c#L4268-L4309)
- [postgres.c#exec_parse_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1314-L1498)
- [postgres.c#exec_bind_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1576-L1898)
- [postgres.c#exec_execute_message](../../raw/postgres-12/src/backend/tcop/postgres.c#L1946-L2096)
- [postgres.c#pg_parse_query](../../raw/postgres-12/src/backend/tcop/postgres.c#L620-L642)
- [postgres.c#pg_plan_queries](../../raw/postgres-12/src/backend/tcop/postgres.c#L937-L975)
- [pquery.c#PortalRun](../../raw/postgres-12/src/backend/tcop/pquery.c#L686-L850)
- [execMain.c#ExecutorStart-Run-Finish](../../raw/postgres-12/src/backend/executor/execMain.c#L130-L447)
- [utility.c#ProcessUtility](../../raw/postgres-12/src/backend/tcop/utility.c#L338-L363)
- [utility.c#standard_ProcessUtility-Explain-Set](../../raw/postgres-12/src/backend/tcop/utility.c#L674-L686)
- [utility.c#ProcessUtilitySlow](../../raw/postgres-12/src/backend/tcop/utility.c#L948-L1395)
- [extension.c#extension-overview](../../raw/postgres-12/src/backend/commands/extension.c#L6-L13)
- [extension.c#parse_extension_control_file](../../raw/postgres-12/src/backend/commands/extension.c#L468-L624)
- [extension.c#execute_sql_string](../../raw/postgres-12/src/backend/commands/extension.c#L698-L779)
- [extension.c#CreateExtensionInternal](../../raw/postgres-12/src/backend/commands/extension.c#L1262-L1541)
- [parser.c#raw_parser](../../raw/postgres-12/src/backend/parser/parser.c#L28-L62)
- [analyze.c#parse_analyze](../../raw/postgres-12/src/backend/parser/analyze.c#L90-L123)
- [rewriteHandler.c#QueryRewrite](../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L3923-L3963)
- [planner.c#planner](../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L270-L350)
- [allpaths.c#make_one_rel](../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L179-L234)
- [createplan.c#create_plan_recurse](../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L320-L430)
- [parsenodes.h#RawStmt](../../raw/postgres-12/src/include/nodes/parsenodes.h#L1470-L1488)
- [parsenodes.h#Query](../../raw/postgres-12/src/include/nodes/parsenodes.h#L97-L150)
- [plannodes.h#PlannedStmt](../../raw/postgres-12/src/include/nodes/plannodes.h#L42-L95)
- [pathnodes.h#RelOptInfo](../../raw/postgres-12/src/include/nodes/pathnodes.h#L632-L723)
- [pathnodes.h#Path](../../raw/postgres-12/src/include/nodes/pathnodes.h#L1104-L1126)
- [execdesc.h#QueryDesc](../../raw/postgres-12/src/include/executor/execdesc.h#L22-L56)
- [execnodes.h#EState](../../raw/postgres-12/src/include/nodes/execnodes.h#L496-L590)
- [execnodes.h#PlanState](../../raw/postgres-12/src/include/nodes/execnodes.h#L936-L1018)
- [rel.h#RelationData](../../raw/postgres-12/src/include/utils/rel.h#L53-L180)
- [tableam.h#TableAmRoutine](../../raw/postgres-12/src/include/access/tableam.h#L162-L245)
- [amapi.h#IndexAmRoutine](../../raw/postgres-12/src/include/access/amapi.h#L159-L233)
- [catalog/Makefile#catalog-generation](../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90)
- [genbki.pl#usage](../../raw/postgres-12/src/backend/catalog/genbki.pl#L960-L974)
- [regress/GNUmakefile#check-targets](../../raw/postgres-12/src/test/regress/GNUmakefile#L125-L147)
- [isolation/Makefile#check-targets](../../raw/postgres-12/src/test/isolation/Makefile#L52-L66)

## Navigation

- [PostgreSQL 12.2 index](index.md)
- [Wiki index](../index.md)
- [Versions](../versions.md)
