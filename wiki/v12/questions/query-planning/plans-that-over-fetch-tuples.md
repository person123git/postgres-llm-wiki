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
  - [Measured example: row-level security and the estimates](#measured-example-row-level-security-and-the-estimates)
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

The exception is a live correction to the histogram. When an inequality estimate reaches the first or last histogram entry, `ineq_histogram_selectivity()` replaces that entry with the column's current minimum or maximum. `get_actual_variable_range()` reads that value from a non-partial, non-hypothetical B-tree index whose first column is the column being estimated, and whose operator family has the histogram's sort operator as its `<` or `>` operator; any other index is skipped ([selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5263-L5325)). So when such an index exists, a range condition on values beyond the stored upper bound is estimated against the live maximum, while the inner histogram entries stay as `ANALYZE` stored them. An equality condition, or a column with no usable B-tree index, gets no such correction.

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
  |-- no statistics --> equality 1/rows if unique, else 1/n_distinct; inequality 1/3 .. F1
  |-- value unknown at plan time --> equality averaged over n_distinct; inequality 1/3 .. F1, F10
  |-- several conditions --> product (independence) unless extended statistics .......... F1
  |-- RLS on the table + non-leakproof operator --> common values and histogram ignored . F1, F8
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
- [3] Row estimate: [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L69-L105), [clausesel.c:119-122](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L119-L122). Equality without statistics: [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425). Inequality without statistics: [selfuncs.c:567-645](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L567-L645). Equality against an unknown value: [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505). Inequality: [selfuncs.c:1174-1185](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1174-L1185). Histogram end re-read: [selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935). Statistics under row-level security: [selfuncs.c:4743-4756](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4743-L4756), [selfuncs.c#statistic_proc_security_check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4968-L4983).
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
| F8 | Security barriers with non-leakproof conditions | Both | `Filter` instead of `Index Cond` on a protected table; `rows=` estimate far from actual on it |
| F9 | LIMIT, OFFSET, EXISTS and cursors | Estimation | A fast-start index scan under `Limit` that reads far more rows than the limit |
| F10 | Generic cached plans | Estimation | A prepared statement whose plan ignores the parameter's actual frequency |
| F11 | Partitions that are not pruned | Structural | An `Append` over partitions that return 0 rows |
| F12 | Cost parameters | Estimation | A sequential scan with a large `Rows Removed by Filter` where an index existed |

### F1. Row-count estimates that are too high or too low

Every later decision uses `rel->rows`. The planner computes it as the table's estimated tuple count times the selectivity of its conditions, then rounds it and forces it to be at least 1 ([costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [costsize.c#clamp_row_est](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L187-L200)). An error in either input becomes an error in the plan choice. Seven inputs go wrong in practice:

| Input | What happens | Source |
|---|---|---|
| Missing statistics | An equality condition on a column with a single-column unique index, partial only if its predicate is proven, gets `1 / rows`, one expected match, before any distinct-count guess ([plancat.c#has_unique_index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L2031-L2054), [selfuncs.c:4468](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4468)). Any other equality condition gets `1 / n_distinct`, where `n_distinct` is 2 for a boolean, the row count for `ctid` and for a `VALUES` column, 1 for `tableoid`, the row count for a table under 200 rows, and otherwise the default 200, so 0.005. An inequality against a constant gets the default 1/3, unless the column is `ctid`, which is estimated from the table size instead. The header also defines 0.005 for pattern matches and for range pairs | [selfuncs.c:324-327](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L324-L327), [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425), [selfuncs.c:5030-5057](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5030-L5057), [selfuncs.c:567-645](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L567-L645), [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5019-L5116), [selfuncs.h:32-49](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L32-L49) |
| A comparison value unknown at plan time, such as a parameter in a generic plan | An equality condition is averaged over all distinct values: `(1 − null fraction) / n_distinct`, capped by the most common value's frequency; a column with a single-column unique index gets `1 / rows` instead; see [F10](#f10-generic-cached-plans). An inequality gets the default 1/3. Equality falls back to the fixed 0.005 only when the condition does not compare an expression in one table's columns with something that contains none of them, for example `a = b + 1` with columns on both sides | [selfuncs.c:269-276](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L269-L276), [selfuncs.c:469-472](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L469-L472), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505), [selfuncs.c:1174-1185](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1174-L1185), [selfuncs.c:258-262](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L258-L262), [selfuncs.c#get_restriction_variable](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4292-L4313) |
| Several conditions on one table | Their selectivities are multiplied, which assumes they are independent; only [extended statistics](../../../glossary.md#extended-statistics) created with `CREATE STATISTICS` correct for correlated columns. `ANALYZE` builds those only for a table's own rows: 12 accepts a statistics object on a partitioned table, but `ANALYZE` runs only its inheritance-tree pass, which never builds extended statistics, so such an object stays empty and the object must be created on each partition instead | [clausesel.c#clauselist_selectivity](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L55-L105), [clausesel.c:119-122](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L119-L122), [statscmds.c:122-125](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L122-L125), [analyze.c:255-268](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268), [analyze.c:578-584](../../../../raw/postgres-12/src/backend/commands/analyze.c#L578-L584) |
| Statistics hidden by row-level security | When the table carries row-level security conditions, the planner may read a column's common values and histogram only through a `LEAKPROOF` operator. For any other operator, such as `textlike` behind `LIKE` or the `numeric` comparisons, the equality estimate skips the common values and divides by the distinct count, a one-sided inequality gets half of the non-null rows, and a range with both bounds collapses to 1 row; see [F8](#f8-security-barriers-and-non-leakproof-conditions) and the [measured example](#measured-example-row-level-security-and-the-estimates) | [selfuncs.c:4743-4756](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4743-L4756), [selfuncs.c#statistic_proc_security_check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4968-L4983), [selfuncs.c:328-330](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L328-L330), [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425), [selfuncs.c:673-686](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L673-L686) |
| Statistics sample size | `ANALYZE` samples 300 rows per unit of the statistics target, whose default is 100; values outside the sample are estimated, not counted | [analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700), [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994) |
| Statistics older than the data | Column statistics change only when `ANALYZE` runs; the table size is scaled from live pages, but the per-column fractions are not. The one live correction is an inequality reaching a histogram end, which is re-read from a B-tree index when one qualifies: non-partial, on the column as its first column, with a matching sort operator; see the [mental model](#mental-model-the-values-the-planner-relies-on) | [heapam_handler.c:2131-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2131-L2157), [analyze.c#update_attstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1417-L1430), [selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935) |
| `relpages = 0` | With `relpages = 0`, fewer than 10 current pages and no inheritance children, the planner assumes 10 pages, so that a newly created table is not assumed to stay small. `relpages = 0` stands in for "never vacuumed", so the rule also fires on a table that was vacuumed or analyzed while empty. With `relpages = 0`, the density comes from the column widths, not from stored counts | [heapam_handler.c:2090-2118](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2090-L2118), [heapam_handler.c:2132-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2132-L2157) |

An estimate that is too **low** causes over-fetching because it makes repeated work look cheap. For example, a nested loop rescans its inner side once per outer row, so its cost grows with the outer estimate ([costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471)). If the outer side was estimated at 1 row and returns 50,000, the inner side is read 50,000 times. Independence multiplication tends to push estimates low, because a product of fractions shrinks with every added condition.

An estimate that is too **high** causes over-fetching under a LIMIT. The planner treats `LIMIT n` as "read n of the estimated rows", so a high estimate makes `n` look like a small fraction and favors a fast-start plan; see [F9](#f9-limit-offset-exists-and-cursors).

### F2. Conditions that cannot become index conditions

A condition helps an index only if the planner can turn it into an index condition. Otherwise the scan fetches every row the index condition admits and tests the condition afterward as a `Filter`. `create_indexscan_plan()` builds that filter list as "the scan's conditions minus those the index handles" ([createplan.c:2798-2848](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L2798-L2848)).

`match_clause_to_index()` and `match_opclause_to_indexcol()` decide whether a condition of the form `column operator value` can be used. Such a condition qualifies only if the first four rows below hold and either the fifth or the sixth does: the support function is asked only when the operator-family and collation test fails. Other condition shapes, such as a boolean column, `= ANY (array)`, a row comparison, or `IS NULL` on an index that can search nulls, go to sibling matchers with their own tests ([indxpath.c:2399-2440](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2399-L2440)).

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

Row-level security also degrades the row estimate. It does so in four steps:

1. The rewriter attaches the policy conditions to the table's range-table entry as its `securityQuals` list ([rewriteHandler.c:2043-2045](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L2043-L2045), [rewriteHandler.c:2097-2098](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L2097-L2098)).
2. `examine_variable()` sets `acl_ok` only when that list is empty and the user can read the column ([selfuncs.c:4743-4756](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4743-L4756)).
3. With `acl_ok` false, `statistic_proc_security_check()` lets only a `LEAKPROOF` operator function read the most common values and the histogram ([selfuncs.c#statistic_proc_security_check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4968-L4983)). `int4lt`, `texteq` and `text_lt` are leakproof; `numeric_eq`, `numeric_lt` and `textlike` are not ([pg_proc.dat:152](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L152), [pg_proc.dat:155](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L155), [pg_proc.dat:1429](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L1429), [pg_proc.dat:4152](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L4152), [pg_proc.dat:4164](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L4164), [pg_proc.dat:1641](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L1641)).
4. For a non-leakproof operator, the estimator skips those lists ([selfuncs.c:328-330](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L328-L330), [selfuncs.c:719-720](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L719-L720), [selfuncs.c:867-868](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L867-L868)) and falls back as the table shows.

| Condition with a non-leakproof operator | Estimate | Source |
|---|---|---|
| `col = value` | `1 / n_distinct`; the stored frequencies of the most common values are ignored | [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425) |
| `col < value` or `col > value` | Half of the non-null rows: `0.5 × (1 − null fraction)` | [selfuncs.c:673-686](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L673-L686) |
| `col BETWEEN a AND b`, or a lower and an upper bound on the same column | 1 row. Each bound is `0.5 × (1 − null fraction)`, which is not the default 1/3 that `clauselist_selectivity()` treats as a guess. So it computes `hi + lo − 1 + null fraction`, which is 0 up to floating-point rounding. A result between −0.01 and 0 is replaced with `1.0e-10` as rounding error; a tiny positive residue is used as it is. Either way `clamp_row_est()` raises the estimate to 1 row | [clausesel.c:294-305](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L294-L305), [clausesel.c:315-332](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L315-L332), [costsize.c#clamp_row_est](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L187-L200) |

So a protected table can get a row estimate far from the real count, and F1's consequences follow: a range that the planner expects to return 1 row makes a nested loop over it look cheap. The [measured example](#measured-example-row-level-security-and-the-estimates) shows all three rows on a `numeric` column.

A partitioned table behaves the same way, with one condition. Each partition's range-table entry gets an empty `securityQuals` list, because only the parent's policies apply ([inherit.c:425-450](../../../../raw/postgres-12/src/backend/optimizer/util/inherit.c#L425-L450)). When the user can read the partition itself, its statistics are therefore used. When the user can read only the parent, `examine_variable()` repeats the check on the parent, whose list is not empty, so the statistics stay hidden ([selfuncs.c:4768-4838](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4768-L4838)).

**Non-default cause.** Default: a table has no row-level security and a view has no `security_barrier` flag, so every user condition is at the lowest security level. Cause: row-level security enabled on the table for the querying role, which `check_enable_rls()` decides ([rls.c#check_enable_rls](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L52-L133), called at [rowsecurity.c:134](../../../../raw/postgres-12/src/backend/rewrite/rowsecurity.c#L134)), or a view created with `security_barrier`. Mechanism: the security conditions take lower `security_level` values, so the promotability test above fails for a non-leakproof user condition. For the estimate, the cause is the non-empty `securityQuals` list that row-level security puts on the table's range-table entry; `acl_ok` is then false, so only leakproof operators see the statistics.

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

A [prepared statement](../../../glossary.md#prepared-statement) can run a [generic plan](../../../glossary.md#custom-and-generic-plan), built without the parameter values. For an equality against an unknown value, `eqsel` uses `var_eq_non_const()`, which assumes the value is equally likely to be any distinct value: `(1 − null fraction) / n_distinct`, capped by the frequency of the most common value ([selfuncs.c:269-276](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L269-L276), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505)). A column with a single-column unique index gets `1 / rows` instead, so the generic plan expects one match ([selfuncs.c:469-472](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L469-L472)). An inequality against an unknown value gets the default 1/3 ([selfuncs.c:1174-1185](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L1174-L1185)). So a generic plan tuned for an average value over-fetches when it runs with a very common value, because it was built for a small match count. For example, it may pick a nested loop whose inner side is then rescanned for each of the many matching outer rows (F6).

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
| `CREATE STATISTICS` object on the table | None | Corrects independence for the covered columns (F1) | Takes effect after `ANALYZE` builds it; never built for an object on a partitioned table, so create it on each partition ([analyze.c:255-268](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268), [analyze.c:578-584](../../../../raw/postgres-12/src/backend/commands/analyze.c#L578-L584)) |
| `work_mem` | 4096 kB ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2241)) | Larger values reduce lossy bitmap pages (F4) | `user`: `SET` |
| `cursor_tuple_fraction` | 0.1 ([guc.c#cursor_tuple_fraction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3318-L3326)) | Lower values favor fast-start plans for cursors (F9) | `user`: `SET` |
| `plan_cache_mode` | `auto` ([guc.c#plan_cache_mode](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4504-L4515)) | `force_generic_plan` removes the custom-plan check (F10) | `user`: `SET` |
| `enable_partition_pruning` | on ([guc.c#enable_partition_pruning](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1044-L1054)) | off keeps all partitions (F11) | `user`: `SET` |
| `constraint_exclusion` | `partition` ([guc.c#constraint_exclusion](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4243-L4253)) | At `partition`, [constraint exclusion](../../../glossary.md#constraint-exclusion) runs only for appendrel members such as inheritance children; `off` skips it, so children whose constraints rule them out are scanned ([plancat.c:1433-1458](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1433-L1458)) | `user`: `SET` |
| `seq_page_cost` and the other cost settings | See F12 | Shift choice between seq and index plans (F12) | `user`: `SET` |
| `autovacuum_analyze_threshold`, `autovacuum_analyze_scale_factor` | 50 and 0.1 ([guc.c#autovacuum_analyze_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2957-L2963), [guc.c#autovacuum_analyze_scale_factor](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3404-L3410)) | Lower values make [autovacuum](../../../glossary.md#autovacuum) refresh statistics sooner (F1) | `sighup`: reload |
| Row-level security or `security_barrier` | Off | Holds back non-leakproof conditions; under row-level security, also hides the common values and histogram from non-leakproof operators (F8) | DDL |
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

1. Compare each node's estimated `rows` with the actual `rows`. A large gap points to F1, F8, F9 or F10.
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

### Measured example: row-level security and the estimates

This example backs the F8 table with numbers from a 12.2 server built from the pin; the [Measurement Script](#measurement-script) produced them. Three tables hold the same 30,000 rows: `id int` and `n numeric` both run from 1 to 30,000, and `k numeric` is 0 on the first 15,000 rows and `id % 1000 + 1` on the rest. `mp` has no row-level security. `mr` has row-level security with one policy, `USING (true)`, that admits every row. `pm` is `mr` split into two range partitions, with the policy on the parent. One non-superuser role, `alice`, runs every query, so the only difference between `mp` and `mr` is the policy.

| Case | Table | Condition | Estimated rows | Actual rows |
|---|---|---|---|---|
| `plain_n_range` | `mp` | `n BETWEEN 1000 AND 1999` | 1000 | 1000 |
| `rls_n_range` | `mr` | `n BETWEEN 1000 AND 1999` | 1 | 1000 |
| `plain_n_lt` | `mp` | `n < 1000` | 999 | 999 |
| `rls_n_lt` | `mr` | `n < 1000` | 15000 | 999 |
| `plain_k_eq` | `mp` | `k = 0` | 15000 | 15000 |
| `rls_k_eq` | `mr` | `k = 0` | 30 | 15000 |
| `plain_id_range` | `mp` | `id BETWEEN 1000 AND 1999` | 1000 | 1000 |
| `rls_id_range` | `mr` | `id BETWEEN 1000 AND 1999` | 1000 | 1000 |
| `part_parent_grant` | `pm` | `n BETWEEN 1000 AND 1999`, `SELECT` granted on the parent only | 2 | 1000 |
| `part_child_grant` | `pm` | the same, after `SELECT` is also granted on both partitions | 1001 | 1000 |

The server reported `proleakproof` false for `numeric_eq`, `numeric_lt`, `numeric_ge` and `numeric_le`, and true for `int4lt`, `int4ge` and `int4le`, as the catalog data defines them ([pg_proc.dat:152](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L152), [pg_proc.dat:398](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L398), [pg_proc.dat:401](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L401), [pg_proc.dat:4152](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L4152), [pg_proc.dat:4161](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L4161), [pg_proc.dat:4164](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L4164), [pg_proc.dat:4167](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L4167)). `BETWEEN` becomes a `>=` and a `<=` condition, as the plan's `Filter: ((n >= '1000'::numeric) AND (n <= '1999'::numeric))` shows. Each `mr` estimate follows from the F8 table. These are derived values; only the estimates in the table above were measured:

| Case | Inputs | Calculation | Result |
|---|---|---|---|
| `rls_k_eq` | 30,000 rows; `k` has 1,001 distinct values by construction (0 and 1 to 1000); no nulls | 30,000 / 1,001 = 29.97, rounded by `clamp_row_est()` | 30, against 15,000 actual, because the stored frequency of 0 (one half) is ignored |
| `rls_n_lt` | 30,000 rows; no nulls | 0.5 × 30,000 | 15,000, against 999 actual |
| `rls_n_range` | Each bound 0.5 | 0.5 + 0.5 − 1 + 0 = 0, replaced by `1.0e-10`; 30,000 × `1.0e-10` rounded up | 1, against 1,000 actual |

The `int4` range on `mr` gets the same estimate as on `mp`, because `int4ge` and `int4le` are leakproof, so the statistics stay visible. The scan itself is the same in every case: a sequential scan with `Rows Removed by Filter`, which shows that the policy did not change what was read here, only what the planner expected. The partitioned case shows the condition in F8: each partition is estimated at 1 row (`rows=2` on the `Append`) until `alice` can read the partitions directly.

**Non-default state in the example.** Default: a table has no row-level security, so `examine_variable()` sets `acl_ok` for a user who can read the column, and every estimator sees the statistics ([selfuncs.c:4743-4756](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4743-L4756)). Cause: `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and a policy on `mr` and `pm`, queried by a role that is neither superuser, nor `BYPASSRLS`, nor the owner. `check_enable_rls()` exempts superusers and `BYPASSRLS` roles always, and the owner unless the table is set to `FORCE ROW LEVEL SECURITY` and the check is not a referential-integrity check, so the script cannot measure as the superuser that owns the tables ([rls.c:87-117](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L87-L117), [aclchk.c#has_bypassrls_privilege](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L5399-L5415)). Mechanism: the policy puts a non-empty `securityQuals` list on the range-table entry, so `acl_ok` is false and only leakproof operators read the statistics, as F8 describes. The `mp` rows are the default leg.

### Final causal summary

Stored statistics and fixed defaults, with only the histogram ends re-read from a B-tree index for inequality estimates → row estimates (F1, F10), with the common values and histogram hidden from non-leakproof operators under row-level security (F8) → the planner prices each path, including partial fetches under LIMIT, EXISTS and cursors (F9), lossy bitmaps (F4), visibility-map heap visits (F5), join rescans and hash builds (F6), and cost parameters (F12) → the planner places each condition as an index condition, a bound, a filter, or above a fence (F2, F3, F7, F8, F11) → it chooses the cheapest plan → the executor reads live data, and every row the estimates or the condition placement did not exclude is fetched and then discarded → `ANALYZE` and `VACUUM` refresh the stored inputs, and only plans made after them see the change.

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Measures the planner's row estimates for the same conditions on the same data with and without row-level security. It backs every number in [Measured example: row-level security and the estimates](#measured-example-row-level-security-and-the-estimates) and the 1-row range claim in F8. It measures estimates and actual row counts only, not timings. |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/of12-run.sh` with the command after this table, then run `bash .wiki-runtime/tmp/of12-run.sh [stage ...]`. Every build, data, socket and output file stays under `.wiki-runtime/tmp/of12/`. |
| Stages | Default order: `build rls report`. `stop` and `clean` tear down and are not in the default order. See [Stages](#stages). |
| Environment | `PORT12` (default `55447`) is the isolated port and `JOBS` (default `8`) the parallel build jobs. The checkout, pin, sandbox path and the 30,000-row fixture size are fixed in the script. |
| Prerequisites | A C toolchain, GNU make, bison and flex; readline and zlib development headers, because 12's `configure` enables both by default ([configure.in:867-868](../../../../raw/postgres-12/configure.in#L867-L868), [configure.in:966-967](../../../../raw/postgres-12/configure.in#L966-L967)); the pinned checkout at `raw/postgres-12/`; about 500 MB of free disk. Locale `C`, encoding `UTF8`, set by the script. No extension is needed. |
| Output | Files under `.wiki-runtime/tmp/of12/out/`. Read `results.txt` first: one line per case with `case|table|condition|estimated_rows|actual_rows`. Then `platform.txt`, `<case>.explain`, `initdb.log` and `server.log`. When the build fails, `configure.log` or `make.log` is copied there too. |
| Runtime | About 50 seconds from an empty sandbox, nearly all of it the build (47 to 59 s on the 2026-10-30 host, Linux x86_64, 22 threads, `JOBS=22`). Re-running `rls report` while the build is present takes 3 to 4 seconds. |
| Cleanup | Run `bash .wiki-runtime/tmp/of12-run.sh stop clean` from the repository root. It stops the cluster if it is still running, checks that no `postmaster.pid`, process naming the data directory, socket or listener on `PORT12` remains, and deletes `.wiki-runtime/tmp/of12/` with its build. Remove the extracted `.wiki-runtime/tmp/of12-run.sh` afterwards. The `rls` stage already stops its cluster at the end. |

To extract the published script without editing it, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/query-planning/plans-that-over-fetch-tuples.md | sed '1d;$d' > .wiki-runtime/tmp/of12-run.sh
bash -n .wiki-runtime/tmp/of12-run.sh
bash .wiki-runtime/tmp/of12-run.sh
```

### Last run

**Latest run (2026-10-30, thirteenth review).** The script text was unchanged since the eleventh-review run. It was extracted from this page with the command above and run from an empty sandbox in 59 s with `JOBS=22`, then `rls report` was re-run with the build present in 3 s. Both runs passed all 13 assertions, and their results were identical to each other and to every earlier run. `platform.txt` recorded the same values as below. `stop clean` again left no PID file, process, socket or listener, and deleted the sandbox with its build.

**2026-10-30, twelfth review.** The script text was unchanged since the eleventh-review run. It was extracted from this page with the command above and run from an empty sandbox in 58 s with `JOBS=22`, then `rls report` was re-run with the build present in 4 s. Both runs passed all 13 assertions, and their results were identical to each other and to the eleventh-review runs. `platform.txt` recorded the same values as below. `stop clean` again left no PID file, process, socket or listener, and deleted the sandbox with its build.

**2026-10-30, eleventh review.** The script ran from an empty sandbox in 47 s with `JOBS=22`, then `rls report` was re-run with the build present in 3 s. Both runs passed all 13 assertions and printed identical results. An earlier run the same day, before the script also recorded the `int4ge`, `int4le`, `numeric_ge` and `numeric_le` flags, gave the same ten estimates and actual counts. The host was Linux 6.18.54.1-microsoft-standard-WSL2 x86_64, with PostgreSQL 12.2 at `45b88269a353ad93744772791feb6d01bc7e1e42`, built with default configure options. `platform.txt` recorded `block_size=8192`, `default_statistics_target=100`, `autovacuum=off`, `shared_buffers=128MB` (chosen by `initdb`) and `max_data_alignment=8`. The estimates do not depend on the block size or the alignment, because each estimate is a fraction of a row count that `ANALYZE` stored from a full read of the table. `stop clean` then confirmed that no PID file, process, socket or listener remained, and deleted the sandbox with its build.

### Stages

| Stage | What it does | Needs |
|---|---|---|
| `build` | Checks the pin, then configures 12.2 out of tree with default options under `$SANDBOX/build`, builds, and installs it. Skipped when a matching build exists. | nothing |
| `rls` | Runs `initdb`, starts the cluster, creates the role and fixtures, runs `ANALYZE`, runs the ten cases as `alice`, stops the cluster, then checks its assertions | `build` |
| `report` | Prints `results.txt` | `rls` |
| `stop` | Stops the cluster if it is still running and checks that no PID file, process, socket or listener remains | none |
| `clean` | Runs `stop`, then deletes `$SANDBOX` | none |

The `rls` stage is idempotent: it deletes and re-creates its data directory and `results.txt`. Inside it, the cases run in a fixed order, because the last case depends on the `GRANT` made just before it. Every role, table and database is disposable and exists only in the sandbox cluster.

### Non-default settings in the measurement

| Setting or state | Default at the pin | Value in the script, why, and which results depend on it |
|---|---|---|
| Row-level security on `mr` and `pm`, queried by `alice` | No row-level security; the default leg is `mp` | Every `rls_*` and `part_*` estimate depends on it; see the non-default note under the [measured example](#measured-example-row-level-security-and-the-estimates). The `plain_*` cases measure the default on the same data. |
| `SELECT` on `pm1` and `pm2` granted to `alice` before `part_child_grant` | `alice` holds only the grant on the parent | Lets the child-level permission check succeed, so the partition statistics become visible ([selfuncs.c:4743-4756](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4743-L4756), [inherit.c:425-450](../../../../raw/postgres-12/src/backend/optimizer/util/inherit.c#L425-L450)). Only `part_child_grant` depends on it. |
| 30,000 rows per table | — | `ANALYZE` samples 300 × the statistics target rows, 30,000 at the default 100 ([analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700), [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994)). At this size it reads every row: the sample size is the largest per-column minimum ([analyze.c:479-480](../../../../raw/postgres-12/src/backend/commands/analyze.c#L479-L480)), the block sampler selects every block when the table has no more blocks than that ([sampling.c:28-30](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L28-L30)), and every row is kept while the reservoir is below it ([analyze.c:1065-1066](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1065-L1066)). So the statistics and every estimate are the same on each run. A larger table would give estimates that vary slightly from run to run. |
| `autovacuum` | `on`, context `PGC_SIGHUP` ([guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434)) | `off` in `postgresql.conf` before the first start, so no auto-analyze runs between the script's own `ANALYZE` and the cases. With full-table samples, a second analyze would store the same statistics, so no result is expected to depend on it. |
| `initdb -U postgres -A trust --locale=C --encoding=UTF8` | Without these, `initdb` names the superuser after the operating-system user ([initdb.c:3286-3288](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3286-L3288)) and takes locale and encoding from the environment ([initdb.c#setlocales](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2300-L2330)) | Fixed so the run does not depend on the host. No result depends on them: the conditions compare `numeric` and `int4` values, which do not use collation. |
| `listen_addresses = ''`, `unix_socket_directories`, `port = 55447` | `localhost`, `/tmp`, `5432`; all `PGC_POSTMASTER` ([guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3958), [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2184)) | Socket-only access in the sandbox on a non-default port, so no other cluster is touched. No result depends on them. |
| `statement_timeout`, `lock_timeout` | `0`, disabled; `PGC_USERSET` ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)) | 5 minutes and 5 seconds for every session, through `PGOPTIONS`. They only bound a runaway statement. |

### The script

```bash
#!/usr/bin/env bash
#
# of12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/query-planning/plans-that-over-fetch-tuples.md
#
# Builds the pinned PostgreSQL 12 checkout out of tree with default configure
# options, starts one isolated cluster, and compares the planner's row
# estimates for the same data in three tables, queried by the same
# non-superuser role:
#   mp  plain table, no row-level security
#   mr  the same rows, row-level security enabled with one permissive policy
#   pm  the same rows in a range-partitioned table with row-level security on
#       the parent; queried before and after SELECT is granted on the partitions
# Every table holds 30000 rows, which is the ANALYZE sample size at the default
# statistics target (300 x 100), so ANALYZE reads every row and the stored
# statistics, and therefore the estimates, are the same on every run.
#
# Every role, table and database is disposable and lives only in this sandbox.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/of12-run.sh [stage ...]
# Stages: build rls report stop clean
# Default: build rls report

set -uo pipefail
set -e

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/of12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55447}"
JOBS="${JOBS:-8}"
ROWS=30000

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
  [ -f "$SANDBOX/.of12-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.of12-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
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
  printf '%s\n' "$1" | sed -E 's/(^|;[[:space:]]*)(SELECT|INSERT|CREATE|ALTER|GRANT|ANALYZE|EXPLAIN|SHOW)([[:space:]]|;|$)/\1\2 \/\* wiki_of12_fixture \*\/\3/g'
}
q() { # q <user> <sql>: run, discard output
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -U "$1" -d of -c "$(tag_sql "$2")" > /dev/null \
    || die "failed as $1: $2"
}
s() { # s <user> <sql>: one unaligned result
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -U "$1" -d of -c "$(tag_sql "$2")" \
    || die "failed as $1: $2"
}

verify_stopped() {
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid still present in $DATA"
  if pgrep -f -- "$DATA" > /dev/null 2>&1; then die "a process still names $DATA"; fi
}
verify_free() {
  [ -z "$(ls -A "$SOCK" 2>/dev/null)" ] || die "socket directory $SOCK is not empty"
  if "$BIN/pg_isready" -q -h 127.0.0.1 -p "$PORT12"; then die "port $PORT12 is occupied"; fi
}

stage_build() {
  say "build 12.2 out of tree from $SRC12 (default configure options)"
  source_pin
  if [ ! -e "$SANDBOX" ]; then
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.of12-owned"
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

# measure <case> <table> <where>: EXPLAIN ANALYZE as role alice, record the
# top plan node's estimated rows and actual rows.
measure() {
  local case=$1 tbl=$2 cond=$3 f est act
  f="$OUT/$case.explain"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -U alice -d of \
      -c "$(tag_sql "EXPLAIN (ANALYZE, TIMING OFF) SELECT * FROM $tbl WHERE $cond")" \
      > "$f" 2>&1 || { cat "$f" >&2; die "EXPLAIN failed for $case"; }
  est=$(sed -n 's/.*cost=[^ ]* rows=\([0-9]*\) width=.*/\1/p' "$f" | head -n 1)
  act=$(sed -n 's/.*(actual rows=\([0-9]*\) loops=.*/\1/p' "$f" | head -n 1)
  [ -n "$est" ] && [ -n "$act" ] || die "could not parse $f"
  printf '%s|%s|%s|%s|%s\n' "$case" "$tbl" "$cond" "$est" "$act" >> "$OUT/results.txt"
  note "$case: $tbl WHERE $cond: estimated rows=$est actual rows=$act"
}
field() { grep -E "^$1\|" "$OUT/results.txt" | tail -n 1 | cut -d'|' -f"$2"; }
expect_range() { # expect_range <case> <field-name> <field-number> <min> <max>
  local got
  got=$(field "$1" "$3")
  [ "$got" -ge "$4" ] && [ "$got" -le "$5" ] || die "$1: $2 = $got, expected $4..$5"
  note "ok: $1: $2 = $got (expected $4..$5)"
}

stage_rls() {
  say "row-level security and the planner's statistics"
  own_sandbox
  source_pin
  [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 default" ] || die "run the build stage first"
  if [ -f "$DATA/postmaster.pid" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  fi
  verify_free
  rm -rf "$DATA"
  : > "$OUT/results.txt"
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
    -c "CREATE /* wiki_of12_fixture */ DATABASE of" > /dev/null || die "CREATE DATABASE failed"
  {
    uname -srm
    "$BIN/postgres" --version
    for g in block_size default_statistics_target autovacuum shared_buffers; do
      s postgres "SELECT '$g=' || current_setting('$g')"
    done
    s postgres "SELECT 'max_data_alignment=' || max_data_alignment FROM pg_control_init()"
    for fn in int4lt int4ge int4le numeric_lt numeric_ge numeric_le numeric_eq; do
      s postgres "SELECT '$fn.proleakproof=' || proleakproof FROM pg_proc WHERE proname = '$fn'"
    done
  } > "$OUT/platform.txt"
  cat "$OUT/platform.txt" >&2

  # Fixtures (disposable).  n is unique; k is 0 on half the rows.
  q postgres "CREATE ROLE alice LOGIN NOSUPERUSER NOBYPASSRLS"
  q postgres "CREATE TABLE mp (id int, n numeric, k numeric)"
  q postgres "INSERT INTO mp SELECT g, g, CASE WHEN g <= $ROWS / 2 THEN 0 ELSE g % 1000 + 1 END FROM generate_series(1, $ROWS) g"
  q postgres "CREATE TABLE mr (LIKE mp); INSERT INTO mr SELECT * FROM mp"
  q postgres "ALTER TABLE mr ENABLE ROW LEVEL SECURITY; CREATE POLICY p_all ON mr FOR SELECT TO alice USING (true)"
  q postgres "CREATE TABLE pm (LIKE mp) PARTITION BY RANGE (id)"
  q postgres "CREATE TABLE pm1 PARTITION OF pm FOR VALUES FROM (1) TO ($ROWS / 2 + 1)"
  q postgres "CREATE TABLE pm2 PARTITION OF pm FOR VALUES FROM ($ROWS / 2 + 1) TO ($ROWS + 1)"
  q postgres "INSERT INTO pm SELECT * FROM mp"
  q postgres "ALTER TABLE pm ENABLE ROW LEVEL SECURITY; CREATE POLICY p_all ON pm FOR SELECT TO alice USING (true)"
  q postgres "GRANT SELECT ON mp, mr, pm TO alice"
  q postgres "ANALYZE"

  # The same four conditions on the plain and the row-level-security table.
  measure plain_n_range  mp "n BETWEEN 1000 AND 1999"
  measure rls_n_range    mr "n BETWEEN 1000 AND 1999"
  measure plain_n_lt     mp "n < 1000"
  measure rls_n_lt       mr "n < 1000"
  measure plain_k_eq     mp "k = 0"
  measure rls_k_eq       mr "k = 0"
  measure plain_id_range mp "id BETWEEN 1000 AND 1999"
  measure rls_id_range   mr "id BETWEEN 1000 AND 1999"
  # Partitioned table: SELECT granted on the parent only, then on the partitions.
  measure part_parent_grant   pm "n BETWEEN 1000 AND 1999"
  q postgres "GRANT SELECT ON pm1, pm2 TO alice"
  measure part_child_grant    pm "n BETWEEN 1000 AND 1999"

  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  STARTED=""
  verify_stopped

  # Assertions.  Actual counts are fixed by the data.
  expect_range plain_n_range actual 5 1000 1000
  expect_range rls_n_range actual 5 1000 1000
  expect_range plain_n_range estimate 4 900 1100
  expect_range rls_n_range estimate 4 1 1
  expect_range plain_n_lt estimate 4 900 1100
  expect_range rls_n_lt estimate 4 14900 15100
  expect_range plain_k_eq estimate 4 14000 16000
  expect_range rls_k_eq estimate 4 1 100
  expect_range rls_k_eq actual 5 15000 15000
  expect_range plain_id_range estimate 4 900 1100
  [ "$(field plain_id_range 4)" = "$(field rls_id_range 4)" ] \
    || die "int4 range estimate differs between mp and mr"
  note "ok: int4 range estimate is the same on mp and mr"
  expect_range part_parent_grant estimate 4 1 2
  expect_range part_child_grant estimate 4 900 1100
}

stage_report() {
  own_sandbox
  say "results ($OUT/results.txt)"
  printf 'case|table|condition|estimated_rows|actual_rows\n' >&2
  cat "$OUT/results.txt" >&2
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

[ "$#" -gt 0 ] || set -- build rls report
for st in "$@"; do
  case "$st" in
    build|rls|report|stop|clean) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- Planner entry and tuple fraction: `standard_planner()`, `grouping_planner()`, `preprocess_limit()`, `get_cheapest_fractional_path()` in `planner.c`; `make_subplan()` and `SS_process_ctes()` in `subselect.c`; `planmain.h`; `gram.y` cursor rule.
- Costing: `costsize.c` header, `clamp_row_est()`, `cost_index()`, `initial_cost_nestloop()`, `set_baserel_size_estimates()`, `compute_bitmap_pages()`; `pathnode.c` fractional comparison, `create_limit_path()` and `adjust_limit_rows_costs()`; `consider_startup` in `relnode.c`; the `Limit` path loop in `grouping_planner()`; `cost.h`.
- Selectivity: `clausesel.c`, `selfuncs.c` (`eqsel_internal`, `var_eq_const`, `var_eq_non_const`, `get_restriction_variable`, `get_variable_numdistinct`, `examine_variable`, `statistic_proc_security_check`, `scalarineqsel`, `scalarineqsel_wrapper`, `ineq_histogram_selectivity`, `get_actual_variable_range`, `btcostestimate`), `selfuncs.h`, the `RestrictInfo` pseudoconstant comment in `pathnodes.h`, `plancat.c` (`estimate_rel_size`, `restriction_selectivity`, `has_unique_index`, function rows), `heapam_estimate_rel_size()`.
- Index matching and plan construction: `indxpath.c` (including the `match_clause_to_indexcol()` dispatch), `restrictinfo.c`, `createplan.c`, `nbtutils.c`.
- Pushdown: `allpaths.c` subquery pushdown rules and security barrier handling; `rowsecurity.c` and `check_enable_rls()` in `rls.c`. The rewriter's `fireRIRrules()` step that attaches policy conditions to `rte->securityQuals` in `rewriteHandler.c`; `has_bypassrls_privilege()` in `aclchk.c`; the child-RTE `securityQuals` reset in `inherit.c`; the parent permission fallback in `examine_variable()`; the range-pair combination in `clauselist_selectivity()`.
- Sub-select and function pull-up: `pull_up_sublinks()` and `inline_set_returning_functions()` in `prepjointree.c`; `convert_EXISTS_sublink_to_join()` in `subselect.c`; `inline_set_returning_function()` in `clauses.c`; the `JoinType` comments in `nodes.h`.
- Executor: `execScan.c`, `indexam.c`, `nodeIndexscan.c`, `nodeIndexonlyscan.c`, `nodeBitmapHeapscan.c`, `nodeNestloop.c`, `nodeHashjoin.c`, `nodeHash.c`, `nodeLimit.c`, `nodeCtescan.c`, `nodeFunctionscan.c`, `execSRF.c`; the `tidbitmap.c` recheck flag; the EXPLAIN join-node counters in `explain.c`.
- Plan cache: `plancache.c` `choose_custom_plan()`; the `CURSOR_OPT_GENERIC_PLAN` and `CURSOR_OPT_CUSTOM_PLAN` flags in `parsenodes.h`, with a search of `src/`, `contrib/` and `src/pl/` for code that sets them.
- Partitioning: `partprune.c`, `inherit.c`.
- Statistics maintenance: `analyze.c` (including where `std_typanalyze()` reads `default_statistics_target`, the two `do_analyze_rel()` passes, and how `acquire_sample_rows()` keeps every row of a table no larger than the sample), `sampling.c` (`BlockSampler_Init()`), `statscmds.c` (accepted relation kinds), `autovacuum.c` (the worker's analyze call and its configuration reload).
- Set operations: `recurse_set_operations()` and the `UNION`/`UNION ALL` fraction handling in `prepunion.c`; the set-operation fraction reset in `grouping_planner()`; every caller of `get_cheapest_fractional_path()` in `src/backend/optimizer/`.
- GUC definitions in `guc.c`; docs `perform.sgml`, `config.sgml`, `ref/explain.sgml`.
- Catalog and extension boundaries: `pg_operator.dat`, `pg_proc.dat` (`proleakproof` of `texteq`, `text_lt`, `textlike`), `catalog/Makefile`, `parser/Makefile`, `Makefile.global.in`, `functioncmds.c`, hook headers.
- Tests: `subselect.out`, `join.out`, `bitmapops.sql`, `select_parallel.out`.
- Measurement: a 12.2 server built from the pin with default configure options, one cluster, and `EXPLAIN (ANALYZE, TIMING OFF)` for ten cases run as a non-superuser role against a plain table, a row-level-security table and a partitioned row-level-security table; see [Measurement Script](#measurement-script). For the setting defaults it relies on, also `guc.c` (`autovacuum`, socket settings, timeouts) and `initdb.c`.
- Wiki pages linked for depth: [plan_cache_mode](plan-cache-mode.md), [partitioning](partitioning-planning-execution-optimizations.md), [statistics sources](query-planner-statistics-sources.md), [EXPLAIN BUFFERS](../observability/explain-analyze-buffers-output.md). No v12 common concept pages exist.

## Evidence Map

| Claim | Evidence |
|---|---|
| The planner picks the cheapest path; `get_cheapest_fractional_path()` applies a fraction for cursors, SubPlans and the branches of a top-level `UNION ALL` under a `LIMIT`, and receives 0 at the top otherwise | [planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5886-L5910), [planner.c:377-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L411), [subselect.c:233](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L233), [prepunion.c:274-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L274-L280), [prepunion.c:553-565](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L553-L565) |
| A `LIMIT` acts through `consider_startup` and the cost of a `Limit` path over each surviving path | [planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c:3598](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3598) |
| Row estimate = tuples × selectivity, clamped to ≥ 1 | [costsize.c#set_baserel_size_estimates](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4404-L4418), [costsize.c#clamp_row_est](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L187-L200) |
| Table size scales stored density by live pages; 10-page minimum when `relpages = 0` and no inheritance children, including a table vacuumed while empty | [heapam_handler.c:2082-2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2082-L2157) |
| Default selectivities | [selfuncs.h:32-49](../../../../raw/postgres-12/src/include/utils/selfuncs.h#L32-L49) |
| Equality without statistics divides by an inferred or default distinct count; against an unknown value it averages over distinct values; a column with a single-column unique index gets `1 / rows` in both cases; the fixed 0.005 applies when no variable side is found | [selfuncs.c:324-327](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L324-L327), [selfuncs.c:469-472](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L469-L472), [selfuncs.c:417-425](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L417-L425), [selfuncs.c#get_variable_numdistinct](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5019-L5116), [selfuncs.c:478-505](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L478-L505), [selfuncs.c:258-262](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L258-L262) |
| Inequality estimates re-read the histogram ends from a non-partial, non-hypothetical B-tree index that leads with the column and matches the sort operator | [selfuncs.c:888-935](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L888-L935), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5263-L5325) |
| Multiplication assumes independence; extended statistics first; a statistics object on a partitioned table is never built | [statscmds.c:122-125](../../../../raw/postgres-12/src/backend/commands/statscmds.c#L122-L125), [analyze.c:255-268](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268), [analyze.c:578-584](../../../../raw/postgres-12/src/backend/commands/analyze.c#L578-L584), [clausesel.c:55-122](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L55-L122) |
| Index condition requirements; operator family or support function as alternatives; other clause shapes use sibling matchers | [indxpath.c:2399-2440](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2399-L2440), [indxpath.c:2262-2300](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2262-L2300), [indxpath.c:2593-2645](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L2593-L2645) |
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
| Security promotability; statistics hidden from non-leakproof operators under row-level security | [selfuncs.c:4743-4756](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4743-L4756), [selfuncs.c:4968-4983](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4968-L4983), [selfuncs.c:673-686](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L673-L686), [restrictinfo.c:376-388](../../../../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L376-L388), [allpaths.c:2221-2227](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2221-L2227) |
| Under row-level security, a non-leakproof range with both bounds is estimated at 1 row; a partition's statistics are visible when the user can read it directly; the rewriter attaches policy conditions to the range-table entry; measured on 12.2 | [clausesel.c:294-305](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L294-L305), [clausesel.c:315-332](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L315-L332), [inherit.c:425-450](../../../../raw/postgres-12/src/backend/optimizer/util/inherit.c#L425-L450), [selfuncs.c:4768-4838](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4768-L4838), [rewriteHandler.c:2043-2045](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L2043-L2045), [rewriteHandler.c:2097-2098](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L2097-L2098), [rls.c:87-117](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L87-L117), [Measurement Script](#measurement-script) |
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
- The worked calculation in F9 uses illustrative costs, not output from a 12.2 server. The page's measurement script does not cover it; a measured example would need a new stage in that script.
- The measured example shows only the estimates under row-level security, not a plan change that follows from them. The claim that a 1-row range estimate makes a nested loop look cheap rests on the nested-loop costing in F6 ([costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471)); the script does not measure a join.
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
- [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5263-L5325)
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
- [selfuncs.c#statistic_proc_security_check](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4968-L4983)
- [analyze.c#analyze_rel](../../../../raw/postgres-12/src/backend/commands/analyze.c#L255-L268)
- [sampling.c#BlockSampler_Init](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L24-L46)
- [clausesel.c#range-pair](../../../../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L294-L332)
- [rewriteHandler.c#fireRIRrules-securityQuals](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L2043-L2098)
- [inherit.c#expand_single_inheritance_child](../../../../raw/postgres-12/src/backend/optimizer/util/inherit.c#L425-L450)
- [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670)
- [relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201)
- [tidbitmap.c#PagetableEntry](../../../../raw/postgres-12/src/backend/nodes/tidbitmap.c#L95-L104)
- [prepunion.c#recurse_set_operations](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L274-L280)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
