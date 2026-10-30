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
| Merge join | Walk two sorted inputs together | Both inputs are already sorted, or the output order is needed |
| Hash join | Load one side into a hash table, probe it with the other | Large unsorted inputs joined on equality |

On this page's measurements, the winner changed from case to case. A 100-row lookup ran about 300 times faster with a nested loop than with a hash join. An unindexed 20,000-row join ran about 2,500 times faster with a hash join than with a nested loop. See [the measured results](#which-is-faster-measured-results).

The [planner](../../../glossary.md#planner) does not choose by rule. It prices the legal methods with estimated row counts and keeps the cheapest ([optimizer/README:40-42](../../../../raw/postgres-12/src/backend/optimizer/README#L40-L42), [joinpath.c#add_paths_to_joinrel](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L116-L327)). Two results follow, and both surprise people:

- A wrong row estimate can pick a slow method. One measured query took about 4.3 to 4.9 seconds instead of 142 ms.
- The lowest [cost](../../../glossary.md#cost) is not always the shortest time. In one measured join with parallelism off, the planner preferred a hash join that ran nearly twice as long as the merge join it rejected.

### Mental model: the values that decide the choice

A join has an **outer** input and an **inner** input. EXPLAIN prints the outer input first. The planner tries each pair of inputs in both directions, so "outer" is the planner's choice, not the order you wrote in the query ([joinrels.c:785-790](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L785-L790)).

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Outer rows | Estimated rows from the outer input | `Path.rows`, built from statistics | Stored estimate; stale when the data changed after the last `ANALYZE`, and wrong when columns are correlated | Multiplies the nested loop's inner cost ([costsize.c:2444-2445](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2444-L2445)) |
| Inner rows | Estimated rows from the inner input | Same | Same | Sets hash table size and sort cost ([costsize.c:3312-3320](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3312-L3320)) |
| Parameterized inner path | An inner index scan that takes the outer row's value as its key | Generated when an index matches the join clause ([optimizer/README:717-741](../../../../raw/postgres-12/src/backend/optimizer/README#L717-L741)) | Plan-time option | Lets a nested loop skip most of the inner side |
| Sort order ([pathkeys](../../../glossary.md#pathkey)) | The order a path already delivers | Index scans and earlier merge or nested loop joins | Plan-time property | Lets a merge join skip a sort ([joinpath.c:605-610](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L605-L610)) |
| [`work_mem`](../../../glossary.md#work_mem) | Memory budget for each hash table or sort; default 4 MB | GUC ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2240)) | Live setting, read at planning and again at execution | Decides whether a hash join needs batches and a sort goes to disk ([nodeHash.c:704](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L704), [nodeHash.c:785](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L785)) |
| Most common value frequency | Share of the inner rows holding the commonest key | [`pg_statistic`](../../../glossary.md#most-common-values-and-histogram), through `estimate_hash_bucket_stats()` ([selfuncs.c#estimate_hash_bucket_stats](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3412)) | Stored | Can price a hash join out of the running ([costsize.c:3503-3506](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3503-L3506)) |
| Cost constants | `seq_page_cost` 1, `random_page_cost` 4, `cpu_tuple_cost` 0.01, `cpu_operator_cost` 0.0025 | [cost.h:24-28](../../../../raw/postgres-12/src/include/optimizer/cost.h#L24-L28) | Live settings | Turn row and page counts into one cost number |

Cost is not time. Its units are "an arbitrary scale", where only relative values matter ([config.sgml#planner-cost-constants](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4670-L4681)).

### Logic map: how a join method is chosen

```text
Two inputs to join (repeated for each pair, in both directions)
│
├─ Is there a usable equality on a mergejoinable operator?      [1]
│    ├─ yes → sort both inputs, or use inputs already sorted → MERGE paths
│    └─ no  → no merge path (except FULL JOIN ON true)          [2]
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
├─ dominated by an existing path on cost and sort order → dropped [6]
└─ otherwise kept
│
▼
Kept paths feed the next join level                              [7]
│
▼
Final choice: cheapest total, or best for the fraction of rows
a LIMIT or cursor needs                                          [8]
```

Key to the citations:

1. [joinpath.c:198-205](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L198-L205), [joinpath.c:268-270](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L268-L270), [joinpath.c#select_mergejoin_clauses](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1931-L2029)
2. [joinpath.c:1115-1127](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1115-L1127)
3. [joinpath.c:1345-1378](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1345-L1378), [joinpath.c:1461-1493](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1461-L1493)
4. [joinpath.c:306-308](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L306-L308), [joinpath.c:1696-1722](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1696-L1722)
5. [joinpath.c:413-428](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L413-L428)
6. [pathnode.c#add_path](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L423-L484), [optimizer/README:425-430](../../../../raw/postgres-12/src/backend/optimizer/README#L425-L430)
7. [allpaths.c#standard_join_search](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2846-L2881)
8. [planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5885-L5910)

### Nested loop join

**How it works.** `ExecNestLoop()` takes one outer row, copies any values the inner scan needs into parameters, rescans the inner plan, and tests each inner row. When the inner plan runs out, it takes the next outer row ([nodeNestloop.c#ExecNestLoop](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L100-L202)).

It comes in two very different forms:

| Form | What each outer row does | Work grows as |
|---|---|---|
| Parameterized inner | Runs an index search for its own key ([optimizer/README:717-741](../../../../raw/postgres-12/src/backend/optimizer/README#L717-L741)) | Outer rows × one index search |
| Plain inner | Reads every inner row and filters ([costsize.c:2465-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2465-L2471)) | Outer rows × inner rows |

The README calls the parameterized form "the only plan type that can avoid fetching all of B" ([optimizer/README:738-741](../../../../raw/postgres-12/src/backend/optimizer/README#L738-L741)). A merge join can stop early, as the next section shows, but only at the ends of the key range.

The planner may put a [Materialize node](../../../glossary.md#materialize-node) on a plain inner side. The inner rows are then read once and replayed from memory, which costs only `cpu_operator_cost` per row on each rescan ([joinpath.c:1403-1412](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1403-L1412), [costsize.c:3774-3785](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3774-L3785)). This shrinks the constant, but the row-pair count stays the same.

**Strengths.** It starts returning rows at once, needs no sort and no hash table, and accepts any join condition, or none. It preserves the outer input's order, so it can feed an `ORDER BY` without a sort ([optimizer/README:596-605](../../../../raw/postgres-12/src/backend/optimizer/README#L596-L605)).

**Weakness.** If the outer row count is higher than estimated, every extra row pays for another inner scan. Nothing corrects this during execution.

### Merge join

**How it works.** Both inputs arrive sorted on the join key. The executor compares the two current rows and advances whichever is lower until they match. It then joins the matching rows ([nodeMergejoin.c:61-85](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L61-L85)).

Duplicates are the awkward case. If the outer side repeats a key, the second outer row must see the same inner rows again. The executor marks the first matching inner row and restores to it ([nodeMergejoin.c:51-58](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L51-L58), [nodeMergejoin.c:1053-1078](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L1053-L1078)). So the inner input must support mark and restore. If it cannot, or if a Material node looks cheaper, the planner puts one above it ([costsize.c:3086-3111](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3086-L3111), [createplan.c:4187-4203](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4187-L4203)). Mark and restore is skipped when each outer row needs only one match and every join clause is a merge clause ([costsize.c:2989-3002](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2989-L3002)).

The sorted order can come from two places ([joinpath.c:1068-1080](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1068-L1080)):

- An explicit Sort node, which must read its whole input before the first row comes out ([nodeSort.c#ExecSort](../../../../raw/postgres-12/src/backend/executor/nodeSort.c#L40-L118)).
- An index scan or another path that already delivers the order, with no sort step.

That difference explains most of merge join's behavior. With sorted inputs it is cheap and starts at once. With unsorted inputs it pays for two sorts.

**Strengths.** Its output is ordered on the join key, so a later merge or `ORDER BY` can reuse that order ([joinpath.c:1027-1029](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1027-L1029)). It stops when one input is exhausted, and the cost model credits that early stop ([costsize.c:2726-2736](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2726-L2736), [perform.sgml:878-882](../../../../raw/postgres-12/doc/src/sgml/perform.sgml#L878-L882)).

**Weakness.** It needs a sort on anything not already ordered. Heavy duplicates on both sides mean repeated re-reads of inner rows, which the cost model charges through `rescannedtuples` ([costsize.c:3010-3051](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3010-L3051)).

### Hash join

**How it works.** The join has two phases:

1. **Build.** The Hash node reads the entire inner input into a hash table on the join key ([nodeHash.c#MultiExecPrivateHash](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L139-L175)). No row can be returned until this finishes.
2. **Probe.** Each outer row hashes its key, finds its bucket and checks that bucket's rows ([nodeHashjoin.c:345-440](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L345-L440)).

Because build reads one side fully and probing visits a short bucket, the work grows with the two input sizes added together, not multiplied. The target is one row per bucket ([nodeHash.c:664-665](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L664-L665)). The planner normally ends up hashing the smaller side, because that direction prices lower: the build charge is per inner row, and the probe charge per outer row is smaller ([costsize.c:3289-3291](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3289-L3291)).

**When the table does not fit `work_mem`.** The join splits into batches:

| Stage | What happens | Source |
|---|---|---|
| Planning | `ExecChooseHashTableSize()` picks a power-of-two batch count | [nodeHash.c:785-838](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L785-L838) |
| Build | Batch 0's rows stay in memory; the other rows go to [temporary files](../../../glossary.md#temporary-file) | [nodeHash.c#ExecHashTableInsert](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1595-L1680) |
| Probe | Outer rows for a later batch go to that batch's file | [nodeHashjoin.c:386-411](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L386-L411) |
| Later batches | Each batch's table is loaded and probed in turn | [nodeHashjoin.c#ExecHashJoinNewBatch](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L947-L1000) |

A hash join also corrects itself during execution. If the table outgrows its budget, the batch count doubles, whatever the planner estimated ([nodeHash.c:1658-1661](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1658-L1661), [nodeHash.c:906](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L906)). In the single-batch case, the bucket count grows too ([nodeHash.c:1640-1652](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1640-L1652)). The measured `misest.hash` plan shows this: `Buckets: 16384 (originally 1024)`.

**Exceptions.**

- Identical keys cannot be split across batches. If a split moves all or none of the rows, the executor stops adding batches and lets memory exceed the budget ([nodeHash.c:1036-1045](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L1036-L1045)).
- Inner rows matching the outer side's most common values go to a small skew table, up to 2% of the budget, which never goes to disk ([hashjoin.h:84-92](../../../../raw/postgres-12/src/include/executor/hashjoin.h#L84-L92), [hashjoin.h:110](../../../../raw/postgres-12/src/include/executor/hashjoin.h#L110)).
- An empty outer side can end the join before the build, when the outer side looks cheap to start. This does not apply to a right join, a full join or Parallel Hash ([nodeHashjoin.c:241-272](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L241-L272)).

**Strengths.** It handles large unsorted inputs well, needs no index, and has the run-time correction described above.

**Weakness.** It needs an equality condition, delays the first row until the build is done, and promises no output order. Batching can reorder rows, so the planner marks every hash join path as having no sort order ([optimizer/README:605-608](../../../../raw/postgres-12/src/backend/optimizer/README#L605-L608)).

### Side-by-side comparison

| Property | Nested loop | Merge join | Hash join |
|---|---|---|---|
| Join condition | Any, or none | Mergejoinable equality | Hashjoinable equality |
| Index needed | Not required; decisive when present | Optional; replaces a sort | Not used on the join key |
| Inner side read | Once per outer row, or by index search | Once, plus re-reads for duplicates | Once, entirely |
| Memory | None for the join itself; Materialize uses up to `work_mem` | Sorts use up to `work_mem` each | Hash table uses up to `work_mem`, unless identical keys force more |
| Spills to temp files | Only through Materialize | Through its sorts | Through batches |
| First row | Immediate, if the inputs start fast | Immediate if presorted, after the sorts if not | After the whole build |
| Output order | Keeps the outer order | Sorted on the join key | None promised |
| `RIGHT`, `FULL` | Not supported | Supported if every join clause, constants aside, is a merge clause | Supported |
| Plan-time row error | Hurts most | A sort or re-read costs more | Corrected by adding batches and buckets |

Sources for the join-type rows: [joinpath.c:1345-1378](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1345-L1378), [joinpath.c:2017-2026](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L2017-L2026), [nodeHashjoin.c:527-551](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L527-L551). A nested loop does not lose a `RIGHT JOIN` outright: the planner also tries that join with the sides swapped, as a `LEFT` join ([joinrels.c:802-807](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L802-L807)).

### Why one gets chosen over another

The planner follows these steps. Each output feeds the next.

1. **Order the joins.** `standard_join_search()` builds join relations level by level, two inputs at a time ([allpaths.c#standard_join_search](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2846-L2881)). At `geqo_threshold` (default 12) or more relations, [GEQO](../../../glossary.md#geqo) takes over when it is enabled ([allpaths.c:2807-2812](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2807-L2812), [guc.c#geqo_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2027-L2029)). See [join order search](../../../glossary.md#join-order-search).
2. **Try both directions.** For each pair, `populate_joinrel_with_paths()` calls `add_paths_to_joinrel()` with each input as outer ([joinrels.c:785-790](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L785-L790)).
3. **Generate candidates.** That function builds merge paths, then nested loop and presorted merge paths, then hash paths ([joinpath.c:264-308](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L264-L308)).
4. **Discard cheaply.** A quick lower-bound cost rejects hopeless paths before their full cost is computed ([joinpath.c:413-428](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L413-L428)).
5. **Compare on three things.** `add_path()` compares total cost and startup cost with a 1% tolerance, then sort order ([pathnode.c#STD_FUZZ_FACTOR](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L48-L52), [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L166-L215), [pathnode.c:477-484](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L477-L484)).
6. **Keep useful losers.** A path survives if it is the only way to get a sort order, even when it costs more overall ([optimizer/README:425-430](../../../../raw/postgres-12/src/backend/optimizer/README#L425-L430)). So a dearer merge join can stay in the running because the next level wants its order.
7. **Pick the final plan.** A query that reads everything takes the cheapest total cost. A `LIMIT` or cursor uses the fraction of rows it needs ([planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5885-L5910)), which favors fast-start plans. See [tuple fraction](../../../glossary.md#tuple-fraction).

So the practical reasons a method wins are:

| Situation | Usually wins | Why, in the cost model |
|---|---|---|
| Few outer rows, indexed inner key | Nested loop | Few index searches cost less than one full inner scan ([costsize.c:2442-2471](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471)) |
| Large unsorted inputs, equality | Hash join | Costs add rather than multiply, and no sort is charged ([costsize.c:3274-3291](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3274-L3291)) |
| Inputs already sorted | Merge join, if it beats the hash join's cheaper scans | No sort charge ([costsize.c:2843-2850](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2843-L2850)) |
| `ORDER BY` with `LIMIT` on the key | Merge join or nested loop | Only they deliver the order without a full read ([planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5885-L5910)) |
| Non-equality condition | Nested loop | It is the only method available |
| One key holds too much of the inner side | Merge join or nested loop | The hash join receives `disable_cost` ([costsize.c:3495-3506](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3495-L3506)) |
| Inner side provably unique | Any; all get cheaper | The executor stops at the first match, and the cost model knows it ([joinpath.c:181-188](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L181-L188), [nodeNestloop.c:322](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L322), [nodeHashjoin.c:682](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L682)) |

The last-but-one row is a special case and needs its default stated. By default a hash join carries no penalty. The cause is a most common inner value whose rows alone would exceed `work_mem`. `final_cost_hashjoin()` tests that size and adds `disable_cost`, 1.0e10, to the startup cost, because batching cannot separate equal keys ([costsize.c:3495-3506](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3495-L3506), [costsize.c:120](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L120)). The path stays available, so it can still win if nothing else exists. The measured `sk` fixture shows it: the same hash join costs exactly 10,000,015,628.00 more at 4 MB than at 256 MB. Of that, 1.0e10 is the penalty and 15,628 is batch I/O ([`skew` legs](#which-is-faster-measured-results)).

### Which methods are possible at all

Cost only ranks what is legal. These conditions decide what is legal:

| Condition | Nested loop | Merge | Hash | Source |
|---|---|---|---|---|
| Equality, normal types | Yes | Yes | Yes | [initsplan.c#check_mergejoinable](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2582-L2607), [initsplan.c#check_hashjoinable](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2619-L2638) |
| `<`, `>`, `<>`, `LIKE` and similar | Yes | No | No | Same |
| No join condition (cross join) | Yes | No, except `FULL JOIN ON true` | No | [joinpath.c:1115-1127](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1115-L1127), [joinpath.c:1721-1722](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1721-L1722) |
| Volatile function in the clause | Yes | No | No | [initsplan.c:2598-2600](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2598-L2600), [initsplan.c:2635-2637](../../../../raw/postgres-12/src/backend/optimizer/plan/initsplan.c#L2635-L2637) |
| Operator is hash-only, such as `xid`, `cid`, `aclitem` | Yes | No | Yes | [pg_operator.dat:180](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L180) |
| Operator is merge-only, such as `bit`, `varbit`, `money`, `tsvector` | Yes | Yes | No | [pg_operator.dat:2082](../../../../raw/postgres-12/src/include/catalog/pg_operator.dat#L2082) |
| Operator has neither flag, such as `box`, `circle`, `line`, `lseg`, `path` | Yes | No | No | Measured from `pg_operator` |
| `FULL JOIN` whose condition is neither mergeable nor hashable, such as `a.k < b.k` | Not available | Not available | Not available; the statement fails with an error | [joinrels.c:825-833](../../../../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L825-L833) |

The measured catalog of this build has 61 `=` operators with typed operands on both sides: 40 support both, 12 only merge, 3 only hash, and 6 neither.

`enable_mergejoin = off` and `enable_hashjoin = off` do not forbid a method everywhere:

- By default all three settings are on.
- `enable_nestloop = off` keeps the paths and adds `disable_cost` ([costsize.c:2531-2532](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2531-L2532)).
- `enable_mergejoin = off` and `enable_hashjoin = off` skip generating those paths, but a `FULL JOIN` overrides that check, because it may have no alternative. Such a path then carries `disable_cost` ([joinpath.c:193-205](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L193-L205), [joinpath.c:301-308](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L301-L308), [costsize.c:2977-2978](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2977-L2978), [costsize.c:3407-3408](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3407-L3408)). A measured equality `FULL JOIN` with both off still ran as a `Hash Full Join`.
- Nested loops cannot be fully suppressed ([config.sgml#guc-enable-nestloop](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4512-L4525)). A measured `a.k < b.k` join with `enable_nestloop = off` stayed a nested loop, priced above 1.0e10, as did an `xid` equality with `enable_nestloop` and `enable_hashjoin` off.

### The cost formulas

Terms used below:

- `Nout`, `Nin`: estimated outer and inner rows.
- `Cout`, `Cin`: total costs of the outer and inner paths.
- `ops`: `cpu_operator_cost`, 0.0025.
- `tup`: `cpu_tuple_cost`, 0.01.
- `Rin`: the cost of one rescan of the inner path.
- `Nmatch`: estimated rows passing the join clause.

These are the common-case formulas for an inner join with one integer key, a single batch, no duplicates forcing re-reads, and inputs read to the end. Semi joins, anti joins and provably unique inner sides use refined formulas that assume an early stop ([costsize.c:2536-2644](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2536-L2644), [costsize.c:3519-3567](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3519-L3567)).

```text
Nested loop  = Cout + Cin + (Nout - 1) * Rin + Nout * Nin * (tup + ops)
Merge join   = sort(Cout, Nout) + sort(Cin, Nin) + ops * (Nout + Nin) + tup * Nmatch
sort(C, N)   = C + 2 * ops * N * log2(N) + ops * N
Hash join    = Cout + Cin + (ops + tup) * Nin + ops * Nout
               + ops * Nout * (rows per bucket) * 0.5 + tup * Nmatch
```

Sources: [costsize.c#initial_cost_nestloop](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2442-L2471), [costsize.c:2650-2657](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L2650-L2657), [costsize.c#cost_sort](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1693-L1790), [costsize.c:3140-3163](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3140-L3163), [costsize.c:3274-3291](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3274-L3291), [costsize.c:3570-3600](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3570-L3600).

In practical terms:

- The nested loop has a multiplication, `Nout * Nin`. With a parameterized index inner, the filtered row count per probe is small, so `Rin` carries the cost instead and that term shrinks.
- The merge join's sort has `N log N`. A presorted input removes `sort()` altogether and charges only the path's own cost.
- The hash join has only additions when buckets hold about one row.

A hash join that needs batches pays extra sequential I/O: `seq_page_cost * (2 * innerpages + 2 * outerpages)`, where the pages are estimated from `(MAXALIGN(width) + 24)` bytes per row ([costsize.c:3329-3338](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3329-L3338), [costsize.c#relation_byte_size](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L5461-L5475)). In the measured `wide` case each side is 1,000,000 rows at width 8, so `ceil(32,000,000 / 8192)` = 3,907 pages per side. The charge is 4 × 3,907 = 15,628, which is the exact gap between the 78,873.02 and 63,245.02 plans.

### Worked calculation: the same join priced three ways

Inputs, from the measured `sweep10000` plans: two tables of 10,000 rows and 55 pages, joined on an unindexed, unordered integer key. Each sequential scan costs 55 + 10,000 × 0.01 = 155.

| Step | Nested loop | Merge join | Hash join |
|---|---|---|---|
| Read outer | 155.00 | 155.00, inside the sort | 155.00 |
| Read inner | 205.00 (155 + 2 × 0.0025 × 10,000 for Materialize) | 155.00, inside the sort | 155.00 |
| Rescans | 9,999 × 25 = 249,975.00 | none | none |
| Sort, two inputs | none | 2 × (0.005 × 10,000 × 13.2877 + 25) = 1,378.77 | none |
| Build | none | none | 12.5 × 10 = 125.00 |
| Probe or merge compare | 1.0e8 pairs × 0.0125 = 1,250,000.00 | 0.0025 × 20,000 = 50.00 | 0.0025 × 10,000 = 25.00, plus 12.50 for bucket checks |
| Output rows | included above | 10,000 × 0.01 = 100.00 | 10,000 × 0.01 = 100.00 |
| **Total** | **1,500,335.00** | **1,838.77** | **572.50** |
| Plan shows | 1500335.00 | 1838.77 | 572.50 |

The three totals match the measured plans exactly. Two details were inferred from those totals, not read from statistics: the merge input fractions round to the full inputs, and the bucket charge uses one row per bucket.

The nested loop is priced about 2,600 times the hash join. Measured time agrees on the order of magnitude: 3,089 ms against 2.5 ms, a factor of about 1,230.

### Which is faster: measured results

All numbers below are medians in milliseconds of five timed runs, or three for the sweep, on PostgreSQL 12.2 built from the pin, on one arm64 Mac with every table in `shared_buffers`. They come from [the script](#the-script). Forced legs use one process. The "planner's pick" column uses the default `max_parallel_workers_per_gather = 2`, except `misest`, which ran with parallelism off.

| Case | Nested loop | Merge | Hash | Planner's pick |
|---|---|---|---|---|
| `lookup`: 100 outer rows, 1,000,000 indexed inner rows | **0.167** | 77.5 | 52.6 | Nested loop, 0.214 |
| `wide`: 1,000,000 × 1,000,000 on two primary keys | 812 | **246** | 459 (16 batches), 407 (256 MB, one batch) | Parallel Hash Join, 219; with parallelism off, Hash Join, 458 |
| `unindexed`: 1,000,000 × 1,000,000, no index, unordered key | not run; cost 1.89e10 | 801, or 695 at 256 MB | **486**, or 866 at 256 MB | Parallel Hash Join, 263; with parallelism off, Hash Join, 490 |
| `topn`: first 10 rows in key order | **0.074** | 0.105 | 459 | Merge join, 0.082 |
| `misest`: estimate 1 outer row, actual 10,000 | 4,327 | 810 | **142** | Nested loop, 4,889; after `CREATE STATISTICS`, Hash Join, 142 |

Growth with input size, both sides the same size, no index:

| Rows per side | Nested loop | Merge | Hash |
|---|---|---|---|
| 5,000 | 774 | 2.52 | 1.34 |
| 10,000 | 3,089 | 5.10 | 2.50 |
| 20,000 | 12,381 | 10.83 | 5.02 |
| Growth per doubling | 3.99×, 4.01× | 2.02×, 2.12× | 1.87×, 2.01× |

What these numbers show:

- **The indexed nested loop has no rival for a small lookup.** It touched 401 buffers. The hash join read the 1,000,000-row table through a sequential scan (10,314 buffers), and the merge join walked almost the whole index (13,050 buffers).
- **The plain nested loop is quadratic.** Each doubling multiplied time by four. Extrapolating the sweep to 1,000,000 rows per side gives about 8.6 hours. That figure is derived, not measured.
- **The ranking among the fast methods depends on existing order.** Merge beat hash on sorted primary keys, and hash beat merge when both inputs had to be sorted.
- **A hash join cannot deliver its first row early.** In `topn` it read everything, taking about 459 ms, while the other two needed under 0.2 ms.
- **The misestimate cost about 30 times.** The planner estimated 1 outer row and got 10,000, so a plan that looked like one bitmap scan ran 10,000 of them over 10,064,370 buffer hits. Correlated columns caused it, because the planner multiplies their [selectivities](../../../glossary.md#selectivity). After `CREATE STATISTICS ... (dependencies)` and `ANALYZE`, the estimate became 10,567, and the planner chose a hash join on its own. See [extended statistics](../../../glossary.md#extended-statistics).
- **More memory is not always faster.** On the primary-key join, one batch at 256 MB was somewhat faster than 16 batches at 4 MB (407 against 459). On the unordered key it was slower (866, range 757 to 1,291, against 486). This disagrees with the intuition that avoiding temp files always helps, and [the cause is an open question](#open-questions).
- **Parallelism changes the comparison.** Three processes running a Parallel Hash Join took 219 ms, close to one process running a merge join at 246 ms.

These are in-memory results. Nothing here measures disk reads, where index-driven random access could rank differently. See [Open Questions](#open-questions).

### Key interaction: estimated cost is not elapsed time

Two measured cases rank the methods differently in cost and in time.

**The `wide` case.** With parallelism off, the planner priced the hash join's aggregate at 78,873 and the merge join's at 95,086, so it chose the hash join. The merge join then ran in 246 ms against 459 ms. The planner uses index-scan cost for the merge inputs, 36,293 each, and sequential-scan cost for the hash inputs, 20,310 each. The merge join therefore starts 31,967 cost units behind. The hash join's added build, probe and batch charges do not make up that gap. The cost constants are fixed at their defaults and are not calibrated to this machine, so the two rankings can differ.

**The `misest` case.** Here the formula was reasonable but its input was wrong. The nested loop's cost of 26,902 assumed one outer row. Multiplying by the true 10,000 outer rows moves the inner side's cost alone to over 30 million, far above the hash join's 52,734. The hash join's cost barely depends on that estimate, which is why it is the safer plan when estimates are unreliable. The planner does not reason about that risk. It compares only the estimated numbers.

So component A, the cost model, ranks plans by estimated rows and fixed constants. Component B, the executor, spends real time according to actual rows and real hardware. Because those inputs differ, the plan with the lowest cost is the planner's best guess, not a measured result.

### Events that change the choice

| Event | State before | State after | Effect on the choice |
|---|---|---|---|
| `ANALYZE` | Old row counts and value statistics | New statistics | Row estimates and hash bucket and MCV estimates change ([selfuncs.c#estimate_hash_bucket_stats](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L3412)). See [the statistics sources page](query-planner-statistics-sources.md) |
| `CREATE STATISTICS`, then `ANALYZE` | Columns treated as independent | Dependencies known | Measured: estimate 1 → 10,567, nested loop → hash join |
| `CREATE INDEX` on the join key | No parameterized inner path | New inner and ordered paths | Nested loop and presorted merge become candidates ([optimizer/README:717-741](../../../../raw/postgres-12/src/backend/optimizer/README#L717-L741)) |
| `SET work_mem` | Old budget | New budget | Measured: skew plan moved from merge to hash at 256 MB, and batch charges disappeared in `wide` |
| A new `LIMIT` | Cheapest total wins | Fractional comparison | Fast-start plans gain ([planner.c#get_cheapest_fractional_path](../../../../raw/postgres-12/src/backend/optimizer/plan/planner.c#L5885-L5910)) |
| A generic cached plan | Chosen for one parameter value | Reused for others | The method stays fixed while the data each call touches can change. See [plan_cache_mode](plan-cache-mode.md) |
| Rows added with no `ANALYZE` | Estimates lag | Execution sees real rows | A hash join adapts its batches and buckets; a nested loop does not change |

### Settings that change the choice

All of these are `user` context, so `SET` works in a session or transaction with no reload or restart.

| Setting | Default | Effect | Apply scope |
|---|---|---|---|
| `enable_nestloop` | on | Off adds `disable_cost` | Session or transaction ([guc.c#enable_nestloop](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L964-L972)) |
| `enable_mergejoin` | on | Off skips merge paths, except `FULL JOIN` | Session or transaction ([guc.c#enable_mergejoin](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L974-L982)) |
| `enable_hashjoin` | on | Off skips hash paths, except `FULL JOIN` | Session or transaction ([guc.c#enable_hashjoin](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L984-L992)) |
| `enable_material` | on | Off drops optional Materialize nodes; a required one stays | Session or transaction ([guc.c#enable_material](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L954-L962), [costsize.c:3093-3111](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L3093-L3111)) |
| `enable_parallel_hash` | on | Off drops the shared hash table | Session or transaction ([guc.c#enable_parallel_hash](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1034-L1042), [joinpath.c:1873-1884](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1873-L1884)) |
| `work_mem` | 4 MB | Sets batching and sort spill | Session or transaction ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2240)) |

Use the `enable_*` settings for diagnosis, as this page does. They apply to every join in the session, including the ones that were fine. Parallel Hash tries the combined `work_mem` of all participants first ([nodeHash.c:706-712](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L706-L712)). Parallel plans exist for all three methods, but not for `RIGHT` or `FULL` joins ([joinpath.c:1510-1524](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1510-L1524), [joinpath.c:1842-1859](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L1842-L1859)). See [parallel query](../../../glossary.md#parallel-query).

### Build, catalog and extension boundaries

- **Catalog.** `oprcanmerge` and `oprcanhash` are `pg_operator` columns ([pg_operator.h:49-52](../../../../raw/postgres-12/src/include/catalog/pg_operator.h#L49-L52)). Their initial values live in `pg_operator.dat`, which [genbki.pl](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L1-L12) turns into the bootstrap data at build time. An extension's operator gets them through `CREATE OPERATOR ... HASHES, MERGES` ([create_operator.sgml:29](../../../../raw/postgres-12/doc/src/sgml/ref/create_operator.sgml#L29)), and the planner reads them through `op_mergejoinable()` and `op_hashjoinable()` ([lsyscache.c:1193](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1193), [lsyscache.c:1244](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L1244)).
- **Path structures.** `JoinPath` is shared; `NestPath` is an alias, and `MergePath` and `HashPath` extend it with sort keys, mark and restore flags, hash clauses and batch counts ([pathnodes.h#JoinPath](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L1482-L1569)). `JoinCostWorkspace` carries results from the initial cost to the final cost ([pathnodes.h#JoinCostWorkspace](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L2476-L2499)).
- **Plan creation.** `create_nestloop_plan()`, `create_mergejoin_plan()` and `create_hashjoin_plan()` turn the winning path into plan nodes ([createplan.c:3996](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L3996), [createplan.c:4072](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4072), [createplan.c:4379](../../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4379)).
- **Extensions.** `set_join_pathlist_hook` can add or remove join paths, and `join_search_hook` can replace the join search ([joinpath.c:321-326](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L321-L326), [allpaths.c:2807-2808](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L2807-L2808)).
- **Foreign tables.** A [foreign data wrapper](../../../glossary.md#foreign-data-wrapper) can push a join to the remote server through `GetForeignJoinPaths`, as `postgres_fdw` does ([joinpath.c:310-319](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L310-L319), [postgres_fdw.c:555](../../../../raw/postgres-12/contrib/postgres_fdw/postgres_fdw.c#L555)).
- **EXPLAIN.** The labels come from `ExplainNode()`, and `show_hash_info()` and `show_sort_info()` print the batch and sort lines ([explain.c:1118-1126](../../../../raw/postgres-12/src/backend/commands/explain.c#L1118-L1126), [explain.c#show_hash_info](../../../../raw/postgres-12/src/backend/commands/explain.c#L2635), [explain.c#show_sort_info](../../../../raw/postgres-12/src/backend/commands/explain.c#L2553)).

Version 12 has no [Memoize](../../../glossary.md#memoize) node and no `hash_mem_multiplier`; a hash join's budget here is plain `work_mem`.

### Tests

- `src/test/regress/sql/join.sql` forces methods with the `enable_*` settings and checks plans and results, for example at [join.sql:524-534](../../../../raw/postgres-12/src/test/regress/sql/join.sql#L524-L534).
- `src/test/regress/sql/join_hash.sql` covers batch counts and growth: the optimal, good and bad memory cases in serial, parallel-oblivious and parallel-aware forms ([join_hash.sql:90-92](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L90-L92), [join_hash.sql:138-140](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L138-L140), [join_hash.sql:186-188](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L186-L188)).
- No test in the tree compares the speed of the methods. The regression suites assert plan shape and output, not elapsed time. This page's script fills that gap for one machine only.

### Final causal summary

Statistics give estimated rows → the planner builds every legal nested loop, merge and hash path for each pair of inputs, in both directions → the cost formulas multiply for a plain nested loop, charge `N log N` for sorts, and add for a hash join → paths that lose on cost and offer no useful order are dropped → the cheapest plan, or the best fast-start plan under a `LIMIT`, runs → the executor spends real time on real rows, so a wrong estimate or an uncalibrated cost constant can make the chosen method slower than a rejected one → `ANALYZE`, `CREATE STATISTICS`, a new index or a new `work_mem` changes the inputs for the next planning cycle.

As a working rule on this evidence: a nested loop with an index is fastest for small lookups; a hash join is fastest for large unsorted equality joins; a merge join is fastest when the order already exists or the query needs it; and a plain nested loop is by far the slowest whenever both inputs are large, but it is the only option for non-equality joins.

## Measurement Script

### Usage

| Item | What to give |
|---|---|
| Purpose | Times all three join methods on the same fixtures. Backs every number under [Which is faster](#which-is-faster-measured-results), the `sweep10000` plans in the [worked calculation](#worked-calculation-the-same-join-priced-three-ways), the batch and `disable_cost` arithmetic, and the catalog counts under [Which methods are possible](#which-methods-are-possible-at-all). |
| Invocation | From the repository root, extract the script with the command below, then run `bash .wiki-runtime/tmp/jm12-run.sh [stage ...]`. |
| Stages | See [Stages](#stages). The default runs every stage from `build` through `stop`. |
| Environment | `PORT12` (default 55448), `JOBS` (default 8, parallel make jobs), `REPS` (default 5, timed runs per leg; the `sweep` stage always uses 3). |
| Prerequisites | A C toolchain, `make`, `bison` and `flex`. The default `configure` options need `readline` or `libedit`, and `zlib`. No extensions. The cluster uses `--no-locale` and UTF8. |
| Output | Under `.wiki-runtime/tmp/jm12/out/`. Read `results.tsv` first: scenario, leg, join node, serial or parallel, top-node cost, median, minimum and maximum ms, buffers, hash batch line, sort methods. Also `*.plan`, `eligibility.txt`, `assertions.txt`, `platform.txt`. The `report` stage prints them all. |
| Runtime | About 5 minutes from an empty sandbox: about 80 s to build and about 3 min 50 s to measure. About 3 min 50 s with the build present. |
| Cleanup | The `stop` stage stops the server and checks teardown. The `clean` stage deletes the sandbox. Run `clean` yourself: the default run keeps the sandbox. |

Extract the script from this page, from the repository root:

```sh
P=wiki/v12/questions/query-planning/join-methods.md
sed -n '/^### The script/,/^## Context Reviewed/p' "$P" \
  | sed -e '1,/^```bash$/d' -e '/^```$/,$d' > .wiki-runtime/tmp/jm12-run.sh
```

### Stages

| Stage | What it does |
|---|---|
| `build` | Checks the pin, and builds and installs 12.2 out of tree. Skipped when the build is present. |
| `init` | Runs `initdb`, starts the cluster, and records platform facts. Replaces any stopped data directory. |
| `fixture` | Creates the disposable tables, runs `VACUUM ANALYZE`, and warms the buffer pool. |
| `lookup` | 100 outer rows against a 1,000,000-row indexed table. |
| `wide` | Whole-table join on two primary keys, at 4 MB and 256 MB. |
| `unindexed` | Whole-table join on an unindexed key, at both memory sizes. The nested loop is priced but not run. |
| `sweep` | 5,000, 10,000 and 20,000 rows per side, no index, all three methods. |
| `topn` | First ten rows in key order. |
| `eligibility` | Inequality, `FULL JOIN` and `xid` plans, the must-fail statement, and the `pg_operator` counts. |
| `skew` | `EXPLAIN` only: one value in half the rows, at both memory sizes. |
| `misestimate` | A one-row estimate on 10,000 real rows, then the same query with `CREATE STATISTICS`. |
| `report` | Prints all results and the assertion count. |
| `stop` | Stops the server and verifies teardown. |
| `clean` | Deletes the sandbox. Not in the default order. |

Each scenario stage needs a running server with the fixture, so re-run one with `bash .wiki-runtime/tmp/jm12-run.sh init fixture wide report stop`. Run the server stages in one invocation.

### Non-default settings in the measurement

| Setting or state | Value here | Default | Context and apply scope | Why the script needs it | Results that depend on it |
|---|---|---|---|---|---|
| `shared_buffers` | 1 GB, command line | Boot value 8 MB ([guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2162)); `initdb` writes up to 128 MB ([initdb.c:959-963](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L959-L963)) | `postmaster`: restart | Keeps all fixtures cached, so runs measure CPU and memory | All timings. At the default, part of the data would need reads through the operating system. No default leg ran. |
| `autovacuum` | off, command line | on ([guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433)) | `sighup`: reload | Stops background `ANALYZE` changing estimates between legs | The stable estimates. Autovacuum would not change the mechanism. |
| `max_parallel_workers_per_gather` | 0 on every leg except `auto`, by `SET` | 2 ([guc.c#max_parallel_workers_per_gather](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3008-L3016)) | `user`: session | Compares one process per method | Forced columns and all `misest` legs. The `auto` legs show the default, which uses Parallel Hash Join for `wide` and `unindexed`. |
| `enable_nestloop`, `enable_mergejoin`, `enable_hashjoin` | Two of three off per forced leg, by `SET` | on | `user`: session | Forces the method. The planner reads each one as described under [Which methods are possible](#which-methods-are-possible-at-all) | Every forced leg. By default the planner picks only the cheapest method; the planner's-pick column shows that choice. |
| `work_mem` | 256 MB on legs so named, by `SET` | 4 MB | `user`: session | Compares batched and in-memory runs | Those legs. At the default, `wide` and `unindexed` hash joins used 16 batches, and the sorts in `unindexed` used disk. |
| `statement_timeout`, `lock_timeout` | 5 min, 5 s, through `PGOPTIONS` | 0, 0 ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2395)) | `user`: session | Bounds every statement | None. |
| `--no-locale`, UTF8 | At `initdb` | Locale from the environment | Fixed at `initdb` | Removes locale effects | None: all join keys are integers or `xid`. |
| Correlated fixture columns in `me` | `c1 = c2 = c3` by construction | Independence is the planner's default assumption | Fixture state | Produces the misestimate on purpose | The `misest` legs. Without correlation the estimate would be right, and the planner would not choose this nested loop. |
| Half the `sk` rows hold one value | Fixture state | No such skew | Fixture state | Triggers the `disable_cost` branch | The `skew` legs. |

The `jit` setting shows `on`, but LLVM support is off unless `configure` gets `--with-llvm`, and this install holds no LLVM library, so these runs compile no expressions ([configure.in:382-385](../../../../raw/postgres-12/configure.in#L382-L385)). Also, `CREATE STATISTICS` in the `misestimate` stage is dropped at the end of that stage.

### Last run

**Filed numbers: 2026-10-30, run 1.** PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`, built with default `configure` options by Apple clang 21 on Darwin arm64, 10 CPUs, 17,179,869,184 bytes of memory. `block_size` 8192; `max_data_alignment` not recorded by the script, and the 8-byte layout shows in the 32-byte estimated row size. `shared_buffers` 1 GB, `work_mem` 4 MB. It ran `init` through `stop` with the build present, in 3 min 47 s, and passed 62 of 62 assertions, including that every leg of a scenario returned the same result.

**Run 2, the same day.** The script was extracted from this page, compared byte for byte with the run 1 text, and run with no arguments from an empty sandbox: 5 min 15 s, 62 of 62 assertions. Every leg chose the same join node. Medians moved by under 10% on nearly every leg that ran for more than 1 ms, and the shorter legs vary more. `unindexed` hash at 256 MB was 803 ms, still slower than the 4 MB leg at 495 ms. `unindexed` auto moved from 263 to 228 ms.

Estimated costs that depend on `ANALYZE`'s sample moved slightly between runs: for example `wide` merge 95,086 then 95,083, and the merge `topn` limit cost 4.13 then 1.86. The tables above use run 1. The `wide`, `sweep` and batch costs used in the calculations were unchanged, and the `skew` hash gap was 10,000,015,628.00 in both runs. The two runs drew different `sk` samples, so the absolute `skew` costs differ. Each sandbox was stopped, checked and deleted.

In `results.tsv` the join column prints `Hash Join` for a `Parallel Hash Join`, because of a greedy match in `join_node()`. The parallel column and the saved `.plan` files show the real node, and the numbers are unaffected.

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
# Stages: build init fixture lookup wide unindexed sweep topn eligibility
#         skew misestimate report stop clean
# Default: every stage from build through stop. clean is never run by default.

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
    | sed -E 's/^.*(Nested Loop( [A-Z][a-z]+ Join)?|Parallel Hash( [A-Z][a-z]+)? Join|Hash( [A-Z][a-z]+)? Join|Merge( [A-Z][a-z]+)? Join).*$/\1/'
}

# leg <scenario> <label> <method> <extra SET text> <SELECT text> [plan]
# Runs EXPLAIN once (ANALYZE, BUFFERS, TIMING OFF unless "plan" is given), then
# one discarded warm-up and REPS timed executions of the plain statement.
leg() {
  local scen="$1" label="$2" method="$3" extra="$4" stmt="$5" mode="${6:-run}"
  local sets tagged plan times med lo hi cost node par buf hb sm res
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
  fi
  cost="$(sed -E -n '1s/^.*cost=[0-9.]+\.\.([0-9.]+) .*$/\1/p' "$plan")"
  node="$(join_node "$plan")"
  par=serial; if grep -q 'Gather' "$plan"; then par=parallel; fi
  buf="$(grep -E '^ *Buffers:' "$plan" | head -n 1 | sed -E 's/^ *Buffers: //' || true)"
  hb="$(grep -E 'Batches:' "$plan" | head -n 1 | sed -E 's/^ *//' || true)"
  sm="$(grep -E 'Sort Method:' "$plan" | sed -E 's/^ *Sort Method: //' | tr '\n' ';' || true)"
  printf '%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\n' \
    "$scen" "$label" "$node" "$par" "$cost" "$med" "$lo" "$hi" "$buf" "$hb" "$sm" >> "$RESULTS"
  note "$scen $label: $node $par cost=$cost median=${med}ms"
  case "$method" in
    nl)    case "$node" in Nested*) r=0 ;; *) r=1 ;; esac; assert "$scen.$label is a nested loop" "$r" ;;
    merge) case "$node" in Merge*)  r=0 ;; *) r=1 ;; esac; assert "$scen.$label is a merge join" "$r" ;;
    hash)  case "$node" in Hash*)   r=0 ;; *) r=1 ;; esac; assert "$scen.$label is a hash join" "$r" ;;
  esac
}

stage_build() {
  say "build 12.2 out of tree (default configure options)"
  source_pin
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
  # shared_buffers (postmaster, restart): 1 GB holds every fixture, so the
  # timed runs read no table block from the operating system.
  # autovacuum (sighup, reload): off, so no background ANALYZE moves the
  # estimates between legs. The fixture runs VACUUM ANALYZE itself.
  # listen_addresses and unix_socket_directories (postmaster): socket only,
  # inside the sandbox. port (postmaster): a non-default port.
  "$BIN/pg_ctl" -D "$DATA" -w -t 60 -l "$OUT/server.log" -o \
    "-c port=$PORT12 -c listen_addresses='' -c unix_socket_directories='$SOCK' -c shared_buffers=1GB -c autovacuum=off" \
    start > /dev/null
  PGDATABASE=postgres sql "CREATE /* wiki_jm12 */ DATABASE jm;"
  sql "SELECT /* wiki_jm12 */ version();" > "$OUT/platform.txt"
  sql "SELECT /* wiki_jm12 */ name || '=' || setting FROM pg_settings
       WHERE name IN ('block_size','max_data_alignment','work_mem','shared_buffers',
                      'max_parallel_workers_per_gather','random_page_cost','seq_page_cost',
                      'cpu_tuple_cost','cpu_operator_cost','jit','effective_cache_size')
       ORDER BY name;" >> "$OUT/platform.txt"
  uname -sm >> "$OUT/platform.txt"
}

stage_fixture() {
  say "disposable fixture tables"
  own_sandbox
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
  # Warm the buffer pool: every table and index read once.
  sql "
SET /* wiki_jm12 */ enable_seqscan = off;
SELECT /* wiki_jm12 */ count(id) FROM ja;
SELECT /* wiki_jm12 */ count(id) FROM jb;
SELECT /* wiki_jm12 */ count(g) FROM jd WHERE g >= 0;
" > /dev/null
  sql "SELECT /* wiki_jm12 */ count(*) FROM ja; SELECT /* wiki_jm12 */ count(*) FROM jb;
       SELECT /* wiki_jm12 */ count(*) FROM jd; SELECT /* wiki_jm12 */ count(*) FROM me;" > /dev/null
}

Q_LOOKUP="SELECT count(*), sum(b.v) FROM js a JOIN jb b ON a.bid = b.id"
Q_WIDE="SELECT count(*), sum(a.v + b.v) FROM ja a JOIN jb b ON a.id = b.id"
Q_UNIDX="SELECT count(*), sum(a.v + b.v) FROM ja a JOIN jb b ON a.k = b.k"
Q_TOPN="SELECT a.id, b.v FROM ja a JOIN jb b ON a.id = b.id ORDER BY a.id LIMIT 10"
Q_MIS="SELECT count(*), sum(d.v) FROM me m JOIN jd d ON m.g = d.g AND m.h = d.h WHERE m.c1 = 1 AND m.c2 = 1 AND m.c3 = 1"
BIGMEM="SET /* wiki_jm12 */ work_mem = '256MB';"

stage_lookup() {
  say "100 outer rows probing a 1,000,000-row indexed inner"
  own_sandbox
  leg lookup auto auto "" "$Q_LOOKUP"
  leg lookup nl nl "" "$Q_LOOKUP"
  leg lookup merge merge "" "$Q_LOOKUP"
  leg lookup hash hash "" "$Q_LOOKUP"
}
stage_wide() {
  say "whole-table join on two primary keys"
  own_sandbox
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
  leg topn auto auto "" "$Q_TOPN"
  leg topn nl nl "" "$Q_TOPN"
  leg topn merge merge "" "$Q_TOPN"
  leg topn hash hash "" "$Q_TOPN"
}
stage_eligibility() {
  say "which clauses each method accepts"
  own_sandbox
  local f="$OUT/eligibility.txt" rc p
  : > "$f"
  p="$OUT/elig.ineq.plan"
  sql "SET /* wiki_jm12 */ enable_nestloop = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM sa_5000 a JOIN sb_5000 b ON a.k < b.k;" > "$p"
  chk "inequality join stays a nested loop with enable_nestloop off" grep -q 'Nested Loop' "$p"
  { echo '-- inequality, enable_nestloop = off'; cat "$p"; } >> "$f"

  p="$OUT/elig.fullineq.err"
  rc=0
  sql "SELECT /* wiki_jm12 */ count(*) FROM sa_5000 a FULL JOIN sb_5000 b ON a.k < b.k;" \
    > /dev/null 2> "$p" || rc=$?
  chk "FULL JOIN on an inequality fails with psql status 3" [ "$rc" = 3 ]
  chk "FULL JOIN on an inequality reports the source error text" \
    grep -q 'FULL JOIN is only supported with merge-joinable or hash-joinable join conditions' "$p"
  { echo '-- FULL JOIN on an inequality'; cat "$p"; } >> "$f"

  p="$OUT/elig.fulleq.plan"
  sql "SET /* wiki_jm12 */ enable_hashjoin = off;
SET /* wiki_jm12 */ enable_mergejoin = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM sa_5000 a FULL JOIN sb_5000 b ON a.k = b.k;" > "$p"
  chk "equality FULL JOIN stays hash or merge with both disabled" \
    grep -E -q 'Hash Full Join|Merge Full Join' "$p"
  { echo '-- equality FULL JOIN, enable_hashjoin and enable_mergejoin off'; cat "$p"; } >> "$f"

  p="$OUT/elig.xid.plan"
  sql "SET /* wiki_jm12 */ enable_hashjoin = off;
SET /* wiki_jm12 */ enable_nestloop = off;
EXPLAIN /* wiki_jm12 */ SELECT count(*) FROM xa a JOIN xb b ON a.x = b.x;" > "$p"
  chk "xid equality cannot merge: hash and nestloop off still gives a nested loop" \
    grep -q 'Nested Loop' "$p"
  { echo '-- xid equality, enable_hashjoin and enable_nestloop off'; cat "$p"; } >> "$f"

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
  set -- build init fixture lookup wide unindexed sweep topn eligibility skew misestimate report stop
fi
for stage in "$@"; do
  case "$stage" in
    build|init|fixture|lookup|wide|unindexed|sweep|topn|eligibility|skew|misestimate|report|stop|clean)
      "stage_$stage" ;;
    *) die "unknown stage: $stage" ;;
  esac
done
```

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/glossary.md`, the last 20 `wiki/log.md` entries and `wiki/v12/index.md`, as navigation.
- `wiki/v12/common-concepts/` does not exist, so this version has no concept page to link.
- Related v12 pages: [Planner Factors That Produce Plans That Over-Fetch Tuples](plans-that-over-fetch-tuples.md), [Query Planner Statistics Sources](query-planner-statistics-sources.md), [plan_cache_mode](plan-cache-mode.md).
- Planner: `joinpath.c` in full, the join sections of `costsize.c`, `add_path()` and `compare_path_costs_fuzzily()` in `pathnode.c`, `joinrels.c`, `allpaths.c`, `initsplan.c`, `createplan.c`, `planner.c`, `optimizer/README`.
- Executor: `nodeNestloop.c`, `nodeHashjoin.c`, `nodeHash.c`, `nodeMergejoin.c`, `execAmi.c`, `hashjoin.h`.
- Catalog and docs: `pg_operator.h`, `pg_operator.dat`, `config.sgml`, `arch-dev.sgml`, `perform.sgml`. Tests: `join.sql`, `join_hash.sql`.

## Evidence Map

| Claim | Evidence |
|---|---|
| Three methods and their mechanics | [optimizer/README:50-58](../../../../raw/postgres-12/src/backend/optimizer/README#L50-L58), [arch-dev.sgml:397-425](../../../../raw/postgres-12/doc/src/sgml/arch-dev.sgml#L397-L425), `ExecNestLoop`, `ExecMergeJoin`, `ExecHashJoinImpl` |
| Candidate generation and legality | [joinpath.c#add_paths_to_joinrel](../../../../raw/postgres-12/src/backend/optimizer/path/joinpath.c#L116-L327) |
| Cost formulas | `initial_cost_*` and `final_cost_*` in `costsize.c`; reproduced exactly by the worked calculation |
| Path comparison | [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L166-L215) |
| Hash batching and growth | [nodeHash.c#ExecChooseHashTableSize](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L668-L845), [nodeHash.c#ExecHashIncreaseNumBatches](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L888-L906) |
| Which method is faster | This page's script: what the built server did on this fixture and host, not a general law |

## Open Questions

- **Cost and time disagree.** In `wide`, the planner ranks the hash join (78,873) below the merge join (95,086), while measured time ranks them the other way (459 ms against 246 ms). Both stand. This is the measured-versus-source disagreement `MANDATORY Measurement Script` asks to keep. The source shows how each cost is computed, but it does not establish which real operations account for the time gap. A CPU-level profile would be needed.
- **More memory made one hash join slower.** `unindexed` hash at 256 MB took 866 ms (range 757 to 1,291) against 486 ms at 4 MB, while the source charges it less (74,370 against 89,998), and the `wide` case moved the other way. The cause is unknown. A plausible but unverified guess is memory-access locality in one large table, and it would need hardware counters to test.
- **Disk behavior was not measured.** Every table stayed in `shared_buffers`, and temp files landed on one Mac's SSD and page cache. The ranking on cold data, on spinning disks, or with tables bigger than memory is unknown. `random_page_cost` models that case, but the script cannot drop the operating system cache without `sudo`, which `AGENTS.md` forbids.
- **The timing evidence covers one host.** Other CPUs, memory sizes and row widths can change the ratios, and `wide` used narrow joined columns over short rows. Treat ratios as examples.
- **Default-parallel `misest` was not run.** That stage ran with parallelism off, so this page does not show whether the default setting would also choose the nested loop.
- **Inferred worked-calculation inputs.** The merge scan fractions and the one-row bucket size were inferred from the totals, as noted there. They were not printed by the server.
- **Same plan, different time.** The `misest` `serial` and `nl` legs use the same plan and the same cost, yet ran at 4,889 and 4,327 ms in run 1, and 4,845 and 4,307 ms in run 2. The gap repeats, and its cause is unknown. It may come from leg order, since `serial` runs first, but the script does not test that.
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
- [nodeNestloop.c#ExecNestLoop](../../../../raw/postgres-12/src/backend/executor/nodeNestloop.c#L60-L256)
- [nodeMergejoin.c:61-85](../../../../raw/postgres-12/src/backend/executor/nodeMergejoin.c#L61-L85)
- [nodeHashjoin.c#ExecHashJoinImpl](../../../../raw/postgres-12/src/backend/executor/nodeHashjoin.c#L164-L576)
- [nodeHash.c#ExecChooseHashTableSize](../../../../raw/postgres-12/src/backend/executor/nodeHash.c#L667-L845)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
