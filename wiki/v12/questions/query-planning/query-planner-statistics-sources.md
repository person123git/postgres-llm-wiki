---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Query Planner Statistics Sources in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Direct answer](#direct-answer)
  - [Mental model: the values the planner reads](#mental-model-the-values-the-planner-reads)
  - [What `pg_stat_all_tables` contains](#what-pg_stat_all_tables-contains)
  - [Core planner call path and data structures](#core-planner-call-path-and-data-structures)
  - [Direct statistics catalogs](#direct-statistics-catalogs)
  - [Relation size path](#relation-size-path)
  - [Single-column, inheritance, and expression-index statistics](#single-column-inheritance-and-expression-index-statistics)
  - [Extended statistics path](#extended-statistics-path)
  - [Extended statistics on inheritance trees and partitioned tables](#extended-statistics-on-inheritance-trees-and-partitioned-tables)
  - [Other catalog metadata that affects planning](#other-catalog-metadata-that-affects-planning)
  - [Missing, stale, inaccessible, and inconsistent statistics](#missing-stale-inaccessible-and-inconsistent-statistics)
  - [How planner statistics are written](#how-planner-statistics-are-written)
  - [The collector's indirect effect through auto-analyze](#the-collectors-indirect-effect-through-auto-analyze)
  - [Events that change the planner's inputs](#events-that-change-the-planners-inputs)
  - [Table AM, FDW, hooks, and extensions](#table-am-fdw-hooks-and-extensions)
  - [Build, generated-header, and cache boundaries](#build-generated-header-and-cache-boundaries)
  - [Regression coverage](#regression-coverage)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [Stages](#stages)
  - [Non-default settings in the measurement](#non-default-settings-in-the-measurement)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, does the query planner use stats from `pg_stat_all_tables`? Does it use stats from any other tables or catalogs?

## Answer

### Direct answer

No. PostgreSQL 12's core [planner](../../../glossary.md#planner) does not use the [cumulative statistics](../../../glossary.md#cumulative-statistics) counters exposed by `pg_stat_all_tables` as estimates for an unrelated query. That view calls `pg_stat_get_*()` functions, and those functions read the statistics collector's `PgStat_StatTabEntry`; the core base-relation path instead loads relation, column, extended-statistics, index, and foreign-key information into planner structures ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [pgstatfuncs.c#table-counter-functions](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L40-L360), [plancat.c#get_relation_info](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149), [plancat.c#catalog-loads](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L445)).

Yes, the planner uses [statistics](../../../glossary.md#statistics) from other [catalogs](../../../glossary.md#catalog):

- [`pg_class`](../../../glossary.md#pg_class) supplies stored relation-level estimates: [`relpages`, `reltuples`](../../../glossary.md#reltuples-and-relpages), and `relallvisible`. For ordinary heap tables, the table access method combines those values with the current physical block count ([pg_class.h#relation-statistics](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171)).
- [`pg_statistic`](../../../glossary.md#pg_statistic) supplies per-column and [expression-index](../../../glossary.md#expression-index) statistics ([pg_statistic.h#FormData_pg_statistic](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L29-L124), [selfuncs.c#column-lookup](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [selfuncs.c#expression-index-lookup](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4524-L4695)).
- `pg_statistic_ext` identifies multicolumn [extended statistics](../../../glossary.md#extended-statistics) objects, and `pg_statistic_ext_data` stores built ndistinct, functional-dependency, and multivariate-[MCV](../../../glossary.md#most-common-values-and-histogram) data ([pg_statistic_ext.h#FormData_pg_statistic_ext](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L33-L56), [pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43), [plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1377)). Unlike `pg_statistic`, these catalogs have no inheritance flag. On a plain-[inheritance](../../../glossary.md#inheritance) parent, the data describes the parent's own rows, but the planner can still apply it to a group-count estimate over the whole tree; see [Extended statistics on inheritance trees and partitioned tables](#extended-statistics-on-inheritance-trees-and-partitioned-tables).

`pg_stats` and `pg_stats_ext` are inspection views, not separate planner inputs. Their SQL definitions join and decode the underlying catalogs with privilege and row-security filtering; planner code uses the underlying catalogs through relcache and syscache lookups ([system_views.sql#pg_stats](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L189-L252), [system_views.sql#pg_stats_ext](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L256-L294), [selfuncs.c#column-lookup](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4756), [plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1377)).

That is the answer for statistical data. If "use" includes all metadata that can change available paths, selectivity, cardinality, or cost, the planner also consults index, constraint, type, operator, function, access-method, and partition metadata. Those inputs are distinct from cumulative monitoring statistics and are summarized below ([plancat.c#get_relation_info-indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L423), [plancat.c#selectivity-and-function-cost](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1760-L2015)).

### Mental model: the values the planner reads

The planner combines one live measurement with several stored values. A stored value is stale only when the data has changed since the last command that wrote it; nothing in the lookup path checks its age ([heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171), [selfuncs.c#column-lookup](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4724-L4737)).

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Current block count | Main-fork pages that exist now | `RelationGetNumberOfBlocks()` ([heapam_handler.c:2082-2083](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2083)) | Live | Becomes `RelOptInfo.pages`; the stored density is scaled to it to give `tuples` |
| `relpages`, `reltuples` | Page and row counts when last measured | `pg_class`, written by [`VACUUM`](../../../glossary.md#vacuum), `ANALYZE`, index builds, [`CLUSTER`](../../../glossary.md#cluster) and [`VACUUM FULL`](../../../glossary.md#vacuum-full), and reset to 0 by [`TRUNCATE`](../../../glossary.md#truncate); see [Events that change the planner's inputs](#events-that-change-the-planners-inputs) ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1135-L1196), [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680)) | Stored | Density `reltuples / relpages` for the size estimate |
| `relallvisible` | Pages marked all-visible in the [visibility map](../../../glossary.md#visibility-map) when last measured | `pg_class`, same writers | Stored, not scaled | `allvisfrac`, used by [index-only scan](../../../glossary.md#index-only-scan) costing ([costsize.c#index-only-allvisfrac](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L607-L664)) |
| `pg_statistic` row, `stainherit = false` | Column distribution of the table alone | `ANALYZE`, non-inherited pass | Stored | [Selectivity](../../../glossary.md#selectivity), distinct counts, width, correlation for a scan without children |
| `pg_statistic` row, `stainherit = true` | Column distribution of the whole inheritance tree | `ANALYZE`, inherited pass | Stored | Selectivity and distinct counts for a scan that includes children. Not width: `get_attavgwidth()` always reads the `stainherit = false` row ([lsyscache.c:2881-2890](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2881-L2890)), and an appendrel's widths are averaged from its children ([allpaths.c:1148-1187](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L1148-L1187)). Not correlation: that is read only in index costing, and a whole-tree parent loads no indexes ([plancat.c:154-160](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L154-L160)) |
| Expression-index `pg_statistic` row | Distribution of an index expression | `ANALYZE` of the table, stored under the index OID | Stored | Expression selectivity, and B-tree or BRIN correlation |
| `pg_statistic_ext_data` | ndistinct, dependencies, multivariate MCV for one object | `ANALYZE`, non-inherited pass only | Stored, no inheritance flag | Clause selectivity and group counts; group counts also for a whole-tree scan |
| `PgStat_StatTabEntry` | Activity counters, including `changes_since_analyze` | [Statistics collector](../../../glossary.md#cumulative-statistics) | Live, asynchronous | Not read by the planner; read by autovacuum to decide when to analyze |

Two values have similar names but different origins. `reltuples` comes from `pg_class`, written by maintenance commands, while `n_live_tup` in `pg_stat_all_tables` comes from the collector entry. The planner reads only the first ([heapam_handler.c:2086-2088](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2086-L2088), [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)). Autovacuum reads both: `reltuples` scales its threshold and `changes_since_analyze` is compared with it ([autovacuum.c#threshold-decision](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080)).

### What `pg_stat_all_tables` contains

`pg_stat_all_tables` joins `pg_class`, `pg_index`, and `pg_namespace`, then exposes scan counts, tuples read or changed, estimated live/dead tuples, modifications since analyze, timestamps, and maintenance counts by calling `pg_stat_get_*()` functions ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)).

Those functions fetch a `PgStat_StatTabEntry`. That collector entry stores scan and tuple counters, live/dead tuple estimates, `changes_since_analyze`, block counters, maintenance timestamps, and maintenance counts ([pgstatfuncs.c#table-counter-functions](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L40-L360), [pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-12/src/include/pgstat.h#L633-L662)). The monitoring documentation likewise defines the view as access and maintenance statistics for each table, rather than optimizer selectivity data ([monitoring.sgml#pg_stat_all_tables](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2704-L2845)).

The collector still matters indirectly: `changes_since_analyze` helps decide when [auto-analyze](../../../glossary.md#autovacuum) should run. Auto-analyze then refreshes the catalogs that the planner does read. The view itself is not the input to that decision; the view and autovacuum read the same underlying collector entry ([autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2927-L3095)).

There is also a reverse-direction monitoring edge. While estimating an inequality, whenever the histogram binary search touches the first or last [histogram](../../../glossary.md#most-common-values-and-histogram) entry, the planner tries to replace that endpoint with the column's current minimum or maximum. When the histogram has exactly two entries, it probes both ends before the search ([selfuncs.c:907-912](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L907-L912)). The probe is not conditional on the histogram being stale, although the monitoring documentation describes it in those terms; see [Open Questions](#open-questions). It scans a matching non-partial, non-hypothetical B-tree whose first column is the variable. The other range lookup, `get_variable_range()` for merge-join estimates, has this probe compiled out under `#ifdef NOT_USED` ([selfuncs.c:5145-5151](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5145-L5151)). The B-tree first-scan path increments that index's cumulative scan counter, and `pg_stat_all_tables.idx_scan` sums the counters of the table's indexes. Planning can therefore increase a counter exposed by the view even though it never reads that counter as an estimate ([selfuncs.c#histogram-endpoint-refresh](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L885-L935), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5250-L5417), [nbtsearch.c#planner-visible-index-scan-count](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L758-L770), [system_views.sql#pg_stat_all_tables-index-scans](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L561), [monitoring.sgml#optimizer-index-access](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2914-L2929)).

### Core planner call path and data structures

For an ordinary planned query, `query_planner()` creates base `RelOptInfo` nodes, matches usable foreign keys, and calls `make_one_rel()`; `build_simple_rel()` dispatches an `RTE_RELATION` to `get_relation_info()` ([planmain.c#query_planner](../../../../raw/postgres-12/src/backend/optimizer/plan/planmain.c#L153-L271), [relnode.c#build_simple_rel](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L175-L293)). The normal path is:

```text
query_planner
  -> add_base_rels_to_query
     -> build_simple_rel
        -> get_relation_info
           -> estimate_rel_size
           -> load IndexOptInfo list
           -> load StatisticExtInfo list
           -> load relevant ForeignKeyOptInfo entries
  -> match_foreign_keys_to_quals
  -> make_one_rel
     -> set_baserel_size_estimates
        -> clauselist_selectivity
           -> extended statistics where applicable
           -> ordinary clause selectivity for the remainder
```

The source establishes those boundaries at the base-relation build, catalog load, foreign-key match, base-row estimate, and clause-selectivity calls ([planmain.c#base-relations-through-make_one_rel](../../../../raw/postgres-12/src/backend/optimizer/plan/planmain.c#L153-L271), [plancat.c#get_relation_info](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149), [plancat.c#index-stats-FK-loads](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L445), [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4390-L4423), [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L53-L106)).

The key planner-side structures are:

| Structure | Statistics-related role |
|---|---|
| [`RelOptInfo`](../../../glossary.md#reloptinfo) | Holds `pages`, `tuples`, `allvisfrac`, cached attribute widths, `indexlist`, `statlist`, and FDW state ([pathnodes.h#RelOptInfo](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L632-L723)). |
| `VariableStatData` | Carries the matched variable or expression, relation, `pg_statistic` tuple, release callback, type data, uniqueness knowledge, and statistics-access result ([selfuncs.h#VariableStatData](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L65-L84)). |
| `IndexOptInfo` | Holds index size, tuple count, B-tree height, keys, expressions, predicate, uniqueness, opfamilies, and AM costing capabilities ([pathnodes.h#IndexOptInfo](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L781-L835)). |
| `StatisticExtInfo` | Identifies one built extended-statistics kind and the columns it covers; one object can produce zero to three entries ([pathnodes.h#StatisticExtInfo](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L868-L882), [plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1332-L1367)). |
| `ForeignKeyOptInfo` | Holds FK columns/operators plus their matches to equivalence classes and join clauses ([pathnodes.h#ForeignKeyOptInfo](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L838-L865)). |

### Direct statistics catalogs

| Source | Direct use in PostgreSQL 12 |
|---|---|
| `pg_class` | Prior page/tuple density and all-visible-page count. The current physical block count is obtained separately, and the heap AM scales the stored density to it ([pg_class.h#relation-statistics](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171)). |
| `pg_statistic` | Null fraction, average width, distinct estimate, and five extensible slots containing MCV, histogram, correlation, element, or range statistics ([pg_statistic.h#fields-and-slots](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L29-L126), [pg_statistic.h#core-slot-kinds](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L136-L271)). |
| `pg_statistic_ext` | Object OID, source relation, key columns, and requested kinds ([pg_statistic_ext.h#FormData_pg_statistic_ext](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L33-L69)). |
| `pg_statistic_ext_data` | Serialized multivariate ndistinct, functional dependencies, and multivariate MCV list ([pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43)). |
| Current relation storage | Current main-fork block count for heap and index relations; it is not a catalog statistic ([heapam_handler.c#current-block-count](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2088), [plancat.c#index-block-count](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407)). |

The PostgreSQL 12 planner-statistics documentation matches this split: relation row/page totals are in `pg_class`, per-column selectivity data is in `pg_statistic`, and `pg_stats` is the recommended human-readable view for inspection ([perform.sgml#single-column-statistics](../../../../raw/postgres-12/doc/src/sgml/perform.sgml#L920-L1059)).

### Relation size path

`get_relation_info()` calls `estimate_rel_size()` for a non-inheritance-parent base relation and stores the result in `RelOptInfo.pages`, `tuples`, and `allvisfrac` ([plancat.c#get_relation_info-size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L105-L152)). For a heap table, the heap AM:

1. reads the current physical block count;
2. reads `pg_class.relpages`, `reltuples`, and `relallvisible` from the relation descriptor;
3. applies a ten-page floor to a small relation whose stored `relpages` is zero and which has no children;
4. scales the old tuple density to the current pages, or derives a fallback density from attribute widths when no prior page count exists; and
5. converts `relallvisible` to a fraction for index-only-scan costing ([heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171), [costsize.c#index-only-allvisfrac](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L607-L664)).

Indexes take a different path. `get_relation_info()` handles an ordinary non-partial index by taking its current physical block count and assigning the parent table's tuple estimate. Only a partial index enters `estimate_rel_size()` to derive an index tuple estimate from its own stored density, after which the result is clamped to the table estimate ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407)). On that partial-index path, the generic estimator discounts one presumed metapage when deriving density; its own comment says that heuristic is suitable for B-tree, hash, and GIN but suspect for GiST ([plancat.c#estimate_rel_size-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L946-L1040)).

An inheritance or partition parent does not use its own physical size as the append relation's size. `get_relation_info()` skips the parent estimate, and `set_append_rel_size()` computes child sizes, sums surviving child output rows, weights widths by child rows, and leaves the parent `pages` at zero to avoid double counting ([plancat.c#inheritance-parent-boundary](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L105-L149), [allpaths.c#set_append_rel_size](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L930-L1039), [allpaths.c#append-size-result](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L1130-L1208)).

The generic size dispatcher returns fixed `1 page / 1 tuple` values for a sequence, initial `pg_class` values for a foreign table, and zero for storage-less relation kinds. Foreign-table estimates are then handed to the FDW, as described below ([plancat.c#estimate_rel_size-relkinds](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1042-L1059), [allpaths.c#set_foreign_size](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L899-L914)). A temporary or unlogged relation is rejected during recovery before these estimates are used ([plancat.c#recovery-error](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L119-L130)).

### Single-column, inheritance, and expression-index statistics

For a plain table column, `examine_simple_variable()` uses the `STATRELATTINH` syscache key `(relation OID, attribute number, rte->inh)`. The `stainherit` key therefore selects table-only statistics for a non-inheritance scan and inheritance-tree statistics for an inherited scan ([selfuncs.c#examine_simple_variable](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [pg_statistic.h#statistics-key](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L29-L35), [syscache.c#STATRELATTINH](../../../../raw/postgres-12/src/backend/utils/cache/syscache.c#L764-L773)).

`pg_statistic` has five physical slots, but a reader searches by `stakind` rather than slot position. Core v12 kinds include MCV, histogram, physical-order correlation, most-common elements, distinct-element-count histogram, range-length histogram, and range-bounds histogram; custom type analysis and selectivity code can agree on additional kind numbers ([pg_statistic.h#slot-contract](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L136-L178), [pg_statistic.h#core-slot-kinds](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L178-L271), [lsyscache.c#get_attstatsslot](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2895-L2965)). For example, B-tree costing reads the first key's correlation slot from the table column or expression-index row and reduces its weight for a multicolumn index ([selfuncs.c#btcostestimate-correlation](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6118-L6213)).

Expression-index statistics are also `pg_statistic` rows, keyed by the index relation OID and index attribute number. During general expression selectivity, `examine_variable()` matches a single-relation expression to an index expression and reads that index's statistics tuple ([selfuncs.c#expression-index-match](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4524-L4604)). That path ignores a [partial index](../../../glossary.md#partial-index)'s expression statistics because they do not describe the whole table; an installed `get_index_stats_hook` must decide for itself whether such statistics are valid ([selfuncs.c#partial-expression-index-boundary](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4574-L4604)). This is not a universal ban: B-tree costing directly looks up the first expression key's correlation row, and BRIN costing does the same for expression columns used by its index clauses. Both can therefore use correlation collected over a partial index's predicate-matching sample ([selfuncs.c#btcostestimate-correlation](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6118-L6213), [selfuncs.c#brincostestimate-correlation](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L7045-L7137), [analyze.c#compute_index_stats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L693-L867)).

`ANALYZE` only prepares separate index-relation statistics for expression columns. A simple index key continues to use the underlying table column's statistics; index setup calls `examine_attribute()` only where the index key number is zero, meaning an expression ([analyze.c#index-expression-setup](../../../../raw/postgres-12/src/backend/commands/analyze.c#L413-L467)). The expression values are evaluated over sampled heap rows that pass the index predicate, and completed rows are written with the index OID as `starelid` ([analyze.c#compute_index_stats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L693-L867), [analyze.c#write-index-statistics](../../../../raw/postgres-12/src/backend/commands/analyze.c#L560-L574)).

Average-width lookup first offers `get_attavgwidth_hook`, then reads `pg_statistic.stawidth`; if neither supplies a positive width, callers fall back to a datatype/typmod estimate from `pg_type` metadata ([lsyscache.c#get_attavgwidth](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2860-L2893), [plancat.c#get_rel_data_width](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1064-L1111), [lsyscache.c#get_typavgwidth](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2347-L2398)).

### Extended statistics path

`RelationGetStatExtList()` scans `pg_statistic_ext` by source relation and caches the object OIDs in the [relcache](../../../glossary.md#relcache). `get_relation_statistics()` then looks up both catalog tuples and creates one `StatisticExtInfo` for each non-NULL built kind. A newly created but not yet analyzed object contributes no planner entry ([relcache.c#RelationGetStatExtList](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4443-L4523), [plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1377)).

For restriction clauses on one relation, `clauselist_selectivity()` estimates compatible clauses with extended statistics first and sends the remaining clauses through ordinary selectivity. The extended path applies a multivariate MCV list before functional dependencies ([clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L53-L106), [extended_stats.c#statext_clauselist_selectivity](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L1169-L1202)). Group-count estimation separately searches for the multivariate ndistinct object matching the most grouping columns and loads its ndistinct data ([selfuncs.c#estimate_num_groups](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3227-L3265), [selfuncs.c#estimate_multivariate_ndistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3562-L3643)).

PostgreSQL 12 extended statistics accept at least two simple user columns whose types have default B-tree ordering. Expressions, system columns, duplicate columns, unsupported types, and excessive dimensions are rejected before catalog insertion ([statscmds.c#extended-statistics-columns](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L182-L273)). `ANALYZE` builds ndistinct, dependency, and MCV data from the same sampled rows as the per-column statistics and writes it to `pg_statistic_ext_data` ([extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L75-L157), [extended_stats.c#statext_store](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L315-L387)).

### Extended statistics on inheritance trees and partitioned tables

Single-column statistics keep a table-only row and an inheritance-tree row apart through `stainherit`. Extended statistics in PostgreSQL 12 have no such flag: `pg_statistic_ext_data` holds one row per object ([pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43)). Three source facts combine:

1. `ANALYZE` builds extended statistics only in the non-inherited pass, from the parent's own rows ([analyze.c:576-584](../../../../raw/postgres-12/src/backend/commands/analyze.c#L576-L584)).
2. `get_relation_info()` loads the object list for every relation, including an inheritance parent planned as the whole tree (`inhparent = true`) ([plancat.c:430](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L430)).
3. The consumers check only that `statlist` is non-empty, not the scan's `inh` flag. `estimate_multivariate_ndistinct()` returns early only for an empty list ([selfuncs.c#estimate_multivariate_ndistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3573-L3585)), and `clauselist_selectivity()` requires only `RTE_RELATION` and a non-empty list ([clausesel.c:84-97](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L84-L97)).

Therefore, for a plain-inheritance parent that has rows of its own, a group-count estimate over the whole tree can use ndistinct data that describes only the parent. The two cases differ along the same dimensions:

| Relation | `ANALYZE` passes ([analyze.c:255-268](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268)) | Extended data built from | Whole-tree group count uses it? |
|---|---|---|---|
| Plain table, no children | Non-inherited only | A sample of its rows | Not applicable |
| Plain-inheritance parent with children | Non-inherited, then inherited | A sample of the parent's own rows only; nothing if the parent has no rows of its own | Yes when built, measured below |
| Partitioned table | Inherited only, because it has no rows | Nothing: the object stays unbuilt | No: an unbuilt kind yields no `StatisticExtInfo` ([plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1324-L1367)) |

`CREATE STATISTICS` accepts a partitioned table as its target ([statscmds.c:120-125](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L120-L125)), so creating such an object succeeds even though `ANALYZE` never fills it.

**Measured example.** The [measurement script](#measurement-script) builds a parent `ih` whose 30,000 own rows have `b = a` with 100 values, so `(a, b)` has 100 combinations. Its child `ih_c` has 30,000 rows where `a` and `b` vary independently, giving 10,000 combinations that include the parent's 100. The script runs `EXPLAIN (ANALYZE) SELECT a, b FROM … GROUP BY a, b` before and after `CREATE STATISTICS ih_s (ndistinct) ON a, b FROM ih` and a new `ANALYZE ih`:

| Case | Scan | Estimated groups (measured) | Actual groups (measured) |
|---|---|---|---|
| `ih_only_before` | `ONLY ih` | 3000 | 100 |
| `ih_tree_before` | `ih` and `ih_c` | 6000 | 10000 |
| `ih_only_after` | `ONLY ih` | 100 | 100 |
| `ih_tree_after` | `ih` and `ih_c` | 100 | 10000 |

The built object stored `{"1, 2": 100}`. For the partitioned table `pt`, the same object definition stayed unbuilt after `ANALYZE pt` (`stxdndistinct IS NULL`), while `ANALYZE pt` still wrote 2 inherited `pg_statistic` rows.

**Worked calculation.** Without an extended object, `estimate_num_groups()` multiplies the per-column distinct counts, then clamps the product. The clamp is the relation's `tuples`, or one tenth of it when more than one value was multiplied. That tenth is never set below the largest single distinct count, or above `tuples` ([selfuncs.c:3275-3295](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3275-L3295)). Here every single count is 100, far below a tenth of `tuples`, so the floor does not bind. For the parent alone, `tuples` is 30,000, so `100 × 100 = 10000` is clamped to `0.1 × 30000 = 3000`. For the tree, the appendrel's `tuples` is the sum of the children's rows, 60,000 ([allpaths.c:1189-1193](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L1189-L1193)), so the clamp gives 6,000. With the object, one ndistinct match covers both columns. The estimate becomes that single value, 100. Only one value was multiplied, so the clamp is `tuples` itself rather than a tenth of it, and 100 is below it ([selfuncs.c:3240-3249](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3240-L3249), [selfuncs.c:3283](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3283)). The result is right for `ONLY ih` and 100 times too low for the tree. The measurement confirms what the source reading predicts. It does not show how often real schemas meet this case.

This is non-default state. By default no statistics object exists, so the planner uses only per-column statistics and the clamp. The cause here is a `CREATE STATISTICS` object on an inheritance parent that holds rows of its own; the mechanism is the three steps above.

### Other catalog metadata that affects planning

The following are not cumulative monitoring statistics, but they can change planner estimates, path eligibility, or cost:

| Catalog or metadata | Planner effect |
|---|---|
| `pg_index` plus index relcache data | Determines live/valid indexes, key columns, expressions, predicates, uniqueness, ordering, opfamilies, physical pages, partial-index tuple estimates, and B-tree height ([relcache.c#RelationGetIndexList](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4318-L4399), [plancat.c#get_relation_info-indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L423)). A proven unique index can override stale or missing distinct-count statistics for equality estimates ([selfuncs.c#unique-override](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5070-L5079)). |
| `pg_constraint` and constraint relcache data | Foreign keys can replace ordinary join-clause selectivity after matching all required equality clauses; validated immutable CHECK constraints can prove a relation scan unnecessary ([plancat.c#get_relation_foreign_keys](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L465-L566), [costsize.c#get_foreign_key_join_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4693-L4906), [plancat.c#get_relation_constraints](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1136-L1287), [plancat.c#relation_excluded_by_constraints](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1380-L1562)). |
| `pg_attribute` and partition relcache data | Attribute type, typmod, collation, dropped-column state, and `attnotnull` affect width and constraint reasoning. Partition constraints come from `RelationGetPartitionQual()`, not from `pg_constraint` ([plancat.c#get_rel_data_width](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1077-L1111), [plancat.c#NOT-NULL-and-partition-constraints](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1224-L1281)). |
| `pg_type` | `typlen` and typmod-aware rules provide fallback widths, while `typanalyze` selects type-specific statistics collection ([pg_type.h#type-width-fields](../../../../raw/postgres-12/src/include/catalog/pg_type.h#L50-L67), [pg_type.h#typanalyze](../../../../raw/postgres-12/src/include/catalog/pg_type.h#L123-L159), [analyze.c#type-specific-analysis](../../../../raw/postgres-12/src/backend/commands/analyze.c#L934-L973)). |
| `pg_operator` | `oprrest` and `oprjoin` select the restriction and join selectivity procedures; a missing procedure yields the generic `0.5` estimate ([pg_operator.h#selectivity-procedures](../../../../raw/postgres-12/src/include/catalog/pg_operator.h#L63-L76), [plancat.c#operator-selectivity](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1760-L1837)). |
| `pg_proc` | `prosupport` may supply selectivity, cost, or row estimates; otherwise `procost` prices function/operator execution and `prorows` estimates set-returning output. `provolatile` constrains constant treatment, `proparallel` constrains parallel paths, and `proleakproof` participates in statistics-value security checks ([pg_proc.h#planner-fields](../../../../raw/postgres-12/src/include/catalog/pg_proc.h#L44-L78), [plancat.c#function-selectivity-cost-rows](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1840-L2015), [costsize.c#function-cost-use](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3909-L3962), [clauses.c#function-volatility](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L632-L654), [clauses.c#parallel-safety](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L895-L940), [selfuncs.c#statistic_proc_security_check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4961-L4982)). |
| `pg_am`, opclass, opfamily, `pg_amop`, and `pg_amproc` data | Relcache resolves an access-method handler, opfamilies, opclass input types, and support functions. `get_relation_info()` copies search, ordering, parallel, bitmap, and `amcostestimate` capabilities into `IndexOptInfo` ([relcache.c#index-AM-and-opclass-load](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1380-L1568), [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L159-L233), [plancat.c#index-AM-capabilities](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L257-L357), [lsyscache.c#opfamily-lookups](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L124-L217)). |

This table is deliberately scoped to metadata that directly affects estimates, costs, or path eligibility; it is not an exhaustive inventory of every catalog consulted during parsing, rewriting, and planning.

### Missing, stale, inaccessible, and inconsistent statistics

PostgreSQL 12 does not fall back from missing `pg_statistic` data to `pg_stat_all_tables`. It uses estimator defaults and structural facts instead. Representative defaults are `0.005` for equality, `1/3` for inequality, `0.005` for pattern matches and for `IS NULL`/unknown, the complementary `0.995` for the corresponding `NOT` tests, and `200` distinct values ([selfuncs.h#default-estimates](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L30-L49)). Equality estimation with no usable statistics divides by the inferred/default distinct count, while null tests use their fixed defaults; unique-index knowledge and special system-column/type cases can still improve those guesses ([selfuncs.c#equality-without-stats](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L385-L425), [selfuncs.c#nulltestsel-fallback](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1450-L1519), [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4985-L5117)).

`pg_class` values are explicitly approximate and not updated for every row change. The heap path mitigates staleness by scaling old tuple density to the current physical page count; `pg_statistic` tuples have no age field or freshness test in the lookup path, so stale distributions remain inputs until statistics are refreshed ([pg_class.h#relation-statistics](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [heapam_handler.c#density-scaling](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2171), [selfuncs.c#column-lookup](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [perform.sgml#stale-relation-statistics](../../../../raw/postgres-12/doc/src/sgml/perform.sgml#L960-L973)).

Statistics values also have a security boundary. The planner loads the tuple, but non-[leakproof](../../../glossary.md#leakproof-function) estimator functions may consume its value arrays only when the effective user can select the table/column and no [security-barrier](../../../glossary.md#security-barrier) or [row-security](../../../glossary.md#row-level-security) qual blocks access. The null fraction, distinct count and average width can still be used, because they expose no column values; see the [cases table](#missing-stale-inaccessible-and-inconsistent-statistics) ([selfuncs.c#statistics-access-check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4739-L4838), [selfuncs.c#nullfrac-and-security-check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L298-L330), [selfuncs.c#statistic_proc_security_check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4961-L4982)). Expression-index statistics require whole-table SELECT permission and no security quals, because the core does not try to derive the expression's exact column set ([selfuncs.c#expression-statistics-access](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4606-L4634)). When the index belongs to an inheritance child that the user cannot read, the check is repeated on the topmost parent named in the query ([selfuncs.c#expression-statistics-parent-check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4636-L4678)). Extended-statistics MCV clauses likewise require table or referenced-column SELECT permission, and a security qual rejects a leaky comparison operator while allowing a leakproof one ([extended_stats.c#extended-statistics-security](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L814-L867), [extended_stats.c#extended-statistics-ACL](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L942-L998)).

An unbuilt extended-statistics value is normally skipped: `statext_is_kind_built()` tests for NULL before `StatisticExtInfo` creation. Missing catalog tuples are treated as internal consistency failures and raise `ERROR`; the individual loaders also error if asked to load a NULL kind ([plancat.c#get_relation_statistics-errors-and-built-checks](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1315-L1367), [extended_stats.c#statext_is_kind_built](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L159-L187), [mvdistinct.c#statext_ndistinct_load](../../../../raw/postgres-12/src/backend/statistics/mvdistinct.c#L137-L165), [dependencies.c#statext_dependencies_load](../../../../raw/postgres-12/src/backend/statistics/dependencies.c#L629-L657), [mcv.c#statext_mcv_load](../../../../raw/postgres-12/src/backend/statistics/mcv.c#L554-L582)). The v12 MCV loader's NULL-data error mistakenly prints the dependency kind character rather than the MCV kind; this affects only that error text, not the normal built-kind guard or selected catalog column ([mcv.c#statext_mcv_load](../../../../raw/postgres-12/src/backend/statistics/mcv.c#L554-L582)).

Operator and function support code validates extension-provided estimates. A restriction or join estimator returning outside `[0,1]`, or a function support routine returning an invalid selectivity, raises `ERROR` rather than admitting that value into planning ([plancat.c#selectivity-validation](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1769-L1895)).

The cases in this section, with the sections above that they depend on:

| Condition / state | Behavior | Consequence |
|---|---|---|
| No `pg_statistic` row for the column | Estimators use the fixed defaults, distinct-count inference, and unique-index knowledge | Never a fallback to `pg_stat_all_tables` counters |
| Row exists, but the user may not read every row of the column | Non-leakproof estimators cannot read the slot values. The null fraction, the distinct count and the average width are still used, because `get_variable_numdistinct()` and `get_attavgwidth()` read them without a permission check ([selfuncs.c:5011-5019](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5011-L5019), [selfuncs.c:328-331](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L328-L331), [selfuncs.c:420-424](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L420-L424), [lsyscache.c:2881-2890](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2881-L2890)) | Equality falls back to `1 / ndistinct` rather than the MCV list; leakproof operators still see the slots |
| Expression statistics of a partial index | `examine_variable()` ignores them unless a hook decides otherwise | B-tree and BRIN correlation lookups still read them |
| `relpages = 0`, fewer than 10 current pages, no children | Heap size estimate assumes 10 pages | A new or never-vacuumed table is not treated as tiny |
| Extended object created but not yet analyzed | No `StatisticExtInfo` entry | Ordinary per-column selectivity applies |
| Extended object on a plain-inheritance parent with rows of its own | Whole-tree group-count estimates can use the parent-only data | See [Extended statistics on inheritance trees and partitioned tables](#extended-statistics-on-inheritance-trees-and-partitioned-tables) |
| Extended catalog tuple missing, or a loader asked for an unbuilt kind | `ERROR` | Internal-consistency failure, not a silent fallback |
| Estimator or support function returns a selectivity outside `[0,1]` | `ERROR` | The value never enters planning |

### How planner statistics are written

For an ordinary heap table, `ANALYZE` takes an approximate random sample. Its two-stage algorithm samples blocks and then applies Vitter reservoir sampling to rows; it extrapolates live/dead row totals from the sampled blocks ([analyze.c#acquire_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L976-L1127)). Type-specific analysis computes `VacAttrStats`, whose valid fields become a `pg_statistic` row through `CatalogTupleUpdate` or `CatalogTupleInsert` ([vacuum.h#VacAttrStats](../../../../raw/postgres-12/src/include/commands/vacuum.h#L44-L137), [analyze.c#update_attstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1394-L1547)).

The same sampled heap rows feed expression-index statistics and extended statistics. A pass writes nothing to `pg_statistic` or `pg_statistic_ext_data` when its sample is empty, so an empty table, or an inheritance parent with no rows of its own, keeps its old or missing statistics from that pass ([analyze.c:512](../../../../raw/postgres-12/src/backend/commands/analyze.c#L512)). The sample holds at most 300 × the largest column statistics target rows ([analyze.c:474-481](../../../../raw/postgres-12/src/backend/commands/analyze.c#L474-L481), [analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700)). Otherwise, what one pass of `do_analyze_rel()` writes depends on whether it is the non-inherited pass or the inherited pass over a parent and its children ([analyze.c#statistics-write-path](../../../../raw/postgres-12/src/backend/commands/analyze.c#L506-L640), [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L75-L157), [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1135-L1196)):

| Write | Non-inherited pass | Inherited pass |
|---|---|---|
| Table-column `pg_statistic` rows | Yes, `stainherit = false` | Yes, `stainherit = true` ([analyze.c:565-566](../../../../raw/postgres-12/src/backend/commands/analyze.c#L565-L566)) |
| Expression-index `pg_statistic` rows | Yes, unless the command named a column list ([analyze.c:439](../../../../raw/postgres-12/src/backend/commands/analyze.c#L439), [analyze.c:568-574](../../../../raw/postgres-12/src/backend/commands/analyze.c#L568-L574)) | No: the inherited pass opens no indexes ([analyze.c:414-426](../../../../raw/postgres-12/src/backend/commands/analyze.c#L414-L426)) |
| `pg_statistic_ext_data` | Yes ([analyze.c:576-584](../../../../raw/postgres-12/src/backend/commands/analyze.c#L576-L584)) | No |
| Table `pg_class` `relpages`, `reltuples`, `relallvisible` | Yes ([analyze.c:587-605](../../../../raw/postgres-12/src/backend/commands/analyze.c#L587-L605)) | No |
| Index `pg_class` rows | Yes, unless this `ANALYZE` is part of `VACUUM ANALYZE`, whose vacuum step already counted the indexes ([analyze.c:607-629](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)) | No |
| Report to the collector | Yes; it resets `changes_since_analyze` only when every column was analyzed ([analyze.c#pgstat_report_analyze](../../../../raw/postgres-12/src/backend/commands/analyze.c#L631-L640)) | No, because the collector tracks only per-table counts |

`ANALYZE` is not the only writer of `pg_class` size statistics. Lazy `VACUUM` writes the table's `relpages`, `reltuples`, and `relallvisible`, and writes each index's counts when the index AM reports an exact count ([vacuumlazy.c:341-363](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L341-L363), [vacuumlazy.c:1803-1815](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1803-L1815)). Every index build, which `CREATE INDEX`, `CREATE INDEX CONCURRENTLY`, and `REINDEX` all reach through `index_build()`, also rewrites both the heap's and the new index's rows through `index_update_stats()` ([index.c:1221](../../../../raw/postgres-12/src/backend/catalog/index.c#L1221), [index.c:1427](../../../../raw/postgres-12/src/backend/catalog/index.c#L1427), [index.c:3530](../../../../raw/postgres-12/src/backend/catalog/index.c#L3530), [index.c:2977-2986](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986), [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680)). `CLUSTER` and `VACUUM FULL` write them from the rewrite, and `TRUNCATE` resets them to 0; see [Events that change the planner's inputs](#events-that-change-the-planners-inputs).

Creating an extended-statistics object inserts metadata into `pg_statistic_ext` and a `pg_statistic_ext_data` row whose three data fields are initially NULL. Only a later `ANALYZE` builds them ([statscmds.c#create-statistics-data-row](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L340-L376), [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L75-L157)). If any requested column was omitted from a partial `ANALYZE` or could not produce per-column statistics, extended-statistics construction warns outside autovacuum and skips rebuilding that object. A new object therefore remains unbuilt; this branch does not clear data from an earlier successful build, so an existing object retains its previous values ([extended_stats.c#missing-column-statistics](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L101-L127), [extended_stats.c#lookup_var_attr_stats](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L259-L313), [extended_stats.c#statext_store](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L315-L387)).

### The collector's indirect effect through auto-analyze

`relation_needs_vacanalyze()` computes the analyze threshold from a base threshold plus a scale factor times `pg_class.reltuples`, then compares it with `PgStat_StatTabEntry.changes_since_analyze`. By default `autovacuum` and `track_counts` are both `on` ([guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1400), [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433)). Threshold-based auto-analyze does not run for a table in these cases:

| Condition | Default | Mechanism |
|---|---|---|
| No collector entry for the table | An entry exists once the table has been used | The threshold branch requires `tabentry` ([autovacuum.c:3060](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060), [autovacuum.c:3082-3090](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3082-L3090)) |
| `autovacuum` or `track_counts` is `off` | Both `on` | `AutoVacuumingActive()` is false, so the threshold branch is skipped ([autovacuum.c#AutoVacuumingActive](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3206-L3212)) |
| The table's `autovacuum_enabled` [storage parameter](../../../glossary.md#storage-parameter) is `false`, and the table is not at wraparound risk | `true` when unset | An early return clears `doanalyze`. At wraparound risk the return is skipped and the threshold branch can still set `doanalyze` ([autovacuum.c:3026](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3026), [autovacuum.c:3045-3051](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3045-L3051)) |
| The table is partitioned | — | The worker's catalog scan keeps only ordinary tables and materialized views ([autovacuum.c:2065-2067](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2065-L2067)) |

The causal chain is therefore:

```text
row changes
  -> collector changes_since_analyze
  -> auto-analyze threshold decision
  -> ANALYZE sample
  -> pg_class / pg_statistic / pg_statistic_ext_data refresh
  -> later plans consume refreshed catalog data
```

Each arrow is present in the collector threshold and `ANALYZE` write paths ([autovacuum.c#collector-threshold](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2932-L2955), [autovacuum.c#threshold-decision](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3095), [analyze.c#statistics-write-path](../../../../raw/postgres-12/src/backend/commands/analyze.c#L506-L640)). This is an indirect influence of cumulative statistics, not planner consumption of `pg_stat_all_tables`.

Inheritance statistics have a separate freshness edge, which depends on the kind of parent:

- **Plain-inheritance parent.** `ANALYZE` of a table with children runs both passes and writes the inherited result with `stainherit = true` ([analyze.c:255-268](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268), [analyze.c#acquire_inherited_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1169-L1390)). Auto-analyze decides from the parent's own `changes_since_analyze`, so changes only in the children do not trigger it. The documentation therefore warns that a manual `ANALYZE` can be needed ([analyze.sgml#inheritance-statistics](../../../../raw/postgres-12/doc/src/sgml/ref/analyze.sgml#L253-L263)).
- **Partitioned table.** Autovacuum never considers it, because its scan keeps only `relkind` `r` and `m` ([autovacuum.c:2065-2067](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2065-L2067)). Its inherited statistics, the only `pg_statistic` rows it can have, are written only by a manual `ANALYZE`: one that names the partitioned table, or a database-wide `ANALYZE`, whose relation list includes partitioned tables ([vacuum.c:834-837](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L834-L837)). A partitioned table also has no row in `pg_stat_all_tables`, whose view keeps `relkind` `r`, `t`, and `m` ([system_views.sql:580](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L580)).

The [measurement script](#measurement-script) shows both. After `autovacuum` was turned on with a 1 s naptime, auto-analyze reached `aa_inh`, `aa_inh_c`, `aa_part1`, and `aa_part2` within about 2 s, and wrote 1 inherited `pg_statistic` row for the plain parent `aa_inh`. It wrote no row for the partitioned table `aa_part`, and `pg_stat_all_tables` had no row for `aa_part`.

### Events that change the planner's inputs

| Event | State before | State after | Effect on later planning |
|---|---|---|---|
| Rows inserted, updated, or deleted | Stored statistics describe the earlier data | Catalog statistics unchanged; the collector's counters and the current block count change | The size estimate follows the new block count at the old density; distributions stay as stored until the next writer runs ([heapam_handler.c#density-scaling](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2171)) |
| `ANALYZE` (manual or auto) | Old or missing statistics | New `pg_statistic`, extended, and `pg_class` data, as in the [pass table](#how-planner-statistics-are-written) | Plans made afterwards read the new values |
| Lazy `VACUUM` | Old `relpages`, `reltuples`, `relallvisible` | Updated table, and exactly counted index, `pg_class` rows | New density and all-visible fraction ([vacuumlazy.c:341-363](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L341-L363)) |
| Index build (`CREATE INDEX`, `REINDEX`) | Old heap counts; no index row | Heap and index `pg_class` rows rewritten | New index size inputs and heap density ([index.c:2977-2986](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986)) |
| `CLUSTER` or `VACUUM FULL` | Old heap counts | The new heap's `relpages` and `reltuples` are written from the rewrite, then swapped into the table's row ([cluster.c:950-959](../../../../raw/postgres-12/src/backend/commands/cluster.c#L950-L959), [cluster.c:1136-1147](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1136-L1147)) | Size estimate uses the rewritten table's density |
| `TRUNCATE` that assigns a new relfilenode, the usual case | Old counts | `relpages`, `reltuples` and `relallvisible` reset to 0 ([tablecmds.c:1819](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1819), [relcache.c:3529-3535](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3529-L3535)) | `relpages = 0` re-arms the 10-page floor and the width-based density fallback. A table created or given a new relfilenode in the same subtransaction is truncated in place without this reset ([tablecmds.c:1786-1798](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1798)) |
| `CREATE STATISTICS` | No object | Object plus a data row whose kinds are NULL; relation's relcache entry invalidated | No planner entry until `ANALYZE` builds a kind ([statscmds.c#create-statistics-data-row](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L340-L378)) |
| `DROP STATISTICS` | Object listed in relcache | Object gone; relation invalidated | The list is rebuilt without it ([statscmds.c#drop-statistics-invalidation](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L417-L466)) |
| Collector counter reset or missing entry | Counters present | No `tabentry` | Auto-analyze stops triggering for the table until an entry exists again; planner inputs are not affected ([autovacuum.c:3082-3090](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3082-L3090)) |

### Table AM, FDW, hooks, and extensions

The heap formula is not mandatory for every table [access method](../../../glossary.md#access-method) (table AM). `estimate_rel_size()` dispatches ordinary table-like relations to `table_relation_estimate_size()`, which calls the relation's `TableAmRoutine.relation_estimate_size` callback; the built-in heap routine is one implementation ([plancat.c#table-AM-dispatch](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L946-L953), [tableam.h#relation_estimate_size](../../../../raw/postgres-12/src/include/access/tableam.h#L586-L603), [tableam.h#table_relation_estimate_size](../../../../raw/postgres-12/src/include/access/tableam.h#L1621-L1637), [heapam_handler.c#heap-AM-registration](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2603-L2663)).

Foreign tables have another boundary. Core seeds a placeholder estimate, then calls the [foreign data wrapper](../../../glossary.md#foreign-data-wrapper)'s mandatory `GetForeignRelSize` callback and clamps the returned row count ([costsize.c#set_foreign_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5174-L5200), [allpaths.c#set_foreign_size](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L899-L914), [fdwapi.h#GetForeignRelSize](../../../../raw/postgres-12/src/include/foreign/fdwapi.h#L23-L32)). By default, shipped `postgres_fdw` estimates from local catalog statistics, because `use_remote_estimate` starts as `false` ([postgres_fdw.c:596-600](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L596-L600)). Setting the option to `true` on the server, or on the table, which overrides the server, makes it execute remote `EXPLAIN` and replace the local row and width estimates ([postgres_fdw.c:5280-5285](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L5280-L5285), [postgres_fdw.c:5308-5313](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L5308-L5313)); shipped `file_fdw` derives size through its own callback ([postgres_fdw.c#postgresGetForeignRelSize](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L563-L720), [postgres-fdw.sgml#use_remote_estimate](../../../../raw/postgres-12/doc/src/sgml/postgres-fdw.sgml#L200-L267), [file_fdw.c#fileGetForeignRelSize](../../../../raw/postgres-12/contrib/file_fdw/file_fdw.c#L498-L523)).

Core exposes four statistics-adjacent [hooks](../../../glossary.md#hook). Each is `NULL` by default, so core behaves as described above until an extension installs one ([plancat.c:61](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L61), [selfuncs.c:147-148](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L147-L148), [lsyscache.c:50](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L50)). Once installed, the two statistics-tuple hooks run before the core lookup, which runs only when the hook returns `false`. `get_attavgwidth_hook` also runs first, and its result is used when it is positive. `get_relation_info_hook` runs last, after core has filled the `RelOptInfo` ([selfuncs.c#relation-hook-call](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4726), [lsyscache.c:2875-2880](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2875-L2880), [plancat.c#hook-call](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L456-L463)):

- `get_relation_info_hook` can change relation size or index metadata after core loading ([plancat.c#get_relation_info_hook](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L57-L62), [plancat.c#hook-call](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L456-L463));
- `get_relation_stats_hook` and `get_index_stats_hook` can supply an alternative statistics tuple ([selfuncs.h#statistics-hooks](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L115-L125), [selfuncs.c#relation-hook-call](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4726), [selfuncs.c#index-hook-call](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4574-L4604)); and
- `get_attavgwidth_hook` can override average width ([lsyscache.h#get_attavgwidth_hook](../../../../raw/postgres-12/src/include/utils/lsyscache.h#L59-L64), [lsyscache.c#get_attavgwidth](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2860-L2893)).

Extensions need not use those hooks to affect estimates. They can define operator selectivity procedures or an index AM cost routine. Shipped `intarray` reads array MCE statistics in `_int_matchsel`, shipped `ltree` implements `ltreeparentsel`, and shipped Bloom installs `blcostestimate` as its index AM estimator ([intarray/_int_selfuncs.c#_int_matchsel](../../../../raw/postgres-12/contrib/intarray/_int_selfuncs.c#L119-L238), [ltree_op.c#ltreeparentsel](../../../../raw/postgres-12/contrib/ltree/ltree_op.c#L562-L614), [bloom/blutils.c#amcostestimate](../../../../raw/postgres-12/contrib/bloom/blutils.c#L118-L139), [bloom/blcost.c#blcostestimate](../../../../raw/postgres-12/contrib/bloom/blcost.c#L20-L45)). External extension behavior beyond the pinned checkout cannot be inferred from core hook declarations alone.

### Build, generated-header, and cache boundaries

The catalog declarations are build inputs. `pg_class.h`, `pg_statistic.h`, `pg_statistic_ext.h`, and `pg_statistic_ext_data.h` include generated `*_d.h` headers that provide relation IDs and attribute-number macros ([pg_class.h#catalog-includes](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L18-L29), [pg_statistic.h#catalog-includes](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L20-L29), [pg_statistic_ext.h#catalog-includes](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L22-L33), [pg_statistic_ext_data.h#catalog-includes](../../../../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L20-L31)). The backend catalog Makefile lists these catalog headers, derives every `*_d.h` name, and runs `genbki.pl` before symlinking generated headers into the build include tree ([catalog/Makefile#catalog-and-generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L27-L57), [catalog/Makefile#genbki-rule](../../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L101)).

Statistics reads use named [syscaches](../../../glossary.md#syscache). `STATRELATTINH`, `STATEXTOID`, and `STATEXTDATASTXOID` are declared in the syscache enum; their cache definitions specify the backing relation, index, and key columns ([syscache.h#statistics-caches](../../../../raw/postgres-12/src/include/utils/syscache.h#L75-L93), [syscache.c#statistics-cache-definitions](../../../../raw/postgres-12/src/backend/utils/cache/syscache.c#L731-L773)). This is why the planner can request a column tuple by `(starelid, staattnum, stainherit)` and extended data by statistics-object OID rather than scanning those catalogs for each clause ([selfuncs.c#column-lookup](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [plancat.c#extended-statistics-lookups](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1304-L1322)).

The list of extended-statistics objects is separately cached in each relation's relcache entry. Shared relcache invalidation clears `rd_statlist`/`rd_statvalid`; `CREATE STATISTICS` and `DROP STATISTICS` explicitly invalidate the associated relation so later planning rebuilds the list ([relcache.c#RelationGetStatExtList](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4443-L4523), [statscmds.c#create-statistics-invalidation](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L340-L378), [statscmds.c#drop-statistics-invalidation](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L417-L466)).

### Regression coverage

The checked-in tests cover the positive sources and the monitoring/source distinction from several angles:

- `stats_ext.sql` compares estimated and actual rows before and after building ndistinct, dependency, and MCV statistics; it also covers invalid definitions, unbuilt data, column drops, partial `ANALYZE`, and supported relation kinds ([stats_ext.sql#definition-and-error-paths](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L7-L119), [stats_ext.sql#ndistinct](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L122-L218), [stats_ext.sql#dependencies-and-MCV](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L220-L491)).
- `join.sql` creates and analyzes a multicolumn FK case before checking its plan ([join.sql#FK-estimation](../../../../raw/postgres-12/src/test/regress/sql/join.sql#L1903-L1928)).
- `privileges.sql` and `inherit.sql` exercise expression-index statistics and the table/column privilege boundaries that decide whether estimators may use them ([privileges.sql#expression-statistics-security](../../../../raw/postgres-12/src/test/regress/sql/privileges.sql#L130-L202), [inherit.sql#inherited-statistics-security](../../../../raw/postgres-12/src/test/regress/sql/inherit.sql#L849-L885)).
- `join_hash.sql` deliberately changes `pg_class.reltuples` and `relpages` to create underestimated relations for hash-join tests, directly exercising planner dependence on those relation statistics ([join_hash.sql#forced-relation-estimates](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L55-L84)). This is a non-default fixture state: by default only maintenance commands write those columns, as listed in [Events that change the planner's inputs](#events-that-change-the-planners-inputs). The test writes them directly with `UPDATE pg_class` ([join_hash.sql:70](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L70), [join_hash.sql:82-84](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L82-L84)), and the heap estimator then scales the false density to the real block count ([heapam_handler.c#density-scaling](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2171)).
- `stats.sql` checks the asynchronously collected scan, tuple, live/dead, and block counters exposed by `pg_stat_user_tables` and related views; it does not present those counters as planner estimates ([stats.sql#collector-setup](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L1-L78), [stats.sql#collector-results](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L154-L175)).
- `postgres_fdw.sql` deliberately uses local statistics on one foreign table and remote-estimate mode on another, and separately analyzes foreign tables that do not use remote estimates ([postgres_fdw.sql#local-vs-remote-estimates](../../../../raw/postgres-12/contrib/postgres_fdw/sql/postgres_fdw.sql#L217-L224), [postgres_fdw.sql#local-foreign-statistics](../../../../raw/postgres-12/contrib/postgres_fdw/sql/postgres_fdw.sql#L376-L382)).

`stats_ext.sql` builds an object on an inheritance parent and accepts one on a partitioned table, but it checks no estimate on either. So no regression test covers the whole-tree use of parent-only extended data, or the unbuilt object on a partitioned table ([stats_ext.sql#inheritance-and-partitioned-objects](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L76-L106)). This page's [measurement script](#measurement-script) is the only evidence for those two cases beyond the source reading.

There is no dedicated regression assertion named for the negative statement "the core planner does not read `pg_stat_all_tables`." That boundary is established by the positive call paths above and by a source-wide search of `src/backend/optimizer/`, which contains no `PgStat_StatTabEntry`, `pgstat_fetch_stat_tabentry`, `pg_stat_get_*`, or `pg_stat_all_tables` reference. The cumulative-statistics regression test verifies the monitoring side, not planner non-use ([stats.sql#collector-setup](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L1-L78), [planmain.c#query_planner](../../../../raw/postgres-12/src/backend/optimizer/plan/planmain.c#L65-L271), [plancat.c#get_relation_info](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149)).

### Final causal summary

1. **Initial state:** maintenance commands have stored `relpages`, `reltuples`, and `relallvisible` in `pg_class`; `ANALYZE` has stored column, expression-index, and built extended statistics. The collector separately keeps activity counters.
2. **Input:** a query reaches `get_relation_info()`, which reads the current block count live and loads the stored catalog data through relcache and syscache. It never reads collector counters.
3. **Transformation:** the table AM scales the stored density to the current blocks. Selectivity code applies extended statistics first, then per-column statistics, or fixed defaults when no usable row exists.
4. **Interaction:** a permission or security-qual boundary can hide slot values, and on an inheritance tree a group-count estimate can use extended data built from the parent's own rows.
5. **Result:** row and group estimates follow the catalogs as last written, so they are stale exactly when the data has changed since then.
6. **Refresh:** the collector's `changes_since_analyze` triggers auto-analyze for ordinary tables and materialized views, but not partitioned tables, and the refreshed catalogs feed later plans. Planning itself can bump an index scan counter through the histogram-endpoint probe, without reading it.

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Measures two statistics-source boundaries. The `extstats` part measures group-count estimates and actual group counts on a plain-inheritance tree, with and without an ndistinct object on its parent, and checks whether `ANALYZE` builds such an object on a partitioned table. The `autoanalyze` part checks which parents auto-analyze writes inherited `pg_statistic` rows for. It backs every number in [Extended statistics on inheritance trees and partitioned tables](#extended-statistics-on-inheritance-trees-and-partitioned-tables) and the measured paragraph in [The collector's indirect effect through auto-analyze](#the-collectors-indirect-effect-through-auto-analyze). It measures estimates, counts, and catalog state, not timings. |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/qs12-run.sh` with the command after this table, then run `bash .wiki-runtime/tmp/qs12-run.sh [stage ...]`. Every build, data, socket, and output file stays under `.wiki-runtime/tmp/qs12/`. |
| Stages | Default order: `build measure report`. `stop` and `clean` tear down and are not in the default order. See [Stages](#stages). |
| Environment | `PORT12` (default `55461`) is the isolated port, `JOBS` (default `8`) the parallel build jobs, and `WAIT_SECS` (default `90`) the longest wait for auto-analyze. The checkout, pin, sandbox path, and the 30,000-row fixture size are fixed in the script. |
| Prerequisites | A C toolchain, GNU make, bison and flex; readline and zlib development headers, because 12's `configure` enables both by default ([configure.in:867-868](../../../../raw/postgres-12/configure.in#L867-L868), [configure.in:966-967](../../../../raw/postgres-12/configure.in#L966-L967)); the pinned checkout at `raw/postgres-12/`; about 500 MB of free disk. Locale `C`, encoding `UTF8`, set by the script. No extension is needed. |
| Output | Files under `.wiki-runtime/tmp/qs12/out/`. Read `results.txt` first: one line per GROUP BY case with `case\|from\|estimated_rows\|actual_rows`. Then `facts.txt` (`name\|value` catalog checks), `platform.txt`, `<case>.explain`, `initdb.log`, and `server.log`. When the build fails, `configure.log` or `make.log` is copied there too. |
| Runtime | About 1 minute from an empty sandbox, nearly all of it the build (58 to 66 s on the 2026-10-30 host, Linux x86_64, 22 threads, `JOBS=22`). Re-running `measure report` while the build is present takes about 11 seconds, most of it the auto-analyze wait. |
| Cleanup | Run `bash .wiki-runtime/tmp/qs12-run.sh stop clean` from the repository root. It stops the cluster if it is still running, checks that no `postmaster.pid`, process naming the data directory, socket, or TCP listener on 127.0.0.1 port `PORT12` remains, and deletes `.wiki-runtime/tmp/qs12/` with its build. Remove the extracted `.wiki-runtime/tmp/qs12-run.sh` afterwards. The `measure` stage already stops its cluster at the end. |

To extract the published script without editing it, check its syntax, and run the default stages, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/query-planning/query-planner-statistics-sources.md | sed '1d;$d' > .wiki-runtime/tmp/qs12-run.sh
bash -n .wiki-runtime/tmp/qs12-run.sh
bash .wiki-runtime/tmp/qs12-run.sh
```

### Last run

**Latest run (2026-10-30, second review).** The script changed in two places since the first run. `verify_free` now detects any TCP listener on 127.0.0.1 at `PORT12` through bash's `/dev/tcp`, instead of only a PostgreSQL server that `pg_isready` reaches. Three values the prose quotes are now asserted exactly rather than as `>= 1` or not at all: `aa_inh_inherited_rows = 1`, `pt_inherited_rows = 2` and `ih_s_ndistinct = {"1, 2": 100}`, which gives 16 assertions. The edited script was extracted from this page with the command above and run from an empty sandbox in 66 s with `JOBS=22`, then `measure report` was re-run with the build present in 11 s. Both runs passed all 16 assertions, and their results were identical to each other and to the first run. Auto-analyze again reached the four `aa_*` tables after about 2 s. `platform.txt` recorded the same values as below. `stop clean` again left no PID file, process, socket or listener, and deleted the sandbox with its build.

**2026-10-30, first run.** The script was extracted from this page with the command above and run from an empty sandbox in 59 s with `JOBS=22`. `measure report` was then re-run with the build present in 11 s. Both runs passed all 15 assertions and printed identical results: the four GROUP BY cases and the catalog facts reported in [Extended statistics on inheritance trees and partitioned tables](#extended-statistics-on-inheritance-trees-and-partitioned-tables) and [The collector's indirect effect through auto-analyze](#the-collectors-indirect-effect-through-auto-analyze). Auto-analyze reached the four `aa_*` tables after about 2 s in each run. The host was Linux 6.18.54.1-microsoft-standard-WSL2 x86_64, with PostgreSQL 12.2 at `45b88269a353ad93744772791feb6d01bc7e1e42`, built with default configure options. `platform.txt` recorded `block_size=8192`, `default_statistics_target=100`, `autovacuum=off` (before the reload), `track_counts=on`, `shared_buffers=128MB` (chosen by `initdb`), and `max_data_alignment=8`. The results do not depend on the block size or the alignment, because each estimate comes from distinct counts and row counts that `ANALYZE` stored. `stop clean` then confirmed that no PID file, process, socket, or listener remained, and deleted the sandbox with its build.

### Stages

| Stage | What it does | Needs |
|---|---|---|
| `build` | Checks the pin, then configures 12.2 out of tree with default options under `$SANDBOX/build`, builds, and installs it. Skipped when a matching build exists. | nothing |
| `measure` | Runs `initdb`, starts the cluster with autovacuum off, creates the fixtures, runs the four GROUP BY cases and the catalog checks, turns autovacuum on with a reload, waits for auto-analyze, records the catalog state, stops the cluster, then checks its assertions | `build` |
| `report` | Prints `results.txt` and `facts.txt` | `measure` |
| `stop` | Stops the cluster if it is still running and checks that no PID file, process, socket, or listener remains | none |
| `clean` | Runs `stop`, then deletes `$SANDBOX` | none |

The `measure` stage is idempotent: it deletes and re-creates its data directory, `results.txt`, and `facts.txt`. Inside it, the steps run in a fixed order, because the `_after` cases depend on the `CREATE STATISTICS` and `ANALYZE` made just before them, and the auto-analyze part needs the reload. Every table, statistics object, and database is disposable and exists only in the sandbox cluster.

### Non-default settings in the measurement

| Setting or state | Default at the pin | Value in the script, why, and which results depend on it |
|---|---|---|
| Statistics objects `ih_s` and `pt_s` | No statistics object exists | Created by `CREATE STATISTICS` to exercise the extended-statistics path. `ih_tree_after`, `ih_only_after`, `ih_s_built`, `ih_s_ndistinct`, and `pt_s_built` depend on them; the `_before` cases are the default leg on the same data. |
| Plain-inheritance parent `ih` holding rows of its own | Tables have no children | The case under test: only a parent with its own rows gives parent-only data that differs from the tree's. Every `ih_*` result depends on it. |
| `autovacuum = off`, then `on` with `autovacuum_naptime = 1s` | `autovacuum` `on`, `autovacuum_naptime` `60` s; both `PGC_SIGHUP`, so a reload applies them ([guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433), [guc.c#autovacuum_naptime](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2938-L2946)) | `off` in `postgresql.conf` before the first start, so no auto-analyze changes the statistics during the `extstats` cases. `ALTER SYSTEM` and `pg_reload_conf()` then turn it on with a 1 s naptime, so the `aa_*` results arrive in seconds rather than minutes. The `aa_*` results depend on autovacuum being on; the naptime only changes how long the wait is. |
| 30,000 rows in each of `ih`, `ih_c` and `pt`; 1,000 or 2,000 in the `aa_*` tables | — | `ANALYZE` samples 300 × the statistics target rows, 30,000 at the default 100 ([analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700), [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994)). So the non-inherited pass reads every row of `ih`, and the parent-only ndistinct value is exactly 100 on every run. The inherited pass samples 30,000 of 60,000 rows, but each column has only 100 values, so every value appears in the sample. The `aa_*` tables only need to pass the default analyze threshold of 50 changed rows. |
| `initdb -U postgres -A trust --locale=C --encoding=UTF8` | Without these, `initdb` names the superuser after the operating-system user ([initdb.c:3286-3288](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3286-L3288)), also uses `trust` but prints a warning ([initdb.c#check_authmethod_unspecified](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2392-L2400), [initdb.c:3333-3336](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3333-L3336)), and takes locale and encoding from the environment ([initdb.c#setlocales](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2300-L2330)) | Fixed so the run does not depend on the host. No result depends on them: the columns are `int4`. |
| `listen_addresses = ''`, `unix_socket_directories`, `port = 55461` | `localhost`, `/tmp`, `5432`; all `PGC_POSTMASTER` ([guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3958), [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2184)). The socket directory is `DEFAULT_PGSOCKET_DIR` ([pg_config_manual.h:182](../../../../raw/postgres-12/src/include/pg_config_manual.h#L182)), and the port is `DEF_PGPORT`, which `configure` sets to 5432 unless `--with-pgport` is given ([configure.in:151-155](../../../../raw/postgres-12/configure.in#L151-L155)) | Socket-only access in the sandbox on a non-default port, so no other cluster is touched. No result depends on them. |
| `statement_timeout`, `lock_timeout` | `0`, disabled; `PGC_USERSET` ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)) | 5 minutes and 5 seconds for every session, set through `PGOPTIONS` at connection start. They only bound a runaway statement. |

### The script

```bash
#!/usr/bin/env bash
#
# qs12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/query-planning/query-planner-statistics-sources.md
#
# Builds the pinned PostgreSQL 12 checkout out of tree with default configure
# options, starts one isolated cluster, and measures two statistics-source
# boundaries:
#   extstats     GROUP BY estimates on a plain-inheritance tree before and after
#                an ndistinct statistics object exists on its parent, compared
#                with the same query on the parent alone (ONLY); plus whether
#                ANALYZE ever builds such an object on a partitioned table
#   autoanalyze  with autovacuum turned on, whether auto-analyze writes
#                inherited (stainherit = true) pg_statistic rows for a
#                plain-inheritance parent and for a partitioned table
# Each parent and child holds 30000 rows or fewer, so ANALYZE at the default
# statistics target (sample size 300 x 100) reads every row of a single table,
# and the parent-only statistics are the same on every run.
#
# Every role, table and database is disposable and lives only in this sandbox.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/qs12-run.sh [stage ...]
# Stages: build measure report stop clean
# Default: build measure report

set -uo pipefail
set -e

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/qs12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55461}"
JOBS="${JOBS:-8}"
ROWS=30000
WAIT_SECS="${WAIT_SECS:-90}"

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; BIN="$INST/bin"
OUT="$SANDBOX/out"; SOCK="$SANDBOX/sock"; DATA="$SANDBOX/data"
export PGPORT="$PORT12" PGHOST="$SOCK"
# Session-scoped bounds for every statement this script sends (user context).
export PGOPTIONS="-c statement_timeout=5min -c lock_timeout=5s"

STARTED=""

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

on_exit() {
  local rc=$?
  trap - EXIT
  if [ "$rc" -ne 0 ] && [ -n "$STARTED" ] && [ -f "$DATA/postmaster.pid" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop >&2 || true
  fi
  exit "$rc"
}
trap on_exit EXIT

own_sandbox() {
  case "$WIKI_ROOT" in
    */postgres-llm-wiki) ;;
    *) die "run from the postgres-llm-wiki repository root" ;;
  esac
  [ -f "$WIKI_ROOT/AGENTS.md" ] || die "repository root not found"
  [ -f "$SANDBOX/.qs12-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.qs12-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
  export TMPDIR="$SANDBOX/tmp"
  mkdir -p "$TMPDIR" "$OUT" "$SOCK"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] \
    || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git --no-optional-locks -C "$SRC12" status --porcelain --untracked-files=no)" ] \
    || die "PostgreSQL 12 checkout has tracked changes"
}

# Add the inline tag comment after each statement's leading verb.
tag_sql() {
  printf '%s\n' "$1" | sed -E 's#(^|;[[:space:]]*)(SELECT|INSERT|CREATE|ALTER|ANALYZE|EXPLAIN)([[:space:]]|;|$)#\1\2 /* wiki_qs12_fixture */\3#g'
}
q() { # q <sql>: run as postgres in database qs, discard output
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -U postgres -d qs -c "$(tag_sql "$1")" > /dev/null \
    || die "failed: $1"
}
s() { # s <sql>: one unaligned result
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -U postgres -d qs -c "$(tag_sql "$1")" \
    || die "failed: $1"
}

verify_stopped() {
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid still present in $DATA"
  if pgrep -f -- "$DATA" > /dev/null 2>&1; then die "a process still names $DATA"; fi
}
verify_free() {
  [ -z "$(ls -A "$SOCK" 2>/dev/null)" ] || die "socket directory $SOCK is not empty"
  # Any TCP listener on the port counts, PostgreSQL or not (bash /dev/tcp).
  if (exec 3<>"/dev/tcp/127.0.0.1/$PORT12") 2>/dev/null; then die "port $PORT12 has a listener"; fi
}

stage_build() {
  say "build 12.2 out of tree from $SRC12 (default configure options)"
  source_pin
  if [ ! -e "$SANDBOX" ]; then
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.qs12-owned"
  fi
  own_sandbox
  mkdir -p "$BUILD"
  if [ -x "$BIN/postgres" ] && [ -x "$BIN/psql" ]; then
    [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 default" ] \
      || die "existing build has a different specification"
    note "matching build already present"
    return 0
  fi
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" > configure.log 2>&1 ) \
    || { cp "$BUILD/configure.log" "$OUT/"; die "configure failed, see $OUT/configure.log"; }
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { cp "$BUILD/make.log" "$OUT/" 2>/dev/null; grep 'error:' "$BUILD/make.log" | head -n 3 >&2
         die "make failed, see $OUT/make.log"; }
  printf '%s default\n' "$PIN12" > "$SANDBOX/.build-spec"
  note "$("$BIN/postgres" --version)"
}

# groups <case> <from>: EXPLAIN ANALYZE a two-column GROUP BY and record the
# top plan node's estimated rows (the group-count estimate) and actual rows.
groups() {
  local case=$1 from=$2 f est act
  f="$OUT/$case.explain"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -U postgres -d qs \
      -c "$(tag_sql "EXPLAIN (ANALYZE, TIMING OFF) SELECT a, b FROM $from GROUP BY a, b")" \
      > "$f" 2>&1 || { cat "$f" >&2; die "EXPLAIN failed for $case"; }
  est=$(sed -n 's/.*cost=[^ ]* rows=\([0-9]*\) width=.*/\1/p' "$f" | head -n 1)
  act=$(sed -n 's/.*(actual rows=\([0-9]*\) loops=.*/\1/p' "$f" | head -n 1)
  [ -n "$est" ] && [ -n "$act" ] || die "could not parse $f"
  printf '%s|%s|%s|%s\n' "$case" "$from" "$est" "$act" >> "$OUT/results.txt"
  note "$case: GROUP BY a, b FROM $from: estimated rows=$est actual rows=$act"
}
# fact <case> <sql>: record one single-value result
fact() {
  local v
  v=$(s "$2")
  printf '%s|%s\n' "$1" "$v" >> "$OUT/facts.txt"
  note "$1 = $v"
}
field() { grep -E "^$1\|" "$OUT/results.txt" | tail -n 1 | cut -d'|' -f"$2"; }
factval() { grep -E "^$1\|" "$OUT/facts.txt" | tail -n 1 | cut -d'|' -f2; }
expect_range() { # expect_range <case> <field-name> <field-number> <min> <max>
  local got
  got=$(field "$1" "$3")
  [ "$got" -ge "$4" ] && [ "$got" -le "$5" ] || die "$1: $2 = $got, expected $4..$5"
  note "ok: $1: $2 = $got (expected $4..$5)"
}
expect_fact() { # expect_fact <case> <value>
  [ "$(factval "$1")" = "$2" ] || die "$1 = $(factval "$1"), expected $2"
  note "ok: $1 = $2"
}

stage_measure() {
  say "extended statistics on inheritance trees, and auto-analyze of parents"
  own_sandbox
  source_pin
  [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 default" ] || die "run the build stage first"
  if [ -f "$DATA/postmaster.pid" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  fi
  verify_free
  rm -rf "$DATA"
  : > "$OUT/results.txt"
  : > "$OUT/facts.txt"
  "$BIN/initdb" -D "$DATA" -U postgres -A trust --locale=C --encoding=UTF8 \
    > "$OUT/initdb.log" 2>&1 || die "initdb failed, see $OUT/initdb.log"
  {
    printf "listen_addresses = ''\n"
    printf "unix_socket_directories = '%s'\n" "$SOCK"
    printf "port = %s\n" "$PORT12"
    printf "autovacuum = off\n"
  } >> "$DATA/postgresql.conf"
  STARTED=yes
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 start > /dev/null 2>&1 \
    || die "pg_ctl start failed, see $OUT/server.log"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -U postgres -d postgres \
    -c "CREATE /* wiki_qs12_fixture */ DATABASE qs" > /dev/null || die "CREATE DATABASE failed"
  {
    uname -srm
    "$BIN/postgres" --version
    for g in block_size default_statistics_target autovacuum track_counts shared_buffers; do
      s "SELECT '$g=' || current_setting('$g')"
    done
    s "SELECT 'max_data_alignment=' || max_data_alignment FROM pg_control_init()"
  } > "$OUT/platform.txt"
  cat "$OUT/platform.txt" >&2

  # --- extstats leg (autovacuum off) -------------------------------------
  # Fixtures (disposable).  On the parent ih, b = a, so (a, b) has 100
  # combinations.  On the child ih_c, a and b vary independently, so (a, b)
  # has 10000 combinations, and they include the parent's 100.
  q "CREATE TABLE ih (a int, b int)"
  q "CREATE TABLE ih_c () INHERITS (ih)"
  q "INSERT INTO ih SELECT g % 100, g % 100 FROM generate_series(1, $ROWS) g"
  q "INSERT INTO ih_c SELECT g % 100, (g / 100) % 100 FROM generate_series(1, $ROWS) g"
  q "ANALYZE ih"
  q "ANALYZE ih_c"
  groups ih_tree_before ih
  groups ih_only_before "ONLY ih"
  q "CREATE STATISTICS ih_s (ndistinct) ON a, b FROM ih"
  q "ANALYZE ih"
  fact ih_s_built "SELECT stxdndistinct IS NOT NULL FROM pg_statistic_ext_data d JOIN pg_statistic_ext e ON e.oid = d.stxoid WHERE e.stxname = 'ih_s'"
  fact ih_s_ndistinct "SELECT stxdndistinct::text FROM pg_statistic_ext_data d JOIN pg_statistic_ext e ON e.oid = d.stxoid WHERE e.stxname = 'ih_s'"
  groups ih_tree_after ih
  groups ih_only_after "ONLY ih"

  # Partitioned table: the same object is accepted but never built.
  q "CREATE TABLE pt (a int, b int) PARTITION BY RANGE (a)"
  q "CREATE TABLE pt1 PARTITION OF pt FOR VALUES FROM (0) TO (50)"
  q "CREATE TABLE pt2 PARTITION OF pt FOR VALUES FROM (50) TO (100)"
  q "INSERT INTO pt SELECT g % 100, g % 100 FROM generate_series(1, $ROWS) g"
  q "CREATE STATISTICS pt_s (ndistinct) ON a, b FROM pt"
  q "ANALYZE pt"
  fact pt_s_built "SELECT stxdndistinct IS NOT NULL FROM pg_statistic_ext_data d JOIN pg_statistic_ext e ON e.oid = d.stxoid WHERE e.stxname = 'pt_s'"
  fact pt_inherited_rows "SELECT count(*) FROM pg_statistic WHERE starelid = 'pt'::regclass AND stainherit"

  # --- autoanalyze leg ----------------------------------------------------
  # Fixtures (disposable): a plain-inheritance parent with rows of its own,
  # and a partitioned table, each with one or two children.  Nothing here is
  # analyzed by hand.
  q "CREATE TABLE aa_inh (a int)"
  q "CREATE TABLE aa_inh_c () INHERITS (aa_inh)"
  q "CREATE TABLE aa_part (a int) PARTITION BY RANGE (a)"
  q "CREATE TABLE aa_part1 PARTITION OF aa_part FOR VALUES FROM (0) TO (1000)"
  q "CREATE TABLE aa_part2 PARTITION OF aa_part FOR VALUES FROM (1000) TO (2000)"
  q "INSERT INTO aa_inh SELECT g FROM generate_series(1, 1000) g"
  q "INSERT INTO aa_inh_c SELECT g FROM generate_series(1, 1000) g"
  q "INSERT INTO aa_part SELECT g FROM generate_series(0, 1999) g"
  # autovacuum and autovacuum_naptime are sighup settings: a reload applies them.
  q "ALTER SYSTEM SET autovacuum = on"
  q "ALTER SYSTEM SET autovacuum_naptime = '1s'"
  s "SELECT pg_reload_conf()" > /dev/null
  local waited=0 done_n=0
  while [ "$waited" -lt "$WAIT_SECS" ]; do
    done_n=$(s "SELECT count(*) FROM pg_stat_user_tables WHERE relname IN ('aa_inh', 'aa_inh_c', 'aa_part1', 'aa_part2') AND last_autoanalyze IS NOT NULL")
    [ "$done_n" = 4 ] && break
    sleep 1
    waited=$((waited + 1))
  done
  [ "$done_n" = 4 ] || die "auto-analyze did not reach all four tables within $WAIT_SECS s"
  # Give the launcher a few more naptime cycles, so a partitioned-table
  # analyze would have had its chance.
  sleep 5
  note "auto-analyze reached the four tables after about $waited s"
  fact aa_inh_autoanalyzed "SELECT last_autoanalyze IS NOT NULL FROM pg_stat_user_tables WHERE relname = 'aa_inh'"
  fact aa_inh_inherited_rows "SELECT count(*) FROM pg_statistic WHERE starelid = 'aa_inh'::regclass AND stainherit"
  fact aa_inh_local_rows "SELECT count(*) FROM pg_statistic WHERE starelid = 'aa_inh'::regclass AND NOT stainherit"
  fact aa_part1_local_rows "SELECT count(*) FROM pg_statistic WHERE starelid = 'aa_part1'::regclass AND NOT stainherit"
  fact aa_part_inherited_rows "SELECT count(*) FROM pg_statistic WHERE starelid = 'aa_part'::regclass AND stainherit"
  fact aa_part_in_stat_view "SELECT count(*) FROM pg_stat_all_tables WHERE relid = 'aa_part'::regclass"

  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  STARTED=""
  verify_stopped

  # Assertions.  Actual counts are fixed by the data.
  expect_range ih_tree_before actual 4 10000 10000
  expect_range ih_only_before actual 4 100 100
  expect_range ih_tree_after actual 4 10000 10000
  expect_range ih_only_after actual 4 100 100
  expect_range ih_tree_before estimate 3 5000 7000
  expect_range ih_only_before estimate 3 2500 3500
  expect_fact ih_s_built t
  expect_range ih_only_after estimate 3 100 100
  expect_range ih_tree_after estimate 3 100 100
  expect_fact pt_s_built f
  expect_fact aa_inh_autoanalyzed t
  expect_fact aa_part_inherited_rows 0
  expect_fact aa_part_in_stat_view 0
  # One column (a) on aa_inh, two columns (a, b) on pt.
  expect_fact aa_inh_inherited_rows 1
  expect_fact pt_inherited_rows 2
  expect_fact ih_s_ndistinct '{"1, 2": 100}'
}

stage_report() {
  own_sandbox
  say "results ($OUT/results.txt)"
  printf 'case|from|estimated_rows|actual_rows\n' >&2
  cat "$OUT/results.txt" >&2
  say "facts ($OUT/facts.txt)"
  cat "$OUT/facts.txt" >&2
}

stage_stop() {
  [ -d "$SANDBOX" ] || { note "no sandbox"; return 0; }
  own_sandbox
  if [ -f "$DATA/postmaster.pid" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "could not stop the cluster"
  fi
  verify_stopped
  verify_free
  note "no server, socket or listener left"
}
stage_clean() {
  [ -d "$SANDBOX" ] || { note "no sandbox"; return 0; }
  stage_stop
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

[ "$#" -gt 0 ] || set -- build measure report
for st in "$@"; do
  case "$st" in
    build|measure|report|stop|clean) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- Pinned checkout: `raw/postgres-12/` at `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- Monitoring side: complete `pg_stat_all_tables` definition, table-facing `pg_stat_get_*()` functions, `PgStat_StatTabEntry`, the planner's B-tree endpoint probe and resulting scan-counter increment, collector tests, and monitoring documentation.
- Normal planner path: `query_planner`, base-relation creation, `get_relation_info`, relation sizing, index/FK/constraint loading, base-row estimation, ordinary and extended selectivity, and relevant planner data structures.
- Statistics reads: plain and inherited columns, expression-index matching, slot extraction, width lookup, extended-statistics object/data loading, security checks, defaults, and consistency errors.
- Statistics writes: ordinary and inherited sampling, type-specific analysis, expression-index statistics, extended-statistics construction, `pg_class` updates, collector reporting, and object create/drop invalidation.
- Other metadata: index and access-method relcache data, FK and exclusion constraints, attributes/types, operator and function estimation procedures, opfamilies, and generated syscache definitions.
- Extension boundaries: table AM sizing, FDW sizing, all four core statistics-adjacent hooks, shipped `postgres_fdw`, `file_fdw`, `intarray`, `ltree`, and Bloom paths. A source-wide search found no assignment to the four statistics-adjacent hooks under the shipped `contrib/` tree.
- Tests and docs: `stats`, `stats_ext`, `join`, `join_hash`, `privileges`, `inherit`, `postgres_fdw`, planner-statistics documentation, `ANALYZE` documentation, monitoring documentation, and FDW documentation.
- Earlier reviews reported a core regression-suite pass in a build tree at `.wiki-runtime/pg12-build/`. That tree no longer exists and no script for it was published, so this page does not rely on it. The page's only run is the [measurement script](#measurement-script).
- 2026-10-30 review: re-read the `ANALYZE` pass split, other `pg_class` writers (lazy `VACUUM`, `index_build()`), autovacuum's relation-kind filter and per-table switch, extended-statistics loading for inheritance parents, the group-count clamp, the expression-index parent permission fallback, the `get_variable_range()` probe exclusion, and the `postgres_fdw` and hook defaults.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| `pg_stat_all_tables` exposes cumulative collector counters, not planner statistics | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-12/src/include/pgstat.h#L633-L662) |
| A planner histogram-endpoint probe can increment an index counter without consuming it as input | [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5250-L5417), [nbtsearch.c#planner-visible-index-scan-count](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L758-L770) |
| Base-relation planning loads catalog/relcache data instead of collector entries | [relnode.c#build_simple_rel](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L175-L293), [plancat.c#get_relation_info](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149) |
| Heap relation size combines current blocks with `pg_class` density and all-visible data | [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171) |
| Ordinary and partial indexes follow different tuple-estimation paths | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L946-L1040) |
| Plain/inherited column statistics use `(relid, attnum, inherit)` | [selfuncs.c#examine_simple_variable](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [syscache.c#STATRELATTINH](../../../../raw/postgres-12/src/backend/utils/cache/syscache.c#L764-L773) |
| Expression-index statistics are separate `pg_statistic` rows; general expression selectivity ignores partial-index rows, while B-tree/BRIN costing can read their correlation | [analyze.c#index-expression-setup](../../../../raw/postgres-12/src/backend/commands/analyze.c#L413-L467), [selfuncs.c#expression-index-match](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4524-L4604), [selfuncs.c#btcostestimate-correlation](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6118-L6213), [selfuncs.c#brincostestimate-correlation](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L7045-L7137) |
| Extended statistics are loaded only for built kinds and used before ordinary clause estimates | [plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1377), [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L53-L106) |
| FK metadata substitutes a join selectivity only after query-clause matching | [planmain.c#FK-match-position](../../../../raw/postgres-12/src/backend/optimizer/plan/planmain.c#L244-L271), [initsplan.c#match_foreign_keys_to_quals](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2413-L2563), [costsize.c#get_foreign_key_join_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4693-L4906) |
| Extended data is built from the parent's own rows but loaded and used for a whole-tree scan; it is never built on a partitioned table | [analyze.c:576-584](../../../../raw/postgres-12/src/backend/commands/analyze.c#L576-L584), [plancat.c:430](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L430), [selfuncs.c#estimate_multivariate_ndistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3573-L3585), [analyze.c:255-268](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268); measured in the [measurement script](#measurement-script) |
| The inherited `ANALYZE` pass writes no extended data, `pg_class` rows, index rows, or collector report | [analyze.c#statistics-write-path](../../../../raw/postgres-12/src/backend/commands/analyze.c#L414-L640) |
| Lazy `VACUUM` and index builds also write `pg_class` size statistics | [vacuumlazy.c:341-363](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L341-L363), [index.c:2977-2986](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986) |
| Autovacuum never analyzes a partitioned table | [autovacuum.c:2065-2067](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2065-L2067); measured in the [measurement script](#measurement-script) |
| Missing column statistics use fixed/default estimates, not cumulative counters | [selfuncs.h#default-estimates](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L30-L49), [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4985-L5117) |
| Collector changes affect planning only indirectly through auto-analyze and catalog refresh | [autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2927-L3095), [analyze.c#statistics-write-path](../../../../raw/postgres-12/src/backend/commands/analyze.c#L506-L640) |
| Table AMs and FDWs can replace core size estimates | [tableam.h#relation_estimate_size](../../../../raw/postgres-12/src/include/access/tableam.h#L586-L603), [allpaths.c#set_foreign_size](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L899-L914) |
| Extensions can supply statistics, selectivity, and AM cost behavior | [selfuncs.h#statistics-hooks](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L115-L125), [intarray/_int_selfuncs.c#_int_matchsel](../../../../raw/postgres-12/contrib/intarray/_int_selfuncs.c#L119-L238), [bloom/blcost.c#blcostestimate](../../../../raw/postgres-12/contrib/bloom/blcost.c#L20-L45) |
| Catalog IDs/attribute numbers and statistics syscaches have generated build plumbing | [catalog/Makefile#catalog-and-generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L27-L57), [catalog/Makefile#genbki-rule](../../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L101), [syscache.c#statistics-cache-definitions](../../../../raw/postgres-12/src/backend/utils/cache/syscache.c#L731-L773) |
| Regression coverage distinguishes collector monitoring from catalog-based planner estimates | [stats.sql#collector-results](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L154-L175), [stats_ext.sql#estimated-row-tests](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L122-L218), [join_hash.sql#forced-relation-estimates](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L55-L84) |

## Open Questions

- None unresolved for PostgreSQL 12 core or the shipped modules at pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42`. The whole-tree use of parent-only extended statistics is settled by the source and by the measurement. This page makes no claim about whether later 12.x minor releases changed it, because a later minor release is outside this pin.
- `clauselist_selectivity()` does not check the `inh` flag either, but this review did not establish whether any v12 path estimates a restriction-clause list against an inheritance-parent appendrel `RelOptInfo`; `set_append_rel_size()` estimates each child separately ([allpaths.c#set_append_rel_size](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L930-L1039)). So the page claims whole-tree use of parent-only extended data only for group counts, the measured case.
- The 2026-10-30 review re-checked the claims it changed or added, not every claim on the page, so `verified_by_agent` stays `not yet`.
- Docs/source discrepancy: the monitoring documentation says the optimizer probes indexes for constants "outside the recorded range of the optimizer statistics because the optimizer statistics might be stale" ([monitoring.sgml:2926-2928](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2926-L2928)). The source probes whenever the histogram search reaches an end entry, and up front when the histogram has two entries, whether or not the constant is outside the range or the statistics are stale ([selfuncs.c:885-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L885-L935)). Source wins; the page follows it.
- An external extension can install the documented hooks or define custom statistics/selectivity behavior. Its implementation is outside this pinned checkout and must be reviewed separately.

## Source References

- [system_views.sql#pg_stats-and-pg_stats_ext](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L189-L294) - user-facing statistics views.
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581) - cumulative monitoring view.
- [pgstatfuncs.c#table-counter-functions](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L40-L360) - collector-backed SQL functions.
- [pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-12/src/include/pgstat.h#L633-L662) - cumulative table entry.
- [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5250-L5417) - planner histogram-endpoint probe.
- [nbtsearch.c#planner-visible-index-scan-count](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L758-L770) - B-tree counter increment reached by that probe.
- [planmain.c#query_planner](../../../../raw/postgres-12/src/backend/optimizer/plan/planmain.c#L65-L271) - base planner sequence.
- [relnode.c#build_simple_rel](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L175-L293) - base relation construction.
- [plancat.c#get_relation_info](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L463) - relation, index, extended-statistics, FDW, and FK loading.
- [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171) - heap size estimate.
- [plancat.c#estimate_rel_size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L926-L1061) - relation-kind dispatcher and index estimate.
- [pathnodes.h#planner-statistics-structures](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L632-L882) - `RelOptInfo`, `IndexOptInfo`, FK, and extended-statistics structures.
- [selfuncs.h#statistics-data-and-defaults](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L30-L131) - defaults, `VariableStatData`, and hooks.
- [selfuncs.c#examine_variable](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4430-L4697) - expression/index statistics matching.
- [selfuncs.c#examine_simple_variable](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4699-L4959) - plain and inherited column lookup and security.
- [pg_statistic.h#FormData_pg_statistic](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L20-L126) - per-attribute catalog.
- [pg_statistic.h#slot-kinds](../../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L136-L271) - core and extension slot contract.
- [plancat.c#get_relation_statistics](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1289-L1377) - extended-statistics planner metadata.
- [extended_stats.c#build-and-store](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L75-L187) - build and built-kind checks.
- [extended_stats.c#statext_clauselist_selectivity](../../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L1169-L1202) - MCV/dependency order.
- [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4390-L4423) - base output rows, width, and qual cost.
- [costsize.c#get_foreign_key_join_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4693-L4906) - FK selectivity substitution.
- [analyze.c#statistics-write-path](../../../../raw/postgres-12/src/backend/commands/analyze.c#L390-L640) - columns, indexes, extended data, relation stats, and collector report.
- [analyze.c#acquire_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L976-L1127) - ordinary sampling.
- [analyze.c#acquire_inherited_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1169-L1390) - inheritance sampling.
- [analyze.c#update_attstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1394-L1547) - `pg_statistic` writes.
- [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1135-L1196) - `pg_class` writes.
- [autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2927-L3095) - indirect collector threshold.
- [tableam.h#relation_estimate_size](../../../../raw/postgres-12/src/include/access/tableam.h#L586-L603) - table-AM size callback.
- [allpaths.c#set_foreign_size](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L899-L914) - FDW size callback.
- [postgres_fdw.c#postgresGetForeignRelSize](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L563-L720) - local/remote foreign estimates.
- [catalog/Makefile#generated-catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L27-L101) - `genbki.pl` and `*_d.h` build path.
- [syscache.c#statistics-cache-definitions](../../../../raw/postgres-12/src/backend/utils/cache/syscache.c#L731-L773) - statistics syscache keys.
- [selfuncs.c#estimate_num_groups](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3049-L3366) - group-count estimate, multivariate ndistinct use, and the multi-column clamp.
- [vacuumlazy.c#lazy_vacuum_rel-relstats](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L341-L363) - `VACUUM` writes to `pg_class`.
- [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2805) - index-build writes to `pg_class`.
- [autovacuum.c#do_autovacuum-relkind-filter](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2055-L2067) - relation kinds autovacuum considers.
- [stats_ext.sql#inheritance-and-partitioned-objects](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L76-L106) - regression coverage of objects on inheritance and partitioned parents.
- [stats_ext.sql#complete-test](../../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L1-L554) - extended-statistics regression coverage.
- [stats.sql#complete-test](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L1-L178) - cumulative collector regression coverage.

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Foreign-Key Join Optimization for Two-Table Joins (unverified)](fk-join-optimization-two-tables.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [PostgreSQL versions](../../../versions.md)
