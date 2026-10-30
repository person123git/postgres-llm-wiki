---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Planner Factors That Produce Plans That Over-Fetch Tuples in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Main idea](#main-idea)
  - [What over-fetching means here](#what-over-fetching-means-here)
  - [Mental model: the values the planner relies on](#mental-model-the-values-the-planner-relies-on)
  - [Logic map](#logic-map)
  - [The factors](#the-factors)
  - [F1. Row-count estimates that are too high or too low](#f1-row-count-estimates-that-are-too-high-or-too-low)
  - [F2. Conditions that cannot become index conditions](#f2-conditions-that-cannot-become-index-conditions)
  - [F3. B-tree conditions that do not bound the scan](#f3-b-tree-conditions-that-do-not-bound-the-scan)
  - [F4. Lossy index results and lossy bitmaps](#f4-lossy-index-results-and-lossy-bitmaps)
  - [F5. Dead tuples and the visibility map](#f5-dead-tuples-and-the-visibility-map)
  - [F6. Join strategy](#f6-join-strategy)
  - [F7. Optimization fences that stop a condition moving down](#f7-optimization-fences-that-stop-a-condition-moving-down)
  - [F8. Security barriers and non-leakproof conditions](#f8-security-barriers-and-non-leakproof-conditions)
  - [F9. LIMIT, OFFSET, EXISTS and cursors](#f9-limit-offset-exists-and-cursors)
  - [F10. Generic cached plans](#f10-generic-cached-plans)
  - [F11. Partitions that are not pruned](#f11-partitions-that-are-not-pruned)
  - [F12. Cost parameters](#f12-cost-parameters)
  - [Settings and states that change the outcome](#settings-and-states-that-change-the-outcome)
  - [Lifecycle events that change the estimates](#lifecycle-events-that-change-the-estimates)
  - [How to see over-fetching in EXPLAIN](#how-to-see-over-fetching-in-explain)
  - [Build, catalog and extension boundaries](#build-catalog-and-extension-boundaries)
  - [Tests](#tests)
  - [Final causal summary](#final-causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

List the factors that make the query planner use a plan that over-fetches tuples during statement execution.

## Answer

### Main idea

The PostgreSQL 12 [planner](../../../glossary.md#planner) never sets out to over-fetch. It picks the [path](../../../glossary.md#path) with the lowest estimated [cost](../../../glossary.md#cost) ([planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5886-L5910)). A plan over-fetches for one of two reasons:

1. **Structural reasons.** The cheapest plan the planner can build reads rows it must then throw away, and the planner knows it. A condition that no index can apply is the typical case.
2. **Estimation reasons.** The planner believes few rows will be read, but the executor reads many, because a row estimate, a statistic, or a cost assumption was wrong.

The twelve factors below fall into one or both groups. The surprise for most readers is that the planner's row counts come from stored, possibly stale statistics and from default guesses, while the executor reads the live data. The only live correction is the current page count, plus, for inequality estimates, the histogram's end values re-read from a B-tree index; see the [mental model](#mental-model-the-values-the-planner-relies-on). When the two disagree, a plan that looked tight on paper reads far more rows than it returns.

### What over-fetching means here

In this page, a plan node over-fetches when it reads more [tuples](../../../glossary.md#tuple) from its storage or its child node than it passes up. The executor counts most of these discards, and [EXPLAIN](../../../glossary.md#explain) `ANALYZE` prints them:

| Discard | Where the executor counts it | EXPLAIN line |
|---|---|---|
| A row read by a scan fails its `Filter` | `ExecScan()` calls `InstrCountFiltered1()` ([execScan.c:229-250](../../../../raw/postgres-12/src/backend/executor/execScan.c#L229-L250)) | `Rows Removed by Filter` ([explain.c:1632-1642](../../../../raw/postgres-12/src/backend/commands/explain.c#L1632-L1642) for sequential, CTE and subquery scans; [explain.c:1588-1591](../../../../raw/postgres-12/src/backend/commands/explain.c#L1588-L1591) for index scans) |
| A heap row returned by a lossy index fails the recheck | `IndexNext()` calls `InstrCountFiltered2()` ([nodeIndexscan.c:135-150](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L135-L150)) | `Rows Removed by Index Recheck` ([explain.c:1584-1586](../../../../raw/postgres-12/src/backend/commands/explain.c#L1584-L1586)) |
| A row on a bitmap page marked for recheck fails the recheck. Every lossy page is marked, and so is an exact page whose index search asked for a recheck | `BitmapHeapNext()` calls `InstrCountFiltered2()` ([nodeBitmapHeapscan.c:315-327](../../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L315-L327), [tidbitmap.c:95-104](../../../../raw/postgres-12/src/backend/nodes/tidbitmap.c#L95-L104)) | `Rows Removed by Index Recheck` ([explain.c:1617-1619](../../../../raw/postgres-12/src/backend/commands/explain.c#L1617-L1619)) |
| A join pair fails the join condition | `ExecNestLoop()` and `ExecHashJoin()` call `InstrCountFiltered1()` ([nodeNestloop.c:214-247](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L214-L247), [nodeHashjoin.c:454-499](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L454-L499)) | `Rows Removed by Join Filter` on nested loop, merge and hash joins ([explain.c:1786-1791](../../../../raw/postgres-12/src/backend/commands/explain.c#L1786-L1791), [explain.c:1797-1817](../../../../raw/postgres-12/src/backend/commands/explain.c#L1797-L1817)) |
| A joined pair that passed the join condition fails the join node's other conditions | `ExecNestLoop()` and `ExecHashJoin()` call `InstrCountFiltered2()` ([nodeNestloop.c:233-244](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L233-L244), [nodeHashjoin.c:493-496](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L493-L496)) | `Rows Removed by Filter` on the join node ([explain.c:1792-1795](../../../../raw/postgres-12/src/backend/commands/explain.c#L1792-L1795)) |
| An index-only scan has to visit the heap | `IndexOnlyNext()` calls `InstrCountTuples2()` ([nodeIndexonlyscan.c:161-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170)) | `Heap Fetches` ([explain.c:1606-1608](../../../../raw/postgres-12/src/backend/commands/explain.c#L1606-L1608)) |
| Rows skipped by `OFFSET` | `ExecLimit()` fetches and discards rows until it passes the offset ([nodeLimit.c:90-108](../../../../raw/postgres-12/src/backend/executor/nodeLimit.c#L90-L108)) | none; compare the child's `rows` with the Limit's |

Some over-fetching has no counter. An index entry whose heap tuple is dead is fetched and skipped inside `index_getnext_slot()` without a discard count ([indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L636)), and B-tree index entries that are scanned over without being returned are not counted at all; see [F3](#f3-b-tree-conditions-that-do-not-bound-the-scan) and [F5](#f5-dead-tuples-and-the-visibility-map).

### Mental model: the values the planner relies on

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| `curpages` | Current number of heap blocks | `RelationGetNumberOfBlocks()` at plan time ([heapam_handler.c:2082-2083](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2083)) | Live | Multiplied by the stored density to get the row count |
| `reltuples / relpages` | Rows per page at the last `VACUUM` or `ANALYZE` | [`pg_class`](../../../glossary.md#reltuples-and-relpages) ([heapam_handler.c:2131-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2131-L2157)) | Stored | Scales `curpages` into `rel->tuples` |
| `relallvisible` | All-visible pages at the last `VACUUM` or `ANALYZE` | `pg_class`, written by both ([vacuumlazy.c:356-363](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L356-L363), [analyze.c:594-603](../../../../raw/postgres-12/src/backend/commands/analyze.c#L594-L603)), used unscaled ([heapam_handler.c:2159-2170](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2159-L2170)) | Stored | `allvisfrac`, the share of heap pages an index-only scan's cost assumes it can skip ([costsize.c:652-653](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L652-L653), and [costsize.c:612-613](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L612-L613) for repeated scans) |
| Column [statistics](../../../glossary.md#statistics) | Null fraction, distinct count, [most common values and histogram](../../../glossary.md#most-common-values-and-histogram), [correlation](../../../glossary.md#correlation) | `pg_statistic`, written by `ANALYZE` ([analyze.c#update_attstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1417-L1430)) | Stored | Each condition's [selectivity](../../../glossary.md#selectivity) |
| Selectivity defaults | Fixed guesses used when statistics are missing, when an inequality's comparison value is unknown at plan time, or when no variable side is found; an equality against an unknown value is averaged over the distinct values instead (F1, F10) | `selfuncs.h` ([selfuncs.h:32-49](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L32-L49)) | Compiled in | Same as column statistics |
| `rel->rows` | Rows a scan is expected to return | `tuples × clauselist_selectivity()`, clamped to at least 1 ([costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [costsize.c#clamp_row_est](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L187-L200)) | Derived | Join sizes, loop counts, LIMIT fractions |
| `startup_cost`, `total_cost` | Cost before the first row and for all rows | Each path's cost function ([costsize.c:37-51](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L37-L51)) | Derived | Path comparison |
| [Tuple fraction](../../../glossary.md#tuple-fraction) | How much of the result the caller is expected to read | `standard_planner()`, `preprocess_limit()`, `make_subplan()` ([planner.c:377-403](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L403), [subselect.c:208-214](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L214)) | Derived from the query | Keeps fast-start paths alive, and through the `Limit` path cost or the cursor and SubPlan fraction, picks one over a cheapest-total path (F9) |

The key contrast is between `curpages`, which is live, and nearly everything else in the row estimate, which is stored; the one exception follows below. The planner corrects the table size for growth since the last `VACUUM` or `ANALYZE`, because it multiplies the stored density by the live page count. It does not refresh the column statistics themselves until the next `ANALYZE`. A stored value is stale only when the data changed after it was written in a way that matters to the condition, such as a shifted correlation between two columns or a new common value.

The exception is a live correction to the histogram. When an inequality estimate reaches the first or last histogram entry, `ineq_histogram_selectivity()` replaces that entry with the column's current minimum or maximum. `get_actual_variable_range()` reads that value from a non-partial, non-hypothetical B-tree index whose first column is the column being estimated ([selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5263-L5310)). So when such an index exists, a range condition on values beyond the stored upper bound is estimated against the live maximum, while the inner histogram entries stay as `ANALYZE` stored them. An equality condition, or a column with no usable B-tree index, gets no such correction.

### Logic map

The map follows one statement through planning. Each `F` label points to the factor section that explains that branch.

```text
SQL statement
  |
  v
[1] Preprocess the query tree
  |-- CTE inlinable? ----------------- no --> CTE Scan + Filter over the CTE's rows ........ F7
  |-- condition pushable into subquery? no --> condition stays above the subquery ......... F7
  |-- RLS policy / security_barrier view? --> non-leakproof user condition waits ......... F8
  |-- partitioned table? ------------ pruning off or no usable condition --> all parts ... F11
  v
[2] Classify each condition against each index
  |-- key column + opfamily operator + collation + no volatile + securely promotable?
  |     yes --> index condition
  |       |-- B-tree boundary condition? ---- no --> scans past entries it rejects ...... F3
  |       '-- lossy for this operator? ------ yes --> heap row rechecked ................ F4
  |     no  --> Filter, evaluated after the heap row is fetched ......................... F2
  v
[3] Estimate rows = tuples x selectivity, clamp to >= 1
  |-- no statistics --> equality 1/n_distinct guess; inequality 1/3 ...................... F1
  |-- value unknown at plan time --> equality averaged over n_distinct; inequality 1/3 .. F1, F10
  |-- several conditions --> product (independence) unless extended statistics .......... F1
  '-- stored statistics older than the data (range ends re-read from a B-tree) .......... F1
  v
[4] Cost each path (startup_cost, total_cost)
  |-- bitmap heap scan: lossy pages when the bitmap outgrows work_mem .................... F4
  |-- index-only scan: heap pages fetched x (1 - allvisfrac) ............................. F5
  |-- nested loop: inner scanned once per outer row; full inner unless unique/semi/anti ... F6
  |-- hash join: whole inner input read into the hash table .............................. F6
  '-- page and CPU cost parameters, correlation ......................................... F12
  v
[5] Choose for a partial fetch
  |-- LIMIT n: fast-start paths kept; each gets a Limit path costed for n rows;
  |            the cheapest total Limit path wins ...................................... F9
  |-- cursor = cursor_tuple_fraction (default 10%), EXISTS SubPlan = 1 row,
  |   ANY SubPlan = 50%, each UNION ALL branch under a LIMIT = the LIMIT's fraction:
  |            cheapest path for that fraction ......................................... F9
  '-- partial cost assumes matches are spread evenly over the input ...................... F9
  v
[6] Cached plan? generic plan estimated without parameter values ......................... F10
  v
Executor: reads live rows; discards appear as Rows Removed by ..., Heap Fetches, OFFSET
```

Key to the map's conditions:

- [1] CTE inlining: [subselect.c:889-899](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L889-L899). Subquery pushdown: [allpaths.c#subquery_is_pushdown_safe](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3004-L3025), [allpaths.c#qual_is_pushdown_safe](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3266-L3288). Security levels: [restrictinfo.c#restriction_is_securely_promotable](../../../../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L376-L388). Pruning: [partprune.c:676-681](../../../../raw/postgres-12/src/backend/partitioning/partprune.c#L676-L681).
- [2] Index matching: [indxpath.c#match_clause_to_index](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2262-L2300), [indxpath.c:2593-2645](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2593-L2645). Filter construction: [createplan.c:2798-2848](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L2798-L2848). Boundary conditions: [selfuncs.c:5940-5975](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5940-L5975).
- [3] Row estimate: [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L69-L105), [clausesel.c:119-122](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L119-L122). Equality without statistics: [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425). Inequality without statistics: [selfuncs.c:567-645](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L567-L645). Equality against an unknown value: [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505). Inequality: [selfuncs.c:1174-1185](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1174-L1185). Histogram end re-read: [selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935).
- [4] Bitmap: [costsize.c:5554-5604](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5554-L5604). Index-only: [costsize.c:652-653](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L652-L653) for a single scan, [costsize.c:612-613](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L612-L613) for repeated scans. Nested loop: [costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471). Hash: [nodeHash.c#MultiExecPrivateHash](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L139-L175).
- [5] LIMIT: [planner.c#preprocess_limit](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2876-L2930), [planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670), [planner.c:398-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L398-L411). Cursor and SubPlan fractions: [planner.c:377-397](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L397), [subselect.c:208-214](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L214). `UNION ALL` branches: [prepunion.c:274-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L274-L280), [prepunion.c:553-565](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L553-L565), [pathnode.c#compare_fractional_path_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L118-L134). An `EXISTS` or `ANY` that the planner turns into a semi join is no SubPlan, so its fraction does not apply; see [F9](#f9-limit-offset-exists-and-cursors).
- [6] Plan cache: [plancache.c#choose_custom_plan](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1063).

### The factors

| # | Factor | Kind | Typical EXPLAIN symptom |
|---|---|---|---|
| F1 | Row-count estimates far from the real count | Estimation | `rows=` estimate far from actual `rows`; a nested loop with many `loops` |
| F2 | A condition no index can apply | Structural | `Filter:` with a large `Rows Removed by Filter` |
| F3 | B-tree conditions that do not bound the scan | Structural | `Index Cond` on a non-leading column; slow scan with few rows |
| F4 | Lossy index results and lossy bitmaps | Structural, worsened by `work_mem` | `Rows Removed by Index Recheck`; `Heap Blocks: lossy=` |
| F5 | Dead tuples and a stale `relallvisible` | Estimation | Large `Heap Fetches` on an index-only scan; slow index scan with few rows |
| F6 | Join strategy | Both | Large `loops` on the inner side; `Rows Removed by Join Filter`; a big `Hash` input |
| F7 | CTE, subquery and set-returning-function fences | Structural | `CTE Scan` or `Subquery Scan` with a `Filter`; a `Function Scan` returning many rows |
| F8 | Security barriers with non-leakproof conditions | Structural | `Filter` instead of `Index Cond` on a protected table |
| F9 | LIMIT, OFFSET, EXISTS and cursors | Estimation | A fast-start index scan under `Limit` that reads far more rows than the limit |
| F10 | Generic cached plans | Estimation | A prepared statement whose plan ignores the parameter's actual frequency |
| F11 | Partitions that are not pruned | Structural | An `Append` over partitions that return 0 rows |
| F12 | Cost parameters | Estimation | A sequential scan with a large `Rows Removed by Filter` where an index existed |

### F1. Row-count estimates that are too high or too low

Every later decision uses `rel->rows`. The planner computes it as the table's estimated tuple count times the selectivity of its conditions, then rounds it and forces it to be at least 1 ([costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [costsize.c#clamp_row_est](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L187-L200)). An error in either input becomes an error in the plan choice. Six inputs go wrong in practice:

| Input | What happens | Source |
|---|---|---|
| Missing statistics | An equality condition gets `1 / n_distinct`, where `n_distinct` is 2 for a boolean, the non-null row count for a column with a unique index, the row count for a table under 200 rows, and otherwise the default 200, so 0.005. An inequality against a constant gets the default 1/3, unless the column is `ctid`, which is estimated from the table size instead. The header also defines 0.005 for pattern matches and for range pairs | [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425), [selfuncs.c:567-645](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L567-L645), [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5019-L5116), [selfuncs.h:32-49](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L32-L49) |
| A comparison value unknown at plan time, such as a parameter in a generic plan | An equality condition is averaged over all distinct values: `(1 − null fraction) / n_distinct`, capped by the most common value's frequency; see [F10](#f10-generic-cached-plans). An inequality gets the default 1/3. Equality falls back to the fixed 0.005 only when the condition does not compare an expression in one table's columns with something that contains none of them, for example `a = b + 1` with columns on both sides | [selfuncs.c:269-276](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L269-L276), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505), [selfuncs.c:1174-1185](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1174-L1185), [selfuncs.c:258-262](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L258-L262), [selfuncs.c#get_restriction_variable](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4292-L4313) |
| Several conditions on one table | Their selectivities are multiplied, which assumes they are independent; only [extended statistics](../../../glossary.md#extended-statistics) created with `CREATE STATISTICS` correct for correlated columns | [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L55-L105), [clausesel.c:119-122](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L119-L122) |
| Statistics sample size | `ANALYZE` samples 300 rows per unit of the statistics target, whose default is 100; values outside the sample are estimated, not counted | [analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700), [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994) |
| Statistics older than the data | Column statistics change only when `ANALYZE` runs; the table size is scaled from live pages, but the per-column fractions are not. The one live correction is an inequality reaching a histogram end, which is re-read from a non-partial B-tree index when one exists | [heapam_handler.c:2131-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2131-L2157), [analyze.c#update_attstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1417-L1430), [selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935) |
| `relpages = 0` | With `relpages = 0`, fewer than 10 current pages and no inheritance children, the planner assumes 10 pages, so that a newly created table is not assumed to stay small. `relpages = 0` stands in for "never vacuumed", so the rule also fires on a table that was vacuumed or analyzed while empty. With `relpages = 0`, the density comes from the column widths, not from stored counts | [heapam_handler.c:2090-2118](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2090-L2118), [heapam_handler.c:2132-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2132-L2157) |

An estimate that is too **low** causes over-fetching because it makes repeated work look cheap. For example, a nested loop rescans its inner side once per outer row, so its cost grows with the outer estimate ([costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471)). If the outer side was estimated at 1 row and returns 50,000, the inner side is read 50,000 times. Independence multiplication tends to push estimates low, because a product of fractions shrinks with every added condition.

An estimate that is too **high** causes over-fetching under a LIMIT. The planner treats `LIMIT n` as "read n of the estimated rows", so a high estimate makes `n` look like a small fraction and favors a fast-start plan; see [F9](#f9-limit-offset-exists-and-cursors).

### F2. Conditions that cannot become index conditions

A condition helps an index only if the planner can turn it into an index condition. Otherwise the scan fetches every row the index condition admits and tests the condition afterward as a `Filter`. `create_indexscan_plan()` builds that filter list as "the scan's conditions minus those the index handles" ([createplan.c:2798-2848](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L2798-L2848)).

`match_clause_to_index()` and `match_opclause_to_indexcol()` decide whether a condition can be used. A condition qualifies only if all of these hold:

| Requirement | Fails for, for example | Source |
|---|---|---|
| Not a pseudoconstant | A condition with no column of the current query level and no volatile function, such as `$1 > 0`; the planner runs it once in a gating `Result` node instead | [indxpath.c:2269-2276](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2269-L2276), [pathnodes.h:1918-1926](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L1918-L1926) |
| Securely promotable | A non-leakproof condition on a table with a security-barrier condition; see [F8](#f8-security-barriers-and-non-leakproof-conditions) | [indxpath.c:2278-2283](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2278-L2283) |
| One side matches an index key column or key expression | `lower(email) = $1` against an index on `email`; any condition on an `INCLUDE` column, because only key columns are tried | [indxpath.c:2285-2286](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2285-L2286), [indxpath.c:2593-2595](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2593-L2595) |
| The other side has no volatile function and no column of the same table | `col > random()`; `a = b + 1` on one table | [indxpath.c:2593-2595](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2593-L2595) |
| The operator belongs to the index column's [operator family](../../../glossary.md#operator-class) and the collations match | A cross-type comparison with no operator in the family; a comparison under a different collation | [indxpath.c:2597-2607](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2597-L2607) |
| Or a planner support function can derive an index condition | Pattern matches whose support function cannot build one | [indxpath.c:2609-2620](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2609-L2620) |

When no index condition can be built at all, the planner usually falls back to a [sequential scan](../../../glossary.md#sequential-scan) with the condition as its filter, which reads the whole table. The documentation's example shows the counter for this case: `Rows Removed by Filter: 3000` on a scan that returned 7000 rows ([perform.sgml:628-646](../../../../raw/postgres-12/doc/src/sgml/perform.sgml#L628-L646)).

### F3. B-tree conditions that do not bound the scan

A [B-tree](../../../glossary.md#b-tree) index condition can be used and still not narrow the scan. Only equality conditions on the leading columns, plus inequality conditions on the next column, decide where the scan starts and stops. The B-tree code marks those keys "required". For quals `x = 1 AND y < 4 AND z < 5`, it rejects `(1,2,7)` but must keep scanning, and with no condition on the first column, "`WHERE y = 4` requires a full-index scan" ([nbtutils.c:685-699](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L685-L699)).

The planner knows this. `btcostestimate()` counts only these boundary conditions when it estimates how many index tuples the scan reads, and it allows the other conditions only to reduce heap visits ([selfuncs.c:5940-5975](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5940-L5975), [selfuncs.c:6036-6075](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6036-L6075)). So the plan reads index entries it rejects, and it was costed to do so. It is still chosen when the alternatives cost more. The executor does not count the rejected index entries, so EXPLAIN shows only the time and buffer traffic.

### F4. Lossy index results and lossy bitmaps

Some index searches return candidates rather than answers. When the index access method sets `xs_recheck`, the index scan re-tests the original condition on each heap row and drops the failures ([nodeIndexscan.c:135-150](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L135-L150)). The documentation's example is a GiST polygon containment search, where the index returns overlapping polygons and the exact test removes one of them ([perform.sgml:650-691](../../../../raw/postgres-12/doc/src/sgml/perform.sgml#L650-L691)).

A [bitmap scan](../../../glossary.md#bitmap-scan) can become lossy because of memory. Its [TID](../../../glossary.md#tid) bitmap is limited by [`work_mem`](../../../glossary.md#work_mem) ([nodeBitmapIndexscan.c:93-95](../../../../raw/postgres-12/src/backend/executor/nodeBitmapIndexscan.c#L93-L95)), and when it grows too large it keeps whole pages instead of single tuples. Every tuple on a [lossy](../../../glossary.md#lossy-bitmap) page is rechecked ([nodeBitmapHeapscan.c:315-327](../../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L315-L327)).

The planner models this. `compute_bitmap_pages()` compares the heap pages the bitmap must cover with the entries that fit in `work_mem`. When they do not fit, it estimates `lossy_pages = heap_pages - maxentries / 2` and charges every tuple on those pages as fetched ([costsize.c:5554-5604](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5554-L5604)). The source calls this a "crude approximation", so the real number of lossy pages can differ from the estimate. `work_mem` defaults to 4096 kB ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2241)).

### F5. Dead tuples and the visibility map

An index points at heap tuples whether or not they are visible. `index_getnext_slot()` fetches the heap tuple for each index entry and, when none is visible to the [snapshot](../../../glossary.md#snapshot), moves on to the next entry ([indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L636)). These [dead tuples](../../../glossary.md#dead-tuple) are fetched and skipped with no EXPLAIN counter. When a whole [HOT](../../../glossary.md#hot) chain is dead to every transaction, the scan marks the index entry [killed](../../../glossary.md#killed-index-tuple) so later scans skip it, except on a standby where the transaction started in recovery ([indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L589)). Until [`VACUUM`](../../../glossary.md#vacuum) removes them, then, dead tuples are a source of over-fetching that the planner does not see in its row count.

An [index-only scan](../../../glossary.md#index-only-scan) has the same problem in a different form. It must visit the heap for every entry whose heap page is not marked all-visible in the [visibility map](../../../glossary.md#visibility-map) ([nodeIndexonlyscan.c:121-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)). The executor reads the map's bit for each entry at run time, so the map itself is live ([nodeIndexonlyscan.c:161-163](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L163)). The planner instead predicts those visits from `allvisfrac`, which comes from the stored `relallvisible`, and scales the heap pages it charges by `1 - allvisfrac`. `relallvisible` is not scaled up for pages added since it was stored, and it does not see bits cleared since then ([heapam_handler.c:2159-2170](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2159-L2170), [costsize.c:652-653](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L652-L653)). `INSERT`, the multi-row insert that `COPY` uses, `DELETE` and `UPDATE` clear the all-visible bit of the pages they change (`heap_insert()` at [heapam.c:1916-1922](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1916-L1922), `heap_multi_insert()` at [heapam.c:2208-2215](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2208-L2215), `heap_delete()` at [heapam.c:2704-2710](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2704-L2710), `heap_update()` at [heapam.c:3667-3681](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3667-L3681)). So after many changes since the last `VACUUM` or `ANALYZE`, fewer pages are all-visible than `relallvisible` says, and `Heap Fetches` is higher than the plan assumed.

### F6. Join strategy

The join methods read their inputs differently, so the chosen method sets how much is read.

| Join method | What it reads | When it over-fetches | Source |
|---|---|---|---|
| [Nested loop](../../../glossary.md#nested-loop-join) | Rescans the inner plan once per outer row | Outer row count underestimated (F1); inner side is a sequential scan | [nodeNestloop.c:100-160](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L100-L160), [costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471) |
| Nested loop or hash join with a non-unique inner side | Keeps reading inner rows after the first match | The inner side is in fact unique, but the planner could not prove it | [nodeNestloop.c:318-322](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L318-L322), [nodeHashjoin.c:682-683](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L682-L683), [nodeHashjoin.c:486-491](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L486-L491), [joinpath.c:146-190](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L146-L190) |
| [Hash join](../../../glossary.md#hash-join) | Reads the whole inner input into the hash table first | Few outer rows find a match; the inner side is large | [nodeHash.c#MultiExecPrivateHash](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L139-L175) |
| Any join with a join condition that is not the hash or merge key | Pairs every match on the key, then tests the rest | Most key matches fail the extra condition | [nodeHashjoin.c:454-499](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L454-L499), [nodeNestloop.c:214-247](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L214-L247) |

The second row needs a note. A [semi join](../../../glossary.md#semi-join-and-anti-join) returns each outer row that has at least one inner match, once; an anti join returns each outer row that has none ([nodes.h#JoinType-semi](../../../../raw/postgres-12/src/include/nodes/nodes.h#L702-L712)). `ExecNestLoop()` stops after the first inner match only when the plan says the inner side is unique or the join is a semi join; an anti join stops on its own when it finds a match ([nodeNestloop.c:214-230](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L214-L230), [nodeNestloop.c:318-322](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L318-L322)). A hash join applies the same single-match rule ([nodeHashjoin.c:682-683](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L682-L683), [nodeHashjoin.c:486-491](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L486-L491)). The planner sets "unique" through `innerrel_is_unique()`, which needs a proof such as a unique index on the join columns ([joinpath.c:146-190](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L146-L190)). Without the proof, the cost model also charges for a full inner scan per outer row, so it does not hide the extra reads ([costsize.c:2450-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2450-L2471)).

The hash join has one shortcut. Before building, it may fetch one outer row, and if the outer side is empty it returns without reading the inner side. It never tries this when the join must emit unmatched inner rows, such as a right or full join, or when it is a parallel hash join. It always tries it for a join that must emit unmatched outer rows, such as a left join. Otherwise it tries it only when the outer side's startup cost is below the hash side's total cost and no previous scan found the outer side non-empty ([nodeHashjoin.c:225-273](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L225-L273)).

### F7. Optimization fences that stop a condition moving down

A condition reduces fetching only if it reaches the scan. Three query shapes stop it:

| Shape | Default behavior in 12 | What stops the condition | Source |
|---|---|---|---|
| Common table expression | A non-recursive `SELECT` CTE referenced once, with no volatile function, is inlined like a subquery | `MATERIALIZED`, a second reference (unless the CTE is written `NOT MATERIALIZED`), recursion, data-modifying content, or a volatile function keeps it as a separate plan that the outer condition only filters. A `NOT MATERIALIZED` CTE referenced more than once is also kept when it refers to a recursive CTE of an outer query level | [subselect.c:895-896](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L895-L896), [subselect.c:870-899](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L870-L899) |
| Subquery in `FROM` | Outer conditions on its output are pushed into it | `LIMIT`/`OFFSET` in the subquery, or an `EXCEPT` in its set operations, blocks all pushdown; an outer condition that contains a sub-select is never pushed; `DISTINCT`, window functions and set-returning functions block volatile conditions; unsafe output columns, including set-operation columns whose types differ between the branches, block conditions that reference them | [allpaths.c:3073](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3073), [allpaths.c:3274](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3274), [allpaths.c:3045](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3045), [allpaths.c#subquery_is_pushdown_safe](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3004-L3025), [allpaths.c#check_output_expressions](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3123-L3170), [allpaths.c#qual_is_pushdown_safe](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3266-L3288) |
| Set-returning function in `FROM` | An SQL-language function that the planner can inline becomes a subquery, so the subquery rules above apply instead. Inlining needs a single function without `WITH ORDINALITY`, no volatile function and no sub-select in its arguments, `EXECUTE` permission, no loaded plugin that hooks the function's entry and exit, a non-strict, non-volatile, non-`SECURITY DEFINER` function with no `SET` clause that does not return `SETOF void`, and a body that is a single `SELECT`. The body must also return the declared result type without changes to its select list, return a whole row when the result type is composite, and match the column list given in the query when it returns `record`. Any other function scan reads the function's entire result into a tuplestore on the first call | For a function scan, any outer condition or `LIMIT` acts only after all rows were produced; the planner's row estimate is `prorows`, 1000 unless the function declares otherwise or has a support function | [prepjointree.c:620-640](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L620-L640), [clauses.c#inline_set_returning_function](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4911-L4982), [clauses.c:5065-5074](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L5065-L5074), [clauses.c:5076-5117](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L5076-L5117), [nodeFunctionscan.c:85-100](../../../../raw/postgres-12/src/backend/executor/nodeFunctionscan.c#L85-L100), [execSRF.c#ExecMakeTableFunctionResult](../../../../raw/postgres-12/src/backend/executor/execSRF.c#L92-L101), [functioncmds.c:1134-1140](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1134-L1140), [plancat.c:1986-2011](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1986-L2011) |

The regression tests show the [CTE](../../../glossary.md#common-table-expression) difference directly. An inlined CTE puts `f1 = 1` on the base table's scan, while the same CTE written `AS MATERIALIZED` produces a `CTE Scan` with `Filter: (x.f1 = 1)` above a sequential scan that has no condition ([subselect.out:1243-1266](../../../../raw/postgres-12/src/test/regress/expected/subselect.out#L1243-L1266)).

A materialized CTE is read on demand, not all at once: `CteScanNext()` asks the CTE's plan for the next row only when its tuplestore runs out ([nodeCtescan.c:87-120](../../../../raw/postgres-12/src/backend/executor/nodeCtescan.c#L87-L120)). So a `LIMIT` above a CTE scan can stop the CTE early. A filter above it cannot reduce what the CTE's own scan reads.

**Non-default cause.** `MATERIALIZED` is an explicit option. Default: a singly referenced, side-effect-free CTE is inlined. Cause: `ctematerialized` is `CTEMaterializeAlways`. Mechanism: the inlining test at [subselect.c:889-899](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L889-L899) accepts only `CTEMaterializeNever`, or `CTEMaterializeDefault` with one reference, so the CTE keeps its own subplan.

### F8. Security barriers and non-leakproof conditions

[Row-level security](../../../glossary.md#row-level-security) policies and [`security_barrier`](../../../glossary.md#security-barrier) views add conditions that must run before any user condition that could leak data. A user condition that calls a function not marked [`LEAKPROOF`](../../../glossary.md#leakproof-function) is therefore held back:

- It cannot become an index condition, because `restriction_is_securely_promotable()` rejects a condition whose security level is above the table's lowest level unless it is leakproof ([restrictinfo.c#restriction_is_securely_promotable](../../../../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L376-L388), [indxpath.c:2278-2283](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2278-L2283)).
- It cannot be pushed into a `security_barrier` view's subquery when it contains leaky variables ([allpaths.c:2221-2227](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2221-L2227), [allpaths.c:3283-3286](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3283-L3286)).

Either way, the scan reads the rows the security condition admits, and only then applies the user's condition as a filter.

**Non-default cause.** Default: a table has no row-level security and a view has no `security_barrier` flag, so every user condition is at the lowest security level. Cause: row-level security enabled on the table for the querying role, which `check_enable_rls()` decides ([rls.c#check_enable_rls](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L52-L133), called at [rowsecurity.c:134](../../../../raw/postgres-12/src/backend/rewrite/rowsecurity.c#L134)), or a view created with `security_barrier`. Mechanism: the security conditions take lower `security_level` values, so the promotability test above fails for a non-leakproof user condition.

### F9. LIMIT, OFFSET, EXISTS and cursors

By default the planner optimizes for reading every row: `standard_planner()` starts with a tuple fraction of 0, and `get_cheapest_fractional_path()` then returns the cheapest-total path ([planner.c:398-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L398-L411), [planner.c:5892-5894](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5892-L5894)). Four constructs change the fraction:

| Construct | Tuple fraction | Source |
|---|---|---|
| Constant `LIMIT n` (plus a constant `OFFSET m`) | `n + m` rows | [planner.c:2876-2930](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2876-L2930) |
| A `LIMIT` whose count or `OFFSET` is not a constant at plan time | 10% of the rows. The branch needs a `LIMIT`; an `OFFSET` with no `LIMIT` changes the fraction only when the caller already passed a fraction | [planner.c:2876-2886](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2876-L2886), [planner.c:2932](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2932) |
| `EXISTS` sub-select planned as a SubPlan | 1 row; an `ANY`/`ALL` SubPlan: 50%. This applies only when the sub-select stays a SubPlan. At the top level of `WHERE` or `JOIN ... ON`, the planner usually turns `EXISTS` and `IN`/`ANY` into a [semi join](../../../glossary.md#semi-join-and-anti-join), which is planned as a join, not with this fraction | [subselect.c:208-214](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L214), [prepjointree.c#pull_up_sublinks](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L168-L186), [subselect.c#convert_EXISTS_sublink_to_join](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L1317) |
| SQL `DECLARE ... CURSOR` | `cursor_tuple_fraction`, default 0.1 | [gram.y:11137](../../../../raw/postgres-12/src/backend/parser/gram.y#L11137), [planner.c:377-397](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L397), [planmain.h:21](../../../../raw/postgres-12/src/include/optimizer/planmain.h#L21) |

With a fraction, the planner prices a path as if it stops early. The cost of reading a fraction `f` of a path's rows is:

```text
startup_cost + f × (total_cost − startup_cost),   f = rows wanted / path rows
```

Here `startup_cost` is the cost before the first row and `total_cost` the cost of all rows ([costsize.c:37-51](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L37-L51)). In plain words: the planner assumes the rows it wants are spread evenly through the input, so reading 10 of 10,000 expected rows costs 0.1% of the run.

The formula reaches the plan choice by two routes: through the cost of a `Limit` path (first row), or through `get_cheapest_fractional_path()` directly (second and third rows). `path rows` means something slightly different in each:

| Construct | How the fraction reaches the choice | `path rows` | Source |
|---|---|---|---|
| `LIMIT` and `OFFSET` in the query | `grouping_planner()` stores the adjusted fraction in `root->tuple_fraction`. A fraction above 0 sets `consider_startup`, so `add_path()` keeps paths with a cheap startup cost ([pathnode.c:155-214](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L155-L214), [pathnode.c:663-687](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L663-L687)). Each surviving path then gets its own `Limit` path, and `adjust_limit_rows_costs()` sets that path's total cost with the formula. `standard_planner()` then calls `get_cheapest_fractional_path()` with fraction 0 and takes the cheapest-total `Limit` path. For a `DECLARE CURSOR` that also has a `LIMIT`, that final call uses `cursor_tuple_fraction` instead of 0, so the `Limit` paths are compared on their partial cost ([planner.c:377-397](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L397), [planner.c:411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L411)) | The row estimate of the path under each `Limit` | [planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c:3598](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3598), [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670), [planner.c:398-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L398-L411) |
| SQL cursor, `EXISTS` or `ANY`/`ALL` SubPlan | `standard_planner()` or `make_subplan()` passes the fraction to `get_cheapest_fractional_path()`, which compares every path's partial cost directly | The row estimate of the relation's cheapest-total path, used to turn an absolute row count into a fraction | [planner.c:401-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L401-L411), [subselect.c:233](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L233), [planner.c:5896-5897](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5896-L5897), [pathnode.c#compare_fractional_path_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L118-L134) |
| Each branch of a top-level `UNION ALL` under a `LIMIT` | `recurse_set_operations()` plans the branch with `root->tuple_fraction`, the `LIMIT`'s fraction, and keeps the single path `get_cheapest_fractional_path()` picks for it. Plain `UNION` resets the fraction to 0 for its branches, and so does an `ORDER BY` on the set operation | The row estimate of the branch's cheapest-total path | [prepunion.c:235-239](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L235-L239), [prepunion.c:274-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L274-L280), [prepunion.c:553-565](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L553-L565), [planner.c:1857-1868](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1857-L1868) |

When the candidate paths have the same row estimate, the two routes rank them the same way, because both divide by the same `path rows`. The distinction still matters when reading the source: for a statement whose `LIMIT` sits directly over a scan or join, the `LIMIT` fraction never reaches `get_cheapest_fractional_path()` at the top, which receives 0, or `cursor_tuple_fraction` for a cursor ([planner.c:398-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L398-L411)). It reaches that function only through the `UNION ALL` branches in the last row.

**Worked calculation.** This example uses illustrative numbers, not measured ones. Take `SELECT ... WHERE status = 'rare' ORDER BY created_at LIMIT 10`, with an index on `created_at` and none on `status`:

| Step | Value |
|---|---|
| Path A: index scan on `created_at` with `status = 'rare'` as a filter | startup 0.4, total 40,000, estimated 10,000 matching rows |
| Fraction for path A | f = 10 / 10,000 = 0.001, the `Limit` count over path A's row estimate |
| Total cost of the `Limit` over A | 0.4 + 0.001 × (40,000 − 0.4) ≈ 40.4 |
| Path B: sequential scan, then a top-N sort | `cost_sort()` starts the sort's startup cost from the total cost of its input ([costsize.c:1678](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1678), [costsize.c:1693-1698](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1693-L1698)); with an assumed scan cost of 20,000, the total cost of the `Limit` over B is above 20,000 |
| Choice | The `Limit` over path A, the cheaper total |

If only 10 rows match and they all have recent `created_at` values, path A reads almost the whole index and table in order before it finds them. The plan was chosen because of two assumptions: the estimate of 10,000 matches (F1) and the even spread. Neither is checked at run time.

`OFFSET` is a simpler case. The `Limit` node must fetch and throw away `m` rows before returning any ([nodeLimit.c:90-108](../../../../raw/postgres-12/src/backend/executor/nodeLimit.c#L90-L108)), and the planner charges those rows to the startup cost ([pathnode.c:3633-3651](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3633-L3651)). The over-fetch is part of the query's meaning, not an estimation error.

**Non-default cause for cursors.** Default: a statement is planned for all its rows (fraction 0). Cause: SQL `DECLARE` adds `CURSOR_OPT_FAST_PLAN` to the cursor options ([gram.y:11137](../../../../raw/postgres-12/src/backend/parser/gram.y#L11137)). Mechanism: `standard_planner()` then uses `cursor_tuple_fraction` instead of 0, turning 1.0 into 0 and 0 into 1e-10 ([planner.c:377-397](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L397)). `cursor_tuple_fraction` is a `user`-context setting, so a session or transaction can `SET` it; 1.0 plans cursors like ordinary queries ([guc.c#cursor_tuple_fraction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3318-L3326), [config.sgml#guc-cursor-tuple-fraction](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5219-L5236)).

### F10. Generic cached plans

A [prepared statement](../../../glossary.md#prepared-statement) can run a [generic plan](../../../glossary.md#custom-and-generic-plan), built without the parameter values. For an equality against an unknown value, `eqsel` uses `var_eq_non_const()`, which assumes the value is equally likely to be any distinct value: `(1 − null fraction) / n_distinct`, capped by the frequency of the most common value ([selfuncs.c:269-276](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L269-L276), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505)). An inequality against an unknown value gets the default 1/3 ([selfuncs.c:1174-1185](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1174-L1185)). So a generic plan tuned for an average value over-fetches when it runs with a very common value, because it was built for a small match count. For example, it may pick a nested loop whose inner side is then rescanned for each of the many matching outer rows (F6).

By default (`plan_cache_mode = auto`), a statement with parameters gets custom plans for its first five executions, then switches to the generic plan when the generic plan's cost is below the average custom-plan cost, which includes a charge for planning. Five earlier checks come first, in this order, so the five-execution rule never applies to the statements they decide ([plancache.c#choose_custom_plan](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1063), [guc.c#plan_cache_mode](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4504-L4515)):

1. A one-shot plan is always custom.
2. A statement with no parameters always uses the generic plan.
3. A transaction-control statement always uses the generic plan.
4. `plan_cache_mode = force_generic_plan` or `force_custom_plan` forces the choice; see the non-default note below.
5. The caller's `CURSOR_OPT_GENERIC_PLAN` or `CURSOR_OPT_CUSTOM_PLAN` option forces the choice. The two flags are defined in `parsenodes.h`, and a search of `src/`, `contrib/` and `src/pl/` found no code in 12 that sets them; `plancache.c` only tests them ([parsenodes.h:2690-2691](../../../../raw/postgres-12/src/include/nodes/parsenodes.h#L2690-L2691), [plancache.c:1038-1041](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1038-L1041)). The [plan_cache_mode page](plan-cache-mode.md) explains that decision in full.

**Non-default cause.** `plan_cache_mode = force_generic_plan` skips the custom-plan comparison ([plancache.c#choose_custom_plan](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1031-L1035)). It is a `user`-context setting, so a session or transaction can `SET` it.

### F11. Partitions that are not pruned

On a partitioned table, [partition pruning](../../../glossary.md#partition-pruning) removes partitions whose bounds cannot match the conditions. It returns every partition when pruning is disabled or there are no conditions to prune with ([partprune.c:676-681](../../../../raw/postgres-12/src/backend/partitioning/partprune.c#L676-L681), [inherit.c:330](../../../../raw/postgres-12/src/backend/optimizer/util/inherit.c#L330)). It also prepares run-time pruning for conditions it can evaluate only during execution ([createplan.c:1211-1222](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L1211-L1222)). A condition on a non-key column, or one that does not compare the key in a way pruning understands, leaves every partition in the plan, and each one is scanned. The [partitioning page](partitioning-planning-execution-optimizations.md) lists the rules.

**Non-default cause.** `enable_partition_pruning` defaults to on ([guc.c#enable_partition_pruning](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1044-L1054)). Setting it off makes `prune_append_rel_partitions()` ([partprune.c:663](../../../../raw/postgres-12/src/backend/partitioning/partprune.c#L663)) return all partitions before it builds any pruning steps for `get_matching_partitions()`, at [partprune.c:680-681](../../../../raw/postgres-12/src/backend/partitioning/partprune.c#L680-L681). It is a `user`-context setting.

### F12. Cost parameters

The cost parameters decide between plans that read different amounts. Their compiled-in defaults are `seq_page_cost` 1.0, `random_page_cost` 4.0, `cpu_tuple_cost` 0.01, `cpu_index_tuple_cost` 0.005 and `cpu_operator_cost` 0.0025, and `effective_cache_size` 524288 pages ([cost.h:24-32](../../../../raw/postgres-12/src/include/optimizer/cost.h#L24-L32)). A high `random_page_cost` makes an index scan look expensive, so the planner may prefer a [sequential scan](../../../glossary.md#sequential-scan) that reads every row and filters most of them away ([seq_page_cost and random_page_cost](../../../glossary.md#seq_page_cost-and-random_page_cost)).

The heap I/O cost of an index scan also depends on the column's correlation with physical order. `cost_index()` interpolates between the uncorrelated and perfectly correlated I/O cost by the square of the correlation ([costsize.c:708-714](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L708-L714)), and `btcostestimate()` takes the correlation from `pg_statistic`, scaled by 0.75 for a multicolumn index ([selfuncs.c:6190-6210](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6190-L6210)). The correlation is stored, so it ages like the other statistics in F1.

All six cost settings are `user`-context settings in 12 ([guc.c:3206-3260](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3206-L3260), [guc.c#effective_cache_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3108-L3118)), so a session can `SET` them to test a different plan. Changing them changes plan choice; it does not make any estimate more accurate.

### Settings and states that change the outcome

| Condition or state | Default | Effect on over-fetching | Change scope |
|---|---|---|---|
| `default_statistics_target` | 100 ([guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994)) | Larger sample and lists, better estimates (F1) | `user`: `SET`; affects only an `ANALYZE` run later in the same session, because `ANALYZE` reads the value in the backend that runs it ([analyze.c:1659](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1659)). Autovacuum runs its analyze inside its own worker process, so it never sees a session's `SET`. The worker re-reads the configuration file when the server is reloaded, so changing the value there and reloading reaches autovacuum ([autovacuum.c:3103-3117](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3103-L3117), [autovacuum.c:2560-2568](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2560-L2568)) |
| `CREATE STATISTICS` object on the table | None | Corrects independence for the covered columns (F1) | Takes effect after `ANALYZE` builds it |
| `work_mem` | 4096 kB ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2241)) | Larger values reduce lossy bitmap pages (F4) | `user`: `SET` |
| `cursor_tuple_fraction` | 0.1 ([guc.c#cursor_tuple_fraction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3318-L3326)) | Lower values favor fast-start plans for cursors (F9) | `user`: `SET` |
| `plan_cache_mode` | `auto` ([guc.c#plan_cache_mode](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4504-L4515)) | `force_generic_plan` removes the custom-plan check (F10) | `user`: `SET` |
| `enable_partition_pruning` | on ([guc.c#enable_partition_pruning](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1044-L1054)) | off keeps all partitions (F11) | `user`: `SET` |
| `constraint_exclusion` | `partition` ([guc.c#constraint_exclusion](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4243-L4253)) | At `partition`, [constraint exclusion](../../../glossary.md#constraint-exclusion) runs only for appendrel members such as inheritance children; `off` skips it, so children whose constraints rule them out are scanned ([plancat.c:1433-1458](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1433-L1458)) | `user`: `SET` |
| `seq_page_cost` and the other cost settings | See F12 | Shift choice between seq and index plans (F12) | `user`: `SET` |
| `autovacuum_analyze_threshold`, `autovacuum_analyze_scale_factor` | 50 and 0.1 ([guc.c#autovacuum_analyze_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2957-L2963), [guc.c#autovacuum_analyze_scale_factor](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3404-L3410)) | Lower values make [autovacuum](../../../glossary.md#autovacuum) refresh statistics sooner (F1) | `sighup`: reload |
| Row-level security or `security_barrier` | Off | Holds back non-leakproof conditions (F8) | DDL |
| `MATERIALIZED` on a CTE | Not written | Fences the CTE (F7) | Query text |

The `user`-context settings can be changed with `SET` in a session or `SET LOCAL` in a transaction, and the two autovacuum settings need a reload, because `PGC_USERSET` and `PGC_SIGHUP` map to those scopes.

### Lifecycle events that change the estimates

| Event | State before | State after | Effect on later planning |
|---|---|---|---|
| Bulk `INSERT`, `UPDATE`, `DELETE` | Statistics describe the old data | Data changed; statistics unchanged; table size is still scaled from live pages | Column fractions may be stale (F1); dead tuples accumulate (F5) |
| `ANALYZE` (manual or autovacuum) | Old `pg_statistic` rows | New sample-based rows; `relpages`, `reltuples` and `relallvisible` updated ([analyze.c#update_attstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1417-L1430), [analyze.c:594-603](../../../../raw/postgres-12/src/backend/commands/analyze.c#L594-L603)) | New selectivities for plans made afterward |
| Autovacuum analyze trigger | Changed tuples below threshold | Analyze runs when changes exceed `threshold + scale_factor × reltuples` ([autovacuum.c:3067-3080](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3067-L3080)) | Same as `ANALYZE` |
| `VACUUM` | Dead tuples in heap and index; visibility bits unset | Dead tuples removed; pages marked all-visible; `relpages`, `reltuples` and `relallvisible` updated ([vacuumlazy.c:1311-1314](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1311-L1314), [vacuumlazy.c:356-363](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L356-L363)) | Fewer skipped fetches (F5); higher `allvisfrac` |
| First `VACUUM`/`ANALYZE` of a table that has rows | `relpages = 0`; planner assumes at least 10 pages when the table has fewer and no inheritance children | Real density and a non-zero `relpages` stored | The 10-page minimum stops applying. If the table was still empty, `relpages` stays 0 and the minimum keeps applying ([heapam_handler.c:2090-2118](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2090-L2118)) |
| Five executions of a prepared statement with parameters | Custom plans | Generic plan may be chosen | Parameter-value-blind estimates (F10) |

### How to see over-fetching in EXPLAIN

`EXPLAIN (ANALYZE, BUFFERS)` runs the statement and prints the actual rows, loops and discard counters described above ([ref/explain.sgml#ANALYZE](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L121-L128)). The [EXPLAIN BUFFERS page](../observability/explain-analyze-buffers-output.md) explains the buffer counters. A template for a read-only `SELECT`:

```sql
SET /* wiki_overfetch_check */ statement_timeout = '60s';
SET /* wiki_overfetch_check */ lock_timeout = '5s';
EXPLAIN /* wiki_overfetch_check */ (ANALYZE, BUFFERS)
SELECT ... ;   -- replace with the statement under investigation
RESET /* wiki_overfetch_check */ statement_timeout;
RESET /* wiki_overfetch_check */ lock_timeout;
```

`EXPLAIN ANALYZE` executes the statement, so run a data-modifying statement inside `BEGIN; ... ROLLBACK;`. Read the output in this order:

1. Compare each node's estimated `rows` with the actual `rows`. A large gap points to F1, F9 or F10.
2. Look for `Rows Removed by Filter`, `Rows Removed by Index Recheck`, `Rows Removed by Join Filter` and `Heap Fetches`. They point to F2, F4, F6 and F5.
3. Multiply the inner side's actual `rows` by `loops` to see how many rows a nested loop read in total (F6).
4. Look for `CTE Scan`, `Subquery Scan` or `Function Scan` nodes that carry a `Filter` (F7).

### Build, catalog and extension boundaries

- **Catalog data.** Each operator's selectivity estimator is its `pg_operator.oprrest` entry, such as `eqsel`, defined in `pg_operator.dat` ([pg_operator.dat:23](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L23)), which the build compiles into the bootstrap catalog ([catalog/Makefile:60-68](../../../../raw/postgres-12/src/backend/catalog/Makefile#L60-L68)). An extension operator with no `oprrest` gets a selectivity of 0.5 ([plancat.c#restriction_selectivity](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1770-L1786)), which can make the planner badly misjudge a condition on that operator.
- **Hooks.** `get_relation_info_hook`, `set_rel_pathlist_hook` and `get_relation_stats_hook` let an extension change the relation data, the paths, or the statistics the planner sees ([plancat.h:21-25](../../../../raw/postgres-12/src/include/optimizer/plancat.h#L21-L25), [paths.h:29-33](../../../../raw/postgres-12/src/include/optimizer/paths.h#L29-L33), [selfuncs.h:116-120](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L116-L120)). This page assumes no hook is loaded.
- **Functions.** `prorows` and planner support functions set a set-returning function's row estimate ([plancat.c:1986-2011](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1986-L2011)); `LEAKPROOF` decides F8.
- **Generated files.** The cursor flag in F9 is set by a grammar action in `gram.y`, which the build turns into `gram.c` with Bison through the generic `%.c: %.y` rule ([Makefile.global.in:741-744](../../../../raw/postgres-12/src/Makefile.global.in#L741-L744), [parser/Makefile:35-38](../../../../raw/postgres-12/src/backend/parser/Makefile#L35-L38)). Apart from that and the catalog data above, the mechanisms on this page live in hand-written C.

### Tests

The regression suite tests the individual mechanisms, not over-fetching as such. A search of `src/test/regress` found no test that checks the LIMIT interpolation against the rows actually read.

| Mechanism | Test |
|---|---|
| CTE inlining versus `MATERIALIZED` | [subselect.out:1243-1266](../../../../raw/postgres-12/src/test/regress/expected/subselect.out#L1243-L1266) |
| Unique inner side recorded on the join | [join.out:5728-5740](../../../../raw/postgres-12/src/test/regress/expected/join.out#L5728-L5740) |
| Lossy bitmaps, with `work_mem` lowered to 64 kB | [bitmapops.sql:1-12](../../../../raw/postgres-12/src/test/regress/sql/bitmapops.sql#L1-L12), [bitmapops.sql:29-30](../../../../raw/postgres-12/src/test/regress/sql/bitmapops.sql#L29-L30) |
| `Rows Removed by Filter` output | [select_parallel.out:556-562](../../../../raw/postgres-12/src/test/regress/expected/select_parallel.out#L556-L562) |

**Non-default setting in the lossy-bitmap test.** Default: `work_mem` is 4096 kB ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2241)). Cause: the test runs `set work_mem = 64`, the minimum, in its session ([bitmapops.sql:29-30](../../../../raw/postgres-12/src/test/regress/sql/bitmapops.sql#L29-L30)). Mechanism: the bitmap index scan sizes its TID bitmap from `work_mem` ([nodeBitmapIndexscan.c:93-95](../../../../raw/postgres-12/src/backend/executor/nodeBitmapIndexscan.c#L93-L95)), so at 64 kB the bitmap runs out of room and stores whole pages, which forces the recheck described in [F4](#f4-lossy-index-results-and-lossy-bitmaps).

### Final causal summary

Stored statistics and fixed defaults, with only the histogram ends re-read from a B-tree index for inequality estimates → row estimates (F1, F10) → the planner prices each path, including partial fetches under LIMIT, EXISTS and cursors (F9), lossy bitmaps (F4), visibility-map heap visits (F5), join rescans and hash builds (F6), and cost parameters (F12) → the planner places each condition as an index condition, a bound, a filter, or above a fence (F2, F3, F7, F8, F11) → it chooses the cheapest plan → the executor reads live data, and every row the estimates or the condition placement did not exclude is fetched and then discarded → `ANALYZE` and `VACUUM` refresh the stored inputs, and only plans made after them see the change.

## Context Reviewed

- Planner entry and tuple fraction: `standard_planner()`, `grouping_planner()`, `preprocess_limit()`, `get_cheapest_fractional_path()` in `planner.c`; `make_subplan()` and `SS_process_ctes()` in `subselect.c`; `planmain.h`; `gram.y` cursor rule.
- Costing: `costsize.c` header, `clamp_row_est()`, `cost_index()`, `initial_cost_nestloop()`, `set_baserel_size_estimates()`, `compute_bitmap_pages()`; `pathnode.c` fractional comparison, `create_limit_path()` and `adjust_limit_rows_costs()`; `consider_startup` in `relnode.c`; the `Limit` path loop in `grouping_planner()`; `cost.h`.
- Selectivity: `clausesel.c`, `selfuncs.c` (`eqsel_internal`, `var_eq_const`, `var_eq_non_const`, `get_restriction_variable`, `get_variable_numdistinct`, `scalarineqsel_wrapper`, `ineq_histogram_selectivity`, `get_actual_variable_range`, `btcostestimate`), `selfuncs.h`, the `RestrictInfo` pseudoconstant comment in `pathnodes.h`, `plancat.c` (`estimate_rel_size`, `restriction_selectivity`, function rows), `heapam_estimate_rel_size()`.
- Index matching and plan construction: `indxpath.c`, `restrictinfo.c`, `createplan.c`, `nbtutils.c`.
- Pushdown: `allpaths.c` subquery pushdown rules and security barrier handling; `rowsecurity.c` and `check_enable_rls()` in `rls.c`.
- Sub-select and function pull-up: `pull_up_sublinks()` and `inline_set_returning_functions()` in `prepjointree.c`; `convert_EXISTS_sublink_to_join()` in `subselect.c`; `inline_set_returning_function()` in `clauses.c`; the `JoinType` comments in `nodes.h`.
- Executor: `execScan.c`, `indexam.c`, `nodeIndexscan.c`, `nodeIndexonlyscan.c`, `nodeBitmapHeapscan.c`, `nodeNestloop.c`, `nodeHashjoin.c`, `nodeHash.c`, `nodeLimit.c`, `nodeCtescan.c`, `nodeFunctionscan.c`, `execSRF.c`; the `tidbitmap.c` recheck flag; the EXPLAIN join-node counters in `explain.c`.
- Plan cache: `plancache.c` `choose_custom_plan()`; the `CURSOR_OPT_GENERIC_PLAN` and `CURSOR_OPT_CUSTOM_PLAN` flags in `parsenodes.h`, with a search of `src/`, `contrib/` and `src/pl/` for code that sets them.
- Partitioning: `partprune.c`, `inherit.c`.
- Statistics maintenance: `analyze.c` (including where `std_typanalyze()` reads `default_statistics_target`), `autovacuum.c` (the worker's analyze call and its configuration reload).
- Set operations: `recurse_set_operations()` and the `UNION`/`UNION ALL` fraction handling in `prepunion.c`; the set-operation fraction reset in `grouping_planner()`; every caller of `get_cheapest_fractional_path()` in `src/backend/optimizer/`.
- GUC definitions in `guc.c`; docs `perform.sgml`, `config.sgml`, `ref/explain.sgml`.
- Catalog and extension boundaries: `pg_operator.dat`, `catalog/Makefile`, `parser/Makefile`, `Makefile.global.in`, `functioncmds.c`, hook headers.
- Tests: `subselect.out`, `join.out`, `bitmapops.sql`, `select_parallel.out`.
- Wiki pages linked for depth: [plan_cache_mode](plan-cache-mode.md), [partitioning](partitioning-planning-execution-optimizations.md), [statistics sources](query-planner-statistics-sources.md), [EXPLAIN BUFFERS](../observability/explain-analyze-buffers-output.md). No v12 common concept pages exist.

## Evidence Map

| Claim | Evidence |
|---|---|
| The planner picks the cheapest path; `get_cheapest_fractional_path()` applies a fraction for cursors, SubPlans and the branches of a top-level `UNION ALL` under a `LIMIT`, and receives 0 at the top otherwise | [planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5886-L5910), [planner.c:377-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L411), [subselect.c:233](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L233), [prepunion.c:274-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L274-L280), [prepunion.c:553-565](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L553-L565) |
| A `LIMIT` acts through `consider_startup` and the cost of a `Limit` path over each surviving path | [planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c:3598](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3598) |
| Row estimate = tuples × selectivity, clamped to ≥ 1 | [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [costsize.c#clamp_row_est](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L187-L200) |
| Table size scales stored density by live pages; 10-page minimum when `relpages = 0` and no inheritance children, including a table vacuumed while empty | [heapam_handler.c:2082-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2157) |
| Default selectivities | [selfuncs.h:32-49](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L32-L49) |
| Equality without statistics divides by an inferred or default distinct count; against an unknown value it averages over distinct values; the fixed 0.005 applies when no variable side is found | [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425), [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5019-L5116), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505), [selfuncs.c:258-262](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L258-L262) |
| Inequality estimates re-read the histogram ends from a non-partial B-tree index | [selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5263-L5310) |
| Multiplication assumes independence; extended statistics first | [clausesel.c:55-122](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L55-L122) |
| Index condition requirements | [indxpath.c:2262-2300](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2262-L2300), [indxpath.c:2593-2645](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2593-L2645) |
| Filter = scan conditions minus index conditions | [createplan.c:2798-2848](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L2798-L2848) |
| B-tree required keys and boundary-qual costing | [nbtutils.c:685-699](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L685-L699), [selfuncs.c:5940-6075](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5940-L6075) |
| Recheck of lossy pages and of exact pages the index marks, and the lossy-page cost estimate | [nodeIndexscan.c:135-150](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L135-L150), [nodeBitmapHeapscan.c:315-327](../../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L315-L327), [tidbitmap.c:95-104](../../../../raw/postgres-12/src/backend/nodes/tidbitmap.c#L95-L104), [costsize.c:5554-5604](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5554-L5604) |
| Dead tuples fetched and skipped; index-only heap fetches read the live visibility map, costed from stored `relallvisible` | [indexam.c:566-636](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L636), [nodeIndexonlyscan.c:121-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170), [costsize.c:652-653](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L652-L653), [costsize.c:612-613](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L612-L613) |
| Nested loop rescans, single-match rule (also for hash joins), cost | [nodeNestloop.c:100-160](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L100-L160), [nodeNestloop.c:318-322](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L318-L322), [nodeHashjoin.c:682-683](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L682-L683), [costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471) |
| Hash join reads the whole inner input; empty-outer shortcut | [nodeHash.c:139-175](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L139-L175), [nodeHashjoin.c:225-273](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L225-L273) |
| CTE inlining rule and test | [subselect.c:870-899](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L870-L899), [subselect.out:1243-1266](../../../../raw/postgres-12/src/test/regress/expected/subselect.out#L1243-L1266) |
| Subquery pushdown rules | [allpaths.c:3004-3288](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3004-L3288) |
| Inlinable SQL set-returning functions become subqueries | [prepjointree.c:620-640](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L620-L640), [clauses.c:4911-4982](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4911-L4982), [clauses.c:5065-5074](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L5065-L5074), [clauses.c:5076-5117](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L5076-L5117) |
| Function scan materializes; 1000-row default | [nodeFunctionscan.c:85-100](../../../../raw/postgres-12/src/backend/executor/nodeFunctionscan.c#L85-L100), [functioncmds.c:1134-1140](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1134-L1140) |
| Security promotability | [restrictinfo.c:376-388](../../../../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L376-L388), [allpaths.c:2221-2227](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2221-L2227) |
| Tuple fractions for LIMIT, EXISTS and ANY SubPlans, cursors; pulled-up sub-selects become semi joins | [planner.c:2876-2932](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2876-L2932), [subselect.c:208-214](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L214), [prepjointree.c:168-186](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L168-L186), [planner.c:377-403](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L403), [gram.y:11137](../../../../raw/postgres-12/src/backend/parser/gram.y#L11137) |
| Fractional cost interpolation | [costsize.c:37-51](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L37-L51), [pathnode.c:118-134](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L118-L134), [pathnode.c:3623-3670](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670) |
| OFFSET rows fetched and discarded | [nodeLimit.c:90-108](../../../../raw/postgres-12/src/backend/executor/nodeLimit.c#L90-L108) |
| Generic plan selectivity and choice | [selfuncs.c:269-276](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L269-L276), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505), [plancache.c:1016-1063](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1063), [parsenodes.h:2690-2691](../../../../raw/postgres-12/src/include/nodes/parsenodes.h#L2690-L2691) |
| Join nodes count join-condition failures and other-condition failures separately; EXPLAIN prints both on nested loop, merge and hash joins | [nodeNestloop.c:214-247](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L214-L247), [nodeHashjoin.c:454-499](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L454-L499), [explain.c:1786-1817](../../../../raw/postgres-12/src/backend/commands/explain.c#L1786-L1817) |
| Subquery pushdown also stops at `EXCEPT` and at conditions with sub-selects; set-returning-function inlining refuses sub-selects in arguments and hooked functions; a multiply referenced CTE with an outer recursive self-reference is not inlined | [allpaths.c:3073](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3073), [allpaths.c:3274](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3274), [clauses.c:4942-4952](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4942-L4952), [subselect.c:895-896](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L895-L896) |
| Partition pruning off or no conditions keeps all partitions | [partprune.c:676-681](../../../../raw/postgres-12/src/backend/partitioning/partprune.c#L676-L681) |
| Cost defaults and correlation | [cost.h:24-32](../../../../raw/postgres-12/src/include/optimizer/cost.h#L24-L32), [costsize.c:708-714](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L708-L714), [selfuncs.c:6190-6210](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6190-L6210) |
| GUC defaults and contexts | [guc.c:1044-1054](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1044-L1054), [guc.c:1986-1994](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994), [guc.c:2231-2241](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2241), [guc.c:2957-2963](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2957-L2963), [guc.c:3206-3260](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3206-L3260), [guc.c:3318-3326](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3318-L3326), [guc.c:3404-3410](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3404-L3410), [guc.c:4243-4253](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4243-L4253), [guc.c:4504-4515](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4504-L4515) |

## Open Questions

- The `lossy_pages = heap_pages - maxentries / 2` estimate is called a "crude approximation" in the source ([costsize.c:5582-5589](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5582-L5589)). This page does not quantify how far it can be from the real number of lossy pages; that would need a measurement.
- The worked calculation in F9 uses illustrative costs, not output from a 12.2 server. A measured example would need a `## Measurement Script`.
- The interpretation of "over-fetch" is this page's: rows read by a node and not passed up. If the question meant another quantity, such as heap pages read, F3 and F12 matter more and the counters differ.
- No v12 common concept page exists for row estimation or join methods; the page explains what it needs inline.

## Source References

- [planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5886-L5910)
- [planner.c#preprocess_limit](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2816-L2983)
- [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418)
- [costsize.c#initial_cost_nestloop](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2416-L2471)
- [costsize.c#compute_bitmap_pages](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5515-L5610)
- [pathnode.c#compare_fractional_path_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L118-L134)
- [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L55-L105)
- [selfuncs.c#btcostestimate](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5940-L6075)
- [selfuncs.c#ineq_histogram_selectivity](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935)
- [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5263-L5310)
- [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5019-L5116)
- [prepjointree.c#pull_up_sublinks](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L168-L186)
- [clauses.c#inline_set_returning_function](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4911-L4982)
- [indxpath.c#match_clause_to_index](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2262-L2300)
- [createplan.c#create_indexscan_plan](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L2798-L2848)
- [allpaths.c#subquery_is_pushdown_safe](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L3004-L3025)
- [subselect.c#SS_process_ctes](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L870-L899)
- [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2072-L2171)
- [nodeNestloop.c#ExecNestLoop](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L61-L250)
- [nodeHash.c#MultiExecPrivateHash](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L139-L175)
- [plancache.c#choose_custom_plan](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1063)
- [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670)
- [relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201)
- [tidbitmap.c#PagetableEntry](../../../../raw/postgres-12/src/backend/nodes/tidbitmap.c#L95-L104)
- [prepunion.c#recurse_set_operations](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L274-L280)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
