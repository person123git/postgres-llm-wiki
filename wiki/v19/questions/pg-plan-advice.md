---
type: question
version: 19
pinned_commit: 01c544e1afb99bc2a76803870010b7cd2907f3b5
verified: false
verified_by_agent: not yet
---

# How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
- [Module Layout](#module-layout)
- [Core Planner Changes (the mechanism)](#core-planner-changes-the-mechanism)
  - [1. The strategy mask `pgs_mask`](#1-the-strategy-mask-pgsmask)
  - [2. Five new planner hooks](#2-five-new-planner-hooks)
  - [3. Per-object extension state](#3-per-object-extension-state)
  - [4. Per-index disabling](#4-per-index-disabling)
- [Module Initialization](#module-initialization)
- [The Advice Language](#the-advice-language)
  - [Relation identifiers](#relation-identifiers)
  - [What each tag family means](#what-each-tag-family-means)
- [Generating Advice (plan -> string)](#generating-advice-plan---string)
- [Enforcing Advice (string -> plan)](#enforcing-advice-string---plan)
- [Feedback and EXPLAIN Output](#feedback-and-explain-output)
- [GUCs](#gucs)
- [Prepared Statements and Plan Caching](#prepared-statements-and-plan-caching)
  - [Enforcement is frozen into the cached plan](#enforcement-is-frozen-into-the-cached-plan)
  - [Generated advice and feedback are usually absent for EXECUTE](#generated-advice-and-feedback-are-usually-absent-for-execute)
- [Round-Trip Testing](#round-trip-testing)
- [Source Commit History](#source-commit-history)
  - [Core planner enabling and fix commits](#core-planner-enabling-and-fix-commits)
  - [`5883ff30` — Add pg_plan_advice contrib module (2026-03-12, Robert Haas)](#5883ff30--add-pgplanadvice-contrib-module-2026-03-12-robert-haas)
  - [`be43c48c` — Initialize variable to placate compiler (2026-03-13, Nathan Bossart; patch by Sami Imseih)](#be43c48c--initialize-variable-to-placate-compiler-2026-03-13-nathan-bossart-patch-by-sami-imseih)
  - [`4f888d0f` — Fix whitespace (2026-03-16, Peter Eisentraut)](#4f888d0f--fix-whitespace-2026-03-16-peter-eisentraut)
  - [`5e72ce24` — Fix failures to accept identifier keywords (2026-03-16, Robert Haas; author Lukas Fittl)](#5e72ce24--fix-failures-to-accept-identifier-keywords-2026-03-16-robert-haas-author-lukas-fittl)
  - [`7560995a` — Fix variable type confusion (2026-03-17, Robert Haas)](#7560995a--fix-variable-type-confusion-2026-03-17-robert-haas)
  - [`59dcc19b` — Always install pg_plan_advice.h, and in the right place (2026-03-17, Robert Haas; author Zsolt Parragi)](#59dcc19b--always-install-pgplanadviceh-and-in-the-right-place-2026-03-17-robert-haas-author-zsolt-parragi)
  - [`01b02c0e` — Avoid a crash under GEQO (2026-03-17, Robert Haas)](#01b02c0e--avoid-a-crash-under-geqo-2026-03-17-robert-haas)
  - [`b335fe56` — Fix multiple copy-and-paste errors in test case (2026-03-18, Robert Haas; reported by Tom Lane)](#b335fe56--fix-multiple-copy-and-paste-errors-in-test-case-2026-03-18-robert-haas-reported-by-tom-lane)
  - [`5dcb15e8` — Refactor to invent pgpa_planner_info (2026-03-26, Robert Haas)](#5dcb15e8--refactor-to-invent-pgpaplannerinfo-2026-03-26-robert-haas)
  - [`6455e55b` — Invent DO_NOT_SCAN(relation_identifier) (2026-03-26, Robert Haas; reviewer Lukas Fittl)](#6455e55b--invent-donotscanrelationidentifier-2026-03-26-robert-haas-reviewer-lukas-fittl)
  - [`874da8b1` — pgindent (2026-03-26, Robert Haas; reported by Lukas Fittl)](#874da8b1--pgindent-2026-03-26-robert-haas-reported-by-lukas-fittl)
  - [`e2ee9523` — Avoid assertion failure with partitionwise aggregate (2026-03-30, Robert Haas; reported by Alexander Lakhin)](#e2ee9523--avoid-assertion-failure-with-partitionwise-aggregate-2026-03-30-robert-haas-reported-by-alexander-lakhin)
  - [`0442f1c9` — Add a guc_check_handler to the EXPLAIN extension mechanism (2026-04-06, Robert Haas)](#0442f1c9--add-a-guccheckhandler-to-the-explain-extension-mechanism-2026-04-06-robert-haas)
  - [`49ce4181` — Improve various new-to-v19 appendStringInfo calls (2026-04-13, David Rowley)](#49ce4181--improve-various-new-to-v19-appendstringinfo-calls-2026-04-13-david-rowley)
  - [`3311ccc3` — Handle non-repeatable TABLESAMPLE scans (2026-04-13, Robert Haas; reported by Alexander Lakhin)](#3311ccc3--handle-non-repeatable-tablesample-scans-2026-04-13-robert-haas-reported-by-alexander-lakhin)
  - [`1faf9dfa` — Add alternatives test to Makefile (2026-04-13, Robert Haas)](#1faf9dfa--add-alternatives-test-to-makefile-2026-04-13-robert-haas)
  - [`0f93ebb3` — Fix a bug when a subquery is pruned away entirely (2026-04-13, Robert Haas; reported by Alexander Lakhin)](#0f93ebb3--fix-a-bug-when-a-subquery-is-pruned-away-entirely-2026-04-13-robert-haas-reported-by-alexander-lakhin)
  - [`c644aca2` — Export feedback-related definitions (2026-04-13, Robert Haas)](#c644aca2--export-feedback-related-definitions-2026-04-13-robert-haas)
  - [`4321dcad` — Fix another unique-semijoin bug (2026-04-17, Robert Haas; reported by Alexander Lakhin)](#4321dcad--fix-another-unique-semijoin-bug-2026-04-17-robert-haas-reported-by-alexander-lakhin)
  - [`228a1f95` — pgindent (2026-04-17, Robert Haas; per buildfarm member koel)](#228a1f95--pgindent-2026-04-17-robert-haas-per-buildfarm-member-koel)
  - [`d3bba041` — Fix a set of typos and grammar issues across the tree (2026-04-21, Michael Paquier)](#d3bba041--fix-a-set-of-typos-and-grammar-issues-across-the-tree-2026-04-21-michael-paquier)
  - [`b1901e28` — DO_NOT_SCAN is a simple tag, not a generic one (2026-05-29, Robert Haas; reported by Nikita Kalinin)](#b1901e28--donotscan-is-a-simple-tag-not-a-generic-one-2026-05-29-robert-haas-reported-by-nikita-kalinin)
  - [`89f5f860cc5` — Don't generate FOREIGN_JOIN advice for a single relation (2026-07-02, Robert Haas; author Mahendra Singh Thalor)](#89f5f860cc5--dont-generate-foreignjoin-advice-for-a-single-relation-2026-07-02-robert-haas-author-mahendra-singh-thalor)
  - [`ea203d371de` — pgindent fix for commit 53e6f51ee (2026-07-03, Robert Haas)](#ea203d371de--pgindent-fix-for-commit-53e6f51ee-2026-07-03-robert-haas)
  - [`da8889ccd7e` — Use PG_MODULE_MAGIC_EXT in newly introduced modules (2026-07-06, Robert Haas; author Andreas Karlsson)](#da8889ccd7e--use-pgmodulemagicext-in-newly-introduced-modules-2026-07-06-robert-haas-author-andreas-karlsson)
  - [Test, documentation, and build-tooling support commits](#test-documentation-and-build-tooling-support-commits)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Source References](#source-references)
- [Open Questions](#open-questions)
- [Related Pages](#related-pages)

## Question

Follow AGENTS.md.
In PostgreSQL 19, do a comprehensive explanation of how `pg_plan_advice` works, explain all commits to the feature.

## Answer

`pg_plan_advice` is a new PostgreSQL 19 `contrib` module that implements a small "plan advice" language for controlling key planner decisions. It does two complementary things, and is designed so they "round-trip": it can read a finished plan and emit a textual advice string that describes the important choices that plan made, and it can take such an advice string and force a future planning cycle to make those same choices [README#Plan-Advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L3-L40). Goals are plan stability and deliberate experimentation; explicit non-goals are controlling every planner decision and forcing plans the optimizer rejects for non-cost reasons [README#goals](../../../raw/postgres-19/contrib/pg_plan_advice/README#L6-L17).

The module is mostly a front end. The main control mechanism lives in the core planner, which v19 extended for this purpose: a per-relation bitmask of allowed "path generation strategies" called `pgs_mask`, seeded from the `enable_*` GUCs, plus planner hooks that let an extension observe planning and clear bits in that mask [pathnodes.h#PGS-strategy-mask](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L25-L97) [planner.c#default_pgs_mask](../../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L493-L529). `pg_plan_advice` installs those hooks and, when advice applies to a relation or join, clears the bits for the strategies the advice forbids; the core path-generation code then skips or disables those strategies [pgpa_planner.c#install_hooks](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L180-L194) [pgpa_planner.c#apply_scan_advice](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1625-L1700). Index-specific advice has one additional core control surface: `pg_plan_advice` marks nonmatching `IndexOptInfo` entries disabled, and core index-path creation converts that into disabled index paths [pgpa_planner.c#index-disable](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1814-L1818) [pathnodes.h#IndexOptInfo.disabled](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L1427-L1428) [pathnode.c#index-disabled](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L1126-L1131).

A guiding rule is that advice only ever *removes* a `pgs_mask` strategy; it never sets a bit the `enable_*` GUCs left off [pgpa_planner.c#joinrel-clear-bits](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L900-L933) [pgpa_planner.c#join-path-clear-bits](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1137-L1149) [pgpa_planner.c#scan-clear-bits](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1845-L1858). It can still deliberately force a cost-disfavored plan by removing cheaper alternatives; the explicit non-goal is forcing plans rejected for reasons other than cost [README#goals](../../../raw/postgres-19/contrib/pg_plan_advice/README#L6-L17) [README#bad-advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L42-L49). Another rule is "no inference from absence": removing a piece of advice only ever widens the planner's freedom, so every constraint must be stated explicitly [README#Advice-Completeness](../../../raw/postgres-19/contrib/pg_plan_advice/README#L191-L202).

The feature's visible module landed in 25 commits that touched `contrib/pg_plan_advice/` (2026-03-12 through 2026-07-06), but the complete scoped source history also includes 20 core planner foundation/enabling/fix commits and test/doc support commits. Those are separated under [Source Commit History](#source-commit-history).

## Module Layout

The module is a loadable library: the make and meson build files define a `MODULE_big` / `shared_module`, and the user-facing docs load it with `LOAD`, `session_preload_libraries`, or `shared_preload_libraries` rather than `CREATE EXTENSION` [Makefile#module](../../../raw/postgres-19/contrib/pg_plan_advice/Makefile#L3-L20) [meson.build#shared_module](../../../raw/postgres-19/contrib/pg_plan_advice/meson.build#L34-L47) [pgplanadvice.sgml#getting-started](../../../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L33-L40) [syntax.sql#LOAD](../../../raw/postgres-19/contrib/pg_plan_advice/sql/syntax.sql#L1). `shared_preload_libraries` requires a server restart, while `session_preload_libraries` applies to new sessions [pgplanadvice.sgml#getting-started](../../../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L33-L40). Its source files split by responsibility:

```text
contrib/pg_plan_advice/
├── pg_plan_advice.c      entry point: GUCs, EXPLAIN option, advisor hooks, output
├── pgpa_scanner.l        flex lexer for the advice language
├── pgpa_parser.y         bison grammar; pgpa_parse() entry
├── pgpa_ast.c/.h         advice AST: tags, targets, relation identifiers, matching
├── pgpa_identifier.c/.h  build/parse "alias#occ/schema.part@plan" identifiers
├── pgpa_trove.c/.h       organize parsed advice for fast lookup during planning
├── pgpa_planner.c/.h     planner hooks: enforce advice + drive advice generation
├── pgpa_walker.c/.h      walk a finished plan to collect scans/joins/features
├── pgpa_scan.c/.h        classify scan nodes into scan strategies
├── pgpa_join.c/.h        "unroll" join trees into ordered/method form
└── pgpa_output.c/.h      render collected plan facts back into an advice string
```

(File roles taken from each file's header comment, e.g. [pgpa_walker.h#purpose](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_walker.h#L4-L5), [pgpa_trove.h#purpose](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_trove.h#L4-L5), [pgpa_output.h#purpose](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_output.h#L4-L5).)

A companion module, `pg_stash_advice`, is a separate `contrib` directory that stores advice strings in shared memory and feeds them back during planning via `pg_plan_advice`'s advisor hook [pg_stash_advice.h#overview](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.h#L4-L11) [pg_stash_advice.c#advisor-registration](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.c#L141-L145) [pg_stash_advice.c#pgsa_advisor](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.c#L153-L211). It remains outside this page's commit-history scope.

## Core Planner Changes (the mechanism)

`pg_plan_advice` could not exist without v19 planner additions. Four pieces matter.

### 1. The strategy mask `pgs_mask`

`pathnodes.h` defines `PGS_*` bits, one per "path generation strategy": scan types (`PGS_SEQSCAN`, `PGS_INDEXSCAN`, `PGS_INDEXONLYSCAN`, `PGS_BITMAPSCAN`, `PGS_TIDSCAN`), join methods (`PGS_FOREIGNJOIN`, `PGS_MERGEJOIN_PLAIN/_MATERIALIZE`, `PGS_NESTLOOP_PLAIN/_MATERIALIZE/_MEMOIZE`, `PGS_HASHJOIN`), append/gather (`PGS_APPEND`, `PGS_MERGE_APPEND`, `PGS_GATHER`, `PGS_GATHER_MERGE`), and three "consider" toggles (`PGS_CONSIDER_INDEXONLY`, `PGS_CONSIDER_PARTITIONWISE`, `PGS_CONSIDER_NONPARTIAL`) [pathnodes.h#PGS-constants](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L66-L97).

The bits travel on three objects: `PlannerGlobal.default_pgs_mask` (query default) [pathnodes.h#default_pgs_mask](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L263), `RelOptInfo.pgs_mask` (per base/join rel) [pathnodes.h#RelOptInfo.pgs_mask](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L1038-L1039), and `JoinPathExtraData.pgs_mask` (per candidate join method) [pathnodes.h#JoinPathExtraData.pgs_mask](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L3616-L3627).

`standard_planner` builds `default_pgs_mask` directly from the `enable_*` GUCs: `enable_seqscan` sets `PGS_SEQSCAN`, `enable_indexscan` sets both `PGS_INDEXSCAN` and `PGS_INDEXONLYSCAN`, `enable_hashjoin` sets `PGS_HASHJOIN`, and so on; a few strategies (Append, Merge Append, foreign join, Gather, non-partial) have no GUC and are always on [planner.c#default_pgs_mask](../../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L493-L524). Each new `RelOptInfo` then copies `default_pgs_mask` into its own `pgs_mask` [relnode.c#baserel-init](../../../raw/postgres-19/src/backend/optimizer/util/relnode.c#L234) [relnode.c#joinrel-init](../../../raw/postgres-19/src/backend/optimizer/util/relnode.c#L840).

Core path-generation code reads the mask to decide what to build. For joins, `add_paths_to_joinrel` copies `joinrel->pgs_mask` into `extra.pgs_mask`, then normally considers merge joins when `PGS_MERGEJOIN_ANY` is set, hash joins when `PGS_HASHJOIN` is set, foreign joins when `PGS_FOREIGNJOIN` is set, and Memoize when `PGS_NESTLOOP_MEMOIZE` is set [joinpath.c#mask-use](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L153-L174) [joinpath.c#mergejoin-gate](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L239-L244) [joinpath.c#hashjoin-gate](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L353-L362) [joinpath.c#memoize-gate](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L724-L738). Full joins are an explicit exception: merge join and hash join paths can still be considered even when disabled because they may be the only legal implementation [joinpath.c#full-join-merge-exception](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L239-L244) [joinpath.c#full-join-hash-exception](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L353-L356). The mask comment spells out the two enforcement styles: in most cases a generated path of the cleared type is marked *disabled* (so it loses on the disabled-node penalty), but in cases where it is safe to skip entirely (an alternative path must exist) path generation is skipped outright [pathnodes.h#two-styles](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L38-L65).

### 2. Five new planner hooks

Core exposes hooks at the exact points `pg_plan_advice` needs: `planner_setup_hook` and `planner_shutdown_hook` (start/end of planning) [planner.h#setup-shutdown-hooks](../../../raw/postgres-19/src/include/optimizer/planner.h#L36-L47), `build_simple_rel_hook` (a base rel was created) and `joinrel_setup_hook` (a join rel was created) [pathnode.h#hooks](../../../raw/postgres-19/src/include/optimizer/pathnode.h#L21-L48), and `join_path_setup_hook` (a proposed outer relation, inner relation, and join type are about to generate join paths) [paths.h#join_path_setup_hook](../../../raw/postgres-19/src/include/optimizer/paths.h#L32-L38) [joinpath.c#join-path-hook-call](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L153-L179). `standard_planner` calls `planner_setup_hook` right after seeding `default_pgs_mask` [planner.c#setup-hook-call](../../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L526-L529).

### 3. Per-object extension state

A new `extendplan` facility lets an extension attach private state to planner objects by integer id: `GetPlannerExtensionId`, plus get/set helpers for `PlannerGlobal`, `PlannerInfo`, and `RelOptInfo` [extendplan.h#api](../../../raw/postgres-19/src/include/optimizer/extendplan.h#L19-L70). `pg_plan_advice` obtains its id once at load and stashes its per-query `pgpa_planner_state` on the `PlannerGlobal` and its per-join `pgpa_join_state` on the `RelOptInfo` [pgpa_planner.c#extension-id](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L183) [pgpa_planner.c#set-global-state](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L280-L288). Generated advice and feedback ride out of planning on `PlannedStmt.extension_state` as `DefElem`s [plannodes.h#extension_state](../../../raw/postgres-19/src/include/nodes/plannodes.h#L158-L165) [pgpa_planner.c#stash-in-pstmt](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L389-L393).

### 4. Per-index disabling

Core also added `IndexOptInfo.disabled`, a flag that marks paths using that index as disabled without hiding the index from other planner reasoning [pathnodes.h#IndexOptInfo.disabled](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L1427-L1428) [pathnode.c#index-disabled](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L1126-L1131). `pg_plan_advice` uses that flag when `INDEX_SCAN` or `INDEX_ONLY_SCAN` names an index: once it finds the matching index, it marks every other index on the relation disabled [pgpa_planner.c#index-disable](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1788-L1819).

## Module Initialization

`_PG_init` defines five `PGC_USERSET` GUCs, reserves the `pg_plan_advice` prefix, registers the `EXPLAIN (PLAN_ADVICE)` option, and installs the planner and EXPLAIN hooks [pg_plan_advice.c#_PG_init](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L67-L139). The EXPLAIN option is registered through v19's extensible-EXPLAIN mechanism with a boolean check handler [pg_plan_advice.c#register-explain](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L128-L133). Long-lived data (advisor list, etc.) goes in a dedicated `TopMemoryContext` child created lazily by `pg_plan_advice_get_mcxt` [pg_plan_advice.c#get_mcxt](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L145-L154).

Advice can come from two sources. Normally it is the `pg_plan_advice.advice` GUC string. But other modules can register an "advisor" callback with `pg_plan_advice_add_advisor`; advisors are consulted first, in registration order, and the first non-NULL string wins [pg_plan_advice.c#advisor-list](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L172-L228) [pg_plan_advice.h#advisor-hook](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.h#L47-L52). This is how `pg_stash_advice` injects saved advice.

## The Advice Language

Advice is whitespace-separated items, each applying one *tag* to one or more *targets*, e.g. `SEQ_SCAN(foo) HASH_JOIN(bar@ss)` [README#syntax](../../../raw/postgres-19/contrib/pg_plan_advice/README#L18-L27). The lexer/grammar (`pgpa_parse` -> flex/bison) builds an AST list of `pgpa_advice_item` objects [pgpa_ast.h#pgpa_advice_item](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h#L104-L116) [pgpa_ast.h#parse-entry](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h#L166-L169).

The recognized tags are an enum of 20 values: scan tags (`SEQ_SCAN`, `INDEX_SCAN`, `INDEX_ONLY_SCAN`, `BITMAP_HEAP_SCAN`, `TID_SCAN`, `DO_NOT_SCAN`), join-method tags (`HASH_JOIN`, `MERGE_JOIN_PLAIN`, `MERGE_JOIN_MATERIALIZE`, `NESTED_LOOP_PLAIN`, `NESTED_LOOP_MATERIALIZE`, `NESTED_LOOP_MEMOIZE`, `FOREIGN_JOIN`), `JOIN_ORDER`, semijoin tags (`SEMIJOIN_UNIQUE`, `SEMIJOIN_NON_UNIQUE`), `PARTITIONWISE`, and parallelism tags (`GATHER`, `GATHER_MERGE`, `NO_GATHER`) [pgpa_ast.h#tags](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h#L80-L102).

A target is a relation identifier, an *ordered* sublist `( ... )`, or an *unordered* sublist `{ ... }` [pgpa_ast.h#target-type](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h#L25-L71). `INDEX_SCAN`/`INDEX_ONLY_SCAN` targets additionally carry an index name [pgpa_ast.h#index_target](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h#L35-L64) [README#Scan-Advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L102-L106).

### Relation identifiers

The hard part is naming exactly the right relation. The canonical form is `alias_name#occurrence_number/partition_schema.partition_name@plan_name`, where every component except the alias is optional and the punctuation is dropped when a component is omitted [README#Relation-Identifiers](../../../raw/postgres-19/contrib/pg_plan_advice/README#L51-L79) [pgpa_identifier.h#pgpa_identifier](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_identifier.h#L19-L26). The occurrence number disambiguates the same alias used twice in one subquery; `@plan_name` disambiguates the same alias across subqueries (it is omitted for the top-level `PlannerInfo`); `/schema.name` names a specific partition child [README#identifier-rules](../../../raw/postgres-19/contrib/pg_plan_advice/README#L66-L79).

### What each tag family means

- **Scan** — pick the scan type for a relation; index scans name the index too; `DO_NOT_SCAN(rel)` forbids scanning a relation at all (used when an alternative version of the relation is chosen instead) [README#Scan-Advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L81-L117).
- **JOIN_ORDER** — `JOIN_ORDER(t1 t2 t3)` makes `t1` the driving table joined to `t2` then `t3` (outer-deep). Parentheses express non-outer-deep shape; curly braces express "joined, but side/order undefined" (e.g. partitionwise or foreign join) [README#Join-Order-Advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L119-L149).
- **Join method** — `HASH_JOIN(x)` etc. name the relation(s) that must sit on the *inner* side of that join method; this also constrains join order because the named side must be buildable as the inner input [README#Join-Method-Advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L151-L189).
- **Semijoin uniqueness** — `SEMIJOIN_UNIQUE()`/`SEMIJOIN_NON_UNIQUE()` record whether a semijoin was turned into "make-unique then inner join" or implemented directly [README#Semijoin-Uniqueness](../../../raw/postgres-19/contrib/pg_plan_advice/README#L204-L213).
- **Partitionwise** — each `PARTITIONWISE(...)` argument is a relation set that should be joined partitionwise (and nothing else partitionwise) [README#Partitionwise](../../../raw/postgres-19/contrib/pg_plan_advice/README#L214-L226).
- **Parallelism** — `GATHER()`/`GATHER_MERGE()` place a Gather/Gather Merge over an exact relation set; `NO_GATHER()` forbids parallelism for a relation [README#Parallel-Query](../../../raw/postgres-19/contrib/pg_plan_advice/README#L228-L235).

## Generating Advice (plan -> string)

When generation is requested, the work happens in `planner_shutdown_hook` -> `pgpa_planner_shutdown` after planning finishes [pgpa_planner.c#shutdown](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L299-L398). It runs the plan-tree walker and then renders a string:

1. **Walk** — `pgpa_plan_walker` traverses the `PlannedStmt` and all subplan roots, collecting facts into a `pgpa_plan_walker_context`: scans bucketed by `pgpa_scan_strategy`, join trees "unrolled" into ordered members with per-member `pgpa_join_strategy`, "query features" (Gather, Gather Merge, and the two semijoin outcomes), the set of scans under no Gather, and `DO_NOT_SCAN` identifiers [pgpa_walker.h#walker-context](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_walker.h#L66-L108) [pgpa_walker.h#query-features](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_walker.h#L19-L64).
   - Scan classification (`pgpa_scan_strategy`) deliberately treats "uninteresting" scans (subquery, VALUES, proven-empty Result) as `PGPA_SCAN_ORDINARY` and emits no advice for them, because there is no planner decision to preserve [pgpa_scan.h#strategies](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_scan.h#L26-L68).
   - Join unrolling flattens outer-deep trees (`((A⋈B)⋈C)⋈D` becomes outer `A`, inner `<B C D>`) and uses substructure for other shapes [pgpa_join.h#unrolled_join](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_join.h#L64-L84). Join strategy is recorded at sub-method precision (merge plain vs. materialized; nested loop plain/materialize/memoize) [pgpa_join.h#join_strategy](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_join.h#L27-L38).
2. **Identifiers** — `pgpa_create_identifiers_for_planned_stmt` builds the unique identifier for every RTI in the flattened range table [pgpa_planner.c#create-identifiers](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L334-L336).
3. **Render** — `pgpa_output_advice` emits items in a fixed order: `JOIN_ORDER` per top-level unrolled join, then join-method advice, then scan advice (skipping ordinary scans), then query-feature advice (Gather/Gather Merge/semijoin), then `NO_GATHER`, then `DO_NOT_SCAN`, wrapping long lines at column 76 [pgpa_output.c#pgpa_output_advice](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_output.c#L79-L164).

The resulting string is stashed in `PlannedStmt.extension_state` under `advice_string` [pgpa_planner.c#advice_string](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L338-L354). Generation is triggered by `EXPLAIN (PLAN_ADVICE)`, by `pg_plan_advice.always_store_advice_details = on`, or by another module calling `pg_plan_advice_request_advice_generation(true)` (a counter, so multiple requesters coexist) [pgpa_planner.c#generate-decision](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L211-L221) [pg_plan_advice.c#request_advice_generation](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L244-L254).

## Enforcing Advice (string -> plan)

When a query is planned and advice applies, `planner_setup_hook` -> `pgpa_planner_setup` parses the supplied advice (re-parsing even GUC advice for safety) and builds a `pgpa_trove` — the parsed advice reorganized for lookup by relation-set, with three lookup flavors: scan, join, and general rel [pgpa_planner.c#setup](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L199-L294) [pgpa_trove.h#lookup-types](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_trove.h#L33-L78). It saves a `pgpa_planner_state` (trove + per-`PlannerInfo` bookkeeping) on the `PlannerGlobal` [pgpa_planner.c#planner_state](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L44-L52).

Enforcement then happens in three hooks. The hooks only clear `pgs_mask` bits, except that index-specific advice also marks nonmatching indexes disabled:

- **Base rels** — `build_simple_rel_hook` -> `pgpa_build_simple_rel` looks up scan and rel advice for the relation's identifier and clears the forbidden scan strategies on `rel->pgs_mask` [pgpa_planner.c#build_simple_rel](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L403-L467). `pgpa_planner_apply_scan_advice` maps each scan tag to the surviving `PGS_*` bit(s) — e.g. `SEQ_SCAN` keeps only `PGS_SEQSCAN`, `INDEX_SCAN` keeps only `PGS_INDEXSCAN`, `DO_NOT_SCAN` keeps nothing — and detects conflicting/duplicate advice [pgpa_planner.c#apply_scan_advice](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1625-L1700). For index advice, it also disables every nonmatching index after locating the named index [pgpa_planner.c#index-disable](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1788-L1819).
- **Join rels** — `joinrel_setup_hook` -> `pgpa_joinrel_setup` clears join/partitionwise bits on `joinrel->pgs_mask` based on advice for the join's relation set [pgpa_planner.c#joinrel_setup](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L477-L521).
- **Join methods** — `join_path_setup_hook` -> `pgpa_join_path_setup` is called per candidate join (with `JoinType` and the proposed outer/inner) and clears method bits on `extra->pgs_mask`, also enforcing join-order and semijoin constraints by denying join orders incompatible with the advice [pgpa_planner.c#join_path_setup](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L527-L622). It is also where the module notices `JOIN_UNIQUE_OUTER`/`JOIN_UNIQUE_INNER` consideration and records it in `sj_unique_rels` for later semijoin advice [pgpa_planner.c#sj_unique_rels](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L536-L578).

Because every `pgs_mask` helper only clears bits, advice can never re-enable a strategy the `enable_*` GUCs disabled, and partitionwise consideration is preserved unless explicitly removed [pgpa_planner.c#clear-only](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L900-L933). With `pg_plan_advice.trace_mask = on`, each mask change is logged as a `WARNING` for debugging [pgpa_planner.c#trace_mask](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L447-L460).

## Feedback and EXPLAIN Output

The module also reports how well advice applied. Each `pgpa_trove_entry` carries a flags field for `PGPA_FB_MATCH_PARTIAL`, `PGPA_FB_MATCH_FULL`, `PGPA_FB_INAPPLICABLE`, `PGPA_FB_CONFLICTING`, and `PGPA_FB_FAILED` [pg_plan_advice.h#feedback-flags](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.h#L18-L45). The enforcement hooks set the match, conflict, and inapplicable flags while planning, and `pgpa_planner_shutdown` then walks the finished plan to add `PGPA_FB_FAILED` when the resulting plan does not contain the advised choice and stores the feedback list in `PlannedStmt.extension_state` [pgpa_planner.c#scan-feedback-flags](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1803-L1843) [pgpa_planner.c#append_feedback](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1866-L1897) [pgpa_planner.c#stash-feedback](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L360-L393). With `pg_plan_advice.feedback_warnings = on`, non-clean application raises warnings via `pgpa_planner_feedback_warning` [pgpa_planner.c#feedback-warning-call](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L384-L387).

`explain_per_plan_hook` -> `pg_plan_advice_explain_per_plan_hook` reads the stash from `PlannedStmt.extension_state` and prints a "Supplied Plan Advice" block (each item annotated with its feedback flags) and, under `EXPLAIN (PLAN_ADVICE)`, a "Generated Plan Advice" block [pg_plan_advice.c#explain-hook](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L346-L410). By default supplied-advice feedback shows even without the option; `pg_plan_advice.always_explain_supplied_advice = off` restricts it to `EXPLAIN (PLAN_ADVICE)` [pg_plan_advice.c#always_explain](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L81-L90).

Example generated advice for a star-schema join, from the regression output [join_order.out#example](../../../raw/postgres-19/contrib/pg_plan_advice/expected/join_order.out#L43-L46):

```text
 Generated Plan Advice:
   JOIN_ORDER(f d2 d1)
   HASH_JOIN(d2 d1)
   SEQ_SCAN(f d2 d1)
```

## GUCs

All five GUCs are `PGC_USERSET`, so they take effect immediately in the session/transaction and need no restart or reload [pg_plan_advice.c#guc-context](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L70-L123):

| GUC | Type | Effect |
|---|---|---|
| `pg_plan_advice.advice` | string | Advice applied to subsequent planning; validated by a check hook that parses it [pg_plan_advice.c#advice-guc](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L70-L79) [pg_plan_advice.c#check-hook](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L415-L445) |
| `pg_plan_advice.always_explain_supplied_advice` | bool (on) | Show supplied-advice feedback in EXPLAIN even without `PLAN_ADVICE` [pg_plan_advice.c#always_explain](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L81-L90) |
| `pg_plan_advice.always_store_advice_details` | bool (off) | Always generate/store advice (e.g. to see advice for prepared statements) [pg_plan_advice.c#always_store](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L92-L101) |
| `pg_plan_advice.feedback_warnings` | bool (off) | Warn when supplied advice does not apply cleanly [pg_plan_advice.c#feedback_warnings](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L103-L112) |
| `pg_plan_advice.trace_mask` | bool (off) | Emit per-relation strategy-mask changes as warnings [pg_plan_advice.c#trace_mask](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L114-L123) |

## Prepared Statements and Plan Caching

`pg_plan_advice` has no plan-cache integration: nothing in the module touches `CachedPlan`/`plancache`, and the `pg_plan_advice.advice` GUC is registered with flag `0`, so changing it carries no plan-cache invalidation hint [pg_plan_advice.c#advice-guc](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L70-L79). Everything the module does happens at plan time: advice is read and enforced while the planner runs, and any generated advice or feedback is stashed into the resulting `PlannedStmt` [pgpa_planner.c#read-advice](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L227) [pgpa_planner.c#stash-in-pstmt](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L389-L393). Because a prepared statement caches its plan and reuses it without re-running the planner, this produces two distinct interactions.

### Enforcement is frozen into the cached plan

Advice is consulted only inside `pgpa_planner_setup`, i.e. only when the planner actually executes [pgpa_planner.c#setup](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L199-L227). The value of `pg_plan_advice.advice` in effect when a cached plan is built is what gets enforced; changing the GUC afterward has no effect until that statement is re-planned, and since the GUC has no invalidation flag, changing it does not by itself invalidate an existing cached plan.

Whether a re-plan ever happens is decided entirely by the core plan cache, not by this module. `choose_custom_plan` defines that policy [plancache.c#choose_custom_plan](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L1174-L1222):

- An **unparameterized** prepared statement (`boundParams == NULL`) uses a generic plan, built once and reused; advice is captured at first execution and frozen thereafter (barring invalidation) [plancache.c#no-params](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L1183-L1185).
- A **parameterized** statement is re-planned as a custom plan for at least its first 5 executions, then may switch to a cached generic plan on cost [plancache.c#five-custom](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L1202-L1221). While custom plans are still being built, an advice change between executions takes effect; once it settles on the generic plan, it does not.
- `plan_cache_mode = force_custom_plan` re-plans every execution (advice always re-read); `force_generic_plan` plans once (advice frozen immediately) [plancache.c#mode-force](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L1191-L1194).
- DDL or relcache/syscache invalidation on a referenced relation forces a re-plan, at which point the current advice is re-applied [plancache.c#invalidation-overview](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L14-L39) [plancache.c#relation-invalidation](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L2137-L2175) [plancache.c#object-invalidation](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L2203-L2274).

So to guarantee a prepared statement reflects the current advice GUC, force a re-plan: set the advice before PREPARE/first execution, use `force_custom_plan`, or re-prepare after changing it.

### Generated advice and feedback are usually absent for EXECUTE

`EXPLAIN (PLAN_ADVICE) EXECUTE ...` can only display advice or feedback that was stored in the `PlannedStmt` during planning. Generation is requested only when the planner runs with a generation trigger present: `EXPLAIN (PLAN_ADVICE)`, `pg_plan_advice.always_store_advice_details`, or a module calling `pg_plan_advice_request_advice_generation(true)` [pgpa_planner.c#generate-decision](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L211-L221) [pg_plan_advice.c#request_advice_generation](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L244-L254). Registering an advisor hook only supplies advice; it does not itself request generated advice [pg_plan_advice.c#advisor-list](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L172-L228). For a prepared statement, the plan was built earlier without that trigger, so nothing was stored and there is nothing to show. The module's own comment states this and prescribes the workaround [pg_plan_advice.c#prepared-note](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L376-L381).

The workaround is `pg_plan_advice.always_store_advice_details = on`, which forces generation at every planning cycle regardless of EXPLAIN, baking the data into the cached plan so a later `EXPLAIN (PLAN_ADVICE) EXECUTE` can display it [pg_plan_advice.c#always_store](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L92-L101). Because it only matters at plan-build time, it must be on when the cached plan is built; enabling it after a generic plan already exists does not retroactively populate that plan.

The dedicated `prepared` regression test demonstrates both halves: with `always_store_advice_details = off`, `EXECUTE pt1`/`pt3` show no Generated/Supplied advice, whereas after enabling it, `EXECUTE pt2`/`pt4` do [prepared.sql](../../../raw/postgres-19/contrib/pg_plan_advice/sql/prepared.sql) [prepared.out#L16-L68](../../../raw/postgres-19/contrib/pg_plan_advice/expected/prepared.out#L16-L68):

```text
-- always_store_advice_details = false
EXPLAIN (COSTS OFF, PLAN_ADVICE) EXECUTE pt1;   -- "Seq Scan on ptab" only, no advice block
-- always_store_advice_details = true
EXPLAIN (COSTS OFF, PLAN_ADVICE) EXECUTE pt2;   -- adds "Generated Plan Advice: SEQ_SCAN(ptab) ..."
```

## Round-Trip Testing

The module's contract is round-trip safety: advice generated from a plan must re-apply cleanly and reproduce that plan absent intervening DDL [README#round-trip](../../../raw/postgres-19/contrib/pg_plan_advice/README#L29-L40). Two test layers guard it. The `contrib` regression suite (`alternatives`, `gather`, `join_order`, `join_strategy`, `partitionwise`, `prepared`, `scan`, `semijoin`, `syntax`) exercises syntax and generated output [Makefile#REGRESS](../../../raw/postgres-19/contrib/pg_plan_advice/Makefile#L22-L24). Separately, `src/test/modules/test_plan_advice` plans *every* query twice — first to generate advice, then to replan under that advice — to catch advice that fails, is inapplicable, or changes the plan [test_plan_advice.c#purpose](../../../raw/postgres-19/src/test/modules/test_plan_advice/test_plan_advice.c#L3-L17). Several of the bug-fix commits below were buildfarm failures surfaced by that second harness.

## Source Commit History

This history has three scopes. First are the 20 same-checkout core planner foundation, enabling, and fix commits that created or repaired infrastructure `pg_plan_advice` uses directly. Second are all 25 commits that touched `contrib/pg_plan_advice/`. Third are the test, docs, and build-tooling support commits that mention or change `pg_plan_advice` surfaces outside the module directory. Commit subjects and bodies are taken from the pinned `raw/postgres-19/` git history. Dates are author dates.

### Core planner enabling and fix commits

- `e2225346` — Treat number of disabled nodes in a path as a separate cost metric (2024-08-21, Robert Haas). This is not `pg_plan_advice`-specific, but it is the foundation for making disabled paths lose before ordinary cost comparison when advice or `enable_*` settings cannot suppress generation outright [pathnode.c#compare-disabled-nodes](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L67-L77).
- `8c49a484` — Assign each subquery a unique name prior to planning it (2025-10-07, Robert Haas). This made subquery `plan_name` stable before planning, which relation identifiers later use to disambiguate subqueries [pathnodes.h#PlannerInfo.plan_name](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L320-L333).
- `0132ddda` — Allow private state in certain planner data structures (2025-10-07, Robert Haas). This added extension state slots to `PlannerGlobal`, `PlannerInfo`, and `RelOptInfo`, later used by `pg_plan_advice` for per-query and per-join state [extendplan.h#api](../../../raw/postgres-19/src/include/optimizer/extendplan.h#L19-L70) [pathnodes.h#extension_state](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L271-L273).
- `c83ac02e` — Add `ExplainState` argument to `pg_plan_query()` and `planner()` (2025-10-08, Robert Haas). This lets planning see EXPLAIN extension options such as `PLAN_ADVICE` [planner.h#standard_planner](../../../raw/postgres-19/src/include/optimizer/planner.h#L58-L61).
- `94f3ad39` — Add `planner_setup_hook` and `planner_shutdown_hook` (2025-10-08, Robert Haas). `pg_plan_advice` uses these as the start/end hooks for parsing supplied advice and storing generated advice [planner.h#setup-shutdown-hooks](../../../raw/postgres-19/src/include/optimizer/planner.h#L36-L47).
- `4685977c` — Add `extension_state` member to `PlannedStmt` (2025-10-08, Robert Haas). This lets the shutdown hook put generated advice and feedback into the final plan so the EXPLAIN hook can read it later [plannodes.h#extension_state](../../../raw/postgres-19/src/include/nodes/plannodes.h#L158-L165) [pgpa_planner.c#stash-in-pstmt](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L389-L393).
- `4020b370` — Allow for plugin control over path generation strategies (2026-01-28, Robert Haas). This is the real origin of `pgs_mask`, `joinrel_setup_hook`, and `join_path_setup_hook`, not the later module commit [pathnodes.h#PGS-strategy-mask](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L25-L97) [pathnode.h#joinrel_setup_hook](../../../raw/postgres-19/src/include/optimizer/pathnode.h#L40-L48) [paths.h#join_path_setup_hook](../../../raw/postgres-19/src/include/optimizer/paths.h#L32-L38).
- `71c11369` — Fix mistakes in commit `4020b370` (2026-01-29, Robert Haas). This corrected early `pgs_mask` fallout in costing/path code, including disabled-node handling.
- `cbdf93d4` — Fix `PGS_CONSIDER_NONPARTIAL` interaction with Materialize nodes (2026-02-10, Robert Haas). This moved the materialized-nestloop decision out of `cost_material()` so planner code either skips an unwanted Materialize path or creates it without an unintended disabled-node penalty [joinpath.c#consider_parallel_nestloop-materialize](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L2107-L2121).
- `0f4c8d33` — Pass `cursorOptions` to `planner_setup_hook` (2026-02-10, Robert Haas). Advisor callbacks receive `cursorOptions` through the setup path [pg_plan_advice.h#advisor-hook](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.h#L47-L52).
- `adbad833` — Store information about range-table flattening in the final plan (2026-02-10, Robert Haas). `PlannedStmt.subrtinfos` records subquery range-table offsets for mapping planned RTIs back to subquery planning cycles [plannodes.h#subrtinfos](../../../raw/postgres-19/src/include/nodes/plannodes.h#L131-L132) [plannodes.h#SubPlanRTInfo](../../../raw/postgres-19/src/include/nodes/plannodes.h#L1846-L1857).
- `0d4391b2` — Store information about elided nodes in the final plan (2026-02-10, Robert Haas). `ElidedNode` preserves RTIs for trivial plan nodes removed during setrefs, which the advice walker needs when reconstructing plan decisions [plannodes.h#elidedNodes](../../../raw/postgres-19/src/include/nodes/plannodes.h#L155-L156) [plannodes.h#ElidedNode](../../../raw/postgres-19/src/include/nodes/plannodes.h#L1860-L1874).
- `7358abcc` — Store information about Append node consolidation in the final plan (2026-02-10, Robert Haas). The `child_append_relid_sets` fields preserve partitionwise-join grouping information after Append/MergeAppend consolidation [pathnodes.h#child_append_relid_sets](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L2267-L2289) [plannodes.h#Append-child-sets](../../../raw/postgres-19/src/include/nodes/plannodes.h#L402-L406).
- `6e466e1e` — Fix `add_partial_path` interaction with `disabled_nodes` (2026-02-19, Robert Haas). This made partial paths keep the same disabled-node-first ordering as ordinary paths, so disabled parallel alternatives cannot be preferred merely because their total cost is lower [pathnode.c#add_partial_path-disabled-order](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L753-L900).
- `8300d3ad` — Consider startup cost as a figure of merit for partial paths (2026-03-09, Robert Haas; co-authored by Tomas Vondra). In addition to the startup-cost change, this fixed `add_partial_path_precheck` to compare `disabled_nodes`, matching `compare_path_costs_fuzzily()` [pathnode.c#add_partial_path_precheck-disabled-nodes](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L912-L985).
- `91f33a2a` — Replace `get_relation_info_hook` with `build_simple_rel_hook` (2026-03-09, Robert Haas). This produced the base-relation hook `pg_plan_advice` installs [pathnode.h#build_simple_rel_hook](../../../raw/postgres-19/src/include/optimizer/pathnode.h#L21-L24).
- `0fbfd37c` — Allow extensions to mark an individual index as disabled (2026-03-10, Robert Haas). `pg_plan_advice` uses this for index-name advice rather than deleting indexes from `RelOptInfo.indexlist` [pathnodes.h#IndexOptInfo.disabled](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L1427-L1428) [pathnode.c#index-disabled](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L1126-L1131).
- `dc47beac` — `get_memoize_path`: don't exit quickly when `PGS_NESTLOOP_PLAIN` is unset (2026-03-24, Robert Haas). This fixed a `test_plan_advice` failure where `NESTED_LOOP_MEMOIZE()` could not be enforced because core exited before costing Memoize [joinpath.c#get_memoize_path](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L723-L739).
- `47c110f7` — Respect `disabled_nodes` in `fix_alternative_subplan` (2026-03-20, Robert Haas). Alternative subplans now carry and compare the disabled-node count, which matters for advice around alternative subplan selection [primnodes.h#SubPlan.disabled_nodes](../../../raw/postgres-19/src/include/nodes/primnodes.h#L1073-L1111) [setrefs.c#fix_alternative_subplan](../../../raw/postgres-19/src/backend/optimizer/plan/setrefs.c#L2235-L2268).
- `26255a32` — Add an `alternative_plan_name` field to `PlannerInfo` (2026-03-26, Robert Haas). This lets advice relate alternative planning roots for `MinMaxAggPath` and hashed subplans back to their original plan names [pathnodes.h#alternative_plan_name](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L323-L333).

The 25 direct module commits are:

### `5883ff30` — Add pg_plan_advice contrib module (2026-03-12, Robert Haas)

The foundational module commit. It introduces the `pg_plan_advice` loadable library: a facility to (1) stabilize plan choices and (2) let knowledgeable users override the planner, by analyzing a finished plan into textual "plan advice" that can be replayed in future planning cycles. The commit message scopes out aggregation choice, sort order, and estimate/cost control as future work, and lists nine reviewers. The core `pgs_mask`, hook, extension-state, and index-disable infrastructure this module uses came from the earlier core commits listed above [README#Plan-Advice](../../../raw/postgres-19/contrib/pg_plan_advice/README#L3-L17).

### `be43c48c` — Initialize variable to placate compiler (2026-03-13, Nathan Bossart; patch by Sami Imseih)

Day-after portability fix: since `5883ff30`, some compilers warned that `rtekind` in `unique_nonjoin_rtekind()` might be used uninitialized. No real risk; the variable is initialized to silence the warning.

### `4f888d0f` — Fix whitespace (2026-03-16, Peter Eisentraut)

Trivial whitespace cleanup in the new files.

### `5e72ce24` — Fix failures to accept identifier keywords (2026-03-16, Robert Haas; author Lukas Fittl)

Grammar bug: `TOK_IDENT` allowed only non-keywords, so any advice component that happened to be a SQL keyword (partition schema, partition name, plan name) was rejected. The fix uses the `identifier` production that accepts both keywords and non-keywords [pgpa_parser.y](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_parser.y).

### `7560995a` — Fix variable type confusion (2026-03-17, Robert Haas)

`pgs_mask` values are 64-bit; two places mistakenly used `uint32`. Corrected to `uint64`, matching the field width on `RelOptInfo`/`JoinPathExtraData` [pathnodes.h#RelOptInfo.pgs_mask](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L1038-L1039).

### `59dcc19b` — Always install pg_plan_advice.h, and in the right place (2026-03-17, Robert Haas; author Zsolt Parragi)

Build fix: the Makefile didn't set `HEADERS_pg_plan_advice`, so the public header wasn't installed; and because this is a loadable module (not an extension), the header belongs in `$(includedir_server)/contrib`, not `.../extension`. The fix also makes the `make` and `meson` install locations agree.

### `01b02c0e` — Avoid a crash under GEQO (2026-03-17, Robert Haas)

GEQO can invoke the module's callbacks in a short-lived memory context. The previous code allocated `pgpa_sj_unique_rel` objects (and their `List` cells) there, risking use-after-free; the fix allocates them in the same long-lived context as the owning `pgpa_planner_state` and copies `uniquerel->relids`. This is why `pgpa_join_path_setup` switches to `pps->mcxt` before recording semijoin rels [pgpa_planner.c#sj_unique_rels](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L554-L577).

### `b335fe56` — Fix multiple copy-and-paste errors in test case (2026-03-18, Robert Haas; reported by Tom Lane)

The feedback half of a test reused leftover prepared statements (`pt2`) instead of the ones it prepared (`pt4`), which produced different output under `debug_discard_caches = 1`. Corrected so the second half actually tests feedback.

### `5dcb15e8` — Refactor to invent pgpa_planner_info (2026-03-26, Robert Haas)

Consolidation. The module tracked two per-`PlannerInfo` things separately (a hash table mapping RTI→identifier for cross-checking, and a list of semijoin make-unique rel sets). This commit unifies them into one `pgpa_planner_info` keyed by `plan_name`, created whenever a new `PlannerInfo` needs associated data [pgpa_planner.h#pgpa_planner_info](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.h#L22-L70).

### `6455e55b` — Invent DO_NOT_SCAN(relation_identifier) (2026-03-26, Robert Haas; reviewer Lukas Fittl)

New tag. `test_plan_advice` could fail when concurrent catalog activity flipped the planner between alternative subplans or a `MinMaxAggPath`, because the cloned subquery has a different plan name and thus different identifiers — so advice referenced relations absent from the replanned tree. `DO_NOT_SCAN` is emitted for relations that should not appear at all, and when supplied it disables every scan method for that relation. The commit also narrows the identifier cross-check to only run when a `pgpa_planner_state` already exists [README#DO_NOT_SCAN](../../../raw/postgres-19/contrib/pg_plan_advice/README#L112-L117).

### `874da8b1` — pgindent (2026-03-26, Robert Haas; reported by Lukas Fittl)

Mechanical reindentation of the module.

### `e2ee9523` — Avoid assertion failure with partitionwise aggregate (2026-03-30, Robert Haas; reported by Alexander Lakhin)

An `Append` that is part of a partitionwise aggregate has no `apprelids`; if such a node was elided, the code called `unique_nonjoin_rtekind()` on a NULL pointer. A NULL check fixes the assertion failure.

### `0442f1c9` — Add a guc_check_handler to the EXPLAIN extension mechanism (2026-04-06, Robert Haas)

Primarily an EXPLAIN-infrastructure commit (so `auto_explain` can validate custom EXPLAIN options before setting them). Within this module it only adds the new check-handler argument, `GUCCheckBooleanExplainOption`, to the `RegisterExtensionExplainOption("plan_advice", ...)` call [pg_plan_advice.c#register-explain](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L128-L133).

### `49ce4181` — Improve various new-to-v19 appendStringInfo calls (2026-04-13, David Rowley)

Tree-wide micro-cleanup choosing the most suitable `appendStringInfo*` variant; it touched `pg_plan_advice.c`, `pgpa_output.c`, and `pgpa_trove.c`. No behavior change.

### `3311ccc3` — Handle non-repeatable TABLESAMPLE scans (2026-04-13, Robert Haas; reported by Alexander Lakhin)

When a tablesample method reports it is not repeatable across scans, `set_tablesample_rel_pathlist` usually materializes it, confusing the plan walker. The fix teaches the walker to treat such a `Material` node as part of the underlying scan [pgpa_walker.h#scan-level-materialize](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_walker.h#L117).

### `1faf9dfa` — Add alternatives test to Makefile (2026-04-13, Robert Haas)

Oversight fix for `6455e55b`: the new `alternatives` regression test wasn't listed in `REGRESS`, so it wasn't run [Makefile#REGRESS](../../../raw/postgres-19/contrib/pg_plan_advice/Makefile#L22-L24).

### `0f93ebb3` — Fix a bug when a subquery is pruned away entirely (2026-04-13, Robert Haas; reported by Alexander Lakhin)

If a proven-empty subquery contained a semijoin whose make-unique strategy was a candidate, advice generation failed with `ERROR: no rtoffset for plan %s`. Fixed so pruned subqueries no longer break identifier offsetting.

### `c644aca2` — Export feedback-related definitions (2026-04-13, Robert Haas)

Prep for letting `test_plan_advice` (and other extensions) inspect feedback. It renames the trove-entry constants from `PGPA_TE_*` to `PGPA_FB_*`, moves them from the internal `pgpa_trove.h` to the installed `pg_plan_advice.h`, and makes `pgpa_planner_feedback_warning` non-static and `PGDLLEXPORT` [pg_plan_advice.h#feedback-flags](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.h#L18-L45) [pgpa_planner.h#feedback-warning](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.h#L79-L81).

### `4321dcad` — Fix another unique-semijoin bug (2026-04-17, Robert Haas; reported by Alexander Lakhin)

A second semijoin bug: when an outer join sits beneath the made-unique side of a semijoin, join RTEs were not factored out of `sj_unique_rels` entries. Fixed, with a test case.

### `228a1f95` — pgindent (2026-04-17, Robert Haas; per buildfarm member koel)

Another mechanical reindent.

### `d3bba041` — Fix a set of typos and grammar issues across the tree (2026-04-21, Michael Paquier)

Tree-wide typo/grammar batch; it touched several `pgpa_*` files' comments and strings. No behavior change in this module.

### `b1901e28` — DO_NOT_SCAN is a simple tag, not a generic one (2026-05-29, Robert Haas; reported by Nikita Kalinin)

Generic tags allow sublists (`MERGE_JOIN((x y))`) but simple tags do not (`SEQ_SCAN((x))` is invalid). `DO_NOT_SCAN` was meant to be simple but was implemented as generic, which could trip assertion failures; this corrects its classification. It was the newest module commit at the previous pin; three more landed on `REL_19_STABLE` after it.

### `89f5f860cc5` — Don't generate FOREIGN_JOIN advice for a single relation (2026-07-02, Robert Haas; author Mahendra Singh Thalor)

A `ForeignScan` can reach the `fs_relids` branch of `pgpa_build_scan()` while targeting only one relation — for example when `postgres_fdw` pushes an aggregate down over a single foreign table. Emitting `FOREIGN_JOIN()` advice for one relation is invalid, so the walker now tests `bms_membership(relids) == BMS_MULTIPLE`: only a genuinely multi-relation foreign scan keeps the `PGPA_SCAN_FOREIGN` strategy, while a single-relation one falls back to `PGPA_SCAN_ORDINARY` [pgpa_scan.c#foreign-single-rel](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_scan.c#L138-L151).

### `ea203d371de` — pgindent fix for commit 53e6f51ee (2026-07-03, Robert Haas)

Mechanical reindentation of the `pgpa_scan.c` change from the preceding FOREIGN_JOIN single-relation fix (`53e6f51ee` names that fix's `master` commit; `89f5f860cc5` is its `REL_19_STABLE` counterpart).

### `da8889ccd7e` — Use PG_MODULE_MAGIC_EXT in newly introduced modules (2026-07-06, Robert Haas; author Andreas Karlsson)

Tree-wide cleanup, backpatched through 19: the newly added modules had used the older `PG_MODULE_MAGIC` macro. This switches `pg_plan_advice` — and its sibling `pg_stash_advice`, plus the `pgrepack` output plugin — to `PG_MODULE_MAGIC_EXT`, which records the module name and version [pg_plan_advice.c#module-magic](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L31-L34).

### Test, documentation, and build-tooling support commits

- `e0e4c132` — Test `pg_plan_advice` using a new `test_plan_advice` module (2026-03-17, Robert Haas). This added the round-trip test harness that replans queries with generated advice [test_plan_advice.c#purpose](../../../raw/postgres-19/src/test/modules/test_plan_advice/test_plan_advice.c#L3-L17).
- `ab697307` — `test_plan_advice`: add `.gitignore` (2026-03-18, Michael Paquier). Test-directory housekeeping.
- `8df3c7a8` — Exclude `contrib/pg_plan_advice/pgpa_parser.h` from headerscheck (2026-03-18, Tom Lane). Build-tooling fix for the Bison-generated header.
- `12444183` — `test_plan_advice`: set TAP test priority 50 in `meson.build` (2026-03-20, Robert Haas; author Matheus Alcantara). This starts the expensive replan-regress TAP test earlier.
- `f4a4f1a7` — doc: fix a couple of mistakes in `pgplanadvice.sgml` (2026-04-13, Robert Haas; author Lakshmi N). This fixed `FOREIGN_JOIN` wording and added the omitted `NESTED_LOOP_MEMOIZE` join method to the docs.
- `5dbb63fc` — REPACK: do not require REPLICATION or LOGIN (2026-04-20, Alvaro Herrera; author Antonin Houska). This is a REPACK commit, but it changed `test_plan_advice`'s TAP script to run under `wal_level = replica`.
- `2fcc8aae` — doc: fix up spacing around verbatim DocBook elements (2026-05-04, Peter Eisentraut). Tree-wide doc formatting; it touched `pgplanadvice.sgml` but did not change feature behavior.
- `736a97bd` — Pre-beta mechanical code beautification, step 2: run `pgperltidy` (2026-05-13, Tom Lane). Mechanical formatting; it touched the `test_plan_advice` TAP script.

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v19/index.md`, and the last ~20 `wiki/log.md` entries for navigation and the v19 pin.
- Module README and all source/headers under `contrib/pg_plan_advice/`.
- Core v19 planner additions and adjacent foundations: `pgs_mask` (`pathnodes.h`), its initialization in `planner.c`/`relnode.c`, its consumption in `joinpath.c` including full-join and Materialize exceptions, the five planner hooks (`planner.h`, `pathnode.h`, `paths.h`), `extendplan.h`, `IndexOptInfo.disabled`, `disabled_nodes` ordering for ordinary paths, partial paths, and alternative subplans, `PlannedStmt.extension_state`, `PlannedStmt.subrtinfos`, `ElidedNode`, `child_append_relid_sets`, and `PlannerInfo.alternative_plan_name`.
- `src/test/modules/test_plan_advice/test_plan_advice.c`, the `contrib` `sql/` and `expected/` regression files (including `prepared`), and the `Makefile` `REGRESS` list.
- Core plan-cache policy in `src/backend/utils/cache/plancache.c` (`choose_custom_plan`) for the prepared-statement interaction.
- Same-checkout docs in `doc/src/sgml/pgplanadvice.sgml`.
- Full `git log` of `contrib/pg_plan_advice/`, `doc/src/sgml/pgplanadvice.sgml`, `src/test/modules/test_plan_advice`, and the core planner files listed in [Source Commit History](#source-commit-history) on the pinned `REL_19_STABLE` commit `01c544e1afb99bc2a76803870010b7cd2907f3b5`. The newest module commit is now `da8889ccd7e` (2026-07-06). In the `cdae794a..01c544e1` repin range, three commits touched `contrib/pg_plan_advice/`: the behavioral fix `89f5f860cc5` (FOREIGN_JOIN single-relation), its `ea203d371de` pgindent follow-up, and the tree-wide `da8889ccd7e` (`PG_MODULE_MAGIC_EXT`); all three are added to the direct-module list above.
- 2026-07-09 repin: upstream `master` had branched to `20devel` and the `REL_19_STABLE` branch was created, so `raw/postgres-19/` was repinned from the former post-`REL_19_BETA1` `master` pin `cdae794a` to the `REL_19_STABLE` tip `01c544e1` (a clean forward move; the old pin is an ancestor of `REL_19_STABLE`). Refreshed shifted `pg_plan_advice.c`/core-planner line anchors, grew the direct-module list from 22 to 25 (50 to 53 counting core and support), and reset `verified_by_agent` to `not yet` because a repin is not a claim-by-claim re-verification.
- 2026-06-26 claim review against the unchanged pin: rechecked the cited source ranges, direct module commit list, support commit list, core planner commit list, GUC/load scopes, prepared-plan regression evidence, and `pg_stash_advice` advisor integration. The only content correction found was the `91f33a2a` commit date (`2026-03-09`, not `2026-03-10`); the stale `pg_stash_advice` Related Pages cross-reference was replaced with direct source citations.
- 2026-06-29 commit-history review: the `raw/postgres-19/` checkout had been left at the prior pin `9a60f295` (which lacks `cdae794a`), so it was fetched from upstream `master` and checked out at the declared pin `cdae794af31b3e9cfc323fc654292d86fa746f77`. Re-verified all 50 listed commits (20 core, 22 module, 8 support) for hash, subject, author, and ancestry, and confirmed the exact 22-commit `contrib/pg_plan_advice/` completeness and the unchanged-since-`REL_19_BETA1` claim against the pin. Corrected the `47c110f7` date to its author date `2026-03-20` (the list uses author dates) and tightened the `b3a95566` repin-range note.

## Evidence Map

| Claim | Evidence |
|---|---|
| Two-way (generate + enforce), round-trip-safe design | [README#L3-L40](../../../raw/postgres-19/contrib/pg_plan_advice/README#L3-L40) |
| Loadable-module build shape | [Makefile#L3-L20](../../../raw/postgres-19/contrib/pg_plan_advice/Makefile#L3-L20) [meson.build#L34-L47](../../../raw/postgres-19/contrib/pg_plan_advice/meson.build#L34-L47) |
| Load methods (`LOAD`, `session_preload_libraries`, `shared_preload_libraries`) | [pgplanadvice.sgml#L33-L40](../../../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L33-L40) |
| `pgs_mask` bits and two enforcement styles | [pathnodes.h#L25-L97](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L25-L97) |
| Mask seeded from `enable_*` GUCs | [planner.c#L493-L524](../../../raw/postgres-19/src/backend/optimizer/plan/planner.c#L493-L524) |
| Core reads mask to gate join paths, with full-join exceptions | [joinpath.c#L153-L179](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L153-L179) [joinpath.c#L239-L244](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L239-L244) [joinpath.c#L353-L362](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L353-L362) |
| Five new planner hooks | [planner.h#L36-L47](../../../raw/postgres-19/src/include/optimizer/planner.h#L36-L47) [pathnode.h#L21-L48](../../../raw/postgres-19/src/include/optimizer/pathnode.h#L21-L48) [paths.h#L32-L38](../../../raw/postgres-19/src/include/optimizer/paths.h#L32-L38) |
| Individual index disabling for index-specific advice | [pgpa_planner.c#L1788-L1819](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1788-L1819) [pathnodes.h#L1427-L1428](../../../raw/postgres-19/src/include/nodes/pathnodes.h#L1427-L1428) [pathnode.c#L1126-L1131](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L1126-L1131) |
| Module installs hooks, GUCs, EXPLAIN option | [pg_plan_advice.c#L67-L139](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L67-L139) |
| Advisor hook chain, GUC fallback | [pg_plan_advice.c#L172-L228](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L172-L228) |
| `pg_stash_advice` feeds saved advice through the advisor hook | [pg_stash_advice.h#L4-L11](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.h#L4-L11) [pg_stash_advice.c#L141-L145](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.c#L141-L145) [pg_stash_advice.c#L153-L211](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.c#L153-L211) |
| Tag enum (20 tags) | [pgpa_ast.h#L80-L102](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h#L80-L102) |
| Relation identifier form | [README#L51-L79](../../../raw/postgres-19/contrib/pg_plan_advice/README#L51-L79) [pgpa_identifier.h#L19-L26](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_identifier.h#L19-L26) |
| Generation triggers and flow (walk + render) | [pgpa_planner.c#L211-L221](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L211-L221) [pgpa_planner.c#L299-L398](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L299-L398) [pg_plan_advice.c#L244-L254](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L244-L254) [pgpa_output.c#L79-L164](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_output.c#L79-L164) |
| Enforcement via scan/joinrel/join-path hooks | [pgpa_planner.c#L403-L622](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L403-L622) |
| `pgs_mask` helpers only clear bits | [pgpa_planner.c#L900-L933](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L900-L933) [pgpa_planner.c#L1137-L1149](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1137-L1149) [pgpa_planner.c#L1845-L1858](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1845-L1858) |
| Feedback flags + EXPLAIN output | [pg_plan_advice.h#L18-L45](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.h#L18-L45) [pgpa_planner.c#L1803-L1843](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1803-L1843) [pgpa_planner.c#L1866-L1897](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L1866-L1897) [pg_plan_advice.c#L346-L410](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L346-L410) |
| No plan-cache integration; advice read at plan time | [pg_plan_advice.c#L70-L79](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L70-L79) [pgpa_planner.c#L199-L227](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L199-L227) |
| Custom-vs-generic re-plan policy and invalidation (when advice re-applies) | [plancache.c#L14-L39](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L14-L39) [plancache.c#L1174-L1222](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L1174-L1222) [plancache.c#L2137-L2175](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L2137-L2175) [plancache.c#L2203-L2274](../../../raw/postgres-19/src/backend/utils/cache/plancache.c#L2203-L2274) |
| Prepared EXECUTE shows advice only with always_store | [pg_plan_advice.c#L376-L381](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L376-L381) [prepared.out#L16-L68](../../../raw/postgres-19/contrib/pg_plan_advice/expected/prepared.out#L16-L68) |
| Round-trip test harness | [test_plan_advice.c#L3-L17](../../../raw/postgres-19/src/test/modules/test_plan_advice/test_plan_advice.c#L3-L17) |

## Source References

- [pg_plan_advice.c](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c) — entry point, GUCs, advisor hooks, EXPLAIN output.
- [pg_plan_advice.h](../../../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.h) — feedback flags, advisor hook type, public prototypes.
- [pgpa_planner.c](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c) / [pgpa_planner.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.h) — planner hooks, enforcement, generation driver.
- [pgpa_ast.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_ast.h) / [pgpa_parser.y](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_parser.y) / [pgpa_scanner.l](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_scanner.l) — advice language.
- [pgpa_walker.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_walker.h) / [pgpa_scan.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_scan.h) / [pgpa_join.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_join.h) — plan analysis.
- [pgpa_trove.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_trove.h) / [pgpa_output.c](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_output.c) / [pgpa_identifier.h](../../../raw/postgres-19/contrib/pg_plan_advice/pgpa_identifier.h) — advice lookup, rendering, identifiers.
- [README](../../../raw/postgres-19/contrib/pg_plan_advice/README) — design rationale and language reference.
- [pgplanadvice.sgml](../../../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml) — user-facing docs for load methods and advice examples.
- [pg_stash_advice.h](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.h) / [pg_stash_advice.c](../../../raw/postgres-19/contrib/pg_stash_advice/pg_stash_advice.c) — sibling-module use of the public advisor hook.
- [pathnodes.h](../../../raw/postgres-19/src/include/nodes/pathnodes.h) / [plannodes.h](../../../raw/postgres-19/src/include/nodes/plannodes.h) / [primnodes.h](../../../raw/postgres-19/src/include/nodes/primnodes.h) / [planner.c](../../../raw/postgres-19/src/backend/optimizer/plan/planner.c) / [joinpath.c](../../../raw/postgres-19/src/backend/optimizer/path/joinpath.c) / [pathnode.c](../../../raw/postgres-19/src/backend/optimizer/util/pathnode.c) / [setrefs.c](../../../raw/postgres-19/src/backend/optimizer/plan/setrefs.c) / [extendplan.h](../../../raw/postgres-19/src/include/optimizer/extendplan.h) — core `pgs_mask`, hooks, extension state, final-plan metadata, disabled-node behavior, and index disable support.
- [plancache.c](../../../raw/postgres-19/src/backend/utils/cache/plancache.c) — `choose_custom_plan` custom-vs-generic policy governing when a prepared statement re-plans (and thus re-reads advice).
- [test_plan_advice.c](../../../raw/postgres-19/src/test/modules/test_plan_advice/test_plan_advice.c) and contrib regression tests ([syntax.sql](../../../raw/postgres-19/contrib/pg_plan_advice/sql/syntax.sql), [prepared.sql](../../../raw/postgres-19/contrib/pg_plan_advice/sql/prepared.sql)).

## Open Questions

- The README's own "Future Work" lists known gaps in the v19 feature: no control over aggregation strategy or sort order, no modeling of eager aggregation, no control over estimates (only outcomes, unlike `pg_hint_plan`), and unverified interaction with GEQO [README#Future-Work](../../../raw/postgres-19/contrib/pg_plan_advice/README#L250-L268). These are limitations of the shipped code, not of this page.
- Scope note: this page covers `pg_plan_advice`. The sibling `contrib/pg_stash_advice` module (advice persistence) has its own commit series and is not analyzed here.

## Related Pages

- [v19/index](../index.md) — PostgreSQL 19 version landing page.
- [versions](../../versions.md) — version index and source pin manifest.
- [wiki index](../../index.md) — global catalog.
