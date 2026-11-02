---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Join Methods in PostgreSQL 12: How Each Works, Why One Is Chosen, and Which Is Faster (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Main idea](#main-idea)
  - [Mental model: the values that decide the choice](#mental-model-the-values-that-decide-the-choice)
  - [Logic map: how a join method is chosen](#logic-map-how-a-join-method-is-chosen)
  - [Nested loop join](#nested-loop-join)
  - [Merge join](#merge-join)
  - [Hash join](#hash-join)
  - [Side-by-side comparison](#side-by-side-comparison)
  - [Why one gets chosen over another](#why-one-gets-chosen-over-another)
  - [Which methods are possible at all](#which-methods-are-possible-at-all)
  - [The cost formulas](#the-cost-formulas)
  - [Worked calculation: the same join priced three ways](#worked-calculation-the-same-join-priced-three-ways)
  - [Which is faster: measured results](#which-is-faster-measured-results)
  - [Key interaction: estimated cost is not elapsed time](#key-interaction-estimated-cost-is-not-elapsed-time)
  - [Events that change the choice](#events-that-change-the-choice)
  - [Settings that change the choice](#settings-that-change-the-choice)
  - [Build, catalog and extension boundaries](#build-catalog-and-extension-boundaries)
  - [Tests](#tests)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Stages](#stages)
  - [Non-default settings in the measurement](#non-default-settings-in-the-measurement)
  - [Last run](#last-run)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, explain each join method, why one gets chosen over another, and which ones are faster than the others?

## Answer

### Main idea

PostgreSQL 12 has three join methods: [nested loop](../../../glossary.md#nested-loop-join), [merge join](../../../glossary.md#merge-join) and [hash join](../../../glossary.md#hash-join) ([optimizer/README:50-58](../../../../raw/postgres-12/src/backend/optimizer/README#L50-L58)).

**No method is always the fastest.** Each one wins on a different shape of work:

| Method | One-line idea | Wins when |
|---|---|---|
| Nested loop | For each outer row, search the inner side | Few outer rows, and an index on the inner join key |
| Merge join | Walk two sorted inputs together | Both inputs are already sorted, in a one-process plan. With [parallel](../../../glossary.md#parallel-query) workers, Parallel Hash beat it in the measured `wide` case |
| Hash join | Load one side into a hash table, probe it with the other | Large unsorted inputs joined on equality, at the default `work_mem` |

On this page's measurements, the winner changed from case to case. A 100-row lookup ran about 130 times faster with a nested loop than with a hash join. An unindexed 20,000-row join ran about 2,100 times faster with a hash join than with a nested loop. See [the measured results](#which-is-faster-measured-results).

The [planner](../../../glossary.md#planner) does not choose by rule. It prices the legal methods with estimated row counts and keeps the cheapest ([optimizer/README:40-42](../../../../raw/postgres-12/src/backend/optimizer/README#L40-L42), [joinpath.c#add_paths_to_joinrel](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L116-L327)). Two results follow, and both surprise people:

- A wrong row estimate can pick a slow method. With parallelism off, one measured query took about 6.1 seconds instead of 92 ms. "Parallelism off" here means `max_parallel_workers_per_gather = 0`, set by the script; the default is 2. See [non-default settings](#non-default-settings-in-the-measurement).
- The lowest [cost](../../../glossary.md#cost) is not always the shortest time. In one measured join with parallelism off, the planner preferred a hash join that ran about 2.1 times as long as the merge join it rejected.

### Mental model: the values that decide the choice

A join has an **outer** input and an **inner** input. [EXPLAIN](../../../glossary.md#explain) prints the outer input first ([explain.c:1985-1993](../../../../raw/postgres-12/src/backend/commands/explain.c#L1985-L1993)). For inner, left, right and full joins the planner tries each pair of inputs in both directions, so "outer" is the planner's choice, not the order you wrote in the query ([joinrels.c:785-790](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L785-L790), [joinrels.c:802-807](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L802-L807), [joinrels.c:816-821](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L816-L821)). An anti join tries one direction only, with the preserved side outer. A semi join does the same, and can also run in either direction as an inner join over a de-duplicated right side ([joinrels.c:851-881](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L851-L881), [joinrels.c:893-895](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L893-L895)).

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Outer rows | Estimated rows from the outer input | `Path.rows`: table size times selectivity | Mixed. After the table has been vacuumed or analyzed (`relpages` above 0), the table size is the live block count times the stored rows-per-page density ([heapam_handler.c:2083](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2083), [heapam_handler.c:2133](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2133), [heapam_handler.c:2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2157)). With `relpages` at 0, the density comes from the column widths instead, and a table with no inheritance children counts as at least 10 pages ([heapam_handler.c:2115-2119](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2115-L2119), [heapam_handler.c:2134-2155](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2134-L2155)). [Selectivity](../../../glossary.md#selectivity) comes from stored [statistics](../../../glossary.md#statistics), so it goes stale when the distribution changes after the last `ANALYZE`. It is also wrong when columns are correlated, unless [extended statistics](../../../glossary.md#extended-statistics) cover them | Multiplies the nested loop's inner cost: one inner rescan per outer row after the first ([costsize.c:2469-2470](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2469-L2470)), plus the rescan startup cost ([costsize.c:2444-2445](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2444-L2445)) |
| Inner rows | Estimated rows from the inner input | Same | Same | Sets hash table size ([costsize.c:3312-3320](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3312-L3320)) and sort cost, through the `tuples` input of `cost_sort()` ([costsize.c:1693-1700](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1693-L1700)) |
| [Parameterized](../../../glossary.md#parameterized-path) inner path | An inner [index scan](../../../glossary.md#index-scan) that takes the outer row's value as its key | Generated when an index matches the join clause ([optimizer/README:717-741](../../../../raw/postgres-12/src/backend/optimizer/README#L717-L741)) | Plan-time option | Lets a nested loop skip most of the inner side |
| Sort order ([pathkeys](../../../glossary.md#pathkey)) | The order a path already delivers | Index scans and earlier merge or nested loop joins | Plan-time property | Lets a merge join skip a sort ([joinpath.c:605-610](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L605-L610)) |
| [`work_mem`](../../../glossary.md#work_mem) | Memory budget for each hash table or sort; default 4 MB | [GUC](../../../glossary.md#guc) ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2240)) | Live setting, read at planning and again at execution | Decides whether a hash join needs batches and a sort goes to disk ([nodeHash.c:704](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L704), [nodeHash.c:785](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L785), [costsize.c:1732](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1732)) |
| Most common value frequency | Share of the inner rows holding the commonest key | [most common values](../../../glossary.md#most-common-values-and-histogram) in [`pg_statistic`](../../../glossary.md#pg_statistic), through `estimate_hash_bucket_stats()` ([selfuncs.c:3412](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3412)) | Stored | Can add [`disable_cost`](../../../glossary.md#disable_cost) to a hash join, so it loses to any unpenalized alternative but stays available; see [the skew case](#why-one-gets-chosen-over-another) ([costsize.c:3503-3506](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3503-L3506)) |
| Cost constants | `seq_page_cost` 1, `random_page_cost` 4, `cpu_tuple_cost` 0.01, `cpu_operator_cost` 0.0025 | [cost.h:24-28](../../../../raw/postgres-12/src/include/optimizer/cost.h#L24-L28) | Live settings | Turn row and page counts into one cost number |

Cost is not time. Its units are "an arbitrary scale", where only relative values matter ([config.sgml#planner-cost-constants](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4670-L4681)).

### Logic map: how a join method is chosen

```text
Two inputs to join (repeated for each pair; both directions except
SEMI and ANTI, see the mental model)
│
├─ Is there a usable equality on a mergejoinable operator?      [1]
│    ├─ yes → sort both inputs, or use inputs already sorted → MERGE paths
│    └─ no  → no merge path (except a FULL JOIN with no clause
│             or only constants, such as ON true or ON false)   [2]
│
├─ Is the join type INNER, LEFT, SEMI or ANTI?                  [3]
│    ├─ yes → one path per outer path and inner choice:
│    │        plain inner, Materialize inner, parameterized index inner
│    │        → NESTED LOOP paths
│    └─ no (RIGHT, FULL) → no nested loop
│
├─ Is there a usable equality on a hashjoinable operator?       [4]
│    ├─ yes → HASH paths
│    └─ no  → no hash path
│
▼
Each path is priced: quick lower bound first, then full cost    [5]
│
├─ dominated by an existing path → dropped                       [6]
│    (on cost, sort order, parameterization, rows, parallel safety)
└─ otherwise kept
│
▼
Kept paths feed the next join level                              [7]
│
▼
Final choice                                                     [8]
├─ LIMIT: each final path already carries a Limit priced for
│         the rows it reads; the cheapest total of those wins
├─ cursor: the cheapest path for cursor_tuple_fraction wins
└─ otherwise: the cheapest total wins
   (this is the top level; EXISTS, ANY and ALL sub-selects kept
    as subplans, and UNION ALL branches under a LIMIT or cursor,
    also choose by a fraction, see step 7)
```

Key to the citations:

1. [joinpath.c:198-205](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L198-L205), [joinpath.c:268-270](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L268-L270), [joinpath.c#select_mergejoin_clauses](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1931-L2029)
2. [joinpath.c:1115-1127](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1115-L1127), [joinpath.c:1963-1969](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1963-L1969)
3. [joinpath.c:1345-1378](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1345-L1378), [joinpath.c:1461-1493](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1461-L1493)
4. [joinpath.c:306-308](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L306-L308), [joinpath.c:1696-1722](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1696-L1722)
5. [joinpath.c:413-428](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L413-L428)
6. [pathnode.c#add_path](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L423-L484), [optimizer/README:425-430](../../../../raw/postgres-12/src/backend/optimizer/README#L425-L430)
7. [allpaths.c#standard_join_search](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2846-L2881)
8. `LIMIT`: [planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670). Cursor and default: [planner.c:377-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L411), [planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5885-L5910). Sub-selects and `UNION ALL` branches: [subselect.c:208-233](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L233), [prepunion.c:271-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L271-L280), [prepunion.c:553-565](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L553-L565)

### Nested loop join

**How it works.** `ExecNestLoop()` takes one outer row, copies any values the inner scan needs into parameters, rescans the inner plan, and tests each inner row. When the inner plan runs out, it takes the next outer row ([nodeNestloop.c:100-202](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L100-L202)).

It comes in two very different forms:

| Form | What each outer row does | Work grows as |
|---|---|---|
| Parameterized inner | Runs an index search for its own key ([optimizer/README:717-741](../../../../raw/postgres-12/src/backend/optimizer/README#L717-L741)) | Outer rows × one index search |
| Plain inner | Reads every inner row and filters ([costsize.c:2465-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2465-L2471)) | Outer rows × inner rows |

The README calls the parameterized form "the only plan type that can avoid fetching all of B" ([optimizer/README:738-741](../../../../raw/postgres-12/src/backend/optimizer/README#L738-L741)). A merge join can stop early, as the next section shows, but only at the high end of the key range. Rows below the first match are still read and skipped, and the planner charges them to startup ([costsize.c:2730-2732](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2730-L2732), [costsize.c:2838-2847](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2838-L2847), [costsize.c:3145-3147](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3145-L3147)).

The planner may put a [Materialize node](../../../glossary.md#materialize-node) on a plain inner side. The inner rows are then read once and replayed, which costs `cpu_operator_cost` per row on each rescan while they fit `work_mem` ([joinpath.c:1403-1412](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1403-L1412), [costsize.c:3774-3785](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3774-L3785)). When they do not fit, each rescan also pays for re-reading the spilled pages ([costsize.c:3786-3796](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3786-L3796)). The 1.89e10 `unindexed` nested loop price below shows that charge: 1.25e10 for row pairs, 2.5e9 for rescans and 3.9e9 for spilled pages. Materialize shrinks the constant, but the row-pair count stays the same.

**Strengths.** It starts returning rows at once when its inputs start fast, because its startup cost is just the two inputs' startup costs ([costsize.c:2442](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442)). It needs no sort and no hash table, and accepts any join condition, or none, because it only evaluates the condition on each row pair ([nodeNestloop.c:214](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L214)). It preserves the outer input's order, so it can feed an `ORDER BY` without a sort ([optimizer/README:596-605](../../../../raw/postgres-12/src/backend/optimizer/README#L596-L605)).

**Weakness.** If the outer row count is higher than estimated, every extra row pays for another inner scan. Nothing corrects this during execution.

### Merge join

**How it works.** Both inputs arrive sorted on the join key. The executor compares the two current rows and advances whichever is lower until they match. It then joins the matching rows ([nodeMergejoin.c:61-85](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L61-L85)).

Duplicates are the awkward case. If the outer side repeats a key, the second outer row must see the same inner rows again. The executor marks the first matching inner row and restores to it ([nodeMergejoin.c:51-58](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L51-L58), [nodeMergejoin.c:1053-1078](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1053-L1078)). So the inner input must support mark and restore, unless the planner proves that no restore can happen. It proves that when each outer row needs only one match, because the join is a semi or anti join or the inner side is provably unique, and every join clause is a merge clause. The path then records `skip_mark_restore` ([costsize.c:2989-3002](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2989-L3002)). As a result the planner adds no Material node for mark and restore ([costsize.c:3080-3084](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3080-L3084)), and the executor never restores the inner position ([nodeMergejoin.c:1074](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1074)).

When mark and restore is still needed, the planner puts a Material node above the inner input if that input cannot mark and restore, or if a Material node looks cheaper ([costsize.c:3086-3111](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3086-L3111), [createplan.c:4187-4203](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4187-L4203)). In the same case it also adds one when the inner input needs a sort that is expected to spill, unless `enable_material` is off, so that the sort's final merge pass need not support mark and restore ([costsize.c:3113-3128](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3113-L3128)).

The sorted order can come from two places ([joinpath.c:1068-1080](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1068-L1080)):

- An explicit Sort node, which must read its whole input before the first row comes out ([nodeSort.c#ExecSort](../../../../raw/postgres-12/src/backend/executor/nodeSort.c#L40-L118)).
- An index scan or another path that already delivers the order, with no sort step.

That difference explains most of merge join's behavior. With sorted inputs it is cheap, and it starts at once unless many rows precede the first match, because those rows are read and skipped first. With unsorted inputs it pays for two sorts.

**Strengths.** Its output is ordered on the join key, so a later merge or `ORDER BY` can reuse that order ([joinpath.c:1027-1029](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1027-L1029)). A `RIGHT` or `FULL` merge join is the exception: it can insert nulls at random points, so the planner records no sort order for it ([optimizer/README:608-610](../../../../raw/postgres-12/src/backend/optimizer/README#L608-L610), [pathkeys.c:1031-1032](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L1031-L1032)). A merge join stops when one input is exhausted, unless it must still return the other side's unmatched rows. Left and anti joins read the outer side to the end, a right join reads the inner side, and a full join reads both. The executor sets which side to null-fill from the join type, and stops at the end of either input unless that fill is still needed ([nodeMergejoin.c:1540-1580](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1540-L1580), [nodeMergejoin.c:1245-1258](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1245-L1258), [nodeMergejoin.c:1313-1325](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1313-L1325)). The cost model mirrors this and credits the early stop: it scans only part of an input unless the join type requires all of it ([costsize.c:2726-2737](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2726-L2737), [costsize.c:2780-2790](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2780-L2790), [perform.sgml:878-882](../../../../raw/postgres-12/doc/src/sgml/perform.sgml#L878-L882)).

**Weakness.** It needs a sort on anything not already ordered. Heavy duplicates on both sides mean repeated re-reads of inner rows, which the cost model charges through `rescannedtuples` ([costsize.c:3010-3051](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3010-L3051)).

### Hash join

**How it works.** The join has two phases:

1. **Build.** The Hash node reads the entire inner input into a hash table on the join key ([nodeHash.c#MultiExecPrivateHash](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L139-L175)). No row can be returned until this finishes.
2. **Probe.** Each outer row hashes its key, finds its bucket and checks that bucket's rows ([nodeHashjoin.c:345-440](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L345-L440)).

Because build reads one side fully and probing visits a short bucket, the work grows with the two input sizes added together, not multiplied. The target is one row per bucket ([nodeHash.c:664-665](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L664-L665)). The planner normally ends up hashing the smaller side, because that direction prices lower: the build charge is per inner row, and the probe charge per outer row is smaller ([costsize.c:3289-3291](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3289-L3291)).

**When the table does not fit `work_mem`.** The join splits into batches:

| Stage | What happens | Source |
|---|---|---|
| Planning, then executor start | `ExecChooseHashTableSize()` picks a power-of-two batch count: once to price the path, and again to create the table | [nodeHash.c:785-838](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L785-L838), [costsize.c:3312](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3312), [nodeHash.c:463](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L463) |
| Build | Batch 0's rows stay in memory; the other rows go to [temporary files](../../../glossary.md#temporary-file) | [nodeHash.c#ExecHashTableInsert](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1595-L1680) |
| Probe | Outer rows for a later batch go to that batch's file | [nodeHashjoin.c:386-411](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L386-L411) |
| Later batches | Each batch's table is loaded and probed in turn | [nodeHashjoin.c#ExecHashJoinNewBatch](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L947-L1000) |

A hash join also corrects itself during execution. If the table outgrows its budget, the batch count doubles, whatever the planner estimated ([nodeHash.c:1658-1661](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1658-L1661), [nodeHash.c:906](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L906)). In the single-batch case, the bucket count grows too ([nodeHash.c:1640-1652](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1640-L1652)). The measured `misest.hash` plan shows this: `Buckets: 16384 (originally 1024)`.

**Exceptions.**

- Identical keys cannot be split across batches. If a split moves all or none of the rows, the executor stops adding batches and lets memory exceed the budget ([nodeHash.c:1036-1045](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1036-L1045)).
- By default all inner rows use the main table. Skew handling is a special case with four conditions: the join is not Parallel Hash, it needs more than one batch, it has one hash clause whose outer side is a simple table column, and that column's most common values cover at least 1% of the outer side ([nodeHash.c:631-648](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L631-L648), [createplan.c:4458-4466](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4458-L4466), [hashjoin.h:111](../../../../raw/postgres-12/src/include/executor/hashjoin.h#L111)). Inner rows matching those values then go to a small skew table, up to 2% of the budget, handled with the first batch ([hashjoin.h:84-92](../../../../raw/postgres-12/src/include/executor/hashjoin.h#L84-L92), [hashjoin.h:110](../../../../raw/postgres-12/src/include/executor/hashjoin.h#L110)). The rows that stay there avoid the disk. When the table passes its budget, the executor evicts a bucket, and those rows move to the main table or to a batch file, so they can reach disk ([nodeHash.c:2444-2445](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L2444-L2445), [nodeHash.c:2524-2525](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L2524-L2525)).
- An empty outer side can end the join before the build. The executor never tries this for a join that must return unmatched inner rows, such as a right or full join, or for Parallel Hash ([nodeHashjoin.c:241-257](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L241-L257)). Outside those cases, it always tries for a join that must return unmatched outer rows, such as a left join. For any other join it tries only when the outer side's startup cost is below the hash side's total cost and no earlier scan of this join found the outer side non-empty ([nodeHashjoin.c:258-272](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L258-L272)).
- An empty inner side ends the join right after the build, unless the join must return unmatched outer rows ([nodeHashjoin.c:298](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L298)).

**Strengths.** It handles large unsorted inputs well, needs no index, and has the run-time correction described above.

**Weakness.** It needs an equality condition, delays the first row until the build is done, and promises no output order. Batching can reorder rows, so the planner marks every hash join path as having no sort order ([optimizer/README:605-608](../../../../raw/postgres-12/src/backend/optimizer/README#L605-L608)).

### Side-by-side comparison

| Property | Nested loop | Merge join | Hash join |
|---|---|---|---|
| Join condition | Any, or none | Mergejoinable equality | Hashjoinable equality |
| Index needed | Not required; decisive when present | Optional; replaces a sort | Not used on the join key |
| Inner side read | Once per outer row, or by index search | Once, plus re-reads for duplicates | Once, entirely |
| Memory | None for the join itself; Materialize uses up to `work_mem` | Sorts use up to `work_mem` each, and so does a Material node over the inner input | Hash table uses up to `work_mem`, unless identical keys force more; Parallel Hash can use more, see [settings](#settings-that-change-the-choice) |
| Spills to temp files | Only through Materialize | Through its sorts, or its Material node when the rows kept since the last mark pass `work_mem` | Through batches |
| First row | Immediate, if the inputs start fast | Immediate if presorted, after the sorts if not | After the whole build |
| Output order | Keeps the outer order | Sorted on the join key, except `RIGHT` and `FULL` | None promised |
| `RIGHT`, `FULL` | Not supported | Supported if every join clause, constants aside, is a merge clause | Supported with at least one hash clause |
| Plan-time row error | Hurts most | A sort or re-read costs more | Batches, and buckets while single-batch, adapt to the real size; the method and build side stay as planned |

Sources for the memory and spill rows: [nodeMaterial.c:64](../../../../raw/postgres-12/src/backend/executor/nodeMaterial.c#L64), [nodeMaterial.c:285](../../../../raw/postgres-12/src/backend/executor/nodeMaterial.c#L285), [tuplesort.c:22-28](../../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L22-L28), [nodeHash.c:704](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L704), [nodeHash.c:1036-1045](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1036-L1045). Sources for the join-type rows: [joinpath.c:1345-1378](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1345-L1378), [joinpath.c:2017-2026](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L2017-L2026), [nodeHashjoin.c:527-551](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L527-L551). A nested loop does not lose a `RIGHT JOIN` outright: the planner also tries that join with the sides swapped, as a `LEFT` join ([joinrels.c:802-807](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L802-L807)).

### Why one gets chosen over another

The planner follows these steps. Each output feeds the next.

1. **Order the joins.** `standard_join_search()` builds join relations level by level, two inputs at a time ([allpaths.c#standard_join_search](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2846-L2881)). When one join list has `geqo_threshold` (default 12) or more items, [GEQO](../../../glossary.md#geqo) takes over if it is enabled and no `join_search_hook` is set ([allpaths.c:2752](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2752), [allpaths.c:2807-2812](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2807-L2812), [guc.c#geqo_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2027-L2029)). Items are not tables. Explicit `JOIN` syntax is folded into one list only up to `join_collapse_limit`, and subqueries in `FROM` only up to `from_collapse_limit`, both 8 by default. So twelve explicitly joined tables do not reach the threshold at the defaults, while twelve tables listed with commas do ([initsplan.c:792-793](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L792-L793), [initsplan.c:1038-1043](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L1038-L1043), [guc.c#from_collapse_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1996-L2005), [guc.c#join_collapse_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2009-L2018)). See [join order search](../../../glossary.md#join-order-search).
2. **Try the legal directions.** For each pair, `populate_joinrel_with_paths()` calls `add_paths_to_joinrel()` with each input as outer, except for the semi and anti join limits in the [mental model](#mental-model-the-values-that-decide-the-choice) ([joinrels.c:785-790](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L785-L790), [joinrels.c:802-807](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L802-L807), [joinrels.c:816-821](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L816-L821)).
3. **Generate candidates.** That function builds merge paths, then nested loop and presorted merge paths, then hash paths ([joinpath.c:264-308](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L264-L308)).
4. **Discard cheaply.** A quick lower-bound cost rejects hopeless paths before their full cost is computed ([joinpath.c:413-428](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L413-L428)).
5. **Compare paths.** `add_path()` compares total cost with a 1% tolerance, then sort order ([pathnode.c#STD_FUZZ_FACTOR](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L48-L52), [pathnode.c:477-484](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L477-L484)). A path that loses on total cost survives on a better startup cost only when its relation allows startup-cost comparison. An unparameterized path checks the relation's `consider_startup` flag, and a parameterized path checks `consider_param_startup` instead ([pathnode.c:169-170](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L169-L170), [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L166-L215)). `consider_startup` is set when `root->tuple_fraction` is above 0, as under a `LIMIT` ([relnode.c:201](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201), [relnode.c:602](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L602)). `consider_param_startup` starts false ([relnode.c:202](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L202), [relnode.c:603](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L603)), and only a base relation that alone forms the right side of a semi or anti join gets it, because only one inner row is needed there ([allpaths.c:250-278](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L250-L278)). Outside an exact tie, a path dominates another only if it also needs no more outer parameters, returns no more rows, and is at least as parallel-safe, so a parameterized join path can survive a cheaper unparameterized one ([pathnode.c:490-549](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L490-L549)). When sort order, parameters and fuzzy cost all tie, parallel safety decides first, row count second, and a near-exact cost comparison third; if that ties too, the old path stays ([pathnode.c:527-542](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L527-L542)).
6. **Keep useful losers.** A path also survives if it is the only way to get a sort order, even when it costs more overall ([optimizer/README:425-430](../../../../raw/postgres-12/src/backend/optimizer/README#L425-L430)). So a dearer merge join can stay in the running because the next level wants its order.
7. **Pick the final plan.** A query that reads everything takes the cheapest total cost. A `LIMIT` works through two earlier steps. Its row count goes into `root->tuple_fraction`, which keeps fast-start paths alive in step 5. Then every surviving path gets a Limit path whose cost covers only the rows the limit lets through ([planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670)). The final call to `get_cheapest_fractional_path()` in `standard_planner()` therefore receives 0 and takes the cheapest total among those Limit paths. At that top level, only a cursor passes it a fraction, `cursor_tuple_fraction` ([planner.c:377-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L411), [planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5885-L5910)). Both routes favor fast-start plans. Lower levels also choose by fraction, in two cases:

   - **A sub-select that stays a subplan.** Under `EXISTS` it is planned for 1 row, and under `ANY` or `ALL` for half its rows ([subselect.c:208-233](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L233)). A join inside it can therefore be chosen for its fast start even when the query has no `LIMIT`. This does not apply when the planner turns an `EXISTS` or `ANY` sub-select into a semi or anti join instead; that join is planned with the rest of the query ([prepjointree.c:390-479](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L390-L479), [prepjointree.c:502-514](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L502-L514), [subselect.c:1212](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L1212), [subselect.c:1317](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L1317)).
   - **A `UNION ALL` branch.** It receives the enclosing fraction unchanged, while a plain `UNION` resets it to 0 ([prepunion.c:271-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L271-L280), [prepunion.c:553-565](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L553-L565)). So a branch favors a fast start only when the enclosing query has a `LIMIT` or is a cursor; otherwise its fraction is 0, as at the top level ([planner.c:399-403](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L399-L403)). See [tuple fraction](../../../glossary.md#tuple-fraction).

So the practical reasons a method wins are:

| Situation | Usually wins | Why, in the cost model |
|---|---|---|
| Few outer rows, indexed inner key | Nested loop | Few index searches cost less than one full inner scan ([costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471)) |
| Large unsorted inputs, equality | Hash join | Costs add rather than multiply, and no sort is charged ([costsize.c:3274-3291](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3274-L3291)) |
| Inputs already sorted | Merge join, if it beats the hash join's cheaper scans | No sort charge ([costsize.c:2843-2850](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2843-L2850)) |
| `ORDER BY` with `LIMIT` on the key | Merge join or nested loop | Only they deliver the order without a full read ([optimizer/README:596-608](../../../../raw/postgres-12/src/backend/optimizer/README#L596-L608)), and the Limit path's cost covers only the rows read, which rewards a cheap start ([pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670)) |
| Non-equality condition | Nested loop | It is the only method available ([initsplan.c#check_mergejoinable](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2582-L2607), [initsplan.c#check_hashjoinable](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2619-L2638)) |
| One key holds too much of the inner side | Merge join or nested loop | The hash join receives [`disable_cost`](../../../glossary.md#disable_cost) ([costsize.c:3495-3506](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3495-L3506)) |
| Inner side provably unique | Nested loop and hash get cheaper; see the measured [`wide` case](#key-interaction-estimated-cost-is-not-elapsed-time). Merge gains only in the three cases given in the paragraph after the effects list below | The executor stops at the first match, and the nested loop and hash cost formulas know it ([joinpath.c:181-188](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L181-L188), [nodeNestloop.c:322](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L322), [nodeHashjoin.c:682](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L682), [costsize.c:2536-2537](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2536-L2537), [costsize.c:3519-3521](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3519-L3521)). For merge, uniqueness with all-merge join clauses sets `skip_mark_restore` ([costsize.c:2995-3000](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2995-L3000)) |

For a merge join, `skip_mark_restore` has three effects:

1. `rescannedtuples` becomes 0 ([costsize.c:3036-3037](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3036-L3037)).
2. No Material node goes over the inner input, so the path is charged `bare_inner_cost` instead of `mat_inner_cost` ([costsize.c:3080-3084](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3080-L3084), [costsize.c:3133-3136](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3133-L3136)).
3. The executor never restores the inner position ([nodeMergejoin.c:1074](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1074)).

So merge gets cheaper only in three cases. Repeated outer keys would otherwise be charged for re-reads. Or the inner input cannot mark and restore and would otherwise need a Material node ([costsize.c:3109-3111](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3109-L3111)). Or, with `enable_material` on, the inner input needs a sort expected to exceed `work_mem`, which would otherwise force a Material node ([costsize.c:3124-3128](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3124-L3128)). In `wide` none of these applies: `rescannedtuples` is already about 0 when keys are distinct, and the inner index scan supports mark and restore and needs no sort, so no Material node was needed.

The row "One key holds too much of the inner side" is a special case and needs its default stated. By default a hash join carries no penalty. The cause is a most common inner value whose rows alone would exceed `work_mem`. `final_cost_hashjoin()` tests that size and adds `disable_cost`, 1.0e10, to the startup cost, because batching cannot separate equal keys ([costsize.c:3495-3506](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3495-L3506), [costsize.c:120](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L120)). The path stays available, so it can still win if nothing else exists. The measured `sk` fixture shows it: the same hash join costs 10,000,015,628.00 more at 4 MB than at 256 MB. Of that, 1.0e10 is the penalty and 15,628 is batch I/O. Run 12 gave the same gap, and run 9 gave 10,000,015,627.99, and that 0.01 is rounding in numbers this large ([`skew` legs](#which-is-faster-measured-results)).

### Which methods are possible at all

Cost only ranks what is legal. These conditions decide what is legal:

| Condition | Nested loop | Merge | Hash | Source |
|---|---|---|---|---|
| Equality, normal types | Yes | Yes | Yes | [initsplan.c#check_mergejoinable](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2582-L2607), [initsplan.c#check_hashjoinable](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2619-L2638) |
| `<`, `>`, `<>`, `LIKE` and similar | Yes | No | No | Same |
| Equality that does not separate the inputs, such as `a.k = b.k + a.v` | Yes | No | No | [restrictinfo.c:157](../../../../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L157), [joinpath.c:1707-1716](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1707-L1716) |
| Inner side parameterized by the outer side, such as a [LATERAL](../../../glossary.md#lateral) subquery that cannot be flattened | Yes | No | No | [joinpath.c:918-920](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L918-L920), [joinpath.c:1380-1386](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1380-L1386), [joinpath.c:1824](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1824) |
| No join condition (cross join) | Yes | No, except a `FULL JOIN` with no clause or only constants, such as `ON true` or `ON false` | No | [joinpath.c:1115-1127](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1115-L1127), [joinpath.c:1963-1969](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1963-L1969), [joinpath.c:1721-1722](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1721-L1722) |
| Volatile function in the clause | Yes | No | No | [initsplan.c:2598-2600](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2598-L2600), [initsplan.c:2635-2637](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2635-L2637) |
| Operator is hash-only, such as `xid`, `cid`, `aclitem` | Yes | No | Yes | [pg_operator.dat:180](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L180), [pg_operator.dat:202](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L202), [pg_operator.dat:1130](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L1130) |
| Operator is merge-only, such as `bit`, `varbit`, `money`, `tsvector` | Yes | Yes | No | [pg_operator.dat:2082](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L2082), [pg_operator.dat:2143](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L2143), [pg_operator.dat:1041](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L1041), [pg_operator.dat:2898](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L2898) |
| Operator has neither flag, such as `box`, `circle`, `line`, `lseg`, `path` | Yes | No | No | [pg_operator.dat:365](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L365), [pg_operator.dat:970](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L970), [pg_operator.dat:1585](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L1585), [pg_operator.dat:1717](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L1717), [pg_operator.dat:1824](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L1824) |
| `FULL JOIN` whose condition is neither mergeable nor hashable, such as `a.k < b.k` | Not available | Not available | Not available; the statement fails with an error | [joinrels.c:825-833](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L825-L833) |

The measured [`pg_operator`](../../../glossary.md#pg_operator) catalog of this build has 61 `=` operators with typed operands on both sides: 40 support both, 12 only merge, 3 only hash, and 6 neither. `pg_operator.dat` gives the same counts. The flags are not the whole test: array equality, and record equality for merge, are decided per type by `op_mergejoinable()` and `op_hashjoinable()` through the type cache ([lsyscache.c:1205-1216](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1205-L1216), [lsyscache.c:1252](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1252)). A set merge flag is also only a hint: if the operator is in no [B-tree](../../../glossary.md#b-tree) [operator family](../../../glossary.md#operator-class), the clause is not mergejoinable ([initsplan.c:2602-2606](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2602-L2606)). A set hash flag is not checked this way; see [Build, catalog and extension boundaries](#build-catalog-and-extension-boundaries).

`enable_mergejoin = off` and `enable_hashjoin = off` do not forbid a method everywhere:

- By default all three settings are on.
- `enable_nestloop = off` keeps the paths and adds `disable_cost` ([costsize.c:2531-2532](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2531-L2532)).
- `enable_mergejoin = off` and `enable_hashjoin = off` skip generating those paths, but a `FULL JOIN` overrides that check, because it may have no alternative. Such a path then carries `disable_cost` ([joinpath.c:193-205](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L193-L205), [joinpath.c:301-308](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L301-L308), [costsize.c:2977-2978](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2977-L2978), [costsize.c:3407-3408](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3407-L3408)). In the script's `eligibility` stage, which runs `EXPLAIN` only, an equality `FULL JOIN` with both off was still planned as a `Hash Full Join`.
- Nested loops cannot be fully suppressed ([config.sgml#guc-enable-nestloop](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4512-L4525)). In the same `EXPLAIN`-only stage, an `a.k < b.k` join with `enable_nestloop = off` was still planned as a nested loop, priced above 1.0e10, as were an `xid` equality with `enable_nestloop` and `enable_hashjoin` off, an `a.k = b.k + a.v` join and a `LATERAL` subquery with `enable_nestloop` off.

### The cost formulas

Terms used below:

- `Nout`, `Nin`: estimated outer and inner rows.
- `Cout`, `Cin`: total costs of the outer and inner paths.
- `ops`: `cpu_operator_cost`, 0.0025.
- `tup`: `cpu_tuple_cost`, 0.01.
- `Rin`: the cost of one rescan of the inner path.
- `Nmatch`: estimated rows passing the join clause.

These are the common-case formulas for an inner join with one integer key, a single batch, sorts that fit `work_mem`, no duplicates forcing re-reads, a non-unique inner side, and inputs read to the end. A sort that spills uses a different formula ([costsize.c:1732-1760](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1732-L1760)). [Semi joins, anti joins](../../../glossary.md#semi-join-and-anti-join) and provably unique inner sides use refined formulas based on the matched outer rows ([costsize.c:2536-2644](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2536-L2644), [costsize.c:3519-3567](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3519-L3567)).

```text
Nested loop  = Cout + Cin + (Nout - 1) * Rin + Nout * Nin * (tup + ops)
Merge join   = sort(Cout, Nout) + sort(Cin, Nin) + ops * (Nout + Nin) + tup * Nmatch
sort(C, N)   = C + 2 * ops * N * log2(N) + ops * N
Hash join    = Cout + Cin + (ops + tup) * Nin + ops * Nout
               + ops * Nout * (rows per bucket) * 0.5 + tup * Nmatch
```

Sources: [costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471), [costsize.c:2650-2657](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2650-L2657), [costsize.c#cost_sort](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1693-L1790), [costsize.c:3140-3163](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3140-L3163), [costsize.c:3274-3291](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3274-L3291), [costsize.c:3570-3600](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3570-L3600).

In practical terms:

- The nested loop has a multiplication, `Nout * Nin`. With a parameterized index inner, the filtered row count per probe is small, so `Rin` carries the cost instead and that term shrinks.
- The merge join's sort has `N log N`. A presorted input removes `sort()` altogether and charges only the path's own cost.
- The hash join has only additions when buckets hold about one row.

The [key interaction](#key-interaction-estimated-cost-is-not-elapsed-time) shows how far a unique inner side can move these totals. A hash join that needs batches pays extra sequential I/O: `seq_page_cost * (2 * innerpages + 2 * outerpages)`, where the pages are estimated from `(MAXALIGN(width) + 24)` bytes per row ([costsize.c:3329-3338](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3329-L3338), [costsize.c#relation_byte_size](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5461-L5475)). In the measured `wide` case each side is 1,000,000 rows at width 8, so `ceil(32,000,000 / 8192)` = 3,907 pages per side. The charge is 4 × 3,907 = 15,628, which is the exact gap between the 78,873.02 and 63,245.02 plans.

### Worked calculation: the same join priced three ways

Inputs, from the measured `sweep10000` plans: two tables of 10,000 rows and 55 pages, joined on an unindexed, unordered integer key. Each [sequential scan](../../../glossary.md#sequential-scan) costs 55 + 10,000 × 0.01 = 155.

| Step | Nested loop | Merge join | Hash join |
|---|---|---|---|
| Read outer | 155.00 | 155.00, inside the sort | 155.00 |
| Read inner | 205.00 (155 + 2 × 0.0025 × 10,000 for Materialize) | 155.00, inside the sort | 155.00 |
| Rescans | 9,999 × 25 = 249,975.00 | none | none |
| Sort, two inputs | none | 2 × (0.005 × 10,000 × 13.2877 + 25) = 1,378.77 | none |
| Build | none | none | (0.0025 + 0.01) × 10,000 = 125.00 |
| Probe or merge compare | 1.0e8 pairs × 0.0125 = 1,250,000.00 | 0.0025 × 20,000 = 50.00 | 0.0025 × 10,000 = 25.00, plus 12.50 for bucket checks |
| Output rows | included above | 10,000 × 0.01 = 100.00 | 10,000 × 0.01 = 100.00 |
| **Total** | **1,500,335.00** | **1,838.77** | **572.50** |
| Plan shows | 1500335.00 | 1838.77 | 572.50 |

The three totals match the measured plans exactly. Two details were inferred from those totals, not read from statistics: the merge input fractions round to the full inputs, and the bucket charge uses one row per bucket.

The nested loop is priced about 2,600 times the hash join. Measured time agrees on the order of magnitude: 2,309 ms against 2.10 ms, a factor of about 1,100.

### Which is faster: measured results

All numbers below are medians in milliseconds of five timed runs, or three for the sweep, on PostgreSQL 12.2 built from the pin, on one Linux x86_64 host (WSL2) with every table in [`shared_buffers`](../../../glossary.md#shared_buffers). They come from [the script](#the-script). Each is the client-side `psql` `\timing` figure, so it includes parsing, planning and the round trip, not just the join. That overhead matters only for the sub-millisecond legs, which is why the `lookup` and `topn` ratios are less exact than the rest. The script checks the buffer claim on each timed leg: the `EXPLAIN ANALYZE` run before the timings must read no shared block from the operating system. Forced legs use one process, because the script sets `max_parallel_workers_per_gather = 0` on them. The `work_mem` and `shared_buffers` values are not defaults either. See [non-default settings](#non-default-settings-in-the-measurement) for each default, cause and affected result. The "planner's pick" column leads with the default `max_parallel_workers_per_gather = 2`. Its entries marked "with parallelism off" for `wide` and `unindexed`, and every `misest` entry, come from legs that set it to 0.

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup`: 100 outer rows, 1,000,000 indexed inner rows | **0.314** | 68.0 | 40.0 | Nested loop, 0.336 |
| `wide`: 1,000,000 × 1,000,000 on two primary keys | 796 | **233** | 515 (16 batches), 463 (256 MB, one batch) | Parallel Hash Join, 184; with parallelism off, Hash Join, 485 |
| `unindexed`: 1,000,000 × 1,000,000, no index, unordered key | not run; cost 1.89e10 | 708, or 740 at 256 MB | **489**, or **690** at 256 MB | Parallel Hash Join, 207; with parallelism off, Hash Join, 501 |
| `topn`: first 10 rows in key order | 0.153 | 0.188 | 426 | Merge join, 0.210 |
| `misest`: estimate 1 outer row, actual 10,000 | 5,899 | 702 | **96.6** | Nested loop, 6,084; after `CREATE STATISTICS`, Hash Join, 92.0 |

Bold marks the fastest forced method at that memory setting. `topn` has no bold entry: the nested loop and merge ranges overlap (0.116 to 0.206 against 0.167 to 0.224), and the single `EXPLAIN ANALYZE` run ranks them the other way, at 0.090 and 0.048 ms of execution time.

The same plan can differ between legs of one run. In run 12 every such pair overlapped:

| Pair, same plan and cost | Medians, ms | Ranges, ms |
|---|---|---|
| `misest` `nl` and planner's pick | 5,899 and 6,084 | 5,828 to 6,027, and 5,906 to 6,351 |
| `wide` `hash` and planner's pick with parallelism off | 515 and 485 | 481 to 570, and 472 to 497 |
| `unindexed` `hash` and planner's pick with parallelism off | 489 and 501 | 484 to 522, and 493 to 535 |

Gaps of this size between such legs are therefore not plan effects. For earlier runs this page keeps the ranges of these pairs only where it quotes them: run 6's `misest` legs, at 6,079 and 6,452 ms, did not overlap. For the other pairs in runs 5 to 11 it keeps only medians, so whether their ranges overlapped is not recorded. The cause of such gaps is open. See [Open Questions](#open-questions).

Growth with input size, both sides the same size, no index:

| Rows per side | Nested loop | Merge | Hash |
|---|---|---|---|
| 5,000 | 584 | 4.94 | 1.12 |
| 10,000 | 2,309 | 4.62 | 2.10 |
| 20,000 | 9,328 | 11.27 | 4.36 |
| Growth per doubling | 3.95×, 4.04× | 0.94×, 2.44× | 1.88×, 2.08× |

Each growth factor divides a median, as shown in the table, by the median one row above it, so it is derived from the timings, not measured separately. The same rule gives every factor this page quotes for earlier runs, from the medians under [Last run](#last-run). The sweep times each leg only three times, so one or two slow runs move a median more than in the five-run cases. In run 12, the 5,000-row merge leg took 3.18, 4.94 and 5.44 ms, so its median sits above the 10,000-row median of 4.62 ms and the first merge factor is below 1. In run 8, the 20,000-row merge leg took 9.15, 12.78 and 24.17 ms, which gave a growth factor of 2.92×.

The `skew` legs are plans only, with one value in half the rows of a 1,000,000-row table joined to itself:

| Leg | Plan | Cost |
|---|---|---|
| Planner's pick, 4 MB | Merge Join | 4,385,335,519.43 |
| Forced hash, 4 MB | Hash Join | 13,757,962,874.25 |
| Forced nested loop, 4 MB | Nested Loop | 19,533,470,279.26 |
| Planner's pick, 256 MB | Hash Join | 3,757,947,246.25 |

What these numbers show:

- **The indexed nested loop has no rival for a small lookup.** It touched 401 buffers. The hash join read the 1,000,000-row table through a sequential scan (10,314 buffers), and the merge join touched 13,050 buffers. It read the whole index from its first entry, because a merge join skips rows below the first match by reading them ([nodeMergejoin.c:1174](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1174), [nodeMergejoin.c:1270](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1270)). The last outer key, 1,000,000, is also the last inner key, so it read to the end. An index scan fetches the table row of every entry it returns, and reads a new buffer only when the table block changes ([heapam_handler.c:130-137](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L130-L137), [bufmgr.c:1533-1536](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1533-L1536)). So it visited every table page as well, which is why it touched more buffers than the sequential scan.
- **The plain nested loop is quadratic.** Its cost quadruples with each doubling, because the row pairs do: the three plans' top-node costs are 375,206, 1,500,410 and 6,000,818. Measured time grew 3.95 and 4.04 times per doubling in run 12, 3.83 and 3.91 in run 11, 4.03 and 3.84 in run 10, 4.09 and 3.69 in run 9, 4.12 and 3.79 in run 8, 3.97 and 4.00 in run 7 and 3.49 and 3.83 in run 6 on the same host, and 4.01 and 4.05 in run 5 on the earlier host. So the measured factor varies by run, between 3.49 and 4.12. Extrapolating the run 12 20,000-row median by the square of 50 to 1,000,000 rows per side gives about 6.5 hours. That figure is derived, not measured.
- **The ranking among the fast methods depends on existing order, and the gap depends on `work_mem`.** Merge beat hash on sorted primary keys. When both inputs had to be sorted, hash beat merge at 4 MB (489 against 708). At 256 MB the sorts stayed in memory, yet merge did not speed up and the hash join slowed down, so hash still led at 690 against 740. On the earlier Darwin arm64 host, merge won at 256 MB (714 against 888); see [Last run](#last-run). So this ranking depends on the host as well.
- **A hash join delivers nothing before its build, and an `ORDER BY` makes it deliver everything first.** With no `ORDER BY`, `LIMIT 10` took 155 ms: the whole build, then only 126 outer rows read. Not all of them were probed: when the join uses batches, as the same forced hash join did in `wide` at 4 MB, an outer row whose hash belongs to a later batch is written to that batch's file instead ([nodeHashjoin.c:386-411](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L386-L411)). With `ORDER BY`, `topn` took 426 ms, because a Sort above the join needed the join's whole output. The other two methods needed under 0.2 ms.
- **The misestimate cost about 66 times.** The planner estimated 1 outer row and got 10,000, so a plan that looked like one [bitmap scan](../../../glossary.md#bitmap-scan) ran 10,000 of them over 10,064,370 buffer hits. Correlated columns caused it, because the planner multiplies their [selectivities](../../../glossary.md#selectivity). After `CREATE STATISTICS ... (dependencies)` and `ANALYZE`, the outer estimate became 9,400, and the planner chose a hash join on its own. See [extended statistics](../../../glossary.md#extended-statistics).
- **More memory is not always faster.** On the primary-key join, one batch at 256 MB had the lower median in run 12 (463, range 424 to 495, against 515, range 481 to 570), but the ranges overlap. It had the higher median in runs 6 to 8, 10 and 11, with overlapping ranges, and in run 9 it was slower with no overlap (471 against 412). On the earlier Mac host the result is inconclusive: 493 ms at 256 MB against 615 and 463 ms on two 4 MB legs that ran the same plan. On the unordered key the hash join was slower with no overlap (690, range 674 to 780, against 489, range 484 to 522), as in runs 6, 7, 10 and 11; in runs 8 and 9 single slow 4 MB runs made the ranges overlap. The merge join did not reliably gain from in-memory sorts either. Here it took 740 against 708 ms, with overlapping ranges (703 to 749 against 693 to 717). Its 256 MB median was also higher in runs 6 and 8 to 11, with no overlap in run 8 (792 against 713). It was slightly lower in run 7 (708 against 711) and lower on the earlier Mac host (714 against 800). This disagrees with the intuition that avoiding temp files always helps, and [the cause is an open question](#open-questions).
- **Parallelism changes the comparison.** A Parallel Hash Join, planned for the leader plus up to two workers at the default `max_parallel_workers_per_gather` of 2 ([guc.c#max_parallel_workers_per_gather](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3008-L3016)), took 184 ms, faster than one process running a merge join at 233 ms. Its `EXPLAIN ANALYZE` run launched both planned workers, so three processes ran the join.

These are in-memory results. Nothing here measures disk reads, where index-driven random access could rank differently. See [Open Questions](#open-questions).

### Key interaction: estimated cost is not elapsed time

Two measured cases rank the methods differently in cost and in time.

**The `wide` case.** With parallelism off, the planner priced the hash join's aggregate at 78,873 and the merge join's at 95,087, so it chose the hash join. The merge join then ran in 233 ms against 485 ms for that hash plan, and 515 ms on a second leg of the same plan. Each charge can be read from the plans and the source:

| Charge | Hash join | Merge join |
|---|---|---|
| Inputs | 40,620.00 (two sequential scans at 20,310) | 72,586.58: the join total less the compare and output charges; each index scan costs 36,293.43, and the 0.28 difference is an [open question](#open-questions) |
| Build | 12,500 | none |
| Probe or compare | 2,500 + 125 | 5,000.00, the full-scan value; an estimated partial scan would lower it slightly |
| Batch I/O | 15,628 | none |
| Output rows | 0.01 | 10,000 |
| Aggregate above the join | 7,500.01 | 7,500.01 |
| **Total** | **78,873.02** | **95,086.59** |

The merge join starts 31,967 units behind on its inputs. The hash join gives back 25,753 of that through its build, batch and probe charges, less merge's compare charge, which leaves merge about 6,214 behind. The output row adds the other 10,000. The plan shows `rows=1000000` for the hash join, yet the join is charged for one output row.

- **Default.** A hash join with a non-unique inner side charges `cpu_tuple_cost` for every estimated output row and the full bucket charge. The fixture proves it: `ua` and `ub` are copies of `ja` and `jb` with no primary key, and the same forced hash join costs 89,998.01, which is 11,125 more.
- **Cause.** `jb.id` is a primary key, so `inner_unique` is true ([joinpath.c:181-188](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L181-L188)), and `final_cost_hashjoin()` takes its early-stop branch. That branch scales both charges by `outer_match_frac`, the share of outer rows expected to match ([costsize.c:3519-3567](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3519-L3567)).
- **Mechanism.** `compute_semi_anti_join_factors()` asks for the `JOIN_SEMI` selectivity, but it passes the `sjinfo` that `make_join_rel()` built with `jointype = JOIN_INNER` ([costsize.c:4187-4191](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4187-L4191), [joinpath.c:212-214](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L212-L214), [joinrels.c:710](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L710)). `eqjoinsel()` switches on `sjinfo->jointype`, so it returns the inner-join selectivity, about 1/1,000,000 ([selfuncs.c:2064](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L2064)). `outer_matched_rows` therefore rounds to 1, although every outer row matches. Only 1 row gets the output and bucket charges. The other 999,999 get the unmatched-row charge: `cpu_operator_cost` times the average bucket size, here 1, times a factor of 0.05, so 0.000125 each and 125 in total, the 125 above ([costsize.c:3558-3560](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3558-L3560)).

The merge join counts its output with `approx_tuple_count()`, not with these factors, so it pays the full 10,000 ([costsize.c:3008](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3008)). The fixture confirms that uniqueness does not help it here: the forced merge join on `ua` and `ub`, through non-unique indexes, costs 95,084.77 against 95,086.59 on the primary keys. A gap that small is the movement of `ANALYZE`'s sample, not a charge. Hash would still win here without this effect, at 89,998 against 95,087, but 11,125 of its 16,214 lead comes from it. Beyond that, the cost constants are fixed at their defaults and are not calibrated to this machine, so the two rankings can differ.

**The `misest` case.** Here the formula was reasonable but its input was wrong. The nested loop's cost of 26,902 assumed one outer row. Multiplying by the true 10,000 outer rows moves the inner side's cost alone to over 30 million, far above the hash join's 52,734. The hash join's cost barely depends on that estimate, which is why it is the safer plan when estimates are unreliable. The planner does not reason about that risk. It compares only the estimated numbers.

So component A, the cost model, ranks plans by estimated rows and fixed constants. Component B, the executor, spends real time according to actual rows and real hardware. Because those inputs differ, the plan with the lowest cost is the planner's best guess, not a measured result.

### Events that change the choice

| Event | State before | State after | Effect on the choice |
|---|---|---|---|
| [`VACUUM`](../../../glossary.md#vacuum) | Old [`relpages`, `reltuples`](../../../glossary.md#reltuples-and-relpages) | New stored size and density ([vacuumlazy.c:341-363](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L341-L363)). Exception: a `VACUUM` that counted tuples on no page, as when the visibility map let it skip every page, writes the old `relpages` and `reltuples` back ([vacuumlazy.c:343-351](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L343-L351)) | Later row estimates start from the stored density, new unless that exception applied |
| `ANALYZE` | Old row counts and value statistics | New statistics | Row estimates and hash bucket and MCV estimates change ([selfuncs.c:3412](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3412)). See [the statistics sources page](query-planner-statistics-sources.md) |
| `CREATE STATISTICS`, then `ANALYZE` | Columns treated as independent | Dependencies known | Measured with parallelism off: outer estimate 1 → 9,400, nested loop → hash join |
| `CREATE INDEX` on the join key | No parameterized inner path | New inner and ordered paths | Parameterized nested loop and presorted merge become candidates ([optimizer/README:717-741](../../../../raw/postgres-12/src/backend/optimizer/README#L717-L741)) |
| `SET work_mem` | Old budget | New budget | Measured: skew plan moved from merge to hash at 256 MB, and batch charges disappeared in `wide` |
| A new `LIMIT` | Cheapest total wins | `root->tuple_fraction` above 0 keeps fast-start paths, and each path's Limit is priced for the rows it reads | Fast-start plans gain ([planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [relnode.c:602](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L602), [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670)) |
| A [generic cached plan](../../../glossary.md#custom-and-generic-plan) | Planned with no parameter values ([plancache.c:866-868](../../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L866-L868)) | Reused for every value | The method stays fixed while the data each call touches can change. See [plan_cache_mode](plan-cache-mode.md) |
| Rows added with no `ANALYZE` | Planned table size follows the live block count at the old density; value statistics stay as stored ([heapam_handler.c:2083](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2083), [heapam_handler.c:2157](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2157)) | Execution sees real rows | Size-driven estimates keep up, so a plan made after the insert already sees a bigger table. Selectivities can lag. At execution, a hash join adapts its batches and buckets; a nested loop does not change |

### Settings that change the choice

All of these have `user` context, so `SET` works in a session or transaction with no reload or restart.

| Setting | Default | Effect | Apply scope |
|---|---|---|---|
| `enable_nestloop` | on | Off adds `disable_cost` | Session or transaction ([guc.c#enable_nestloop](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L964-L972)) |
| `enable_mergejoin` | on | Off skips merge paths, except `FULL JOIN` | Session or transaction ([guc.c#enable_mergejoin](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L974-L982)) |
| `enable_hashjoin` | on | Off skips hash paths, except `FULL JOIN` | Session or transaction ([guc.c#enable_hashjoin](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L984-L992)) |
| `enable_material` | on | Off drops optional Materialize nodes; a required one stays | Session or transaction ([guc.c#enable_material](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L954-L962), [costsize.c:3093-3111](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3093-L3111)) |
| `enable_parallel_hash` | on | Off drops the shared hash table | Session or transaction ([guc.c#enable_parallel_hash](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1034-L1042), [joinpath.c:1873-1884](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1873-L1884)) |
| `work_mem` | 4 MB | Sets batching and sort spill | Session or transaction ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2240)) |
| `max_parallel_workers_per_gather` | 2 | 0 stops the planner from allowing parallel plans ([planner.c:343](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L343)) | Session or transaction ([guc.c#max_parallel_workers_per_gather](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3008-L3016)) |
| `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_operator_cost` | 1, 4, 0.01, 0.0025 ([cost.h:24-28](../../../../raw/postgres-12/src/include/optimizer/cost.h#L24-L28)) | Change the relative prices in [the cost formulas](#the-cost-formulas), so they can change the ranking | Session or transaction ([guc.c:3207](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3207), [guc.c:3218](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3218), [guc.c:3229](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3229), [guc.c:3251](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3251)) |

Use the `enable_*` settings for diagnosis, as this page does. They apply to every join in the session, including the ones that were fine. Parallel Hash tries the combined `work_mem` of all participants first, and falls back to `work_mem` per participant if that would still need batches ([nodeHash.c:706-712](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L706-L712), [nodeHash.c:793-806](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L793-L806)). Parallel plans exist for all three methods, but not for `RIGHT` or `FULL` joins ([joinpath.c:1510-1524](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1510-L1524), [joinpath.c:1842-1859](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1842-L1859)). See [parallel query](../../../glossary.md#parallel-query).

### Build, catalog and extension boundaries

- **Catalog.** `oprcanmerge` and `oprcanhash` are `pg_operator` columns ([pg_operator.h:49-52](../../../../raw/postgres-12/src/include/catalog/pg_operator.h#L49-L52)). Their initial values live in `pg_operator.dat`, which [genbki.pl:1-12](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L1-L12) turns into the bootstrap data at build time. A user-defined operator gets them through `CREATE OPERATOR ... HASHES, MERGES` ([create_operator.sgml:29](../../../../raw/postgres-12/doc/src/sgml/ref/create_operator.sgml#L29)). The planner reads them through `op_mergejoinable()` and `op_hashjoinable()` ([lsyscache.c:1193](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1193), [lsyscache.c:1244](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1244)).
- **Operator families.** The two flags behave differently when the operator family is missing. The merge flag is only a hint: if the operator is in no B-tree operator family, the clause is simply not mergejoinable ([initsplan.c:2602-2606](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2602-L2606), [xoper.sgml:444-451](../../../../raw/postgres-12/doc/src/sgml/xoper.sgml#L444-L451)). The planner trusts the hash flag alone ([lsyscache.c:1260](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1260), [initsplan.c:2635-2637](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2635-L2637)). So a `HASHES` operator with no hash operator family still gets hash join paths, and the join then fails at run time with "could not find hash function for hash operator" ([nodeHash.c:556-558](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L556-L558), [xoper.sgml:357-362](../../../../raw/postgres-12/doc/src/sgml/xoper.sgml#L357-L362)).
- **Path structures.** `JoinPath` is shared; `NestPath` is an alias, and `MergePath` and `HashPath` extend it with sort keys, mark and restore flags, hash clauses and batch counts ([pathnodes.h#JoinPath](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L1482-L1569)). `JoinCostWorkspace` carries results from the initial cost to the final cost ([pathnodes.h#JoinCostWorkspace](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L2476-L2499)).
- **Plan creation.** `create_nestloop_plan()`, `create_mergejoin_plan()` and `create_hashjoin_plan()` turn the winning path into plan nodes ([createplan.c:3996](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L3996), [createplan.c:4072](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4072), [createplan.c:4379](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4379)).
- **Extensions.** `set_join_pathlist_hook` can add or remove join paths, and `join_search_hook` can replace the join search ([joinpath.c:321-326](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L321-L326), [allpaths.c:2807-2808](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2807-L2808)).
- **Foreign tables.** A [foreign data wrapper](../../../glossary.md#foreign-data-wrapper) can push a join to the remote server through `GetForeignJoinPaths`, as `postgres_fdw` does ([joinpath.c:310-319](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L310-L319), [postgres_fdw.c:555](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L555)).
- **EXPLAIN.** The labels come from `ExplainNode()`, and `show_hash_info()` and `show_sort_info()` print the batch and sort lines ([explain.c:1118-1126](../../../../raw/postgres-12/src/backend/commands/explain.c#L1118-L1126), [explain.c:2635](../../../../raw/postgres-12/src/backend/commands/explain.c#L2635), [explain.c:2553](../../../../raw/postgres-12/src/backend/commands/explain.c#L2553)).

Version 12 has no [Memoize](../../../glossary.md#memoize) node and no `hash_mem_multiplier`: a search of the pinned `src/` tree finds neither name. A serial hash join's budget here is plain `work_mem` ([nodeHash.c:704](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L704)), and only Parallel Hash can try more, as the previous section says.

### Tests

- `src/test/regress/sql/join.sql` forces methods with the `enable_*` settings and checks plans and results, for example at [join.sql:524-534](../../../../raw/postgres-12/src/test/regress/sql/join.sql#L524-L534).
- `src/test/regress/sql/join_hash.sql` covers batch counts and growth: the optimal, good and bad memory cases in serial, parallel-oblivious and parallel-aware forms ([join_hash.sql:90-92](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L90-L92), [join_hash.sql:138-140](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L138-L140), [join_hash.sql:186-188](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L186-L188)).
- No test in the tree compares the speed of the methods. The regression suites assert plan shape and output, not elapsed time. This page's script fills that gap for one machine only.

### Final causal summary

Statistics give estimated rows → the planner builds every legal nested loop, merge and hash path for each pair of inputs, in each legal direction → the cost formulas multiply for a plain nested loop, charge `N log N` for sorts, and add for a hash join → paths that lose on cost and offer no useful order are dropped → the cheapest plan, or the best fast-start plan under a `LIMIT`, runs → the executor spends real time on real rows, so a wrong estimate or an uncalibrated cost constant can make the chosen method slower than a rejected one → `ANALYZE`, `CREATE STATISTICS`, a new index or a new `work_mem` changes the inputs for the next planning cycle.

As a working rule on this evidence: a nested loop with an index is fastest for small lookups; a hash join is fastest for large unsorted equality joins at the default `work_mem`; a merge join is fastest in a one-process plan when the order already exists, though Parallel Hash can beat it with workers; when a `LIMIT` query needs that order, an ordered merge join or an indexed nested loop both finish in under 0.2 ms; and a plain nested loop is by far the slowest whenever both inputs are large, but it is the only option for non-equality joins.

## Measurement Script

### Usage

| Item | What to give |
|---|---|
| Purpose | Times all three join methods on the same fixtures. Backs every number under [Which is faster](#which-is-faster-measured-results), the charges under [Key interaction](#key-interaction-estimated-cost-is-not-elapsed-time), including the unique-side legs, the `sweep10000` plans in the [worked calculation](#worked-calculation-the-same-join-priced-three-ways), the batch and `disable_cost` arithmetic, the catalog counts and the `EXPLAIN`-only plans under [Which methods are possible](#which-methods-are-possible-at-all), the ratios in the [Main idea](#main-idea), the 1.89e10 nested loop breakdown, the `misest.hash` bucket line and the 63,245.02 one-batch plan cost. |
| Invocation | From the repository root, extract the script with the command below, then run `bash .wiki-runtime/tmp/jm12-run.sh [stage ...]`. |
| Stages | See [Stages](#stages). The default runs every stage from `build` through `stop`, except `start`. |
| Environment | `PORT12` (default 55448), `JOBS` (default 8, parallel make jobs), `REPS` (default 5, timed runs per leg; the `sweep` stage always uses 3). Keep `REPS` odd: the script takes the middle sorted value as the median, so an even count reports the lower of the two middle values. |
| Prerequisites | A C toolchain, `make`, `bison`, `flex` and `perl`; a git checkout runs Perl programs such as `genbki.pl` during the build. The script itself uses no Perl. The default `configure` options need `readline` or `libedit`, and `zlib`. No extensions. The cluster uses `--no-locale` and UTF8. The repository directory must be named `postgres-llm-wiki`, because the script refuses to run from any other root. |
| Output | Under `.wiki-runtime/tmp/jm12/out/`. Read `results.tsv` first: scenario, leg, join node, serial or parallel, top-node cost, median, minimum and maximum ms, buffers, hash batch line, sort methods, and the planning and execution times of the one `EXPLAIN ANALYZE` run. Also `*.plan`, `eligibility.txt`, `assertions.txt`, `platform.txt` and `relations.txt`. The `report` stage prints `platform.txt`, `relations.txt`, `results.tsv`, `eligibility.txt` and every `*.plan`, then only the pass and fail counts from `assertions.txt`; read that file for the assertion names. |
| Runtime | From an empty sandbox, 4 min 18 s to 4 min 26 s on the Linux host with `JOBS=16` (runs 6 to 11), and 5 min 23 s on the earlier Mac with the default `JOBS` (run 5). The measuring stages take about 4 min, so a re-run with the build present takes about 4 min. |
| Cleanup | The `stop` stage stops the server and checks teardown. The `clean` stage deletes the sandbox. Run `clean` yourself: the default run keeps the sandbox. |

Extract the script from this page, from the repository root. The command builds the Markdown fence at run time, so that it contains no literal fence itself:

```sh
P=wiki/v12/questions/query-planning/join-methods.md
fence=$(printf '\140\140\140')
mkdir -p .wiki-runtime/tmp
sed -n '/^### The script/,/^## Context Reviewed/p' "$P" \
  | sed -e "1,/^${fence}bash\$/d" -e "/^${fence}\$/,\$d" > .wiki-runtime/tmp/jm12-run.sh
```

### Stages

| Stage | What it does |
|---|---|
| `build` | Builds and installs 12.2 out of tree. Skipped when the build is present. |
| `init` | Runs `initdb`, starts the cluster, and records platform facts. Replaces any stopped data directory and clears the results. |
| `start` | Starts the existing cluster, if it is stopped. Not in the default order. |
| `fixture` | Drops and recreates the disposable tables, runs `VACUUM ANALYZE`, and warms the buffer pool: every table, and the leaf and internal pages of the indexes the timed legs use. |
| `lookup` | 100 outer rows against a 1,000,000-row indexed table. |
| `wide` | Whole-table join on two primary keys, at 4 MB and 256 MB. |
| `unindexed` | Whole-table join on an unindexed key, at both memory sizes. The nested loop is priced but not run. |
| `sweep` | 5,000, 10,000 and 20,000 rows per side, no index, all three methods. |
| `topn` | First ten rows in key order, then the same join with no `ORDER BY`. |
| `eligibility` | Inequality, `FULL JOIN`, `xid`, mixed-side equality and `LATERAL` plans, the must-fail statement, and the `pg_operator` counts. |
| `skew` | `EXPLAIN` only: one value in half the rows, at both memory sizes. |
| `misestimate` | A one-row estimate on 10,000 real rows, then the same query with `CREATE STATISTICS`. |
| `unique` | `EXPLAIN` only: the `wide` hash and merge joins on copies of the tables with non-unique indexes. |
| `report` | Prints all results and the assertion count. |
| `stop` | Stops the server and verifies teardown. |
| `clean` | Deletes the sandbox. Not in the default order. |

Every invocation checks the pin before its first stage other than `stop` or `clean`, and stops if the checkout has moved or has tracked changes. Each scenario stage needs a running server with the fixture loaded and warmed, and a re-run replaces that scenario's earlier rows. A restart empties the buffer pool, and only `fixture` warms it, so include `fixture` when re-running after a `stop`: `bash .wiki-runtime/tmp/jm12-run.sh start fixture wide report stop`. Without it, the "found every shared block in the buffer pool" assertions fail on the first legs to touch each table and index. `fixture` also runs a new `ANALYZE`, so sample-dependent costs can move slightly, as [Last run](#last-run) shows. Run the server stages in one invocation.

### Non-default settings in the measurement

| Setting or state | Value here | Default | Context and apply scope | Why the script needs it | Results that depend on it |
|---|---|---|---|---|---|
| `shared_buffers` | 1 GB, command line | Boot value 8 MB ([guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2162)); `initdb` writes up to 128 MB ([initdb.c:959-963](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L959-L963)) | `postmaster`: restart | Keeps all fixtures cached, so runs measure CPU and memory | All timings. At the default, part of the data would need reads through the operating system. No default leg ran. |
| `autovacuum` | off, command line | on ([guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433)) | `sighup`: reload | Stops background `ANALYZE` changing estimates between legs | The stable estimates. Autovacuum would not change the mechanism. |
| `max_parallel_workers_per_gather` | 0 by `SET` on every leg except the `auto` legs: the timed legs other than `auto`, the `skew` legs, and the plan-only `unique` and `unindexed` `nl-plan` legs. The `auto` legs, timed in `lookup`, `wide`, `unindexed` and `topn`, and the `eligibility` plans do not set it, so they plan at the default | 2 ([guc.c#max_parallel_workers_per_gather](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3008-L3016)) | `user`: session | Compares one process per method | Every forced column; the "with parallelism off" planner's picks (485 and 501 ms); all `misest` legs; every `skew` plan and cost; the `unique` costs; and the 78,873 and 95,087 costs under [Key interaction](#key-interaction-estimated-cost-is-not-elapsed-time). At 0, `standard_planner()` leaves `parallelModeOK` false, so no [partial path](../../../glossary.md#partial-path) is built ([planner.c:339-349](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L339-L349)). The `auto` legs show the default, which uses Parallel Hash Join for `wide` and `unindexed`. |
| `enable_nestloop`, `enable_mergejoin`, `enable_hashjoin` | Two of three off per forced leg, by `SET`; in `eligibility`, one or two off per plan, as each plan is labeled | on | `user`: session | Forces the method. The planner reads each one as described under [Which methods are possible](#which-methods-are-possible-at-all) | Every forced leg, and the `eligibility` plans. By default the planner picks only the cheapest method; the planner's-pick column shows that choice. |
| `work_mem` | 256 MB on legs so named, by `SET` | 4 MB | `user`: session | Compares batched and in-memory runs | Those legs. At the default, `wide` and `unindexed` hash joins used 16 batches, and the sorts in `unindexed` used disk. |
| `enable_seqscan`, and `enable_hashjoin`, `enable_mergejoin` | off for one warm-up statement list, by `SET` | on ([guc.c#enable_seqscan](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L892)) | `user`: session | Makes the warm-up read the indexes, including the internal pages that a walk of the leaf pages skips | None directly: it warms the buffer pool and affects no timed leg. |
| `statement_timeout`, `lock_timeout` | 5 min, 5 s, through `PGOPTIONS` | 0, 0 ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2395)) | `user`: session | Bounds every statement | None. |
| `--no-locale`, UTF8 | At `initdb` | Locale from the environment | Fixed at `initdb` | Removes locale effects | None: all join keys are integers or `xid`. |
| Correlated fixture columns in `me` | `c1 = c2 = c3` by construction | Independence is the planner's default assumption | Fixture state | Produces the misestimate on purpose | The `misest` legs. Without correlation the estimate would be right, and the planner would not choose this nested loop. |
| Half the `sk` rows hold one value | Fixture state | No such skew | Fixture state | Triggers the `disable_cost` branch | The `skew` legs. |
| `ua`, `ub` have no primary key, only non-unique indexes on `id` | Fixture state, copies of `ja`, `jb` | Not applicable: `ja` and `jb` are the measured pair | Fixture state | Turns off `inner_unique` while keeping the same pages, rows and ordered index paths | The `unique` legs: hash 89,998.01, merge 95,084.77. |

These runs compile no expressions, although the [`jit`](../../../glossary.md#jit-compilation) setting shows `on`, its default ([guc.c#jit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1869-L1876)). The cause is the default build: LLVM support is off unless `configure` gets `--with-llvm` ([configure.in:382-385](../../../../raw/postgres-12/configure.in#L382-L385)), so this install holds no JIT provider library. The mechanism: before loading the provider, the server checks that its library file exists, and when it does not, it disables JIT for the session ([jit.c:95-100](../../../../raw/postgres-12/src/backend/jit/jit.c#L95-L100)). Also, `CREATE STATISTICS` in the `misestimate` stage is dropped at the end of that stage.

### Last run

**Filed numbers: 2026-11-02, run 12, after the thirteenth review's fixes.** PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`, built with default `configure` options. The script recorded `x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0` from `version()`, `Linux x86_64` from `uname`, 22 CPUs, `block_size` 8192, `Maximum data alignment` 8, `shared_buffers` 1 GB and `work_mem` 4 MB. The CPU count comes from `getconf _NPROCESSORS_ONLN`, a variable POSIX does not specify, so another platform may record `unknown`. Three facts were read outside the script: the kernel is WSL2 (`6.18.54.1-microsoft-standard-WSL2`), the host has 31 GB of memory, and the sandbox, including temp files, sits on an ext4 file system. The script was extracted from this page, byte-identical to the run 5 text, and run with `JOBS=16` and no stage arguments from an empty sandbox: 4 min 24 s, 103 of 103 assertions, including that every leg of a scenario returned the same result and that every timed leg found its shared blocks in the buffer pool. The `stop` stage confirmed teardown, and `clean` then deleted the sandbox.

**Earlier runs.** Runs 6 to 11 ran on the same Linux host earlier on 2026-11-02, with the same script text, and each passed 103 of 103, in 4 min 24 s, 4 min 18 s, 4 min 26 s, 4 min 21 s, 4 min 20 s and 4 min 20 s. Runs 1 to 5 ran on another host, Darwin arm64, with Apple clang 21, 10 CPUs and 17 GB of memory. Runs 1 to 3, on 2026-10-30, passed 62, 62 and 67 assertions. Run 4, on 2026-10-31, used the new buffer check and passed 101 of 103. The two failures were `lookup.auto` with 8 blocks read and `misest.serial` with 9, each the first leg to probe its index. The warm-up had walked only the leaf pages, so it was changed to probe across the key range as well. Run 5, on 2026-10-31 with that text, passed 103 of 103 in 5 min 23 s. All twelve runs chose the same join nodes on every shared leg.

Run 11's medians on the same host, in ms, for comparison with the tables above:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.306 | 71.0 | 42.3 | 0.333 |
| `wide` | 804 | 230 | 398, or 434 at 256 MB | Parallel 161; serial 390 |
| `unindexed` | not run | 682, or 703 at 256 MB | 406, or 635 at 256 MB | Parallel 187; serial 479 |
| `topn` | 0.130 | 0.141 | 397 | 0.161 |
| `misest` | 6,069 | 696 | 93.3 | 6,193; fixed 93.6 |
| `sweep` 5,000 / 10,000 / 20,000 | 597 / 2,288 / 8,937 | 2.24 / 5.17 / 10.55 | 1.00 / 1.90 / 3.86 | not run |

Run 10's medians on the same host, in ms:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.305 | 69.2 | 41.9 | 0.376 |
| `wide` | 814 | 217 | 394, or 432 at 256 MB | Parallel 174; serial 394 |
| `unindexed` | not run | 718, or 721 at 256 MB | 435, or 634 at 256 MB | Parallel 174; serial 413 |
| `topn` | 0.110 | 0.165 | 374 | 0.170 |
| `misest` | 6,046 | 707 | 93.9 | 6,124; fixed 93.1 |
| `sweep` 5,000 / 10,000 / 20,000 | 587 / 2,368 / 9,098 | 2.20 / 4.48 / 9.65 | 1.04 / 1.86 / 3.63 | not run |

Run 9's medians on the same host, in ms:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.325 | 69.1 | 42.0 | 0.332 |
| `wide` | 791 | 226 | 412, or 471 at 256 MB | Parallel 168; serial 413 |
| `unindexed` | not run | 703, or 720 at 256 MB | 438, or 628 at 256 MB | Parallel 191; serial 441 |
| `topn` | 0.110 | 0.141 | 393 | 0.156 |
| `misest` | 5,850 | 662 | 91.8 | 6,311; fixed 90.2 |
| `sweep` 5,000 / 10,000 / 20,000 | 593 / 2,427 / 8,965 | 2.33 / 4.56 / 9.08 | 0.96 / 1.99 / 3.82 | not run |

Run 8's medians on the same host, in ms:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.270 | 71.6 | 43.1 | 0.279 |
| `wide` | 822 | 236 | 416, or 423 at 256 MB | Parallel 168; serial 422 |
| `unindexed` | not run | 713, or 792 at 256 MB | 485, or 643 at 256 MB | Parallel 187; serial 438 |
| `topn` | 0.105 | 0.132 | 396 | 0.165 |
| `misest` | 6,260 | 706 | 94.3 | 6,545; fixed 93.4 |
| `sweep` 5,000 / 10,000 / 20,000 | 585 / 2,411 / 9,137 | 2.08 / 4.38 / 12.78 | 0.95 / 1.92 / 3.74 | not run |

Run 7's medians on the same host, in ms:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.295 | 70.3 | 41.6 | 0.262 |
| `wide` | 776 | 222 | 384, or 408 at 256 MB | Parallel 173; serial 428 |
| `unindexed` | not run | 711, or 708 at 256 MB | 428, or 629 at 256 MB | Parallel 195; serial 408 |
| `topn` | 0.115 | 0.134 | 371 | 0.158 |
| `misest` | 5,826 | 675 | 93.6 | 6,258; fixed 90.2 |
| `sweep` 5,000 / 10,000 / 20,000 | 573 / 2,274 / 9,107 | 2.14 / 4.28 / 9.42 | 1.01 / 1.91 / 3.83 | not run |

Run 6's medians on the same host, in ms:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.284 | 63.4 | 39.2 | 0.313 |
| `wide` | 785 | 229 | 399, or 404 at 256 MB | Parallel 185; serial 399 |
| `unindexed` | not run | 706, or 713 at 256 MB | 401, or 640 at 256 MB | Parallel 172; serial 415 |
| `topn` | 0.112 | 0.151 | 398 | 0.169 |
| `misest` | 6,079 | 697 | 95.7 | 6,452; fixed 94.3 |
| `sweep` 5,000 / 10,000 / 20,000 | 683 / 2,386 / 9,128 | 2.18 / 4.25 / 9.69 | 1.05 / 1.83 / 3.72 | not run |

Run 5's medians on the earlier host, in ms:

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup` | 0.202 | 75.9 | 53.9 | 0.198 |
| `wide` | 810 | 240 | 615, or 493 at 256 MB | Parallel 191; serial 463 |
| `unindexed` | not run | 800, or 714 at 256 MB | 503, or 888 at 256 MB | Parallel 217; serial 556 |
| `topn` | 0.082 | 0.097 | 480 | 0.109 |
| `misest` | 4,567 | 817 | 137 | 4,569; fixed 138 |
| `sweep` 5,000 / 10,000 / 20,000 | 778 / 3,122 / 12,630 | 2.43 / 5.10 / 10.75 | 1.30 / 2.58 / 5.03 | not run |

The two hosts agree on every forced-method ranking except one: `unindexed` at 256 MB, where merge won on the Mac (714 against 888) and hash won on the Linux host (690 against 740 in run 12, 635 against 703 in run 11, 634 against 721 in run 10, 628 against 720 in run 9, 643 against 792 in run 8, 629 against 708 in run 7, 640 against 713 in run 6). `unindexed` hash at 256 MB had the higher median in all twelve runs (866, 803, 826, 907, 888, 640, 629, 643, 628, 634, 635 and 690 ms against 486, 495, 498, 487, 503, 401, 428, 485, 438, 435, 406 and 489 ms).

Estimated costs that depend on `ANALYZE`'s sample move slightly between runs: `wide` merge was 95,086, 95,083, 95,086, 95,084, 95,086, 95,085, 95,084, 95,086, 95,083, 95,086, 95,078 and 95,087, the merge `topn` limit cost was 4.13, 1.86, 3.25, 2.75, 2.94, 3.68, 2.79, 1.79, 2.65, 2.40, 3.82 and 2.48, the `skew` plan costs moved by up to 4%, and the outer estimate after `CREATE STATISTICS` was 9,367 in run 5, 9,000 in run 6, 9,967 in run 7, 10,733 in run 8, 10,667 in run 9, 9,133 in run 10, 9,533 in run 11 and 9,400 in run 12. The `wide` hash, `sweep` and batch costs were unchanged, and the `skew` hash gap stayed at 10,000,015,628 to within 0.01. The tables above use run 12.

**Re-run check.** After run 5, the sandbox was re-run with `start fixture lookup unique report stop`, which confirmed that the stages can repeat: `results.tsv` kept 44 rows and `assertions.txt` kept 103 passing lines. The `unique` hash cost was unchanged and the merge cost moved to 95,084.57, because `fixture` runs a new `ANALYZE`. Those medians are not filed.

### The script

```bash
#!/usr/bin/env bash
#
# jm12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/query-planning/join-methods.md
#
# Builds the pinned PostgreSQL 12 checkout out of tree with default configure
# options, starts one isolated cluster, and runs the same join statements with
# each join method forced in turn, so the three methods are compared on the
# same data, in the same server, with a warm buffer pool.
#
# Every role, table and database is disposable and lives only in this sandbox.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/jm12-run.sh [stage ...]
# Stages: build init start fixture lookup wide unindexed sweep topn eligibility
#         skew misestimate unique report stop clean
# Default: every stage from build through stop, except start.
# start and clean are never run by default.

set -uo pipefail
set -e

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/jm12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55448}"
JOBS="${JOBS:-8}"
REPS="${REPS:-5}"

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; BIN="$INST/bin"
OUT="$SANDBOX/out"; SOCK="$SANDBOX/sock"; DATA="$SANDBOX/data"
RESULTS="$OUT/results.tsv"; ASSERTS="$OUT/assertions.txt"
export PGPORT="$PORT12" PGHOST="$SOCK" PGUSER=postgres PGDATABASE=jm
# Session-scoped bounds for every statement this script sends (user context).
export PGOPTIONS="-c statement_timeout=5min -c lock_timeout=5s"

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

own_sandbox() {
  case "$WIKI_ROOT" in
    */postgres-llm-wiki) ;;
    *) die "run from the postgres-llm-wiki repository root" ;;
  esac
  [ -f "$WIKI_ROOT/AGENTS.md" ] || die "repository root not found"
  [ -f "$SANDBOX/.jm12-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.jm12-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
  export TMPDIR="$SANDBOX/tmp"
  mkdir -p "$TMPDIR" "$OUT" "$SOCK"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] \
    || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git --no-optional-locks -C "$SRC12" status --porcelain --untracked-files=no)" ] \
    || die "PostgreSQL 12 checkout has tracked changes"
}

start_server() {
  # shared_buffers (postmaster, restart): 1 GB holds every fixture, so the
  # timed runs read no table block from the operating system.
  # autovacuum (sighup, reload): off, so no background ANALYZE moves the
  # estimates between legs. The fixture runs VACUUM ANALYZE itself.
  # listen_addresses and unix_socket_directories (postmaster): socket only,
  # inside the sandbox. port (postmaster): a non-default port.
  "$BIN/pg_ctl" -D "$DATA" -w -t 60 -l "$OUT/server.log" -o \
    "-c port=$PORT12 -c listen_addresses='' -c unix_socket_directories='$SOCK' -c shared_buffers=1GB -c autovacuum=off" \
    start > /dev/null
}
clear_scenario() { # clear_scenario <name>: a re-run replaces that scenario's rows
  local tab
  tab="$(printf '\t')"
  grep -v "^$1$tab" "$RESULTS" > "$RESULTS.new" || true
  mv "$RESULTS.new" "$RESULTS"
  grep -v -E "^(PASS|FAIL) $1\." "$ASSERTS" > "$ASSERTS.new" || true
  mv "$ASSERTS.new" "$ASSERTS"
  rm -f "$OUT/$1.reference"
}

sql() { # sql <text>: run with unaligned output, stop on the first error
  printf '%s\n' "$1" | "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1
}
assert() { # assert <name> <condition result 0|1>
  if [ "$2" = 0 ]; then printf 'PASS %s\n' "$1" >> "$ASSERTS"
  else printf 'FAIL %s\n' "$1" >> "$ASSERTS"; note "FAIL $1"; fi
}
chk() { # chk <name> <command ...>: a failed command is a failed assertion
  local name="$1"; shift
  if "$@"; then assert "$name" 0; else assert "$name" 1; fi
}

method_sets() {
  case "$1" in
    auto)   : ;;
    serial) printf 'SET /* wiki_jm12 */ max_parallel_workers_per_gather = 0;\n' ;;
    nl)     printf 'SET /* wiki_jm12 */ max_parallel_workers_per_gather = 0;\nSET /* wiki_jm12 */ enable_mergejoin = off;\nSET /* wiki_jm12 */ enable_hashjoin = off;\n' ;;
    merge)  printf 'SET /* wiki_jm12 */ max_parallel_workers_per_gather = 0;\nSET /* wiki_jm12 */ enable_nestloop = off;\nSET /* wiki_jm12 */ enable_hashjoin = off;\n' ;;
    hash)   printf 'SET /* wiki_jm12 */ max_parallel_workers_per_gather = 0;\nSET /* wiki_jm12 */ enable_nestloop = off;\nSET /* wiki_jm12 */ enable_mergejoin = off;\n' ;;
    *) die "unknown method $1" ;;
  esac
}
join_node() { # join_node <plan file>: the first join node in the plan text
  grep -E 'Nested Loop|Hash( [A-Z][a-z]+)? Join|Merge( [A-Z][a-z]+)? Join' "$1" | head -n 1 \
    | sed -E -e 's/^ *(-> *)?//' -e 's/ +\(cost=.*$//'
}
no_shared_read() { # no_shared_read <plan file>: true when no node read a shared block from the OS
  ! grep -E -q 'shared (hit=[0-9]+ )?read=' "$1"
}

# leg <scenario> <label> <method> <extra SET text> <SELECT text> [plan]
# Runs EXPLAIN once (ANALYZE, BUFFERS, TIMING OFF unless "plan" is given), then
# one discarded warm-up and REPS timed executions of the plain statement.
leg() {
  local scen="$1" label="$2" method="$3" extra="$4" stmt="$5" mode="${6:-run}"
  local sets tagged plan times med lo hi cost node par buf hb sm res pt et
  sets="$(method_sets "$method")$extra"
  tagged="$(printf '%s' "$stmt" | sed 's#^SELECT#SELECT /* wiki_jm12 */#')"
  plan="$OUT/$scen.$label.plan"; res="$OUT/$scen.$label.result"
  if [ "$mode" = plan ]; then
    sql "$sets
EXPLAIN /* wiki_jm12 */ $stmt;" > "$plan"
    med=-; lo=-; hi=-
  else
    sql "$sets
EXPLAIN /* wiki_jm12 */ (ANALYZE, BUFFERS, TIMING OFF) $stmt;" > "$plan"
    times="$TMPDIR/times.$$"
    { printf '%s\n\\o %s\n' "$sets" "$res"
      printf '%s;\n\\o /dev/null\n' "$tagged"
      printf '\\timing on\n'
      i=0; while [ "$i" -lt "$REPS" ]; do printf '%s;\n' "$tagged"; i=$((i + 1)); done
    } | "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 \
      | sed -E -n 's/^Time: ([0-9.]+) ms.*$/\1/p' | sort -n > "$times"
    [ "$(wc -l < "$times" | tr -d ' ')" = "$REPS" ] || die "$scen.$label: missing timings"
    med="$(sed -n "$(((REPS + 1) / 2))p" "$times")"
    lo="$(head -n 1 "$times")"; hi="$(tail -n 1 "$times")"
    rm -f "$times"
    if [ -f "$OUT/$scen.reference" ]; then
      chk "$scen.$label returns the same result as the first leg" cmp -s "$res" "$OUT/$scen.reference"
    else
      cp "$res" "$OUT/$scen.reference"
    fi
    # The EXPLAIN ANALYZE run comes before the timed runs, so a clean plan means
    # the timed runs found the same blocks in the buffer pool.
    chk "$scen.$label found every shared block in the buffer pool" no_shared_read "$plan"
  fi
  cost="$(sed -E -n '1s/^.*cost=[0-9.]+\.\.([0-9.]+) .*$/\1/p' "$plan")"
  node="$(join_node "$plan")"
  par=serial; if grep -q 'Gather' "$plan"; then par=parallel; fi
  buf="$(grep -E '^ *Buffers:' "$plan" | head -n 1 | sed -E 's/^ *Buffers: //' || true)"
  hb="$(grep -E 'Batches:' "$plan" | head -n 1 | sed -E 's/^ *//' || true)"
  sm="$(grep -E 'Sort Method:' "$plan" | sed -E 's/^ *Sort Method: //' | tr '\n' ';' || true)"
  pt="$(sed -E -n 's/^ *Planning Time: ([0-9.]+) ms$/\1/p' "$plan")"
  et="$(sed -E -n 's/^ *Execution Time: ([0-9.]+) ms$/\1/p' "$plan")"
  printf '%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\n' \
    "$scen" "$label" "$node" "$par" "$cost" "$med" "$lo" "$hi" "$buf" "$hb" "$sm" \
    "${pt:--}" "${et:--}" >> "$RESULTS"
  note "$scen $label: $node $par cost=$cost median=${med}ms"
  case "$method" in
    nl)    case "$node" in Nested*) r=0 ;; *) r=1 ;; esac; assert "$scen.$label is a nested loop" "$r" ;;
    merge) case "$node" in Merge*)  r=0 ;; *) r=1 ;; esac; assert "$scen.$label is a merge join" "$r" ;;
    hash)  case "$node" in Hash*)   r=0 ;; *) r=1 ;; esac; assert "$scen.$label is a hash join" "$r" ;;
  esac
}

stage_build() {
  say "build 12.2 out of tree (default configure options)"
  if [ ! -e "$SANDBOX" ]; then
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.jm12-owned"
  fi
  own_sandbox
  if [ -x "$BIN/postgres" ] && [ -x "$BIN/psql" ]; then
    note "build already present"; return 0
  fi
  mkdir -p "$BUILD"
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" > configure.log 2>&1 ) \
    || { tail -n 5 "$BUILD/configure.log" >&2; die "configure failed"; }
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { grep 'error:' "$BUILD/make.log" | head -n 3 >&2; die "make failed"; }
}

stage_init() {
  say "initdb and start on port $PORT12"
  own_sandbox
  [ ! -f "$DATA/postmaster.pid" ] || die "a cluster is already running on $DATA"
  rm -rf "$DATA"
  : > "$RESULTS"; : > "$ASSERTS"
  "$BIN/initdb" -D "$DATA" -U postgres -A trust -E UTF8 --no-locale > "$OUT/initdb.log"
  start_server
  PGDATABASE=postgres sql "CREATE /* wiki_jm12 */ DATABASE jm;"
  sql "SELECT /* wiki_jm12 */ version();" > "$OUT/platform.txt"
  sql "SELECT /* wiki_jm12 */ name || '=' || setting FROM pg_settings
       WHERE name IN ('block_size','work_mem','shared_buffers',
                      'max_parallel_workers_per_gather','random_page_cost','seq_page_cost',
                      'cpu_tuple_cost','cpu_operator_cost','jit','effective_cache_size')
       ORDER BY name;" >> "$OUT/platform.txt"
  "$BIN/pg_controldata" -D "$DATA" | grep 'Maximum data alignment' >> "$OUT/platform.txt"
  uname -sm >> "$OUT/platform.txt"
  printf 'cpus=%s\n' "$(getconf _NPROCESSORS_ONLN 2>/dev/null || echo unknown)" >> "$OUT/platform.txt"
}

stage_start() {
  say "start the existing cluster on port $PORT12"
  own_sandbox
  [ -d "$DATA" ] || die "no data directory; run init first"
  if [ -f "$DATA/postmaster.pid" ]; then note "already running"; else start_server; fi
}

stage_fixture() {
  say "disposable fixture tables"
  own_sandbox
  sql "DROP /* wiki_jm12_fixture */ TABLE IF EXISTS ja, jb, js, me, jd, sk, xa, xb, ua, ub,
       sa_5000, sb_5000, sa_10000, sb_10000, sa_20000, sb_20000;" 2> /dev/null
  sql "
CREATE /* wiki_jm12_fixture */ TABLE ja (id int, k int, v int, pad text);
INSERT /* wiki_jm12_fixture */ INTO ja
  SELECT i, (i::bigint * 7919 % 1000003)::int, i % 1000, repeat('a', 40)
  FROM generate_series(1, 1000000) i;
ALTER /* wiki_jm12_fixture */ TABLE ja ADD PRIMARY KEY (id);
CREATE /* wiki_jm12_fixture */ TABLE jb (id int, k int, v int, pad text);
INSERT /* wiki_jm12_fixture */ INTO jb
  SELECT i, (i::bigint * 104729 % 1000003)::int, i % 1000, repeat('b', 40)
  FROM generate_series(1, 1000000) i;
ALTER /* wiki_jm12_fixture */ TABLE jb ADD PRIMARY KEY (id);
CREATE /* wiki_jm12_fixture */ TABLE js (id int, bid int);
INSERT /* wiki_jm12_fixture */ INTO js SELECT i, i * 10000 FROM generate_series(1, 100) i;
CREATE /* wiki_jm12_fixture */ TABLE me (id int, c1 int, c2 int, c3 int, g int, h int);
INSERT /* wiki_jm12_fixture */ INTO me
  SELECT i, i % 100, i % 100, i % 100, i % 1000, i FROM generate_series(1, 1000000) i;
CREATE /* wiki_jm12_fixture */ TABLE jd (id int, g int, h int, v int, pad text);
INSERT /* wiki_jm12_fixture */ INTO jd
  SELECT i, i % 1000, i, i % 1000, repeat('d', 40) FROM generate_series(1, 1000000) i;
CREATE /* wiki_jm12_fixture */ INDEX jd_g ON jd (g);
CREATE /* wiki_jm12_fixture */ TABLE sk (id int, k int);
INSERT /* wiki_jm12_fixture */ INTO sk
  SELECT i, CASE WHEN i % 2 = 0 THEN 1 ELSE i END FROM generate_series(1, 1000000) i;
CREATE /* wiki_jm12_fixture */ TABLE xa (x xid);
INSERT /* wiki_jm12_fixture */ INTO xa SELECT i::text::xid FROM generate_series(1, 1000) i;
CREATE /* wiki_jm12_fixture */ TABLE xb (x xid);
INSERT /* wiki_jm12_fixture */ INTO xb SELECT i::text::xid FROM generate_series(1, 1000) i;
CREATE /* wiki_jm12_fixture */ TABLE ua AS SELECT * FROM ja;
CREATE /* wiki_jm12_fixture */ TABLE ub AS SELECT * FROM jb;
CREATE /* wiki_jm12_fixture */ INDEX ua_id ON ua (id);
CREATE /* wiki_jm12_fixture */ INDEX ub_id ON ub (id);
"
  for pair in 5000:5003 10000:10007 20000:20011; do
    n="${pair%:*}"; p="${pair#*:}"
    sql "
CREATE /* wiki_jm12_fixture */ TABLE sa_$n AS
  SELECT i AS id, (i::bigint * 7919 % $p)::int AS k, i AS v FROM generate_series(1, $n) i;
CREATE /* wiki_jm12_fixture */ TABLE sb_$n AS
  SELECT i AS id, (i::bigint * 104729 % $p)::int AS k, i AS v FROM generate_series(1, $n) i;
"
  done
  sql "VACUUM /* wiki_jm12_fixture */ ANALYZE;"
  sql "SELECT /* wiki_jm12 */ c.relname, c.relpages, c.reltuples::bigint
       FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
       WHERE n.nspname = 'public' ORDER BY 1;" > "$OUT/relations.txt"
  # Warm the buffer pool: every table once, plus the indexes the timed legs use.
  # enable_seqscan (user, this statement list only) forces the index reads that
  # bring those indexes in. A full index scan walks the leaf pages but descends
  # only once, so the nested loops probe across the key range to read the
  # internal pages too; enable_hashjoin and enable_mergejoin (user, same list)
  # force those probes. ua_id and ub_id serve only a plan leg.
  sql "
SET /* wiki_jm12 */ enable_seqscan = off;
SELECT /* wiki_jm12 */ count(id) FROM ja;
SELECT /* wiki_jm12 */ count(id) FROM jb;
SELECT /* wiki_jm12 */ count(g) FROM jd WHERE g >= 0;
SET /* wiki_jm12 */ enable_hashjoin = off;
SET /* wiki_jm12 */ enable_mergejoin = off;
SELECT /* wiki_jm12 */ count(*) FROM generate_series(1, 1000000, 100) s JOIN ja x ON x.id = s;
SELECT /* wiki_jm12 */ count(*) FROM generate_series(1, 1000000, 100) s JOIN jb x ON x.id = s;
SELECT /* wiki_jm12 */ count(*) FROM generate_series(0, 999) s JOIN jd x ON x.g = s;
" > /dev/null
  for t in ja jb js me jd sk xa xb ua ub sa_5000 sb_5000 sa_10000 sb_10000 sa_20000 sb_20000; do
    sql "SELECT /* wiki_jm12 */ count(*) FROM $t;" > /dev/null
  done
}

Q_LOOKUP="SELECT count(*), sum(b.v) FROM js a JOIN jb b ON a.bid = b.id"
Q_WIDE="SELECT count(*), sum(a.v + b.v) FROM ja a JOIN jb b ON a.id = b.id"
Q_UNIDX="SELECT count(*), sum(a.v + b.v) FROM ja a JOIN jb b ON a.k = b.k"
Q_TOPN="SELECT a.id, b.v FROM ja a JOIN jb b ON a.id = b.id ORDER BY a.id LIMIT 10"
Q_FIRST="SELECT a.id, b.v FROM ja a JOIN jb b ON a.id = b.id LIMIT 10"
Q_NOUNIQ="SELECT count(*), sum(a.v + b.v) FROM ua a JOIN ub b ON a.id = b.id"
Q_MIS="SELECT count(*), sum(d.v) FROM me m JOIN jd d ON m.g = d.g AND m.h = d.h WHERE m.c1 = 1 AND m.c2 = 1 AND m.c3 = 1"
BIGMEM="SET /* wiki_jm12 */ work_mem = '256MB';"

stage_lookup() {
  say "100 outer rows probing a 1,000,000-row indexed inner"
  own_sandbox
  clear_scenario lookup
  leg lookup auto auto "" "$Q_LOOKUP"
  leg lookup nl nl "" "$Q_LOOKUP"
  leg lookup merge merge "" "$Q_LOOKUP"
  leg lookup hash hash "" "$Q_LOOKUP"
}
stage_wide() {
  say "whole-table join on two primary keys"
  own_sandbox
  clear_scenario wide
  leg wide auto auto "" "$Q_WIDE"
  leg wide serial serial "" "$Q_WIDE"
  leg wide nl nl "" "$Q_WIDE"
  leg wide merge merge "" "$Q_WIDE"
  leg wide hash hash "" "$Q_WIDE"
  leg wide hash-256MB hash "$BIGMEM" "$Q_WIDE"
}
stage_unindexed() {
  say "whole-table join on an unindexed, unordered key"
  own_sandbox
  clear_scenario unindexed
  leg unindexed auto auto "" "$Q_UNIDX"
  leg unindexed serial serial "" "$Q_UNIDX"
  leg unindexed nl-plan nl "" "$Q_UNIDX" plan
  leg unindexed merge merge "" "$Q_UNIDX"
  leg unindexed hash hash "" "$Q_UNIDX"
  leg unindexed merge-256MB merge "$BIGMEM" "$Q_UNIDX"
  leg unindexed hash-256MB hash "$BIGMEM" "$Q_UNIDX"
}
stage_sweep() {
  say "growth with input size, no index"
  own_sandbox
  local saved="$REPS"; REPS=3
  for n in 5000 10000 20000; do
    clear_scenario "sweep$n"
    q="SELECT count(*), sum(a.v + b.v) FROM sa_$n a JOIN sb_$n b ON a.k = b.k"
    leg "sweep$n" nl nl "" "$q"
    leg "sweep$n" merge merge "" "$q"
    leg "sweep$n" hash hash "" "$q"
  done
  REPS="$saved"
}
stage_topn() {
  say "first ten rows in key order"
  own_sandbox
  clear_scenario topn
  clear_scenario firstrows
  leg topn auto auto "" "$Q_TOPN"
  leg topn nl nl "" "$Q_TOPN"
  leg topn merge merge "" "$Q_TOPN"
  leg topn hash hash "" "$Q_TOPN"
  chk "topn.hash sorts the hash join output" grep -q 'Sort Key' "$OUT/topn.hash.plan"
  # The same join with no ORDER BY: no Sort, so the LIMIT stops the probe early.
  leg firstrows hash hash "" "$Q_FIRST"
}
stage_eligibility() {
  say "which clauses each method accepts"
  own_sandbox
  local f="$OUT/eligibility.txt" rc p
  clear_scenario eligibility
  : > "$f"
  p="$OUT/elig.ineq.plan"
  sql "SET /* wiki_jm12 */ enable_nestloop = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM sa_5000 a JOIN sb_5000 b ON a.k < b.k;" > "$p"
  chk "eligibility.inequality join stays a nested loop with enable_nestloop off" grep -q 'Nested Loop' "$p"
  { echo '-- inequality, enable_nestloop = off'; cat "$p"; } >> "$f"

  p="$OUT/elig.fullineq.err"
  rc=0
  sql "SELECT /* wiki_jm12 */ count(*) FROM sa_5000 a FULL JOIN sb_5000 b ON a.k < b.k;" \
    > /dev/null 2> "$p" || rc=$?
  chk "eligibility.FULL JOIN on an inequality fails with psql status 3" [ "$rc" = 3 ]
  chk "eligibility.FULL JOIN on an inequality reports the source error text" \
    grep -q 'FULL JOIN is only supported with merge-joinable or hash-joinable join conditions' "$p"
  { echo '-- FULL JOIN on an inequality'; cat "$p"; } >> "$f"

  p="$OUT/elig.fulleq.plan"
  sql "SET /* wiki_jm12 */ enable_hashjoin = off;
SET /* wiki_jm12 */ enable_mergejoin = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM sa_5000 a FULL JOIN sb_5000 b ON a.k = b.k;" > "$p"
  chk "eligibility.equality FULL JOIN stays hash or merge with both disabled" \
    grep -E -q 'Hash Full Join|Merge Full Join' "$p"
  { echo '-- equality FULL JOIN, enable_hashjoin and enable_mergejoin off'; cat "$p"; } >> "$f"

  p="$OUT/elig.xid.plan"
  sql "SET /* wiki_jm12 */ enable_hashjoin = off;
SET /* wiki_jm12 */ enable_nestloop = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM xa a JOIN xb b ON a.x = b.x;" > "$p"
  chk "eligibility.xid equality cannot merge: hash and nestloop off still gives a nested loop" \
    grep -q 'Nested Loop' "$p"
  { echo '-- xid equality, enable_hashjoin and enable_nestloop off'; cat "$p"; } >> "$f"

  p="$OUT/elig.mixed.plan"
  sql "SET /* wiki_jm12 */ enable_nestloop = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM sa_5000 a JOIN sb_5000 b ON a.k = b.k + a.v;" > "$p"
  chk "eligibility.equality with outer columns on both sides stays a nested loop" \
    grep -q 'Nested Loop' "$p"
  { echo '-- equality that mixes the inputs on one side, enable_nestloop = off'; cat "$p"; } >> "$f"

  p="$OUT/elig.lateral.plan"
  sql "SET /* wiki_jm12 */ enable_nestloop = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM sa_5000 a,
  LATERAL (SELECT b.v FROM sb_5000 b WHERE b.k = a.k OFFSET 0) s;" > "$p"
  chk "eligibility.LATERAL subquery stays a nested loop" grep -q 'Nested Loop' "$p"
  { echo '-- LATERAL subquery with OFFSET 0, enable_nestloop = off'; cat "$p"; } >> "$f"

  { echo '-- equality operators by capability'
    sql "SELECT /* wiki_jm12 */ oprcanmerge, oprcanhash, count(*),
                string_agg(DISTINCT format_type(oprleft, NULL), ', ' ORDER BY format_type(oprleft, NULL))
                  FILTER (WHERE NOT (oprcanmerge AND oprcanhash) AND oprleft = oprright)
         FROM pg_operator WHERE oprname = '=' AND oprleft <> 0 AND oprright <> 0
         GROUP BY 1, 2 ORDER BY 1, 2;"
  } >> "$f"
}
stage_skew() {
  say "one value in half the inner rows"
  own_sandbox
  clear_scenario skew
  q="SELECT count(*) FROM sk a JOIN sk b ON a.k = b.k"
  leg skew serial serial "" "$q" plan
  leg skew hash hash "" "$q" plan
  leg skew nl nl "" "$q" plan
  leg skew merge merge "" "$q" plan
  leg skew hash-256MB hash "$BIGMEM" "$q" plan
  leg skew serial-256MB serial "$BIGMEM" "$q" plan
}
stage_misestimate() {
  say "a one-row estimate that is really 10,000 rows"
  own_sandbox
  clear_scenario misest
  sql "DROP /* wiki_jm12_fixture */ STATISTICS IF EXISTS me_dep;" 2> /dev/null
  leg misest serial serial "" "$Q_MIS"
  leg misest nl nl "" "$Q_MIS"
  leg misest merge merge "" "$Q_MIS"
  leg misest hash hash "" "$Q_MIS"
  sql "CREATE /* wiki_jm12_fixture */ STATISTICS me_dep (dependencies) ON c1, c2, c3 FROM me;
ANALYZE /* wiki_jm12_fixture */ me;"
  leg misest fixed serial "" "$Q_MIS"
  sql "DROP /* wiki_jm12_fixture */ STATISTICS me_dep;
ANALYZE /* wiki_jm12_fixture */ me;"
}
stage_unique() {
  say "the wide join on non-unique copies, plan only"
  own_sandbox
  clear_scenario unique
  leg unique hash hash "" "$Q_NOUNIQ" plan
  leg unique merge merge "" "$Q_NOUNIQ" plan
}
stage_report() {
  say "results"
  own_sandbox
  cat "$OUT/platform.txt"
  cat "$OUT/relations.txt"
  cat "$RESULTS"
  cat "$OUT/eligibility.txt"
  for p in "$OUT"/*.plan; do printf -- '-- %s\n' "${p##*/}"; cat "$p"; done
  printf 'assertions: %s passed, %s failed\n' \
    "$(grep -c '^PASS' "$ASSERTS" || true)" "$(grep -c '^FAIL' "$ASSERTS" || true)"
}
stage_stop() {
  say "stop the server and confirm teardown"
  own_sandbox
  if [ -f "$DATA/postmaster.pid" ]; then "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null; fi
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid still present"
  if pgrep -f -- "$DATA" > /dev/null 2>&1; then die "a process still names $DATA"; fi
  [ -z "$(ls -A "$SOCK" 2>/dev/null)" ] || die "socket directory is not empty"
  if "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT12"; then die "port $PORT12 still answers"; fi
  note "stopped; no postmaster.pid, no process, socket directory empty"
}
stage_clean() {
  say "delete the sandbox"
  own_sandbox
  [ ! -f "$DATA/postmaster.pid" ] || die "stop the server first"
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

if [ "$#" -eq 0 ]; then
  set -- build init fixture lookup wide unindexed sweep topn eligibility skew misestimate unique report stop
fi
pinned=no
for stage in "$@"; do
  case "$stage" in
    stop|clean) ;;
    *) if [ "$pinned" = no ]; then source_pin; pinned=yes; fi ;;
  esac
  case "$stage" in
    build|init|start|fixture|lookup|wide|unindexed|sweep|topn|eligibility|skew|misestimate|unique|report|stop|clean)
      "stage_$stage" ;;
    *) die "unknown stage: $stage" ;;
  esac
done
```

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/glossary.md`, the last 20 `wiki/log.md` entries and `wiki/v12/index.md`, as navigation.
- `wiki/v12/common-concepts/` does not exist, so this version has no concept page to link.
- Related v12 pages: [Planner Factors That Produce Plans That Over-Fetch Tuples](plans-that-over-fetch-tuples.md), [Query Planner Statistics Sources](query-planner-statistics-sources.md), [plan_cache_mode](plan-cache-mode.md).
- Planner: `joinpath.c` in full, the join sections of `costsize.c`, `add_path()`, `compare_path_costs_fuzzily()`, `create_limit_path()` and `adjust_limit_rows_costs()` in `pathnode.c`, `consider_startup` in `relnode.c`, `joinrels.c`, `allpaths.c`, `initsplan.c` including the join-list collapse limits, `createplan.c`, `planner.c`, `optimizer/README`, `pathnodes.h`, `cost.h`.
- Executor: `nodeNestloop.c`, `nodeHashjoin.c`, `nodeHash.c`, `nodeMergejoin.c`, `nodeMaterial.c`, `execAmi.c`, `hashjoin.h`, `jit.c`.
- Estimation, access and caches: `heapam_estimate_rel_size()` and `heapam_index_fetch_tuple()` in `heapam_handler.c`, `ReleaseAndReadBuffer()` in `bufmgr.c`, `build_join_pathkeys()` in `pathkeys.c`, the statistics update in `vacuumlazy.c`, `compute_semi_anti_join_factors()` and `cost_rescan()` in `costsize.c`, `eqjoinsel()` in `selfuncs.c`, `op_mergejoinable()` and `op_hashjoinable()` in `lsyscache.c`, `plancache.c`, `restrictinfo.c`, `explain.c`, `nodeSort.c`, `tuplesort.c`, `guc.c`.
- Catalog, build and docs: `pg_operator.h`, `pg_operator.dat`, `genbki.pl`, `configure.in`, `initdb.c`, `config.sgml`, `arch-dev.sgml`, `perform.sgml`, `xoper.sgml`, `ref/create_operator.sgml`, and `postgres_fdw.c`. Tests: `join.sql`, `join_hash.sql`.
- Fifth review, 2026-11-02: every source link was re-audited for file, range and label, and every cited range was re-read against its claim.
- Sixth review, 2026-11-02: `set_base_rel_consider_startup()` in `allpaths.c`, the `tuple_fraction` handling in `make_subplan()` (`subselect.c`) and in `prepunion.c`, and the merge join startup charges in `initial_cost_mergejoin()` and `final_cost_mergejoin()`.

## Evidence Map

| Claim | Evidence |
|---|---|
| Three methods and their mechanics | [optimizer/README:50-58](../../../../raw/postgres-12/src/backend/optimizer/README#L50-L58), [arch-dev.sgml:397-425](../../../../raw/postgres-12/doc/src/sgml/arch-dev.sgml#L397-L425), [nodeNestloop.c#ExecNestLoop](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L60-L256), [nodeMergejoin.c#ExecMergeJoin](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L600-L1429), [nodeHashjoin.c#ExecHashJoinImpl](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L164-L576) |
| Candidate generation and legality | [joinpath.c#add_paths_to_joinrel](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L116-L327) |
| Operator flags: merge flag is a planner hint, hash flag is trusted until run time | [initsplan.c:2602-2606](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2602-L2606), [lsyscache.c:1260](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1260), [nodeHash.c:556-558](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L556-L558), [xoper.sgml:357-362](../../../../raw/postgres-12/doc/src/sgml/xoper.sgml#L357-L362) |
| Cost formulas | [costsize.c#initial_cost_nestloop](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2415-L2480), [costsize.c#final_cost_nestloop](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2491-L2665), [costsize.c#initial_cost_mergejoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2698-L2898), [costsize.c#final_cost_mergejoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2927-L3171), [costsize.c#initial_cost_hashjoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3255-L3350), [costsize.c#final_cost_hashjoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3363-L3608); reproduced exactly by the worked calculation |
| Path comparison, including `consider_startup` and `consider_param_startup` | [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L166-L215), [relnode.c:201-202](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L201-L202), [relnode.c:602-603](../../../../raw/postgres-12/src/backend/optimizer/util/relnode.c#L602-L603), [allpaths.c:250-278](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L250-L278) |
| `LIMIT`, cursor, sub-select and `UNION ALL` routes to a fractional choice | [subselect.c:208-233](../../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L208-L233), [prepjointree.c:390-479](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L390-L479), [planner.c:399-403](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L399-L403), [prepunion.c:271-280](../../../../raw/postgres-12/src/backend/optimizer/prep/prepunion.c#L271-L280), [planner.c:377-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L411), [planner.c:1840-1855](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L1840-L1855), [planner.c:2306-2313](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L2306-L2313), [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670) |
| Unique-inner merge effects | [costsize.c:2995-3000](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2995-L3000), [costsize.c:3080-3136](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3080-L3136) |
| Hash batching and growth | [nodeHash.c#ExecChooseHashTableSize](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L667-L845), [nodeHash.c#ExecHashIncreaseNumBatches](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L888-L906) |
| Unique-inner hash charges | [costsize.c:3519-3567](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3519-L3567), [selfuncs.c:2064](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L2064), [joinrels.c:710](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L710); the `unique` legs confirm the 11,125 hash gap and no merge gain |
| Which method is faster | This page's script: what the built server did on this fixture and host, not a general law |

## Open Questions

- **Cost and time disagree.** In `wide`, the planner ranks the hash join (78,873) below the merge join (95,087), while measured time ranks them the other way (485 and 515 ms on two legs of the same plan, against 233 ms; run 11 gave 390 and 398 against 230, run 10 gave 394 and 394 against 217, run 9 413 and 412 against 226, run 8 422 and 416 against 236, run 7 428 and 384 against 222, run 6 399 and 399 against 229, and run 5 on the earlier host 463 and 615 against 240). Both stand. This is the measured-versus-source disagreement `MANDATORY Measurement Script` asks to keep. The source shows how each cost is computed, but it does not establish which real operations account for the time gap. A CPU-level profile would be needed.
- **More memory did not reliably help.** `unindexed` hash at 256 MB took 690 ms (range 674 to 780) against 489 ms at 4 MB (range 484 to 522), while the source charges it less (74,370 against 89,998); every run on both hosts had the higher median at 256 MB. The `wide` hash join had overlapping ranges at the two sizes in runs 6 to 8 and 10 to 12, with the higher median at 256 MB in all of them except run 12 (463 against 515), was slower at 256 MB with no overlap in run 9 (471 against 412), and is inconclusive on the Mac, where 493 ms at 256 MB sits between two 4 MB legs of one plan at 463 and 615 ms. The merge join on the unordered key, whose sorts fit in memory at 256 MB, was also not reliably faster: 740 against 708 in run 12, 703 against 682 in run 11, 721 against 718 in run 10, 720 against 703 in run 9, 792 against 713 in run 8, 713 against 706 in run 6, but slightly faster in run 7 (708 against 711) and faster on the Mac (714 against 800). The cause is unknown. A plausible but unverified guess is memory-access locality in one large table, and it would need hardware counters to test.
- **Disk behavior was not measured.** Every table stayed in `shared_buffers`, and temp files landed in the page cache of each host: an SSD on the Mac, and an ext4 virtual disk under WSL2 on the Linux host. The ranking on cold data, on spinning disks, or with tables bigger than memory is unknown. `random_page_cost` models that case, but the script cannot drop the operating system cache without `sudo`, which `AGENTS.md` forbids.
- **The timing evidence covers two hosts.** The filed tables come from run 12 on one Linux x86_64 host. [Last run](#last-run) keeps the medians of runs 6 to 11 from the same host and of run 5 from a Darwin arm64 host; the two hosts disagree on one ranking. Other CPUs, memory sizes and row widths can change the ratios, and `wide` used narrow joined columns over short rows. Treat ratios as examples.
- **Default-parallel `misest` was not run.** That stage ran with parallelism off, so this page does not show whether the default setting would also choose the nested loop.
- **Merge input charge in `wide`.** The table under [Key interaction](#key-interaction-estimated-cost-is-not-elapsed-time) derives merge's input charge, 72,586.58, by subtracting the compare, output and aggregate charges from the total, taking the compare charge at its full-scan value of 5,000. The two index scans cost 36,293.43 each, 72,586.86 together, so 0.28 is unaccounted for; runs 6 to 11 left 1.41, 2.57, 1.21, 3.50, 0.71 and 9.36, and the gap moves with each `ANALYZE` sample. The source charges each input's run cost only for the fraction between its estimated start and end selectivities, and it counts the compared rows over the same fraction ([costsize.c:2805-2806](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2805-L2806), [costsize.c:3145-3150](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3145-L3150), [costsize.c:2838-2849](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2838-L2849), [costsize.c:2862-2875](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2862-L2875)), which come from `mergejoinscansel()` through `cached_scansel()` ([costsize.c:2761](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2761), [costsize.c:3199](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3199)). An end fraction slightly below 1 would lower both the input and the compare charges, so the gap belongs to the two together, not to the inputs alone. That would explain the gap, but the plans do not print those fractions, so this page does not confirm it.
- **Inferred worked-calculation inputs.** The merge scan fractions and the one-row bucket size were inferred from the totals, as noted there. They were not printed by the server.
- **Same plan, different time.** The `misest` `serial` and `nl` legs use the same plan and the same cost. They ran at 4,889 and 4,327 ms in run 1, 4,845 and 4,307 ms in run 2, 4,343 and 4,332 ms in run 3, 4,569 and 4,567 ms in run 5, 6,452 and 6,079 ms in run 6, with no overlap between the run 6 ranges, 6,258 and 5,826 ms in run 7, 6,545 and 6,260 ms in run 8, 6,311 and 5,850 ms in run 9, 6,124 and 6,046 ms in run 10, 6,193 and 6,069 ms in run 11, and 6,084 and 5,899 ms in run 12, with overlapping ranges. In run 5 the `wide` `serial` and `hash` legs, also one plan, ran at 463 and 615 ms, and `unindexed` at 556 and 503 ms; in run 6 they ran at 399 and 399 ms, and 415 and 401 ms; in run 7 at 428 and 384 ms, and 408 and 428 ms; in run 8 at 422 and 416 ms, and 438 and 485 ms; in run 9 at 413 and 412 ms, and 441 and 438 ms; in run 10 at 394 and 394 ms, and 413 and 435 ms; in run 11 at 390 and 398 ms, and 479 and 406 ms; in run 12 at 485 and 515 ms, and 501 and 489 ms, each pair with overlapping ranges. Only run 12 and run 6's `misest` pair have their ranges filed here. So these gaps are not a stable property of a plan. Every leg found its blocks in the buffer pool, so reads do not explain them, and their cause is unknown.
- **Is the one-row match estimate intended?** Calling `compute_semi_anti_join_factors()` for a unique inner side is deliberate: its header says a join type other than `JOIN_SEMI` or `JOIN_ANTI` is assumed to be `inner_unique` ([costsize.c:4140](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4140)). What stays open is the selectivity it gets. It asks `clauselist_selectivity()` for `JOIN_SEMI` ([costsize.c:4187-4191](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4187-L4191)), but `eqjoinsel()` switches on `sjinfo->jointype` ([selfuncs.c:2064](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L2064)), which `make_join_rel()` set to `JOIN_INNER` for a plain inner join ([joinrels.c:710](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L710)). So `outer_match_frac` receives the inner-join selectivity, although the function's comment describes the fraction of outer rows with a match ([costsize.c:4220-4225](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L4220-L4225)). The source shows what happens, measured here in `wide`, but not whether that routing is deliberate. This page does not measure other row counts or the nested loop's matching branch.
- **`topn` ranking.** `\timing` puts the nested loop first (0.153 against 0.188 ms in run 12, 0.130 against 0.141 in run 11, 0.110 against 0.165 in run 10, 0.110 against 0.141 in run 9, 0.105 against 0.132 in run 8, 0.115 against 0.134 in run 7, 0.112 against 0.151 in run 6, 0.082 against 0.097 in run 5), while one `EXPLAIN ANALYZE` run puts the merge join first (0.090 against 0.048 ms of execution time in run 12, 0.079 against 0.048 in run 11, 0.081 against 0.046 in run 10, 0.083 against 0.047 in run 9, 0.083 against 0.047 in run 8, 0.083 against 0.048 in run 7, 0.089 against 0.050 in run 6, 0.117 against 0.048 in run 5), and the merge join touched 8 buffers against 35. The difference between the two instruments is not explained. `EXPLAIN ANALYZE` adds per-row counting overhead, and `\timing` adds parsing and planning, but the script does not separate them for the timed runs.
- **Common concept pages.** Version 12 has none. A `join-methods` concept page could hold the mechanics for reuse, if wanted.

## Source References

- [joinpath.c#add_paths_to_joinrel](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L116-L327)
- [joinpath.c#match_unsorted_outer](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1330-L1550)
- [joinpath.c#hash_inner_and_outer](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1676-L1907)
- [joinrels.c#populate_joinrel_with_paths](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L754-L835)
- [costsize.c#initial_cost_nestloop](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2415-L2480)
- [costsize.c#final_cost_mergejoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2927-L3171)
- [costsize.c#initial_cost_hashjoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3255-L3350)
- [costsize.c#final_cost_hashjoin](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3363-L3608)
- [pathnode.c#add_path](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L423-L484)
- [pathnode.c#adjust_limit_rows_costs](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L3623-L3670)
- [planner.c:377-411](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L377-L411)
- [nodeNestloop.c#ExecNestLoop](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L60-L256)
- [nodeMergejoin.c:61-85](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L61-L85)
- [nodeHashjoin.c#ExecHashJoinImpl](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L164-L576)
- [nodeHash.c#ExecChooseHashTableSize](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L667-L845)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
