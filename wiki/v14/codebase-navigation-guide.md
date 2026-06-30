---
type: codebase-navigation-guide
version: 14
pinned_commit: 5c00f4e2e3bcee6931ae93429d53f7c2a4f46156
verified: false
verified_by_agent: not yet
---

# PostgreSQL 14 Codebase Navigation Guide (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Fast Map](#fast-map)
  - [SQL Statement Path](#sql-statement-path)
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

Create a codebase navigation guide document for PostgreSQL 14.

Prompt note: this is the canonical mandatory guide prompt for `type: codebase-navigation-guide`.

## Answer

Use the PostgreSQL 14 checkout as a set of subsystem maps. The top-level `src/Makefile` names the major source families, and the backend makefile narrows server code into `access`, `bootstrap`, `catalog`, `parser`, `commands`, `executor`, `foreign`, `nodes`, `optimizer`, `partitioning`, `postmaster`, `replication`, `rewrite`, `statistics`, `storage`, `tcop`, `tsearch`, `utils`, and `jit` [src/Makefile#SUBDIRS](../../raw/postgres-14/src/Makefile#L15-L32) [backend/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/Makefile#L20-L24).

### Fast Map

| Need | Start here | Evidence |
|---|---|---|
| Server subsystems | `src/backend/` | `src/backend/Makefile` is the short ownership map for parser, planner, executor, storage, replication, commands, and utility code [backend/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/Makefile#L20-L24). |
| Access methods | `src/backend/access/` | The access makefile separates BRIN, common AM code, GIN, GiST, hash, heap, index, B-tree, resource-manager descriptions, SP-GiST, table AM, table sampling, and transaction code [access/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/access/Makefile#L11-L12). |
| Storage and locks | `src/backend/storage/` | Storage is split into buffer, file, freespace, IPC, large-object, lock-manager, page, smgr, and sync directories [storage/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/storage/Makefile#L11). |
| Client programs | `src/bin/` | Client and maintenance programs are listed under `src/bin`, including the v14-new `pg_amcheck`, with Windows `pgevent` added conditionally [bin/Makefile#SUBDIRS](../../raw/postgres-14/src/bin/Makefile#L16-L41). |
| Frontend APIs | `src/interfaces/` | PostgreSQL 14 builds `libpq` and `ecpg` from `src/interfaces` [interfaces/Makefile#SUBDIRS](../../raw/postgres-14/src/interfaces/Makefile#L15). |
| Extensions | `contrib/` | The contrib makefile lists built contrib modules and adds platform/language-conditional modules such as `sslinfo`, `uuid-ossp`, `xml2`, `sepgsql`, and PL transform modules [contrib/Makefile#SUBDIRS](../../raw/postgres-14/contrib/Makefile#L7-L88). |

### SQL Statement Path

1. For the simple-query protocol, start in `exec_simple_query()`. It parses the query string with `pg_parse_query()`, then runs each raw statement through analysis, rewrite, planning, portal setup, and execution [postgres.c#exec_simple_query](../../raw/postgres-14/src/backend/tcop/postgres.c#L923-L1010) [postgres.c#pg_parse_query](../../raw/postgres-14/src/backend/tcop/postgres.c#L596-L633).
2. Parse analysis and rewrite live behind `pg_analyze_and_rewrite()`, which calls `pg_rewrite_query()`. The normal statement path reaches `QueryRewrite()` before planning [postgres.c#pg_analyze_and_rewrite](../../raw/postgres-14/src/backend/tcop/postgres.c#L644-L674) [rewriteHandler.c#QueryRewrite](../../raw/postgres-14/src/backend/rewrite/rewriteHandler.c#L4365-L4455).
3. Planning begins at `pg_plan_queries()`. Normal queries enter `planner()`, which either uses `planner_hook` or `standard_planner()`, and `subquery_planner()` builds the per-query planner state [postgres.c#pg_plan_queries](../../raw/postgres-14/src/backend/tcop/postgres.c#L882-L915) [planner.c#planner](../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L263-L274) [planner.c#subquery_planner](../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L590-L640).
4. Execution is portal-driven. `PortalRun()` dispatches portal execution, and the executor entry points are `ExecutorStart()` and `ExecutorRun()` with hook-aware standard implementations [pquery.c#PortalRun](../../raw/postgres-14/src/backend/tcop/pquery.c#L682-L720) [execMain.c#ExecutorStart](../../raw/postgres-14/src/backend/executor/execMain.c#L127-L144) [execMain.c#ExecutorRun](../../raw/postgres-14/src/backend/executor/execMain.c#L300-L309).
5. Utility commands are routed outside normal plan execution. `ProcessUtility()` exposes `ProcessUtility_hook`; without a hook it calls `standard_ProcessUtility()` [utility.c#ProcessUtility](../../raw/postgres-14/src/backend/tcop/utility.c#L502-L530).

### Generated And Catalog Files

Treat generated headers and catalog artifacts as part of the navigation surface. The backend `generated-headers` target depends on `parser/gram.h`, the generated LWLock-names header, and the catalog and utility header submakes [backend/Makefile#generated-headers](../../raw/postgres-14/src/backend/Makefile#L163-L165). In PostgreSQL 14 the catalog headers are generated by the backend catalog makefile's `generated-header-symlinks` target, which drives `genbki.pl` over the `pg_*.h`/`pg_*.dat` inputs [backend/catalog/Makefile#generated-header-symlinks](../../raw/postgres-14/src/backend/catalog/Makefile#L88-L94). Each catalog header follows the `CATALOG()` pattern; `pg_index.h` shows the macro plus generated index declarations [pg_index.h#CATALOG](../../raw/postgres-14/src/include/catalog/pg_index.h#L29-L62).

### Key Data Structures

Keep these structs open when tracing behavior:

| Structure | Why it matters |
|---|---|
| `RawStmt` and `Query` | Raw parse trees become analyzed `Query` trees before rewrite and planning [parsenodes.h#Query](../../raw/postgres-14/src/include/nodes/parsenodes.h#L116-L194) [parsenodes.h#RawStmt](../../raw/postgres-14/src/include/nodes/parsenodes.h#L1556-L1562). |
| `PlannedStmt` | Planner output and utility wrappers are represented as `PlannedStmt` nodes [plannodes.h#PlannedStmt](../../raw/postgres-14/src/include/nodes/plannodes.h#L42-L91). |
| `PlannerGlobal`, `RelOptInfo`, and `Path` | These are the planner's global state, relation state, and alternative access/join path records [pathnodes.h#PlannerGlobal](../../raw/postgres-14/src/include/nodes/pathnodes.h#L89-L132) [pathnodes.h#RelOptInfo](../../raw/postgres-14/src/include/nodes/pathnodes.h#L674-L773) [pathnodes.h#Path](../../raw/postgres-14/src/include/nodes/pathnodes.h#L1174-L1196). |
| `QueryDesc`, `EState`, and `PlanState` | Executor entry points receive a `QueryDesc`, build query-wide `EState`, and execute a `PlanState` tree [execdesc.h#QueryDesc](../../raw/postgres-14/src/include/executor/execdesc.h#L33-L56) [execnodes.h#EState](../../raw/postgres-14/src/include/nodes/execnodes.h#L566-L672) [execnodes.h#PlanState](../../raw/postgres-14/src/include/nodes/execnodes.h#L986-L1076). |
| `RelationData`, `TableAmRoutine`, and `IndexAmRoutine` | Relation cache state points at table and index access-method callback tables [rel.h#RelationData](../../raw/postgres-14/src/include/utils/rel.h#L54-L135) [tableam.h#TableAmRoutine](../../raw/postgres-14/src/include/access/tableam.h#L265-L320) [amapi.h#IndexAmRoutine](../../raw/postgres-14/src/include/access/amapi.h#L210-L287). |
| `MemoryContextData` | Memory contexts form the allocation tree used by backend code [memnodes.h#MemoryContextData](../../raw/postgres-14/src/include/nodes/memnodes.h#L78-L93). |

### Tests And Docs

Use `src/test/Makefile` to choose between Perl/TAP, postmaster, regression, isolation, modules, authentication, recovery, and subscription test areas [test/Makefile#SUBDIRS](../../raw/postgres-14/src/test/Makefile#L15). SQL regression tests build `pg_regress` and expose `check`/`installcheck` targets, while isolation tests build `isolationtester` and `pg_isolation_regress` and expose their own `check`/`installcheck` targets [regress/GNUmakefile#pg_regress](../../raw/postgres-14/src/test/regress/GNUmakefile#L37-L40) [regress/GNUmakefile#check](../../raw/postgres-14/src/test/regress/GNUmakefile#L125-L132) [isolation/Makefile#targets](../../raw/postgres-14/src/test/isolation/Makefile#L20-L64). Test-only extension modules live under `src/test/modules` [test/modules/Makefile#SUBDIRS](../../raw/postgres-14/src/test/modules/Makefile#L7-L39). Developer docs in `doc/src/sgml/filelist.sgml` include BKI, catalogs, table AM, index AM, source-layout, storage, and contrib docs [filelist.sgml#developer-docs](../../raw/postgres-14/doc/src/sgml/filelist.sgml#L82-L110).

### Navigation Checklist

1. Confirm the target version and pin before reading source.
2. Use makefiles to find ownership before following call graphs.
3. For SQL behavior, separate the normal query path from `ProcessUtility()`.
4. Keep the relevant node structs open while reading planner or executor code.
5. Check generated-header, catalog, and test surfaces before drafting behavioral claims.

## Context Reviewed

- Version navigation: `wiki/versions.md`, `wiki/index.md`, recent `wiki/log.md`, and `wiki/v14/index.md`.
- Source layout: top-level, backend, access, storage, bin, interfaces, contrib, and test makefiles.
- Query path: `postgres.c`, `pquery.c`, `utility.c`, `rewriteHandler.c`, `planner.c`, and `execMain.c`.
- Data structures: parser, planner, executor, relation cache, table AM, index AM, and memory-context headers.
- Generated and test surfaces: backend generated-header rules, the backend catalog header-generation target, the `pg_index.h` catalog header, regression tests, isolation tests, test modules, and SGML file lists.

## Evidence Map

| Claim area | Primary evidence |
|---|---|
| Source layout | `src/Makefile`, `src/backend/Makefile`, subsystem makefiles, `contrib/Makefile`. |
| Normal SQL path | `exec_simple_query()`, `pg_parse_query()`, `pg_analyze_and_rewrite()`, `pg_rewrite_query()`/`QueryRewrite()`, `pg_plan_queries()`, `PortalRun()`, executor entry points. |
| Utility path | `ProcessUtility()` and `standard_ProcessUtility()`. |
| Generated/catalog artifacts | backend generated-header target, backend catalog `generated-header-symlinks` target, `pg_index.h`. |
| Key structs | `RawStmt`, `Query`, `PlannedStmt`, planner structs, executor structs, `RelationData`, AM routines, memory contexts. |
| Tests/docs | regression, isolation, test modules, and SGML file-list entries. |

## Open Questions

None for this navigation-scope page. Subsystem pages still need fresh caller/callee, struct, macro, generated-file, error-path, and test review for their exact behavior.

## Source References

- [src/Makefile#SUBDIRS](../../raw/postgres-14/src/Makefile#L15-L32)
- [backend/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/Makefile#L20-L24)
- [access/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/access/Makefile#L11-L12)
- [storage/Makefile#SUBDIRS](../../raw/postgres-14/src/backend/storage/Makefile#L11)
- [bin/Makefile#SUBDIRS](../../raw/postgres-14/src/bin/Makefile#L16-L41)
- [interfaces/Makefile#SUBDIRS](../../raw/postgres-14/src/interfaces/Makefile#L15)
- [contrib/Makefile#SUBDIRS](../../raw/postgres-14/contrib/Makefile#L7-L88)
- [postgres.c#exec_simple_query](../../raw/postgres-14/src/backend/tcop/postgres.c#L923-L1010)
- [postgres.c#pg_parse_query](../../raw/postgres-14/src/backend/tcop/postgres.c#L596-L633)
- [postgres.c#pg_analyze_and_rewrite](../../raw/postgres-14/src/backend/tcop/postgres.c#L644-L674)
- [postgres.c#pg_plan_queries](../../raw/postgres-14/src/backend/tcop/postgres.c#L882-L915)
- [rewriteHandler.c#QueryRewrite](../../raw/postgres-14/src/backend/rewrite/rewriteHandler.c#L4365-L4455)
- [planner.c#planner](../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L263-L274)
- [planner.c#subquery_planner](../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L590-L640)
- [pquery.c#PortalRun](../../raw/postgres-14/src/backend/tcop/pquery.c#L682-L720)
- [execMain.c#ExecutorStart](../../raw/postgres-14/src/backend/executor/execMain.c#L127-L144)
- [execMain.c#ExecutorRun](../../raw/postgres-14/src/backend/executor/execMain.c#L300-L309)
- [utility.c#ProcessUtility](../../raw/postgres-14/src/backend/tcop/utility.c#L502-L530)
- [backend/Makefile#generated-headers](../../raw/postgres-14/src/backend/Makefile#L163-L165)
- [backend/catalog/Makefile#generated-header-symlinks](../../raw/postgres-14/src/backend/catalog/Makefile#L88-L94)
- [pg_index.h#CATALOG](../../raw/postgres-14/src/include/catalog/pg_index.h#L29-L62)
- [parsenodes.h#Query](../../raw/postgres-14/src/include/nodes/parsenodes.h#L116-L194)
- [parsenodes.h#RawStmt](../../raw/postgres-14/src/include/nodes/parsenodes.h#L1556-L1562)
- [plannodes.h#PlannedStmt](../../raw/postgres-14/src/include/nodes/plannodes.h#L42-L91)
- [pathnodes.h#RelOptInfo](../../raw/postgres-14/src/include/nodes/pathnodes.h#L674-L773)
- [execnodes.h#PlanState](../../raw/postgres-14/src/include/nodes/execnodes.h#L986-L1076)
- [rel.h#RelationData](../../raw/postgres-14/src/include/utils/rel.h#L54-L135)
- [tableam.h#TableAmRoutine](../../raw/postgres-14/src/include/access/tableam.h#L265-L320)
- [amapi.h#IndexAmRoutine](../../raw/postgres-14/src/include/access/amapi.h#L210-L287)
- [regress/GNUmakefile#check](../../raw/postgres-14/src/test/regress/GNUmakefile#L125-L132)
- [isolation/Makefile#targets](../../raw/postgres-14/src/test/isolation/Makefile#L20-L64)
- [filelist.sgml#developer-docs](../../raw/postgres-14/doc/src/sgml/filelist.sgml#L82-L110)

## Navigation

- [PostgreSQL 14.23 index](index.md)
- [Wiki index](../index.md)
- [Versions](../versions.md)
