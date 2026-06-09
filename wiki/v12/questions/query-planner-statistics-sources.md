---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: gpt-5 2026-06-09T14:53:48Z
---

# Query Planner Statistics Sources in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, does the query planner use stats from `pg_stat_all_tables`? Does it use stats from any other tables or catalogs?

## Answer Up Front

No. PostgreSQL 12's core planner does not use `pg_stat_all_tables` as planner input. `pg_stat_all_tables` is a monitoring view over cumulative statistics functions such as `pg_stat_get_numscans`, `pg_stat_get_tuples_returned`, `pg_stat_get_live_tuples`, and analyze/vacuum timestamp and count functions [system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581). Those functions read `PgStat_StatTabEntry` counters from the statistics collector, not the planner's relation-size or selectivity catalogs [pgstatfuncs.c#pg_stat_get_numscans](../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L40-L84) [pgstat.h#PgStat_StatTabEntry](../../../raw/postgres-12/src/include/pgstat.h#L633-L662).

The PostgreSQL 12 planner's normal relation-size path reads `pg_class` fields such as `relpages`, `reltuples`, and `relallvisible`, while its selectivity path reads `pg_statistic` and extended-statistics catalogs [pg_class.h#relpages-reltuples-relallvisible](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66) [heapam_handler.c#heapam_estimate_rel_size](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171) [selfuncs.c#examine_simple_variable](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737) [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1374).

The planner also does not read the user-facing `pg_stats` or `pg_stats_ext` views directly. Those views expose `pg_statistic`, `pg_statistic_ext`, and `pg_statistic_ext_data` with access filtering for SQL users; the planner paths below use syscache and relcache access to the underlying catalogs instead [system_views.sql#pg_stats](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L189-L256) [system_views.sql#pg_stats_ext](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L256-L285) [selfuncs.c#examine_simple_variable](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737) [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1374).

## What `pg_stat_all_tables` Contains

`pg_stat_all_tables` is defined as a SQL view that joins `pg_class`, `pg_index`, and `pg_namespace`, then calls `pg_stat_get_*()` functions for scan counters, tuple-change counters, live/dead tuple counters, modification-since-analyze counters, and last-vacuum/last-analyze timestamps [system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581).

The underlying table-statistics entry stores counters such as `numscans`, `tuples_returned`, `tuples_fetched`, `tuples_inserted`, `tuples_updated`, `tuples_deleted`, `n_live_tuples`, `n_dead_tuples`, `changes_since_analyze`, block hit/fetch counters, and analyze/vacuum counters [pgstat.h#PgStat_StatTabEntry](../../../raw/postgres-12/src/include/pgstat.h#L633-L662). The planner code paths cited below do not consume that `PgStat_StatTabEntry` structure; they consume relation, column, extended-statistics, index, constraint, and foreign-key metadata through relcache/syscache paths [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149) [plancat.c#get_relation_info-catalog-load](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L428-L445).

## Planner Statistics and Metadata Sources

| Source | Planner use in PostgreSQL 12 | Evidence |
|---|---|---|
| `pg_class` | Relation page count, tuple count, and all-visible page fraction through `relpages`, `reltuples`, and `relallvisible`. | [pg_class.h#relpages-reltuples-relallvisible](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [heapam_handler.c#heapam_estimate_rel_size](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171) |
| `pg_statistic` | Per-column null fraction, average width, distinct estimate, MCV/histogram slots, and selectivity inputs. | [pg_statistic.h#FormData_pg_statistic](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L29-L124), [selfuncs.c#examine_simple_variable](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [lsyscache.c#get_attavgwidth](../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2870-L2893) |
| `pg_statistic_ext` | Extended-statistics object metadata: relation, key columns, and requested statistic kinds. | [pg_statistic_ext.h#FormData_pg_statistic_ext](../../../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L33-L56), [relcache.c#RelationGetStatExtList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4448-L4517) |
| `pg_statistic_ext_data` | Built extended-statistics data: multivariate ndistinct, functional dependencies, and multivariate MCV lists. | [pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../../../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43), [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1315-L1370) |
| `pg_stats` and `pg_stats_ext` | User-facing inspection views, not direct planner inputs. They expose the same underlying statistics catalogs through SQL views. | [system_views.sql#pg_stats](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L189-L256), [system_views.sql#pg_stats_ext](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L256-L285) |
| `pg_index` and index relcache data | Index OIDs, validity, keys, opfamilies, predicates, expressions, uniqueness, index size, and B-tree height. | [relcache.c#RelationGetIndexList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4318-L4399), [plancat.c#get_relation_info-indexes](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L423) |
| `pg_constraint` and relcache constraint data | Foreign-key metadata for join selectivity, plus CHECK, NOT NULL, and partition constraints for relation exclusion. | [relcache.c#RelationGetFKeyList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4226-L4315), [plancat.c#get_relation_constraints](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1137-L1287), [plancat.c#relation_excluded_by_constraints](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1380-L1560) |
| Relation tuple descriptor / type metadata | Attribute type, typmod, collation, and fallback width information when no `pg_statistic.stawidth` is available. | [plancat.c#get_rel_data_width](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1064-L1111) |

## Relation Size Path

For an `RTE_RELATION` base relation, `build_simple_rel()` calls `get_relation_info()` with the comment "retrieve statistics from the system catalogs" [relnode.c#build_simple_rel](../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L287-L293). `get_relation_info()` opens the relation, fills relation attributes, and calls `estimate_rel_size()` to populate `RelOptInfo.pages`, `RelOptInfo.tuples`, and `RelOptInfo.allvisfrac` for non-inheritance-parent baserels [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149).

For heap relations, `heapam_estimate_rel_size()` obtains the current physical block count with `RelationGetNumberOfBlocks()`, coerces `rel->rd_rel->relpages`, `rel->rd_rel->reltuples`, and `rel->rd_rel->relallvisible` from the relation descriptor, uses a 10-page floor for small never-vacuumed relations, estimates current tuples from old tuple density when `relpages > 0`, falls back to tuple-width-derived density when no old page count exists, and converts `relallvisible` to a fraction for costing [heapam_handler.c#heapam_estimate_rel_size](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171).

For index relations, `estimate_rel_size()` uses `RelationGetNumberOfBlocks()` for current pages, reads the index relation's `pg_class` `relpages`, `reltuples`, and `relallvisible`, discounts an index metapage for tuple-density estimation when possible, falls back to tuple-width-derived density when no old page count exists, and reports the resulting index pages and tuple estimate to the caller [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L936-L1061).

## Column Statistics Path

For a plain relation column, `examine_simple_variable()` looks up the `pg_statistic` tuple with `SearchSysCache3(STATRELATTINH, relid, attnum, inh)` and stores it in `VariableStatData.statsTuple` for selectivity functions [selfuncs.c#examine_simple_variable](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737). For average width, `get_attavgwidth()` performs the same `STATRELATTINH` syscache lookup and returns `pg_statistic.stawidth` when it is positive [lsyscache.c#get_attavgwidth](../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2870-L2893).

`pg_statistic` stores `stanullfrac`, `stawidth`, `stadistinct`, five statistic-kind slots, operator and collation IDs, `stanumbersN`, and `stavaluesN`; those slots are the source for common selectivity inputs such as most-common-values lists and histograms [pg_statistic.h#FormData_pg_statistic](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L29-L124) [lsyscache.c#get_attstatsslot](../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2895-L2944).

## Extended Statistics Path

`get_relation_statistics()` loads extended statistics for a relation by calling `RelationGetStatExtList()`, then looking up `pg_statistic_ext` with `STATEXTOID` and `pg_statistic_ext_data` with `STATEXTDATASTXOID` [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1323). It creates one `StatisticExtInfo` entry per built statistic kind: ndistinct, dependencies, and MCV [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1332-L1370) [pathnodes.h#StatisticExtInfo](../../../raw/postgres-12/src/include/nodes/pathnodes.h#L868-L882).

Restriction-clause selectivity calls `statext_clauselist_selectivity()` when all clauses reference a single relation that has `rel->statlist`; the extended-statistics path estimates compatible clauses first, then normal selectivity handles the remaining clauses [clausesel.c#clauselist_selectivity](../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L69-L106). The extended-statistics selectivity function applies multivariate MCV statistics first and functional dependency statistics afterward [extended_stats.c#statext_clauselist_selectivity](../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L1170-L1202). Grouping distinct estimates can also use multivariate ndistinct statistics through `estimate_multivariate_ndistinct()` and `statext_ndistinct_load()` [selfuncs.c#estimate_num_groups-ndistinct](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3235-L3259) [selfuncs.c#estimate_multivariate_ndistinct](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3573-L3642).

## Index and Foreign-Key Metadata

For indexes, `get_relation_info()` calls `RelationGetIndexList()`, opens each candidate index relation, ignores indexes that are not `indisvalid`, partitioned indexes, and indexes whose `indcheckxmin` makes the plan transient, then copies keys, collations, opfamilies, access-method capabilities, index expressions, predicates, uniqueness flags, current index pages/tuples, and B-tree height into `IndexOptInfo` [plancat.c#get_relation_info-indexes](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L423). `RelationGetIndexList()` scans `pg_index` for rows whose `indrelid` matches the relation and returns OIDs for indexes that are still `indislive`; planner usability is filtered by `get_relation_info()` afterward [relcache.c#RelationGetIndexList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4318-L4399).

For foreign keys, `RelationGetFKeyList()` scans `pg_constraint` for constraints whose `conrelid` is the relation, filters `CONSTRAINT_FOREIGN`, and deconstructs the FK columns and equality operators into `ForeignKeyCacheInfo` entries [relcache.c#RelationGetFKeyList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4226-L4299). `get_relation_foreign_keys()` converts relevant cached FK entries into planner `ForeignKeyOptInfo` objects [plancat.c#get_relation_foreign_keys](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L476-L566), and `match_foreign_keys_to_quals()` matches those FKs to query equality clauses for join selectivity [initsplan.c#match_foreign_keys_to_quals](../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2399-L2558). `get_foreign_key_join_selectivity()` then substitutes FK-based selectivity for the matched join clauses, especially when normal `pg_statistic` stats are missing or stale [costsize.c#get_foreign_key_join_selectivity](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4694-L4906).

## How Planner Statistics Get Written

`ANALYZE` emits completed per-column statistics into `pg_statistic` with `update_attstats()` [analyze.c#do_analyze_rel-stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L555-L584). `update_attstats()` constructs or replaces `pg_statistic` tuples containing `starelid`, `staattnum`, `stainherit`, `stanullfrac`, `stawidth`, `stadistinct`, statistic-kind fields, operators, collations, numeric arrays, and value arrays [analyze.c#update_attstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L1395-L1534).

`ANALYZE` also calls `BuildRelationExtStatistics()` for non-inheritance relations; that function builds requested ndistinct, dependency, and MCV statistics and stores them through `statext_store()` [analyze.c#do_analyze_rel-extended-stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L576-L584) [extended_stats.c#BuildRelationExtStatistics](../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L83-L150). `statext_store()` serializes those values into `pg_statistic_ext_data.stxdndistinct`, `stxddependencies`, and `stxdmcv` [extended_stats.c#statext_store](../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L317-L378).

Relation-level `pg_class` statistics are written by `vac_update_relstats()`, which updates `relpages`, `reltuples`, and `relallvisible` [vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1157-L1196). `ANALYZE` invokes that path after counting all-visible pages for the relation [analyze.c#do_analyze_rel-relstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L587-L604).

## Regression Coverage

PostgreSQL 12's regression tests exercise extended statistics as planner input. `stats_ext.sql` defines `check_estimated_rows()` using `EXPLAIN ANALYZE`, compares row estimates before and after `CREATE STATISTICS` plus `ANALYZE` for multivariate ndistinct, and directly queries `pg_statistic_ext` joined to `pg_statistic_ext_data` [stats_ext.sql#check_estimated_rows](../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L7-L26) [stats_ext.sql#ndistinct](../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L122-L218). The same file exercises functional-dependency and MCV extended statistics through row-estimate checks before and after statistics creation [stats_ext.sql#dependencies-and-mcv](../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L220-L360).

`join.sql` has a foreign-key join-estimation test that creates a multicolumn FK, runs `ANALYZE`, and checks the resulting join plan with `EXPLAIN` [join.sql#fk-join-estimation-test](../../../raw/postgres-12/src/test/regress/sql/join.sql#L1903-L1928). `stats.sql` exercises the cumulative monitoring counters exposed by `pg_stat_user_tables` and related views, including scan, tuple, and block counters; that supports the distinction that these are monitoring counters, while the planner-source claims above are established by the planner code paths [stats.sql#statistics-collector](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L1-L25) [stats.sql#counter-checks](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L154-L175).

## Extension Hooks Caveat

The answer above describes PostgreSQL 12 core planner behavior. The source exposes planner statistics hooks: `get_relation_info_hook` can alter relation info after catalog loading, `get_relation_stats_hook` and `get_index_stats_hook` can supply alternative variable statistics, and `get_attavgwidth_hook` can override average-width lookup [plancat.c#get_relation_info_hook](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L57-L62) [plancat.c#get_relation_info-hook-call](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L456-L463) [selfuncs.c#stats-hooks](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L146-L149) [selfuncs.h#stats-hooks](../../../raw/postgres-12/src/include/utils/selfuncs.h#L115-L125) [lsyscache.h#get_attavgwidth_hook](../../../raw/postgres-12/src/include/utils/lsyscache.h#L61-L63).

## Evidence Map

| Claim | Primary evidence |
|---|---|
| `pg_stat_all_tables` is a monitoring view over `pg_stat_get_*()` functions. | [system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581) |
| The `pg_stat_get_*()` table functions read `PgStat_StatTabEntry` counters. | [pgstatfuncs.c#pg_stat_get_numscans](../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L40-L84), [pgstat.h#PgStat_StatTabEntry](../../../raw/postgres-12/src/include/pgstat.h#L633-L662) |
| Planner relation size uses `pg_class` `relpages`, `reltuples`, and `relallvisible`. | [pg_class.h#relpages-reltuples-relallvisible](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [heapam_handler.c#heapam_estimate_rel_size](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171) |
| Planner column selectivity reads `pg_statistic`. | [selfuncs.c#examine_simple_variable](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737), [lsyscache.c#get_attstatsslot](../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2895-L2944) |
| Planner extended statistics use `pg_statistic_ext` and `pg_statistic_ext_data`. | [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1374) |
| `pg_stats` and `pg_stats_ext` are SQL inspection views over the underlying statistics catalogs. | [system_views.sql#pg_stats](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L189-L256), [system_views.sql#pg_stats_ext](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L256-L285) |
| Planner uses index metadata from `pg_index` and index relcache data. | [relcache.c#RelationGetIndexList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4318-L4399), [plancat.c#get_relation_info-indexes](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L423) |
| Planner uses FK metadata from `pg_constraint` for join selectivity. | [relcache.c#RelationGetFKeyList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4226-L4299), [costsize.c#get_foreign_key_join_selectivity](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4694-L4906) |

## Open Questions

- None for the scope of PostgreSQL 12 core planner behavior at pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42`. Extension-provided hook behavior depends on installed extension code and is outside this pinned core-source answer.

## Source References

- [system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581) - monitoring view definition.
- [system_views.sql#pg_stats](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L189-L256) - user-facing per-column statistics view.
- [system_views.sql#pg_stats_ext](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L256-L285) - user-facing extended-statistics view.
- [pgstatfuncs.c#pg_stat_get_numscans](../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L40-L84) - examples of `pg_stat_get_*()` functions reading cumulative table counters.
- [pgstat.h#PgStat_StatTabEntry](../../../raw/postgres-12/src/include/pgstat.h#L633-L662) - cumulative table-statistics entry.
- [pg_class.h#relpages-reltuples-relallvisible](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66) - relation-level planner statistics fields.
- [relnode.c#build_simple_rel](../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L287-L293) - base-relation call into `get_relation_info()`.
- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L83-L149) - relation catalog loading and relation-size estimate call.
- [heapam_handler.c#heapam_estimate_rel_size](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171) - heap relation size estimation.
- [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L936-L1061) - index relation size estimation.
- [pg_statistic.h#FormData_pg_statistic](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L29-L124) - per-column statistics catalog structure.
- [selfuncs.c#examine_simple_variable](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4709-L4737) - plain-column `pg_statistic` lookup.
- [lsyscache.c#get_attavgwidth](../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2870-L2893) - average-width lookup from `pg_statistic`.
- [lsyscache.c#get_attstatsslot](../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L2895-L2944) - statistic-slot extraction.
- [pg_statistic_ext.h#FormData_pg_statistic_ext](../../../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L33-L56) - extended-statistics metadata catalog.
- [pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../../../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43) - extended-statistics data catalog.
- [plancat.c#get_relation_statistics](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1290-L1374) - extended-statistics loading into planner structures.
- [clausesel.c#clauselist_selectivity](../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L69-L106) - extended-statistics use during clause selectivity.
- [extended_stats.c#statext_clauselist_selectivity](../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L1170-L1202) - MCV/dependency selectivity path.
- [selfuncs.c#estimate_multivariate_ndistinct](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3573-L3642) - multivariate ndistinct use.
- [relcache.c#RelationGetIndexList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4318-L4399) - `pg_index` scan for relation indexes.
- [plancat.c#get_relation_info-indexes](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L164-L423) - index planner metadata load.
- [relcache.c#RelationGetFKeyList](../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4226-L4299) - `pg_constraint` scan for foreign keys.
- [plancat.c#get_relation_foreign_keys](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L476-L566) - FK metadata conversion into planner structures.
- [initsplan.c#match_foreign_keys_to_quals](../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2399-L2558) - FK-to-query-clause matching.
- [costsize.c#get_foreign_key_join_selectivity](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4694-L4906) - FK join selectivity.
- [plancat.c#get_relation_constraints](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1137-L1287) - CHECK, NOT NULL, and partition constraint metadata use.
- [plancat.c#relation_excluded_by_constraints](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1380-L1560) - relation exclusion using constraints and restrictions.
- [analyze.c#update_attstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L1395-L1534) - writes per-column `pg_statistic` rows.
- [extended_stats.c#statext_store](../../../raw/postgres-12/src/backend/statistics/extended_stats.c#L317-L378) - writes `pg_statistic_ext_data` values.
- [vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1157-L1196) - writes relation-level `pg_class` stats.
- [stats_ext.sql#check_estimated_rows](../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L7-L26) - helper for comparing planned and actual rows.
- [stats_ext.sql#ndistinct](../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L122-L218) - extended-statistics ndistinct regression coverage.
- [stats_ext.sql#dependencies-and-mcv](../../../raw/postgres-12/src/test/regress/sql/stats_ext.sql#L220-L360) - functional-dependency and MCV regression coverage.
- [join.sql#fk-join-estimation-test](../../../raw/postgres-12/src/test/regress/sql/join.sql#L1903-L1928) - FK join-estimation regression coverage.
- [stats.sql#statistics-collector](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L1-L25) - cumulative statistics collector test setup.
- [stats.sql#counter-checks](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L154-L175) - monitoring counter checks.

## Related Pages

- [v12/index](../index.md)
- [versions](../../versions.md)
