---
type: codebase-navigation-guide
version: 19
pinned_commit: e18b0cb7344cb4bd28468f6c0aeeb9b9241d30aa
verified: false
verified_by_agent: not yet
---

# PostgreSQL 19 Codebase Navigation Guide (unverified)

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

Create a codebase navigation guide document for PostgreSQL 19.

Prompt note: this is the canonical mandatory guide prompt for `type: codebase-navigation-guide`.

## Answer

Use the PostgreSQL 19 checkout as a set of subsystem maps. The top-level `src/Makefile` names the major source families, and the backend makefile narrows server code into backend subsystems, helper libraries, generated headers, and the `postgres` target [src/Makefile#SUBDIRS](../../raw/postgres-19/src/Makefile#L15-L36) [backend/Makefile#SUBDIRS](../../raw/postgres-19/src/backend/Makefile#L19-L31) [backend/Makefile#all](../../raw/postgres-19/src/backend/Makefile#L84-L84).

### Fast Map

| Need | Start here | Evidence |
|---|---|---|
| Server subsystems | `src/backend/` | `src/backend/Makefile` is the short ownership map for backend directories and build-time generated headers [backend/Makefile#SUBDIRS](../../raw/postgres-19/src/backend/Makefile#L19-L31). |
| Access methods | `src/backend/access/` | The access makefile separates access-method and transaction subdirectories under `src/backend/access` [access/Makefile#SUBDIRS](../../raw/postgres-19/src/backend/access/Makefile#L11-L28). |
| Storage and locks | `src/backend/storage/` | Storage is split by the storage makefile into directories such as buffer, file, IPC, lock-manager, page, smgr, and sync code [storage/Makefile#SUBDIRS](../../raw/postgres-19/src/backend/storage/Makefile#L11-L27). |
| Client programs | `src/bin/` | Client and maintenance programs are listed under `src/bin`, with Windows `pgevent` added conditionally [bin/Makefile#SUBDIRS](../../raw/postgres-19/src/bin/Makefile#L16-L40). |
| Frontend APIs | `src/interfaces/` | PostgreSQL 19 builds `libpq` and `ecpg`, and conditionally adds `libpq-oauth`, from `src/interfaces` [interfaces/Makefile#SUBDIRS](../../raw/postgres-19/src/interfaces/Makefile#L15-L18). |
| Extensions | `contrib/` | The contrib makefile lists built contrib modules and adds platform/language-conditional modules such as `pgcrypto`, `uuid-ossp`, `xml2`, `sepgsql`, and PL transform modules [contrib/Makefile#SUBDIRS](../../raw/postgres-19/contrib/Makefile#L7-L89). |

### SQL Statement Path

1. For the simple-query protocol, start in `exec_simple_query()`. It parses the query string with `pg_parse_query()`, then proceeds through analysis, rewrite, planning, portal setup, and execution for each raw statement [postgres.c#exec_simple_query](../../raw/postgres-19/src/backend/tcop/postgres.c#L1029-L1225) [postgres.c#pg_parse_query](../../raw/postgres-19/src/backend/tcop/postgres.c#L616-L645).
2. Parse analysis and rewrite live behind the `pg_analyze_and_rewrite_*()` wrappers and `pg_rewrite_query()`. The normal statement path reaches `QueryRewrite()` before planning [postgres.c#pg_analyze_and_rewrite](../../raw/postgres-19/src/backend/tcop/postgres.c#L682-L851) [rewriteHandler.c#QueryRewrite](../../raw/postgres-19/src/backend/rewrite/rewriteHandler.c#L4781-L4835).
3. Planning begins at `pg_plan_queries()`. Normal queries enter `planner()`, which either uses `planner_hook` or `standard_planner()`, and `subquery_planner()` builds the per-query planner state [postgres.c#pg_plan_queries](../../raw/postgres-19/src/backend/tcop/postgres.c#L987-L1025) [planner.c#planner](../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L333-L375) [planner.c#subquery_planner](../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L775-L850).
4. Execution is portal-driven. `PortalRun()` dispatches portal execution, and the executor entry points are `ExecutorStart()` and `ExecutorRun()` with hook-aware standard implementations [pquery.c#PortalRun](../../raw/postgres-19/src/backend/tcop/pquery.c#L681-L850) [execMain.c#ExecutorStart-Run](../../raw/postgres-19/src/backend/executor/execMain.c#L124-L345).
5. Utility commands are routed outside normal plan execution. `ProcessUtility()` exposes `ProcessUtility_hook`; without a hook it calls `standard_ProcessUtility()` [utility.c#ProcessUtility](../../raw/postgres-19/src/backend/tcop/utility.c#L504-L600).

### Generated And Catalog Files

Treat generated headers and catalog artifacts as part of the navigation surface. The backend `generated-headers` target depends on LWLock names, catalog headers, node headers, utility headers, and `parser/gram.h` [backend/Makefile#generated-headers](../../raw/postgres-19/src/backend/Makefile#L183-L188). Catalog headers generate through `src/include/catalog/Makefile`, and `pg_index.h` shows the catalog-header pattern with `CATALOG()` plus generated index declarations [catalog/Makefile#generated-headers](../../raw/postgres-19/src/include/catalog/Makefile#L132-L136) [pg_index.h#CATALOG](../../raw/postgres-19/src/include/catalog/pg_index.h#L31-L78).

### Key Data Structures

Keep these structs open when tracing behavior:

| Structure | Why it matters |
|---|---|
| `RawStmt` and `Query` | Raw parse trees become analyzed `Query` trees before rewrite and planning [parsenodes.h#Query](../../raw/postgres-19/src/include/nodes/parsenodes.h#L117-L170) [parsenodes.h#RawStmt](../../raw/postgres-19/src/include/nodes/parsenodes.h#L2187-L2210). |
| `PlannedStmt` | Planner output and utility wrappers are represented as `PlannedStmt` nodes [plannodes.h#PlannedStmt](../../raw/postgres-19/src/include/nodes/plannodes.h#L59-L135). |
| `PlannerGlobal`, `RelOptInfo`, and `Path` | These are the planner's global state, relation state, and alternative access/join path records [pathnodes.h#PlannerGlobal](../../raw/postgres-19/src/include/nodes/pathnodes.h#L168-L245) [pathnodes.h#RelOptInfo](../../raw/postgres-19/src/include/nodes/pathnodes.h#L1009-L1095) [pathnodes.h#Path](../../raw/postgres-19/src/include/nodes/pathnodes.h#L1964-L2005). |
| `QueryDesc`, `EState`, and `PlanState` | Executor entry points receive a `QueryDesc`, build query-wide `EState`, and execute a `PlanState` tree [execdesc.h#QueryDesc](../../raw/postgres-19/src/include/executor/execdesc.h#L33-L80) [execnodes.h#EState](../../raw/postgres-19/src/include/nodes/execnodes.h#L690-L785) [execnodes.h#PlanState](../../raw/postgres-19/src/include/nodes/execnodes.h#L1195-L1265). |
| `RelationData`, `TableAmRoutine`, and `IndexAmRoutine` | Relation cache state points at table and index access-method callback tables [rel.h#RelationData](../../raw/postgres-19/src/include/utils/rel.h#L55-L135) [tableam.h#TableAmRoutine](../../raw/postgres-19/src/include/access/tableam.h#L321-L420) [amapi.h#IndexAmRoutine](../../raw/postgres-19/src/include/access/amapi.h#L233-L320). |
| `MemoryContextData` | Memory contexts form the allocation tree used by backend code [memnodes.h#MemoryContextData](../../raw/postgres-19/src/include/nodes/memnodes.h#L117-L170). |

### Tests And Docs

Use `src/test/Makefile` to choose between Perl/TAP, postmaster, regression, isolation, modules, authentication, recovery, and subscription test areas [test/Makefile#SUBDIRS](../../raw/postgres-19/src/test/Makefile#L15-L35). SQL regression tests build `pg_regress` and expose `check`/`installcheck` targets, while isolation tests build `isolationtester` and `pg_isolation_regress` [regress/GNUmakefile#targets](../../raw/postgres-19/src/test/regress/GNUmakefile#L36-L103) [isolation/Makefile#targets](../../raw/postgres-19/src/test/isolation/Makefile#L21-L64). Test-only extension modules live under `src/test/modules` [test/modules/Makefile#SUBDIRS](../../raw/postgres-19/src/test/modules/Makefile#L7-L85). Developer docs in `doc/src/sgml/filelist.sgml` include BKI, catalogs, table AM, index AM, source-layout, storage, and contrib docs [filelist.sgml#developer-docs](../../raw/postgres-19/doc/src/sgml/filelist.sgml#L87-L120).

### Navigation Checklist

1. Confirm the target version and pin before reading source.
2. Use makefiles to find ownership before following call graphs.
3. For SQL behavior, separate the normal query path from `ProcessUtility()`.
4. Keep the relevant node structs open while reading planner or executor code.
5. Check generated-header, catalog, and test surfaces before drafting behavioral claims.

## Context Reviewed

- Version navigation: `wiki/versions.md`, `wiki/index.md`, recent `wiki/log.md`, and `wiki/v19/index.md`.
- Source layout: top-level, backend, access, storage, bin, interfaces, contrib, and test makefiles.
- Query path: `postgres.c`, `pquery.c`, `utility.c`, `rewriteHandler.c`, `planner.c`, and `execMain.c`.
- Data structures: parser, planner, executor, relation cache, table AM, index AM, and memory-context headers.
- Generated and test surfaces: backend generated-header rules, catalog generation rules, catalog headers, regression tests, isolation tests, test modules, and SGML file lists.

## Evidence Map

| Claim area | Primary evidence |
|---|---|
| Source layout | `src/Makefile`, `src/backend/Makefile`, subsystem makefiles, `contrib/Makefile`. |
| Normal SQL path | `exec_simple_query()`, `pg_parse_query()`, `pg_analyze_and_rewrite_*()`, `pg_rewrite_query()`, `pg_plan_queries()`, `PortalRun()`, executor entry points. |
| Utility path | `ProcessUtility()` and `standard_ProcessUtility()`. |
| Generated/catalog artifacts | backend generated-header target, catalog generated-header target, `pg_index.h`. |
| Key structs | `RawStmt`, `Query`, `PlannedStmt`, planner structs, executor structs, `RelationData`, AM routines, memory contexts. |
| Tests/docs | regression, isolation, test modules, and SGML file-list entries. |

## Open Questions

None for this navigation-scope page. Subsystem pages still need fresh caller/callee, struct, macro, generated-file, error-path, and test review for their exact behavior.

## Source References

- [src/Makefile#SUBDIRS](../../raw/postgres-19/src/Makefile#L15-L36)
- [backend/Makefile#SUBDIRS](../../raw/postgres-19/src/backend/Makefile#L19-L31)
- [postgres.c#exec_simple_query](../../raw/postgres-19/src/backend/tcop/postgres.c#L1029-L1225)
- [postgres.c#pg_plan_queries](../../raw/postgres-19/src/backend/tcop/postgres.c#L987-L1025)
- [pquery.c#PortalRun](../../raw/postgres-19/src/backend/tcop/pquery.c#L681-L850)
- [utility.c#ProcessUtility](../../raw/postgres-19/src/backend/tcop/utility.c#L504-L600)
- [planner.c#planner](../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L333-L375)
- [planner.c#subquery_planner](../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L775-L850)
- [execMain.c#ExecutorStart-Run](../../raw/postgres-19/src/backend/executor/execMain.c#L124-L345)
- [backend/Makefile#generated-headers](../../raw/postgres-19/src/backend/Makefile#L183-L188)
- [pg_index.h#CATALOG](../../raw/postgres-19/src/include/catalog/pg_index.h#L31-L78)
- [parsenodes.h#RawStmt](../../raw/postgres-19/src/include/nodes/parsenodes.h#L2187-L2210)
- [pathnodes.h#RelOptInfo](../../raw/postgres-19/src/include/nodes/pathnodes.h#L1009-L1095)
- [execnodes.h#PlanState](../../raw/postgres-19/src/include/nodes/execnodes.h#L1195-L1265)
- [tableam.h#TableAmRoutine](../../raw/postgres-19/src/include/access/tableam.h#L321-L420)
- [amapi.h#IndexAmRoutine](../../raw/postgres-19/src/include/access/amapi.h#L233-L320)
- [regress/GNUmakefile#targets](../../raw/postgres-19/src/test/regress/GNUmakefile#L36-L103)
- [isolation/Makefile#targets](../../raw/postgres-19/src/test/isolation/Makefile#L21-L64)

## Navigation

- [PostgreSQL 19 index](index.md)
- [Wiki index](../index.md)
- [Versions](../versions.md)
