---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Reading EXPLAIN (ANALYZE, BUFFERS, SETTINGS) to Find Missing, Unusable and Bloated Indexes in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The short answer](#the-short-answer)
  - [How to run it and what each option turns on](#how-to-run-it-and-what-each-option-turns-on)
  - [Where every number comes from](#where-every-number-comes-from)
  - [From statement to printed plan](#from-statement-to-printed-plan)
  - [Reading one plan node](#reading-one-plan-node)
  - [Loops, averages and totals](#loops-averages-and-totals)
  - [The Buffers lines](#the-buffers-lines)
  - [The Settings line](#the-settings-line)
  - [Other lines that matter for index diagnosis](#other-lines-that-matter-for-index-diagnosis)
  - [A decision procedure for index problems](#a-decision-procedure-for-index-problems)
  - [Case 1: the index is missing](#case-1-the-index-is-missing)
  - [Case 2: the index cannot match the predicate](#case-2-the-index-cannot-match-the-predicate)
  - [Case 3: the index is invalid](#case-3-the-index-is-invalid)
  - [Case 4: the index is usable but not chosen](#case-4-the-index-is-usable-but-not-chosen)
  - [Case 5: index bloat changes cost and reads](#case-5-index-bloat-changes-cost-and-reads)
  - [How the planner prices a B-tree index](#how-the-planner-prices-a-b-tree-index)
  - [What the executor reads in a B-tree scan](#what-the-executor-reads-in-a-b-tree-scan)
  - [The bloat scenarios built with the mandatory B-tree bloat tests](#the-bloat-scenarios-built-with-the-mandatory-b-tree-bloat-tests)
  - [What EXPLAIN showed in each phase](#what-explain-showed-in-each-phase)
  - [Worked calculation: why the plan flips](#worked-calculation-why-the-plan-flips)
  - [The false-positive trap: a low fillfactor looks like bloat](#the-false-positive-trap-a-low-fillfactor-looks-like-bloat)
  - [Confirming and fixing bloat](#confirming-and-fixing-bloat)
  - [Branches and exceptional cases](#branches-and-exceptional-cases)
  - [Lifecycle events that change what EXPLAIN shows](#lifecycle-events-that-change-what-explain-shows)
  - [The key interaction](#the-key-interaction)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [How to use the script](#how-to-use-the-script)
  - [Stages](#stages)
  - [Settings, fixtures and non-default behavior](#settings-fixtures-and-non-default-behavior)
  - [The script](#the-script)
  - [Last run](#last-run)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

The prompt is restated below after the corrections `MANDATORY Prompt Hygiene` requires; the original wording and each correction are recorded in this filing's `wiki/log.md` entry. It sits in a text block because it quotes a file name as plain text.

```text
Follow AGENTS.md. In PostgreSQL 17, question: create a comprehensive tutorial on how to
read EXPLAIN (ANALYZE, BUFFERS, SETTINGS) and how to use it to find issues in plans when
indexes are missing or not in use because of index bloat. Use the common concept
mandatory-btree-bloat-tests.md to generate test scenarios with bloated indexes.
```

## Answer

### The short answer

`EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` runs the statement and prints, for every [plan node](../../../glossary.md#plan-tree), two kinds of numbers side by side. The [planner](../../../glossary.md#planner)'s estimates come first: startup and total [cost](../../../glossary.md#cost), rows and width. The [executor](../../../glossary.md#executor)'s measurements follow: time to first and last row, rows, and the number of times the node ran. `BUFFERS` adds the node's buffer accesses, and `SETTINGS` adds one line naming every planner-relevant setting whose value differs from its built-in default ([explain.c#ExplainNode](../../../../raw/postgres-17/src/backend/commands/explain.c#L1807-L1872), [explain.c:2280-2282](../../../../raw/postgres-17/src/backend/commands/explain.c#L2280-L2282), [explain.c#ExplainPrintSettings](../../../../raw/postgres-17/src/backend/commands/explain.c#L807-L863)).

Three rules make the output readable. First, time and rows on the node line are averages per loop, while the `Buffers` line is a total over all loops ([explain.c:1841-1858](../../../../raw/postgres-17/src/backend/commands/explain.c#L1841-L1858), [explain.c:2280-2282](../../../../raw/postgres-17/src/backend/commands/explain.c#L2280-L2282)). Second, a parent node's time and buffers include its children's, because the counters are snapshots taken around the parent's own call ([instrument.c#InstrStartNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L66-L80), [instrument.c#InstrStopNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L82-L128)). Third, `hit` means the block was already in [shared_buffers](../../../glossary.md#shared_buffers), and `read` means it was not and was requested from the kernel; a `read` can still be served by the [OS page cache](../../../glossary.md#os-page-cache) ([ref/explain.sgml:185-201](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L185-L201), [bufmgr.c:1452-1463](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1452-L1463)).

With those rules, the index problems this page measures look like this:

| Problem | What `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` shows | Measured example |
|---|---|---|
| Missing index | a Seq Scan (often Parallel) whose `Rows Removed by Filter` dwarfs its output rows, and whose buffers equal the table's pages | 20 rows out of 1,000,000, `Rows Removed by Filter: 333327` per loop over 3 loops, `Buffers: shared hit=7352` (the whole table) |
| Index cannot match the predicate | the same Seq Scan, with a cast or a function wrapped around the indexed column in `Filter` | `Filter: ((customer_id)::text = '4242'::text)`, estimate 5,000 rows against 20 actual |
| Invalid index | the same Seq Scan; nothing in the plan mentions the index | `orders_amount_uq` has `indisvalid = f`; a valid index on the same column gives a 6-buffer Bitmap Index Scan |
| Index usable but not chosen | a different scan, with the cause either in the `Settings` line or in the row estimate | `Settings: random_page_cost = '1000'`; or 799,980 of 1,000,000 rows wanted |
| Index bloat | an index node that reads many buffers per returned entry, an index cost estimate inflated by the index's physical size, and sometimes a plan that abandons the index | 277 index buffers for 10,116 entries against 30 after `REINDEX`; a 90 % range count chose a 7,353-buffer Seq Scan over a 249-buffer Index Only Scan until the index was rebuilt |

The bloat rows come from scenarios built and maintained under the wiki's [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md) protocol, with a measured `REINDEX INDEX` as the oracle. Every number on this page was produced by the script under [Measurement Script](#measurement-script).

### How to run it and what each option turns on

`ExplainQuery()` turns the option list into an `ExplainState`. In PostgreSQL 17 only `COSTS` starts on; `TIMING` and `SUMMARY` then default to whatever `ANALYZE` is, and every other option, `BUFFERS` and `SETTINGS` included, stays off unless named ([explain.c#NewExplainState](../../../../raw/postgres-17/src/backend/commands/explain.c#L368-L382), [explain.c:284-285](../../../../raw/postgres-17/src/backend/commands/explain.c#L284-L285), [explain.c:305-306](../../../../raw/postgres-17/src/backend/commands/explain.c#L305-L306), [ref/explain.sgml:181-208](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L181-L208)).

| Option | Default in 17 | What it adds | Constraint | Evidence |
|---|---|---|---|---|
| `ANALYZE` | off | runs the statement and adds actual time, rows and loops | the statement's side effects happen | [explain.c:678-709](../../../../raw/postgres-17/src/backend/commands/explain.c#L678-L709), [ref/explain.sgml:90-124](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L90-L124) |
| `BUFFERS` | off | per-node `Buffers:` and `I/O Timings:` lines, and a `Planning:` group | without `ANALYZE`, only the planning group can be non-zero | [explain.c:206-207](../../../../raw/postgres-17/src/backend/commands/explain.c#L206-L207), [explain.c:485-506](../../../../raw/postgres-17/src/backend/commands/explain.c#L485-L506) |
| `SETTINGS` | off | one `Settings:` line after the plan tree | none | [explain.c:210-211](../../../../raw/postgres-17/src/backend/commands/explain.c#L210-L211), [explain.c:909-913](../../../../raw/postgres-17/src/backend/commands/explain.c#L909-L913) |
| `TIMING` | same as `ANALYZE` | node times; off still counts rows | requires `ANALYZE` | [explain.c:284-291](../../../../raw/postgres-17/src/backend/commands/explain.c#L284-L291), [explain.c:633-636](../../../../raw/postgres-17/src/backend/commands/explain.c#L633-L636) |
| `SUMMARY` | same as `ANALYZE` | `Planning Time` and `Execution Time` | none | [explain.c:305-306](../../../../raw/postgres-17/src/backend/commands/explain.c#L305-L306), [explain.c:747-752](../../../../raw/postgres-17/src/backend/commands/explain.c#L747-L752), [explain.c:795-797](../../../../raw/postgres-17/src/backend/commands/explain.c#L795-L797) |
| `WAL`, `SERIALIZE` | off | WAL records/bytes per node; output conversion cost | both require `ANALYZE` | [explain.c:278-297](../../../../raw/postgres-17/src/backend/commands/explain.c#L278-L297) |
| `GENERIC_PLAN` | off | a plan for a statement with `$n` parameters | cannot be combined with `ANALYZE` | [explain.c:299-303](../../../../raw/postgres-17/src/backend/commands/explain.c#L299-L303) |
| `FORMAT` | `TEXT` | XML, JSON or YAML with the same information | none | [explain.c:251-269](../../../../raw/postgres-17/src/backend/commands/explain.c#L251-L269), [ref/explain.sgml:296-306](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L296-L306) |

Because `ANALYZE` executes the statement, the documentation's advice for a data-modifying statement is to wrap it in `BEGIN` and `ROLLBACK` ([ref/explain.sgml:90-109](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L90-L109)). The query's result rows are discarded: unless `SERIALIZE` is given, the destination is `None_Receiver` ([explain.c:665-670](../../../../raw/postgres-17/src/backend/commands/explain.c#L665-L670)).

The page's measured form is always the same three options, which PostgreSQL 17 never enables by default:

```sql
EXPLAIN /* wiki_explain_tutorial */ (ANALYZE, BUFFERS, SETTINGS)
SELECT * FROM orders WHERE customer_id = 4242;
```

`EXPLAIN (BUFFERS, SETTINGS)` without `ANALYZE` is legal in 17 and still useful. It runs nothing, so no node prints actual numbers or node buffers, but the planner's own buffer use is printed. The script's run printed this, with 73 planning buffer hits:

```text
Bitmap Heap Scan on orders  (cost=4.58..81.70 rows=20 width=26)
  Recheck Cond: (customer_id = 4242)
  ->  Bitmap Index Scan on orders_customer_id_idx  (cost=0.00..4.58 rows=20 width=0)
        Index Cond: (customer_id = 4242)
Planning:
  Buffers: shared hit=73
```

### Where every number comes from

The single most useful habit is to know, for each number, which component produced it and whether it describes the planner's model or what happened. This table is the mental model the rest of the page uses.

| Value as printed | Meaning | Produced by | Estimate or measured | Later use |
|---|---|---|---|---|
| `cost=S..T` | startup and total cost in planner units; a parent includes its children | the path's cost functions, copied into the plan | estimate ([explain.c:1807-1826](../../../../raw/postgres-17/src/backend/commands/explain.c#L1807-L1826), [perform.sgml:143-152](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L143-L152)) | compared between alternative paths when planning |
| `rows=N width=W` (left) | rows the node is expected to emit; per process for a parallel node | selectivity times the estimated table size | estimate ([perform.sgml:154-162](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L154-L162), [costsize.c:343-346](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L343-L346)) | compared with actual rows |
| `actual time=a..b` | milliseconds to first and to last row, averaged per loop | `Instrumentation.startup` and `.total` divided by `nloops` | measured ([explain.c:1844-1854](../../../../raw/postgres-17/src/backend/commands/explain.c#L1844-L1854)) | finding where time goes |
| `rows=N` (right) | rows emitted, averaged per loop and rounded | `Instrumentation.ntuples / nloops` | measured ([explain.c:1847](../../../../raw/postgres-17/src/backend/commands/explain.c#L1847)) | judging the row estimate |
| `loops=L` | how many times the node was started | `InstrEndLoop()` increments `nloops` at every rescan and at the end | measured ([instrument.c#InstrEndLoop](../../../../raw/postgres-17/src/backend/executor/instrument.c#L138-L165), [execAmi.c#ExecReScan](../../../../raw/postgres-17/src/backend/executor/execAmi.c#L75-L80)) | multiplying averages back to totals |
| `Rows Removed by Filter` | rows the node rejected with its `Filter`, averaged per loop | `nfiltered1 / nloops` | measured ([explain.c#show_instrumentation_count](../../../../raw/postgres-17/src/backend/commands/explain.c#L3622-L3645), [execScan.c:247-255](../../../../raw/postgres-17/src/backend/executor/execScan.c#L247-L255)) | spotting a scan that reads far more than it returns |
| `Heap Fetches` | index-only scan entries that needed a heap visit, total over loops | `ntuples2`, bumped when the visibility map is not set | measured ([explain.c:1992-1994](../../../../raw/postgres-17/src/backend/commands/explain.c#L1992-L1994), [nodeIndexonlyscan.c:161-170](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L161-L170)) | checking whether the visibility map is set |
| `Buffers: shared hit=H read=R ...` | buffer accesses by the node and its children, total over loops | `BufferUsage` deltas around the node | measured ([explain.c:2280-2282](../../../../raw/postgres-17/src/backend/commands/explain.c#L2280-L2282), [instrument.h#BufferUsage](../../../../raw/postgres-17/src/include/executor/instrument.h#L24-L42)) | the main evidence for index page reads |
| `Settings:` | `GUC_EXPLAIN` settings that differ from their compiled-in boot value | `get_explain_guc_options()` | live session state ([guc.c#get_explain_guc_options](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L5332-L5432)) | explaining a plan choice |
| index pages, index tuples, tree height | the physical size, the table's tuple estimate, the fast-root level | `get_relation_info()` at plan time | live size, stored row count ([plancat.c:463-500](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L500)) | index cost estimate; never printed directly |

Two of the planner's inputs need a precise source, because index bloat acts through them. The index's page count is `RelationGetNumberOfBlocks()` on the index, read at plan time, so it is current and includes every deleted or empty page still in the file. The index's tuple count, for a non-partial index, is the table's own estimate, `rel->tuples` ([plancat.c:471-477](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L477)). That table estimate is itself the stored density `reltuples / relpages` scaled to the table's current block count ([tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L666-L713), [reltuples and relpages](../../../glossary.md#reltuples-and-relpages)). The stored values describe the table as of the `VACUUM`, `ANALYZE` or `CREATE INDEX` that last wrote them, while the physical page count is read fresh for every plan.

### From statement to printed plan

This map shows where each part of the output is produced, in the order the code runs. It is the [EXPLAIN](../../../glossary.md#explain) call chain for one plannable statement in 17.

```text
ExplainQuery()                              explain.c:183-366
  parse options into ExplainState          (BUFFERS, SETTINGS, ANALYZE ... default off)
  QueryRewrite()
  standard_ExplainOneQuery()                explain.c:455-512
    if BUFFERS: snapshot pgBufferUsage
    pg_plan_query()  ------------------->  planning: the planner may read catalog and
    if BUFFERS: planning buffers = delta     index pages (metapage, histogram endpoints)
  ExplainOnePlan()                          explain.c:617-800
    instrument_option = TIMER|ROWS (+BUFFERS)(+WAL)
    ExecutorStart()   -> every PlanState gets an Instrumentation   execProcnode.c:409-412
    if ANALYZE: ExecutorRun(), ExecutorFinish()
         each node call goes through ExecProcNodeInstr():           execProcnode.c:473-485
           InstrStartNode()  snapshot timer and pgBufferUsage
           node does its work (children are called from inside it)
           InstrStopNode()   add time, one row, buffer delta
    ExplainPrintPlan()                      explain.c:877-930
      ExplainNode() for every node          explain.c:1367-2423
        name, cost line, actual line, quals, counters, Buffers, children
      ExplainPrintSettings()                explain.c:807-863   -> "Settings:"
    "Planning:" group if planning buffers > 0                    explain.c:723-745
    "Planning Time"                                               explain.c:747-752
    triggers, JIT, serialization
    ExecutorEnd(); "Execution Time" includes executor start-up and shut-down
```

Each step's evidence: option parsing ([explain.c#ExplainQuery](../../../../raw/postgres-17/src/backend/commands/explain.c#L183-L306)); the planning snapshot ([explain.c#standard_ExplainOneQuery](../../../../raw/postgres-17/src/backend/commands/explain.c#L455-L512)); instrumentation options and execution ([explain.c:633-709](../../../../raw/postgres-17/src/backend/commands/explain.c#L633-L709)); per-node allocation and the wrapper ([execProcnode.c:409-412](../../../../raw/postgres-17/src/backend/executor/execProcnode.c#L409-L412), [execProcnode.c#ExecProcNodeInstr](../../../../raw/postgres-17/src/backend/executor/execProcnode.c#L468-L485)); printing order ([explain.c#ExplainPrintPlan](../../../../raw/postgres-17/src/backend/commands/explain.c#L877-L930), [explain.c:718-797](../../../../raw/postgres-17/src/backend/commands/explain.c#L718-L797)). The documentation says the same about the two summary times: planning time excludes parsing and rewriting, and execution time includes executor start-up and shut-down and triggers ([perform.sgml:941-961](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L941-L961)).

Some nodes do not return tuples one at a time, so the generic wrapper does not apply. A Bitmap Index Scan instruments itself around `index_getbitmap()` and nothing else, which is why its `Buffers` line counts index pages only ([nodeBitmapIndexscan.c#MultiExecBitmapIndexScan](../../../../raw/postgres-17/src/backend/executor/nodeBitmapIndexscan.c#L48-L121), [execProcnode.c:488-499](../../../../raw/postgres-17/src/backend/executor/execProcnode.c#L488-L499)). This page uses that property repeatedly to measure index reads in isolation.

### Reading one plan node

Here is the script's first plan: a lookup on a column with no index, in a 1,000,000-row table of 7,352 pages.

```text
Gather  (cost=1000.00..13562.33 rows=20 width=26) (actual time=0.937..17.311 rows=20 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  Buffers: shared hit=7352
  ->  Parallel Seq Scan on orders  (cost=0.00..12560.33 rows=8 width=26) (actual time=0.631..13.105 rows=7 loops=3)
        Filter: (customer_id = 4242)
        Rows Removed by Filter: 333327
        Buffers: shared hit=7352
Planning:
  Buffers: shared hit=35
Planning Time: 0.314 ms
Execution Time: 17.344 ms
```

Read it top-down, one node at a time:

1. **The node name.** `->` marks a child, indented under its parent; `Parallel` is prepended to a parallel-aware node ([explain.c:1631-1651](../../../../raw/postgres-17/src/backend/commands/explain.c#L1631-L1651)). An index scan prints `using <index>`, a Bitmap Index Scan prints `on <index>` ([explain.c:1691-1723](../../../../raw/postgres-17/src/backend/commands/explain.c#L1691-L1723)).
2. **The estimate.** `cost=0.00..12560.33 rows=8 width=26` is printed with `%.2f` costs and `%.0f` rows ([explain.c:1811-1813](../../../../raw/postgres-17/src/backend/commands/explain.c#L1811-L1813)). For a parallel scan the row estimate is per process: 20 expected rows divided by the parallel divisor of 2.4 gives 8 ([costsize.c:343-346](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L343-L346), [costsize.c#get_parallel_divisor](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6380-L6405)).
3. **The measurement.** `actual time=0.631..13.105 rows=7 loops=3`: three processes ran the node (the leader and two [parallel query](../../../glossary.md#parallel-query) workers), and each returned 7 rows on average, which is 20 / 3 rounded ([explain.c:1844-1854](../../../../raw/postgres-17/src/backend/commands/explain.c#L1844-L1854)).
4. **The node's details.** `Filter` and `Rows Removed by Filter: 333327` (per loop, so about 999,980 in total) say the scan inspected almost a million rows to return twenty ([explain.c:2018-2028](../../../../raw/postgres-17/src/backend/commands/explain.c#L2018-L2028)).
5. **The buffers.** `shared hit=7352` is a total over the three processes, because worker instrumentation is added into the leader's node and worker buffer usage into the leader's counters ([execParallel.c:1039-1043](../../../../raw/postgres-17/src/backend/executor/execParallel.c#L1039-L1043), [execParallel.c:1167-1172](../../../../raw/postgres-17/src/backend/executor/execParallel.c#L1167-L1172)). It equals the table's 7,352 pages.
6. **The parent.** The Gather's buffers repeat the child's 7,352, because a parent's counters include its children. `Workers Planned` comes from the plan, `Workers Launched` from execution ([explain.c:2029-2052](../../../../raw/postgres-17/src/backend/commands/explain.c#L2029-L2052)).
7. **The trailer.** `Planning: Buffers: shared hit=35` is the planner's own buffer use on this first run; on the repeated run it was zero and the group was not printed. There is no `Settings:` line, which means every planner setting was at its built-in default ([explain.c:723-745](../../../../raw/postgres-17/src/backend/commands/explain.c#L723-L745), [explain.c:839-841](../../../../raw/postgres-17/src/backend/commands/explain.c#L839-L841)).

The estimate is reproducible by hand, which is a good check that you are reading the right numbers. For `cost_seqscan()`, the [sequential scan](../../../glossary.md#sequential-scan) cost function, disk cost is [`seq_page_cost`](../../../glossary.md#seq_page_cost-and-random_page_cost) `* pages` and CPU cost is `(cpu_tuple_cost + qual cost) * tuples`, the CPU part divided by the parallel divisor ([costsize.c#cost_seqscan](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L284-L351)). With the defaults of 1.0, 0.01 and 0.0025 ([cost.h:24-28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L24-L28)) and the divisor 2 + (1 - 0.3 x 2) = 2.4:

```text
disk  = 1.0 x 7352                          = 7352.00
cpu   = (0.01 + 0.0025) x 1,000,000 / 2.4   = 5208.33
Parallel Seq Scan total                     = 12560.33
Gather = 12560.33 + 1000 setup + 0.1 x 20   = 13562.33
```

The Gather terms are `parallel_setup_cost` and `parallel_tuple_cost` per row, defaults 1000 and 0.1 ([costsize.c#cost_gather](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L436-L461), [cost.h:29-30](../../../../raw/postgres-17/src/include/optimizer/cost.h#L29-L30)).

### Loops, averages and totals

A node can run more than once: the inner side of a [nested loop](../../../glossary.md#nested-loop-join) runs once per outer row, and each parallel process runs its copy once. The [plan node instrumentation](../../../glossary.md#plan-node-instrumentation) keeps running totals and counts the runs as loops. Mixing per-loop and total numbers is the most common misreading of `EXPLAIN ANALYZE`, so this table lists which is which.

| Field | Per loop or total | Evidence |
|---|---|---|
| `actual time`, actual `rows` | average per loop | [explain.c:1844-1858](../../../../raw/postgres-17/src/backend/commands/explain.c#L1844-L1858), [perform.sgml:719-730](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L719-L730) |
| `Rows Removed by Filter`, `Rows Removed by Index Recheck`, `Rows Removed by Join Filter` | average per loop | [explain.c:3631-3644](../../../../raw/postgres-17/src/backend/commands/explain.c#L3631-L3644) |
| `Buffers`, `I/O Timings` | total over loops, including children | [explain.c:2280-2282](../../../../raw/postgres-17/src/backend/commands/explain.c#L2280-L2282), [instrument.c#InstrStopNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L104-L107) |
| `Heap Fetches` | total over loops | [explain.c:1992-1994](../../../../raw/postgres-17/src/backend/commands/explain.c#L1992-L1994) |
| `Heap Blocks: exact/lossy` | total for this process only; in a parallel bitmap scan, the leader's share | [nodeBitmapHeapscan.c:250-255](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L250-L255), [explain.c#show_tidbitmap_info](../../../../raw/postgres-17/src/backend/commands/explain.c#L3592-L3614) |
| estimated `rows` | per execution; per process in a parallel node | [perform.sgml:719-730](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L719-L730), [costsize.c:343-346](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L343-L346) |

The nested loop the script ran shows all of it at once:

```text
->  Index Only Scan using orders_customer_id_idx on orders o  (cost=0.42..4.70 rows=20 width=4) (actual time=0.001..0.001 rows=20 loops=50)
      Index Cond: (customer_id = c.id)
      Heap Fetches: 0
      Buffers: shared hit=151
```

The node ran 50 times and returned 20 rows each time, so it produced 1,000 rows in total, which is what its parent reported. Its 151 buffer hits are the total for all 50 descents, about three per descent, not 151 per loop.

The parallel lossy-bitmap control shows the one counter that is neither: with three processes, `Rows Removed by Index Recheck: 274428` is a per-loop average (the serial run of the same query printed 823,284, exactly three times as much), the `Buffers` total of 7,450 matches the serial run, but `Heap Blocks: exact=183 lossy=2159` is only the leader's share of the serial run's `exact=626 lossy=6726`. In 17 the exact and lossy counters live in each process's own scan state and nothing adds the workers' counts in ([execnodes.h:1803-1824](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L1803-L1824), [nodeBitmapHeapscan.c:250-255](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L250-L255)).

### The Buffers lines

The [EXPLAIN BUFFERS](../../../glossary.md#explain-buffers) counters are fields of one `BufferUsage` struct that the [buffer manager](../../../glossary.md#buffer-manager) increments as it works ([instrument.h#BufferUsage](../../../../raw/postgres-17/src/include/executor/instrument.h#L24-L42)). Knowing exactly what increments each one is what turns them into evidence.

| Counter | Incremented when | Evidence |
|---|---|---|
| `shared hit` | a block was found in [shared_buffers](../../../glossary.md#shared_buffers) and [pinned](../../../glossary.md#buffer-pin); every pin counts, so a root page visited by 50 loops counts 50 times | [bufmgr.c#PinBufferForBlock](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1148-L1160) |
| `shared read` | the block was not in shared buffers and was requested from the kernel; the code counts it as read even if another backend finished the read first | [bufmgr.c:1452-1463](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1452-L1463) |
| `shared dirtied` | this backend made a clean buffer [dirty](../../../glossary.md#dirty-buffer), including by setting [hint bits](../../../glossary.md#hint-bits) on a read | [bufmgr.c:2590-2596](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2590-L2596), [bufmgr.c:5174-5180](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5174-L5180) |
| `shared written` | this backend had to write a dirty buffer out, typically to evict it for a new block | [bufmgr.c:2052-2054](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2052-L2054), [bufmgr.c:3975](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3975) |
| `local ...` | the same, for temporary tables in backend-local buffers | [bufmgr.c:1150-1152](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1150-L1152), [bufmgr.c:1460-1461](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1460-L1461) |
| `temp read/written` | blocks of [temporary files](../../../glossary.md#temporary-file) used by sorts, hashes and similar | [buffile.c:477-483](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L477-L483), [buffile.c:551-557](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L551-L557) |
| `I/O Timings` | time spent in those reads and writes, collected only when `track_io_timing` is on | [pgstat_io.c:125-147](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_io.c#L125-L147), [ref/explain.sgml:185-190](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L185-L190) |

In text format only positive counters are printed, and the line disappears when all are zero ([explain.c#show_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3743-L3817)). JSON prints every counter, zeros included ([explain.c:3862-3905](../../../../raw/postgres-17/src/backend/commands/explain.c#L3862-L3905)), and always prints the `Planning` group. The script's JSON run shows it as `"Shared Hit Blocks": 73` with every other planning counter at 0 ([explain.c#peek_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3702-L3716)).

Five properties of these counters decide how to read them:

1. **They count accesses, not distinct pages.** A heap fetch that lands on the block it already holds pinned is not counted again, because `ReleaseAndReadBuffer()` returns the same buffer without a new pin ([bufmgr.c#ReleaseAndReadBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2599-L2645), [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L125-L140)). That is why the correlated table's baseline index-only scan made 100,000 heap fetches but only 1,012 buffer accesses: consecutive entries pointed into the same heap page.
2. **They include the children.** The planning group, the node lines and the parent lines are all deltas of the same running counter ([instrument.c#BufferUsageAccumDiff](../../../../raw/postgres-17/src/backend/executor/instrument.c#L246-L274)).
3. **`read` does not prove device I/O.** The counter is bumped before the kernel is asked for the data ([bufmgr.c:1452-1463](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1452-L1463)). After the script restarted the server, the first run read 30 index blocks with `I/O Timings: shared read=0.107`, about 3.6 microseconds per block. A restart empties shared buffers but not the OS page cache, and the script never drops that cache.
4. **A repeated run can still read.** A sequential scan of a table larger than a quarter of shared buffers uses a 256 kB [ring buffer](../../../glossary.md#ring-buffer), so its pages do not stay cached for the next run ([heapam.c:433-459](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L459)). The bloated fixture's 7,353-page Seq Scan read 5,275 of its blocks again on the repeated run, with the default 16,384-buffer pool.
5. **A fresh index is not cached.** `CREATE INDEX` and `REINDEX` write a B-tree through the [bulk writer](../../../glossary.md#bulk-writer), bypassing shared buffers, so "the pages will need to be re-read into shared buffers on first use" ([bulk_write.c:12-17](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L12-L17), [nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149)). Right after the index was created, the first lookup showed `Bitmap Index Scan ... Buffers: shared read=3` and `Planning: Buffers: shared hit=18 read=1`; the repeat showed `hit=3`.

For comparisons, this page adds `hit` and `read` together and calls the sum buffer accesses. The split depends on cache state; the sum depends on the plan and the data. The engine's own regression test drops text-mode `Buffers:` lines from its expected output for the same reason, because they vary "depending on the system state" ([explain.sql:25-30](../../../../raw/postgres-17/src/test/regress/sql/explain.sql#L25-L30)).

`Planning: Buffers` deserves its own note. It covers everything `pg_plan_query()` touched: catalog and [relcache](../../../glossary.md#relcache) loads on a first run, the index metapage when the planner reads the tree height ([nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L704)), and index pages read to refresh a histogram endpoint. The planner probes the index's actual minimum or maximum only when a comparison reaches the first or last histogram bucket ([selfuncs.c:1099-1136](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1099-L1136)), and gives up after 100 heap pages of invisible entries ([selfuncs.c:6437-6455](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6437-L6455)). That is why `k BETWEEN 1 AND 900000` printed `Planning: Buffers: shared hit=4` even on its repeated run, while `k BETWEEN 100001 AND 200000` printed nothing.

### The Settings line

[EXPLAIN SETTINGS](../../../glossary.md#explain-settings) answers one question: did someone change a setting the planner reads? `get_explain_guc_options()` walks only settings whose source is not the built-in default, keeps those flagged `GUC_EXPLAIN` and visible to the user, and reports one only if its current value differs from its boot value ([guc.c#get_explain_guc_options](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L5338-L5432), [guc.h:215](../../../../raw/postgres-17/src/include/utils/guc.h#L215)).

```text
                     current value of each GUC
                                |
             source is PGC_S_DEFAULT? --- yes ---> not reported
                                | no
              flagged GUC_EXPLAIN? ------- no ----> not reported (track_io_timing, autovacuum, ...)
                                | yes
              visible to this user? ------ no ----> not reported
                                | yes
         value equals the boot value? --- yes ---> not reported (set, but to the default)
                                | no
                    "name = 'value'" appended to "Settings:"
```

Evidence for each branch: the non-default list and the three tests ([guc.c:5352-5424](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L5352-L5424)); the text output, which prints nothing at all when no setting qualifies ([explain.c:835-862](../../../../raw/postgres-17/src/backend/commands/explain.c#L835-L862)); the non-text output, which prints an empty `Settings` object instead ([explain.c:819-834](../../../../raw/postgres-17/src/backend/commands/explain.c#L819-L834)), as the script's JSON run did.

Four consequences follow for diagnosis:

- **No `Settings:` line is evidence too.** In the script's cluster, `EXPLAIN (SETTINGS) SELECT 1` printed only the plan, so every `GUC_EXPLAIN` setting was at its boot value, although `autovacuum` had been turned off in the configuration file. A standard build has 59 such settings: the GUC table flags 60, but `optimize_bounded_sort` exists only when the server is compiled with `DEBUG_BOUNDED_SORT` ([guc_tables.c:1686-1699](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1686-L1699)). `autovacuum` is not flagged `GUC_EXPLAIN` ([guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)).
- **The comparison is with the compiled-in default, not with `postgresql.conf`.** A value set in the configuration file to something other than the boot value is reported on every query, which is useful when comparing plans across servers.
- **What qualifies.** The flagged settings include the planner's method switches (`enable_*`), the cost constants (`seq_page_cost`, `random_page_cost`, the `cpu_*` and `parallel_*` costs, `effective_cache_size`), the parallel limits, the join-search limits and GEQO settings, `work_mem`, `hash_mem_multiplier`, `temp_buffers`, `effective_io_concurrency`, the JIT thresholds, `search_path`, `constraint_exclusion` and `plan_cache_mode` ([guc_tables.c:784-1017](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L784-L1017), [guc_tables.c:3673-3729](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3673-L3729), [guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2457), [guc_tables.c#search_path](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4279-L4288)).
- **What never appears.** `track_io_timing` is not flagged ([guc_tables.c#track_io_timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1420-L1428)). Its effect is visible only as `I/O Timings` lines.

The engine's regression test checks the same thing from SQL: `set local plan_cache_mode = force_generic_plan` must show up in the text and JSON `Settings` ([explain.sql:79-89](../../../../raw/postgres-17/src/test/regress/sql/explain.sql#L79-L89)).

### Other lines that matter for index diagnosis

| Line | Printed for | What it tells you | Evidence |
|---|---|---|---|
| `Index Cond` | Index Scan, Index Only Scan, Bitmap Index Scan | conditions the index itself searches; anything absent from it was not used to bound the scan | [explain.c:1967-1999](../../../../raw/postgres-17/src/backend/commands/explain.c#L1967-L1999) |
| `Filter` / `Rows Removed by Filter` | most scan and upper nodes | conditions applied row by row after fetching, and how many rows they rejected | [explain.c:1975-1978](../../../../raw/postgres-17/src/backend/commands/explain.c#L1975-L1978), [explain.c:2024-2027](../../../../raw/postgres-17/src/backend/commands/explain.c#L2024-L2027) |
| `Recheck Cond` / `Rows Removed by Index Recheck` | Bitmap Heap Scan, lossy index scans | conditions re-evaluated on heap rows; large removals mean a lossy bitmap | [explain.c:2000-2011](../../../../raw/postgres-17/src/backend/commands/explain.c#L2000-L2011) |
| `Heap Blocks: exact=E lossy=L` | Bitmap Heap Scan with `ANALYZE` | heap pages visited with exact tuple lists, and pages whose bitmap lost precision under `work_mem` | [explain.c#show_tidbitmap_info](../../../../raw/postgres-17/src/backend/commands/explain.c#L3592-L3614), [costsize.c:6483-6510](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6483-L6510) |
| `Heap Fetches` | Index Only Scan | entries whose heap page was not all-visible in the [visibility map](../../../glossary.md#visibility-map) | [nodeIndexonlyscan.c:161-170](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L161-L170) |
| `Sort Method: external merge  Disk: NkB` | Sort | a sort that spilled to temporary files because of `work_mem` | [explain.c:2964-2967](../../../../raw/postgres-17/src/backend/commands/explain.c#L2964-L2967) |
| `(never executed)` | any node | the parent never asked this node for a row | [explain.c:1873-1876](../../../../raw/postgres-17/src/backend/commands/explain.c#L1873-L1876) |

Three controls in the script exercise these lines. With [`work_mem`](../../../glossary.md#work_mem) `= '1MB'` and parallelism off, sorting all orders by `note` printed `Sort Method: external merge  Disk: 26344kB` and `Buffers: shared hit=7352, temp read=9873 written=9944`. With `work_mem = '64kB'`, a 100,020-row range through a forced [bitmap scan](../../../glossary.md#bitmap-scan) printed `Heap Blocks: exact=626 lossy=6726` and `Rows Removed by Index Recheck: 823284`: the bitmap became [lossy](../../../glossary.md#lossy-bitmap), so every row on a lossy page had to be rechecked. A join whose outer side found no customer printed `(never executed)` on the whole inner Bitmap Heap Scan subtree. Each of those changed settings is listed in its own `Settings:` line, as the [Measurement Script](#settings-fixtures-and-non-default-behavior) section explains.

One last warning about Bitmap Heap Scans in this pin. Even a `count(*)` that needs no columns visits every heap page in its bitmap, because commit `78cb2466f75` (2025-04-02) disabled the skip-fetch optimization in the 17 branch by always passing `need_tuples = true` ([nodeBitmapHeapscan.c:186-219](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L186-L219)).

### A decision procedure for index problems

Start from a plan where you expected an index and did not get one, or got one that is slow. Walk this tree from the top; each branch names the `EXPLAIN` evidence and the source behavior behind it.

```text
Is the table accessed by Seq Scan / Parallel Seq Scan with a large "Rows Removed by Filter"?
|
+-- yes: is there an index whose leading column appears in the Filter?
|        |
|        +-- no  ......................................... Case 1: the index is missing
|        |
|        +-- yes: does the Filter wrap that column in a cast or function,
|                 or compare it with another type?
|                 +-- yes ...................................... Case 2: cannot match
|                 +-- no: is pg_index.indisvalid false? 
|                          +-- yes ............................ Case 3: invalid index
|                          +-- no  ............................ Case 4: usable, not chosen
|                                     check "Settings:", then estimated vs actual rows
|
+-- no: an index node is used, or a Seq Scan wins only at wide ranges.
         Does the index node read many more buffers per returned entry than a
         freshly built index would, or does the plan change after REINDEX?
         +-- yes ......................................... Case 5: index bloat
         |      confirm with pgstatindex (leaf density, deleted pages) and the fillfactor
         +-- no  ......................................... the index is healthy;
                look at heap access (Heap Fetches, Heap Blocks, correlation)
```

| Branch | `EXPLAIN` evidence | Engine behavior behind it | Evidence |
|---|---|---|---|
| Case 1 | Seq Scan, filter rejects almost everything, buffers = table pages | without a matching index the planner has only scan paths that read the table | [costsize.c#cost_seqscan](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L284-L351) |
| Case 2 | the column appears inside a cast or function in `Filter`; row estimate is a default | an index clause needs the index's opfamily operator applied to the indexed key itself | [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2392-L2460) |
| Case 3 | Seq Scan, no mention of the index | the planner skips indexes with `indisvalid = false` | [plancat.c:256-267](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267) |
| Case 4 | a `Settings:` line, or an estimate that makes the index costlier | `enable_*` adds `disable_cost` or removes the path; cost constants and row estimates decide | [costsize.c:606-607](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L606-L607), [indxpath.c:1738-1740](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1738-L1740) |
| Case 5 | buffers per entry far above a fresh index's; inflated Bitmap Index Scan cost | the planner prorates index I/O over all physical index pages | [selfuncs.c:6717-6732](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732) |

### Case 1: the index is missing

The plan in [Reading one plan node](#reading-one-plan-node) is the canonical symptom. Three numbers carry the diagnosis together:

| Signal | Value | Why it matters |
|---|---|---|
| output rows | 20 (7 per loop x 3 loops, rounded) | the query wants very little |
| `Rows Removed by Filter` | 333,327 per loop, about 999,980 in total | the scan inspected everything to find it |
| `Buffers` | 7,352 shared hits | every heap page was read |

After `CREATE INDEX orders_customer_id_idx ON orders (customer_id)`, the same statement became:

```text
Bitmap Heap Scan on orders  (cost=4.58..81.70 rows=20 width=26) (actual time=0.008..0.015 rows=20 loops=1)
  Recheck Cond: (customer_id = 4242)
  Heap Blocks: exact=20
  Buffers: shared hit=23
  ->  Bitmap Index Scan on orders_customer_id_idx  (cost=0.00..4.58 rows=20 width=0) (actual time=0.005..0.005 rows=20 loops=1)
        Index Cond: (customer_id = 4242)
        Buffers: shared hit=3
```

The index part now costs 3 buffer accesses, consistent with one page per level of a descent from root to leaf ([nbtsearch.c#_bt_search](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L95-L179)); the run did not record the index's height. `Heap Blocks: exact=20` says the 20 matching rows sit on 20 different heap pages, so the heap part costs 20, and the parent's 23 includes the child's 3.

### Case 2: the index cannot match the predicate

An index is only considered for a condition of the form `indexed_key operator expression` where the operator belongs to the index's operator family, the group of [operator classes](../../../glossary.md#operator-class) the index column uses, or a commuted form of it ([indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269), [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2392-L2460)). A cast on the column changes the left operand into an expression the index does not store, so no index clause is built and only a scan path remains. Two of the script's controls show the two common ways this happens:

| Statement | `Filter` printed | Estimated rows | Actual rows | Plan |
|---|---|---|---|---|
| `WHERE customer_id::text = '4242'` | `((customer_id)::text = '4242'::text)` | 5,000 | 20 | Parallel Seq Scan, 7,352 buffers |
| `WHERE customer_id = 4242.0` | `((customer_id)::numeric = 4242.0)` | 5,000 | 20 | Parallel Seq Scan, 7,352 buffers |

The second row is the easy one to miss. The literal `4242.0` is `numeric`, so the parser casts the integer column up to `numeric`, as the printed `(customer_id)::numeric` shows, and the `=` becomes `numeric = numeric`. The B-tree `integer_ops` family holds only operators between `int2`, `int4` and `int8` ([pg_amop.dat:15-168](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L15-L168)), so that operator cannot drive the integer index. The `Filter` line shows the cast; the query text does not.

The estimate is a second clue. For an expression with no statistics, `var_eq_const()` falls back to one over the default distinct count, `1 / 200`, so 1,000,000 rows become 5,000 ([selfuncs.c:441-449](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L441-L449), [selfuncs.c:5957-5966](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5957-L5966), [selfuncs.h:52](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L52)). A round default estimate on a filtered column is often the sign of an expression the planner cannot see into. The fix is to write the predicate against the bare column with a matching type, or to create an [expression index](../../../glossary.md#expression-index) on the expression.

### Case 3: the index is invalid

A failed `CREATE INDEX` [`CONCURRENTLY`](../../../glossary.md#concurrently) leaves its index behind, marked invalid in [`pg_index`](../../../glossary.md#pg_index), and the planner ignores it; `psql`'s `\d` shows it as `INVALID` ([ref/create_index.sgml:646-661](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L661), [plancat.c:256-267](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267), [Invalid index](../../../glossary.md#invalid-index)). The script provokes this with a unique concurrent build over duplicate amounts, which fails with `could not create unique index "orders_amount_uq"`. The catalog then holds:

```text
         index          | indisvalid | indisready | indislive
------------------------+------------+------------+-----------
 orders_customer_id_idx | t          | t          | t
 orders_amount_uq       | f          | f          | t
```

`EXPLAIN` of `WHERE amount = 42.0` printed a Parallel Seq Scan with `Rows Removed by Filter: 333000` per loop and no trace of `orders_amount_uq`. After the invalid index was dropped and a valid, non-unique index built on the same column, the same statement used a Bitmap Index Scan of 6 buffers and a Bitmap Heap Scan of 1,006 in total. Because the plan itself never mentions an invalid index, the check is a catalog query:

```sql
SET /* wiki_explain_index_flags */ statement_timeout = '30s';
SET /* wiki_explain_index_flags */ lock_timeout = '2s';
SELECT /* wiki_explain_index_flags */ i.indexrelid::regclass AS index, i.indisvalid, i.indisready
  FROM pg_index i
 WHERE i.indrelid = 'orders'::regclass
 ORDER BY 1;
```

### Case 4: the index is usable but not chosen

Here an index path exists, but another path is cheaper in the planner's model. Either a setting changed the model, which the `Settings:` line reports, or the row estimate made the index genuinely worse.

| Control | `Settings:` line | Plan | Buffers | Mechanism |
|---|---|---|---|---|
| methods disabled | `enable_indexscan = 'off', enable_bitmapscan = 'off'` | Parallel Seq Scan | 7,352 | index paths get `disable_cost`; the bitmap path is penalized the same way ([costsize.c:606-607](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L606-L607), [costsize.c:1041-1042](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042), [disable_cost](../../../glossary.md#disable_cost)) |
| expensive random reads | `random_page_cost = '1000'` | Parallel Seq Scan | 7,352 | each index and heap page fetched by index is priced at 1000 ([selfuncs.c:6780-6787](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787), [costsize.c:728-731](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L728-L731)) |
| cheap random reads | `random_page_cost = '1.1', effective_cache_size = '8GB'` | Index Scan (instead of Bitmap) | 23 | the same 20 rows; the plain index scan now wins |
| low selectivity | none | Seq Scan | 7,352 | 799,980 of 1,000,000 rows wanted; reading the table is cheaper than 800,000 index fetches |

All four were set with `set_config(..., true)` inside the statement that ran `EXPLAIN`, so they lasted only for that statement. In a session you would use `SET`, because every [GUC](../../../glossary.md#guc) in this table has [context](../../../glossary.md#guc-context) `user` and applies to the session or transaction ([guc_tables.c#enable_indexscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L793-L802), [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)). The defaults are on for every `enable_*` switch, 4.0 for `random_page_cost` and 524,288 blocks for `effective_cache_size` ([costsize.c:134-137](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L134-L137), [cost.h:25](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25), [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34)).

`enable_indexscan = off` in 17 does not remove index paths; it adds the fixed `disable_cost` of 1.0e10 to their startup cost, so a disabled index can still win when everything else is disabled too ([costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130), [costsize.c:606-607](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L606-L607)). `enable_indexonlyscan = off` works differently: `check_index_only()` returns false, so no index-only path is built at all ([indxpath.c:1738-1740](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1738-L1740)).

The low-selectivity row is the planner being right. A filter that keeps 80 % of the rows, an estimate of 802,673 rows close to the actual 799,980, and `Rows Removed by Filter: 200020` together say that a Seq Scan was the correct choice. That plan is serial because a parallel plan would pay `parallel_tuple_cost` of 0.1 for each of 802,673 rows sent to the leader, about 80,000 cost units on top of the 19,852 the serial scan costs ([costsize.c#cost_gather](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L436-L461)).

### Case 5: index bloat changes cost and reads

Index [bloat](../../../glossary.md#bloat) is space in the index file that holds no entries the index still needs. In a [B-tree](../../../glossary.md#b-tree) it takes two shapes. VACUUM deletes a page only when it is completely empty, and B-tree never merges partly full pages ([nbtree/README:235-245](../../../../raw/postgres-17/src/backend/access/nbtree/README#L235-L245)), so deleting most but not all keys in each key range leaves sparse [leaf pages](../../../glossary.md#leaf-page). Completely emptied pages become [deleted pages](../../../glossary.md#b-tree-page-deletion), which stay in the file until a later page split recycles them ([nbtree/README:279-282](../../../../raw/postgres-17/src/backend/access/nbtree/README#L279-L282), [nbtree/README:319-325](../../../../raw/postgres-17/src/backend/access/nbtree/README#L319-L325)). The documentation names the same usage pattern: "if all but a few index keys on a page have been deleted, the page remains allocated" ([maintenance.sgml:1032-1039](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1039)).

Bloat reaches `EXPLAIN` through two independent channels, and the rest of this page is about telling them apart:

| Channel | Component | What it uses | Where it shows in `EXPLAIN` |
|---|---|---|---|
| planner cost | `btcostestimate()` and `genericcostestimate()` | the index's physical page count, all pages included | the cost of the index node, the plan choice |
| executor reads | the B-tree scan code | the live leaf pages in the key range, walked through sibling links | the index node's `Buffers` |

### How the planner prices a B-tree index

The cost of the index part of a scan is computed from four inputs, which the planner keeps in the index's [IndexOptInfo](../../../glossary.md#indexoptinfo) or derives from the [selectivity](../../../glossary.md#selectivity) of the query's conditions:

| Input | Meaning | Source | Live or stored |
|---|---|---|---|
| `index->pages` | the index file's block count, metapage, internal, leaf, deleted and empty pages alike | `RelationGetNumberOfBlocks()` | live ([plancat.c:473-477](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L473-L477)) |
| `index->tuples` | for a non-partial index, the table's tuple estimate | `rel->tuples`, scaled from `reltuples / relpages` | stored density, live size ([plancat.c:476](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L476), [tableam.c:711-747](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747)) |
| `numIndexTuples` | index entries the scan is expected to visit | boundary-qual selectivity x `index->rel->tuples` | estimate ([selfuncs.c:7003-7019](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7003-L7019)) |
| `tree_height` | the [fast root](../../../glossary.md#fast-root)'s level, from the cached [metapage](../../../glossary.md#metapage) | `_bt_getrootheight()` | cached metapage ([plancat.c:488-495](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L495), [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)) |

With those inputs, the logic is a short linear chain:

1. **Pages to visit.** `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)`, a pro-rata share of all index pages ([selfuncs.c:6717-6732](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)).
2. **I/O cost.** For a single scan, `numIndexPages * random_page_cost` ([selfuncs.c:6780-6787](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787)); a repeated inner scan uses the [Mackert-Lohman formula](../../../glossary.md#mackert-lohman-formula) instead ([selfuncs.c:6756-6779](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6756-L6779)).
3. **CPU cost.** `numIndexTuples * (cpu_index_tuple_cost + cpu_operator_cost * number_of_index_quals)` ([selfuncs.c:6803-6810](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6803-L6810)).
4. **Descent.** `ceil(log2(index->tuples)) * cpu_operator_cost` plus `(tree_height + 1) * 50 * cpu_operator_cost`; the source says the second term exists so that "bloated indexes would appear to have the same search cost as unbloated ones" without it ([selfuncs.c:7086-7106](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7086-L7106), [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).
5. **Heap cost.** `cost_index()` adds the heap: for an [index-only scan](../../../glossary.md#index-only-scan), heap pages are multiplied by `1 - allvisfrac`, so a fully all-visible table adds no heap I/O ([costsize.c:714-747](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L714-L747)); the CPU cost of `cpu_tuple_cost` per fetched tuple remains ([costsize.c:795-800](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800)).

The non-obvious consequence of step 1 is a cancellation. `numIndexTuples` is `selectivity * tuples`, and the formula divides by `tuples` again, so `numIndexPages = ceil(selectivity * index->pages)`. The estimated index page reads depend only on the fraction of the key range and on the physical size of the file. They do not depend on how many entries the index still holds, which is exactly the property bloat changes. Doubling the file through bloat doubles the estimated page reads for every range query.

`EXPLAIN` exposes this index-only estimate directly in one place. A Bitmap Index Scan node's total cost is set to the index path's `indextotalcost`, the amcostestimate result, with startup 0 ([createplan.c:3476-3485](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3476-L3485)). Comparing that number before and after `REINDEX` isolates the price the planner puts on bloat.

### What the executor reads in a B-tree scan

The executor's reads follow a different rule. `_bt_search()` reads one page per level from the fast root down to a leaf, and the cached metapage saves the metapage read on every search after the first ([nbtsearch.c#_bt_search](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L95-L179), [nbtpage.c#_bt_getroot](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L356-L378)). The scan then walks right one leaf at a time through sibling links until the key range ends ([nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2180-L2243)). Deleted pages have been unlinked from their siblings, so a scan that starts after the deletion never visits them ([nbtree/README:279-282](../../../../raw/postgres-17/src/backend/access/nbtree/README#L279-L282)).

| Bloat shape | Planner's estimated index pages | Executor's index buffers |
|---|---|---|
| sparse leaves (most keys deleted from every leaf) | high, because the file is large | high, because the same leaves span the same key range but hold few entries |
| deleted pages (whole leaves emptied) | high, because deleted pages still count in `index->pages` | normal, because the scan walks only live leaves |
| mixed | high | between the two |

That table is the reason the deleted-page shape is so misleading. The index node's buffers look healthy, yet its cost is inflated and the planner may refuse to use it.

### The bloat scenarios built with the mandatory B-tree bloat tests

The fixtures follow the five phases of [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md): **build** (create, load, `ANALYZE`, create the index), **baseline** (record size and counts), **churn** (rule 2's uniform drain, then the maintenance `VACUUM ANALYZE` with its proofs), **decide** (run the published `EXPLAIN` statements unmodified) and **oracle** (`REINDEX INDEX`, measure the file again). This page does not score a rebuild-decision method; the `EXPLAIN` statements stand in the decide phase and the oracle shows what a rebuild would have given back.

Each fixture is one table of 1,000,000 rows, `(id int, k int, f1 bigint, f2 bigint, f3 bigint)`, 7,353 heap pages, with no variable-length column so it has no TOAST table. The B-tree on `k` holds keys 1 to 1,000,000.

| Fixture | Heap order of `k` | Churn | What it models | Concept page source |
|---|---|---|---|---|
| `drand` | hash order, so key and heap position are uncorrelated | rule 2 drain: delete every row outside one heap block in ten | most keys removed from every leaf: sparse leaves | rule 2 and family 2's calibration idea |
| `dseq` | sequential, perfectly correlated | the same drain | whole leaves emptied: deleted pages, plus sparse survivors | rule 2's "volume, not distribution" warning |
| `ffctl` | hash order | none (drain-exempt), index built `WITH (fillfactor = 50)` | a fresh index that must not be rebuilt | fillfactor controls 53 to 55 and family 3's fresh-index idea |

The drain deletes by heap block number taken from each row's `ctid`, `((ctid::text::point)[0])::int % 10 <> 0`, which leaves 100,096 rows on 736 blocks spread over the whole heap. Because the drain selects heap blocks while a B-tree's leaves are ordered by key, the shape it leaves in the index depends on each fixture's key-to-heap correlation, as the concept page warns.

The maintenance step followed the protocol's assumption, which stands in for [autovacuum](../../../glossary.md#autovacuum), and its no-defeat rule, which forbids anything that holds back the [xmin horizon](../../../glossary.md#xmin-horizon) or cancels the statement. Each drained table got one [`VACUUM`](../../../glossary.md#vacuum) `(VERBOSE, ANALYZE)` in a session whose four [settable timeouts](../../../glossary.md#statement_timeout-and-lock_timeout) were 0, after the churn's [cumulative statistics](../../../glossary.md#cumulative-statistics) were flushed. Every proof came back clean:

| Proof | `drand` | `dseq` |
|---|---|---|
| statement completed, no skip or cancel line in the log | yes | yes |
| `tuples: ... removed, ... remain, N are dead but not yet removable` | 899,904 removed, 100,096 remain, **0** | 899,904 removed, 100,096 remain, **0** |
| `tuples missed ... due to cleanup lock contention` | none (**0**) | none (**0**) |
| other backends with an xmin or open transaction, slots, prepared transactions | 0, 0, 0, 0 | 0, 0, 0, 0 |
| index line of the VERBOSE report | `2745 in total, 0 newly deleted` | `2745 in total, 1721 newly deleted` |
| timeouts in force | all four 0 | all four 0 |

The missed-tuple check goes beyond the concept page's list. On an earlier run of this script one `dseq` heap page was skipped: a non-aggressive VACUUM that cannot get a cleanup lock processes the page without pruning and counts its dead tuples as missed, leaving them and their index entries in place ([vacuumlazy.c:929-961](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L929-L961), [vacuumlazy.c:1762-1769](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769), [vacuumlazy.c:664-668](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668)). The script now runs a [`CHECKPOINT`](../../../glossary.md#checkpoint) between the drain and the maintenance `VACUUM`, which flushes every data file ([ref/checkpoint.sgml:31-37](../../../../raw/postgres-17/doc/src/sgml/ref/checkpoint.sgml#L31-L37)), so the pages holding the drain's [dead tuples](../../../glossary.md#dead-tuple) are already clean when `VACUUM` reaches them. It also refuses to continue if any tuple is missed; the filed run missed none. Rule 3's census then found `n_mod_since_analyze = 0` on all three tables against thresholds of 10,059.6, 10,059.6 and 100,050, so nothing was analyzed again.

The page classes after each phase, from [`pgstatindex`](../../../glossary.md#pgstatindex) in the [pgstattuple](../../../glossary.md#pgstattuple) extension:

| Index | Phase | Bytes | Blocks | Level | Leaf pages | Deleted pages | `avg_leaf_density` |
|---|---|---|---|---|---|---|---|
| `drand_k` | built | 22,487,040 | 2,745 | 2 | 2,733 | 0 | 90.06 |
| `drand_k` | churned | 22,487,040 | 2,745 | 2 | 2,733 | 0 | 9.28 |
| `drand_k` | rebuilt | 2,260,992 | 276 | 1 | 274 | 0 | 89.92 |
| `dseq_k` | built | 22,487,040 | 2,745 | 2 | 2,733 | 0 | 90.06 |
| `dseq_k` | churned | 22,487,040 | 2,745 | 2 | 1,012 | 1,721 | 24.56 |
| `dseq_k` | rebuilt | 2,260,992 | 276 | 1 | 274 | 0 | 89.92 |
| `ffctl_k` | every phase | 40,722,432 | 4,971 | 2 | 4,951 | 0 | 49.85 |

`pgstatindex` counts deleted and half-dead pages separately and computes density from leaf free space ([pgstatindex.c:304-326](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L326), [pgstatindex.c:363-367](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367)). The oracle's measured `REINDEX INDEX` gave back 89.95 % of `drand_k` and of `dseq_k`, and 0.00 % of `ffctl_k`.

### What EXPLAIN showed in each phase

Each fixture ran the same five statements in every phase. `q1` to `q3` are left to the planner; `p1` and `p2` are probes that force one path and say so in their `Settings:` line.

| Statement | Text |
|---|---|
| `q1_narrow_rows` | `SELECT * FROM t WHERE k BETWEEN 100001 AND 110000` (1 % of keys) |
| `q2_mid_count` | `SELECT count(*) FROM t WHERE k BETWEEN 100001 AND 200000` (10 %) |
| `q3_wide_count` | `SELECT count(*) FROM t WHERE k BETWEEN 1 AND 900000` (90 %) |
| `p1_bitmap_probe` | `q2` with `enable_seqscan = off, enable_indexscan = off`: a Bitmap Index Scan isolates index page reads and prints the index cost estimate |
| `p2_ios_probe` | `q3` with `enable_seqscan = off, enable_bitmapscan = off`: the index-only path the planner would otherwise reject |

The results of the repeated run, with buffers as `hit + read`:

| Fixture, phase | `q2` plan, cost, buffers | `p1` Bitmap Index Scan cost, buffers, entries | `q3` plan, cost, buffers | `p2` forced Index Only Scan cost, buffers |
|---|---|---|---|---|
| `drand` baseline | Bitmap Heap Scan, 10,908.03, 7,629 | 2,060.47, 276, 100,000 | Parallel Seq Scan, 13,603.00, 7,353 | 52,103.01, 902,328 (900,000 heap fetches) |
| `drand` decide | Index Only Scan, 1,324.52, **277** | **1,222.47, 276, 10,116** | **Seq Scan, 8,854.44, 7,353** | 11,662.16, 2,463 |
| `drand` oracle | Index Only Scan, 320.39, **30** | **218.34, 29, 10,116** | **Index Only Scan, 2,790.03, 249** | 2,790.03, 249 |
| `dseq` baseline | Index Only Scan, 3,880.95, 1,012 (100,000 heap fetches) | 2,123.19, 276, 100,000 | Parallel Seq Scan, 13,603.00, 7,353 | 29,206.49, 11,437 (900,000 heap fetches) |
| `dseq` decide | Index Only Scan, 1,277.14, **104** | **1,178.78, 103, 10,008** | **Seq Scan, 8,854.44, 7,353** | 11,662.38, **912** |
| `dseq` oracle | Index Only Scan, 309.01, **30** | **210.65, 29, 10,008** | **Index Only Scan, 2,790.25, 248** | 2,790.25, 248 |
| `ffctl` (all phases) | Bitmap Heap Scan, 11,827.27, 7,851 | 2,963.26, **498**, 100,000 | Parallel Seq Scan, 13,603.00, 7,353 | 60,006.70, 904,322 (900,000 heap fetches) |

How to read the bloat signature out of that table, fixture by fixture:

1. **`drand`, sparse leaves.** In the decide phase the 10 % count read 277 index buffers for 10,116 entries, 36.5 entries per buffer; after `REINDEX` it read 30, 337 per buffer. The Bitmap Index Scan cost fell from 1,222.47 to 218.34. Both channels moved together, because the leaves the scan walks are the same leaves that make the file big.
2. **`dseq`, deleted pages.** The decide-phase 10 % count read only 104 index buffers, because the 1,721 deleted pages are unlinked and the scan never visits them. The Bitmap Index Scan estimate still rose to 1,178.78, close to `drand`'s, because the file still has 2,745 pages. Against the rebuilt index, its buffers are 3.5 times higher (104 against 30) while its index cost estimate is 5.6 times higher (1,178.78 against 210.65).
3. **The plan flip.** In the decide phase both bloated fixtures answered the 90 % count with a Seq Scan of 7,353 buffers. After `REINDEX` both used an Index Only Scan of about 249 buffers. Nothing else changed between those two phases: same heap, same statistics, same row estimates (89,887 and 89,898). The forced probe `p2` shows what the planner rejected: 2,463 buffers on `drand`, three times fewer than the Seq Scan's 7,353, and only 912 on `dseq`, eight times fewer.
4. **The baseline differs for another reason.** The build phase runs `ANALYZE` but no `VACUUM`, so the visibility map is empty, the planner's `allvisfrac` is 0, and every index-only plan needs heap fetches. That is why `drand`'s baseline `q2` is a Bitmap Heap Scan and `dseq`'s baseline index-only scan shows `Heap Fetches: 100000` ([tableam.c:749-760](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L749-L760), [costsize.c:725-737](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L725-L737)). The maintenance `VACUUM` set the map, and every decide- and oracle-phase index-only scan shows `Heap Fetches: 0`.

The decide-phase plan pair for `drand` is the clearest single picture of index bloat in `EXPLAIN`:

```text
Aggregate  (cost=9079.16..9079.17 rows=1 width=8) (actual time=15.914..15.915 rows=1 loops=1)
  Buffers: shared hit=2078 read=5275
  ->  Seq Scan on drand  (cost=0.00..8854.44 rows=89887 width=0) (actual time=0.004..12.758 rows=90178 loops=1)
        Filter: ((k >= 1) AND (k <= 900000))
        Rows Removed by Filter: 9918
        Buffers: shared hit=2078 read=5275
Planning:
  Buffers: shared hit=4
```

and, after `REINDEX INDEX drand_k`, the same statement:

```text
Aggregate  (cost=3014.75..3014.76 rows=1 width=8) (actual time=9.065..9.065 rows=1 loops=1)
  Buffers: shared hit=249
  ->  Index Only Scan using drand_k on drand  (cost=0.29..2790.03 rows=89887 width=0) (actual time=0.012..5.716 rows=90178 loops=1)
        Index Cond: ((k >= 1) AND (k <= 900000))
        Heap Fetches: 0
        Buffers: shared hit=249
Planning:
  Buffers: shared hit=3
```

The first plan reads the whole heap to count rows an index could count from 249 pages. Nothing in the first plan names the index or says why it lost, which is why the forced probe and the `REINDEX` comparison are part of the method.

### Worked calculation: why the plan flips

Inputs for `drand` in the decide phase, all from the filed run: `index->pages = 2745` before and `276` after the rebuild; `index->tuples = rel->tuples = 100,096`; tree height 2 before and 1 after; for the 90 % range the estimate is 89,887 entries; the heap has 7,353 pages, all-visible after the maintenance `VACUUM`; defaults `random_page_cost = 4.0`, `cpu_index_tuple_cost = 0.005`, `cpu_operator_cost = 0.0025`, `cpu_tuple_cost = 0.01` ([cost.h:24-28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L24-L28)).

```text
Index Only Scan, bloated index (decide)
  numIndexPages  = ceil(89887 x 2745 / 100096)      = ceil(2465.0)  = 2466
  index I/O      = 2466 x 4.0                                       = 9864.00
  index CPU      = 89887 x (0.005 + 2 x 0.0025)                     =  898.87
  descent        = ceil(log2 100096) x 0.0025 + (2 + 1) x 50 x 0.0025 = 0.0425 + 0.375
  heap I/O       = 0 (allvisfrac = 1)
  heap CPU       = 89887 x 0.01                                     =  898.87
  total                                                             = 11662.16   (printed: 11662.16)

Seq Scan, same table
  7353 x 1.0 + 100096 x (0.01 + 2 x 0.0025)                         =  8854.44   (printed: 8854.44)

Index Only Scan, rebuilt index (oracle)
  numIndexPages  = ceil(89887 x 276 / 100096)       = ceil(247.9)   =  248
  total = 248 x 4.0 + 898.87 + 0.0425 + (1 + 1) x 50 x 0.0025 + 898.87 = 2790.03   (printed: 2790.03)
```

Every term comes from the chain in [How the planner prices a B-tree index](#how-the-planner-prices-a-b-tree-index) and the Seq Scan formula in [Reading one plan node](#reading-one-plan-node); the Seq Scan has two quals, so its operator cost per row is `2 x 0.0025`. What the calculation demonstrates:

- The decision turned on one term. With the bloated file, index I/O alone (9,864) exceeds the whole Seq Scan (8,854.44); with the rebuilt file it is 992. The CPU terms are identical in both phases.
- The estimated index pages track the file, not the entries. 2,466 estimated pages became 248 although the entry count and the selectivity never changed.
- The estimate was right for one bloat shape and wrong for the other. For `drand` the forced probe read 2,463 buffers against 2,466 estimated; for `dseq` it read 912 against the same 2,466, because the deleted pages the estimate counts are pages the scan skips.

The same arithmetic reproduces the Bitmap Index Scan probe: `ceil(10205 x 2745 / 100096) = 280` pages, `280 x 4.0 + 10205 x 0.01 + 0.0425 + 0.375 = 1222.47` before, and `ceil(10205 x 276 / 100096) = 29` pages, `29 x 4.0 + 102.05 + 0.0425 + 0.25 = 218.34` after, both as printed.

### The false-positive trap: a low fillfactor looks like bloat

"Many buffers per returned entry" is a symptom, not a diagnosis. The `ffctl` index was built fresh with [`fillfactor`](../../../glossary.md#fillfactor) `= 50` and never churned. Its probe read 498 index buffers for 100,000 entries, 201 entries per buffer, against 362 for the as-built default index (276 buffers for 100,000 entries). By the buffers-per-entry signal alone it looks 1.8 times bloated.

The oracle disagrees: `REINDEX INDEX ffctl_k` produced a file of exactly the same 40,722,432 bytes, 0.00 % given back. A B-tree build leaves `BLCKSZ * (100 - fillfactor) / 100` bytes free on every leaf ([nbtree.h:1138-1145](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145), [nbtsort.c:661-665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665)), and a rebuild uses the same reloption, so a low-fillfactor index is as dense as it is ever going to be. Its `avg_leaf_density` of 49.85 matches its fillfactor of 50, while the default is 90 ([nbtree.h:200](../../../../raw/postgres-17/src/include/access/nbtree.h#L200)).

| Fixture | Index buffers per 1,000 entries in `p1` | `avg_leaf_density` | Fillfactor | Rebuild gives back |
|---|---|---|---|---|
| `drand` decide | 27.3 | 9.28 | 90 | 89.95 % |
| `dseq` decide | 10.3 | 24.56 (plus 1,721 deleted pages) | 90 | 89.95 % |
| `ffctl` | 5.0 | 49.85 | 50 | 0.00 % |
| as-built default (`drand` baseline) | 2.8 | 90.06 | 90 | not measured (no churn yet) |
| rebuilt default (`drand` oracle) | 2.9 | 89.92 | 90 | already rebuilt |

So `EXPLAIN` can raise the suspicion, but only the index's density compared with its own fillfactor, or a measured rebuild, settles it. This is the same distinction the concept page draws between family 3's fresh indexes that must not be rebuilt and family 4's reclaimable ones.

### Confirming and fixing bloat

The `EXPLAIN` evidence above leads to two confirmations and a fix that `EXPLAIN` then verifies.

1. **Compare the physical size with what the entries need.** `pgstatindex` reports leaf, internal, deleted and empty pages and `avg_leaf_density` ([pgstattuple.sgml:226-272](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L226-L272), [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L367)). Read the density against the index's fillfactor, not against 100.
2. **Run the index-isolating probe.** Force a bitmap plan for a representative range and read the Bitmap Index Scan's own `Buffers` and cost, as `p1` does. `SET enable_seqscan = off` and `SET enable_indexscan = off` have context `user`, so they apply to your session only ([guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792), [guc_tables.c#enable_indexscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L793-L802)); the `Settings:` line records that you set them.
3. **Rebuild and compare.** [`REINDEX`](../../../glossary.md#reindex) `INDEX CONCURRENTLY` rebuilds without blocking writes, while plain `REINDEX` locks out writes until it is done ([ref/reindex.sgml:24](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L24), [ref/reindex.sgml:166-175](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L166-L175)). Then re-run the same `EXPLAIN`. Expect `read=` on the first run, because the new index was written past shared buffers.

```sql
-- Diagnostic and repair statements; run them in a session with these limits.
SET /* wiki_explain_bloat_check */ statement_timeout = '5min';
SET /* wiki_explain_bloat_check */ lock_timeout = '5s';
SELECT /* wiki_explain_bloat_check */ s.tree_level, s.leaf_pages, s.deleted_pages,
       s.empty_pages, s.avg_leaf_density,
       coalesce((SELECT option_value FROM pg_options_to_table(c.reloptions)
                  WHERE option_name = 'fillfactor'), '90 (default)') AS fillfactor
  FROM pg_class c, pgstatindex(c.oid::regclass) s
 WHERE c.oid = 'drand_k'::regclass;
REINDEX /* wiki_explain_bloat_fix */ INDEX CONCURRENTLY drand_k;
```

`pgstatindex(regclass)` and the column names are documented for 17 ([pgstattuple.sgml:166](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L166), [pgstattuple.sgml:226-261](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L226-L261)), and the `REINDEX` form follows the synopsis ([ref/reindex.sgml:24](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L24)). How the concurrent rebuild works, including its waits and failure states, is on [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](../indexing/reindex-index-concurrently.md). For estimating reclaimable space from `pgstatindex` alone, see [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)](../indexing/btree-bloat-with-pgstatindex.md).

### Branches and exceptional cases

| Condition | Behavior | Consequence for reading `EXPLAIN` |
|---|---|---|
| node ran in several loops | time, rows and removed rows are per-loop averages; buffers and heap fetches are totals | multiply averages by `loops` before comparing with buffers |
| parallel plan | worker instrumentation and buffers are added into the leader's node | `Buffers` covers all processes; per-worker lines need `VERBOSE` ([explain.c:2287-2306](../../../../raw/postgres-17/src/backend/commands/explain.c#L2287-L2306)) |
| parallel bitmap heap scan | `Heap Blocks` counts the leader only | do not compare it with the serial plan's `Heap Blocks` |
| node never executed | `(never executed)` replaces the actual numbers | its estimate was never tested |
| `LIMIT` stops a node early | actual rows below estimated rows is not an error | compare against the parent's limit ([perform.sgml:1006-1032](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L1006-L1032)) |
| BitmapAnd / BitmapOr | actual rows always 0 | ignore their row counts ([perform.sgml:1050-1052](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L1050-L1052)) |
| first run after a restart, a `CREATE INDEX` or a `REINDEX` | `read` instead of `hit`, and planning buffers | compare repeated runs, or compare `hit + read` |
| seq scan of more than a quarter of shared buffers | ring buffer; repeats keep reading | `read` on a warm system is normal here |
| `track_io_timing = off` (the default) | no `I/O Timings` lines | turn it on per session as superuser to see read time; it has context `superuser` ([guc_tables.c#track_io_timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1420-L1428)) |
| range at a histogram end | the planner probes the index's endpoint | `Planning: Buffers` appears even on a repeated run |
| VACUUM could not get a cleanup lock | dead tuples and their index entries stay until the next VACUUM | an index can keep dead entries although VACUUM "ran"; check `VACUUM VERBOSE`'s `tuples missed` line |
| index deleted pages | counted by the planner, skipped by the scan | buffers look healthy while the plan avoids the index |
| low fillfactor | sparse by design | buffers per entry look bloated; a rebuild returns nothing |

### Lifecycle events that change what EXPLAIN shows

| Event | State before | State after | Effect on later `EXPLAIN` |
|---|---|---|---|
| bulk load + `ANALYZE` (build phase) | empty table | rows, statistics; visibility map not set | index-only plans need heap fetches; bitmap plans may win |
| `CREATE INDEX` | no index | index written past shared buffers | first run shows `read` for index pages and the metapage in planning |
| drain `DELETE` | dense index | dead heap tuples, index entries still present | not measured here: the protocol forbids scoring unvacuumed churn |
| maintenance `VACUUM ANALYZE` | dead tuples | dead tuples and index entries removed, empty leaves deleted, map set, statistics refreshed | sparse or deleted-page bloat; `Heap Fetches: 0`; the file keeps its size |
| `REINDEX INDEX` | bloated file | new compact file; both `pg_class` rows refreshed ([index.c:3126-3135](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135)) | lower index cost and buffers; first run reads the new pages |
| server restart | warm shared buffers | empty shared buffers, warm OS page cache | `read` with small `I/O Timings` |
| `SET` of a `GUC_EXPLAIN` setting | default plan | possibly another plan | the `Settings:` line names it |

The rebuild changed no input other than the index itself in this run. `index_update_stats()` rewrote the table's `reltuples` and `relpages` with the build's exact counts ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2857)), but they equalled what `ANALYZE` had stored. `ANALYZE` reads every block when the table has no more blocks than its 30,000-row sample target, and then `totalrows` is the exact live count; the maintenance log printed `scanned 7353 of 7353 pages, containing 100096 live rows and 0 dead rows; 30000 rows in sample, 100096 estimated total rows` for both drained tables ([analyze.c:1185-1187](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1187), [sampling.c#BlockSampler_Init](../../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L39-L55), [analyze.c:1285-1289](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1285-L1289)). The census and the snapshot tables record 100,096 before and after.

### The key interaction

The planner's cost model uses the index's **physical page count**, while the executor reads only the **live leaves in the key range**. Because `numIndexPages` reduces to `selectivity x index->pages`, every page in the file raises the estimate, including pages that hold one entry and pages that hold none. Therefore sparse-leaf bloat raises both the estimate and the reads, and the `EXPLAIN` buffers confirm the cost; deleted-page bloat raises only the estimate, and the buffers of an index scan look nearly normal. In both shapes the inflated estimate can push the planner to a Seq Scan, as it did for the 90 % count on both fixtures, and the plan output never mentions the index it gave up.

### Final causal summary

Dense B-tree, built at fillfactor 90 → most keys deleted from each key range, then `VACUUM` removes the entries and deletes only completely empty leaves → the file keeps all 2,745 pages (sparse leaves in `drand`, 1,721 deleted pages in `dseq`) → the planner prorates index I/O over all 2,745 pages, so a 90 % range is priced at 2,466 index pages → that alone exceeds the whole-table Seq Scan, so `EXPLAIN` shows a Seq Scan of 7,353 buffers instead of an Index Only Scan → the index node's own buffers, seen through a forced probe, are high for sparse leaves and nearly normal for deleted pages → `REINDEX INDEX` writes a 276-page file, the estimate falls to 248 pages, and the same statement becomes an Index Only Scan of about 249 buffers. A low fillfactor produces the same many-buffers-per-entry symptom without any reclaimable space, so the diagnosis is confirmed with `pgstatindex` density against the fillfactor, or with a measured rebuild.

## Measurement Script

### How to use the script

| Item | Value |
|---|---|
| Purpose | Builds PostgreSQL 17 from this page's pin, runs its regression checks, and captures every `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` output and every number quoted under [Answer](#answer): the controls (missing, unusable, invalid and not-chosen indexes, loops, JSON, temp and lossy lines), the three bloat fixtures through the five phases of [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md) with their maintenance proofs, census and oracle, and one cold-cache read. |
| Invocation | `bash explain_bloat_v17.sh [stage ...]` from the repository root, after saving the block below under that name. With no argument every default stage runs in order. |
| Stages | `build check cluster harness controls fixtures baseline churn decide oracle coldread report`, plus `stop` and `clean`; see [Stages](#stages). |
| Environment | `WIKI_ROOT` (default `$PWD`), `SRC` (default `$WIKI_ROOT/raw/postgres-17`), `SANDBOX` (default `$WIKI_ROOT/.wiki-runtime/tmp/exbt`), `PORT` (default `55437`), `JOBS` (default `4`, `make -j`). |
| Prerequisites | a C toolchain with `make`, `bison`, `flex` and `perl` for the PostgreSQL build, `git`, `pgrep`, and the pinned checkout at `raw/postgres-17` (checked against the pin before building). The build uses `--without-icu --without-readline --without-zlib`, so none of those libraries is needed. `initdb` runs with `--locale=C --encoding=UTF8`. |
| Output | `$SANDBOX/out`. Read `summary.txt` first (one row per captured plan), then `plans.txt` (every plan text, both runs), `maint_proofs.txt`, `census.txt`, `snap.txt`, `oracle.txt`, `ctl_index_flags.txt`, `ctl_json.txt`, `ctl_plain_explain.txt`, `summary_run1.txt`, `platform.txt`, `checks.txt`, `log_audit.txt` and `timing.txt`. |
| Runtime | On the last run (22-core Linux x86_64, `JOBS=20`), 74 seconds end to end: configure 11 s, make 37 s, `make check` 12 s, all measurement stages 14 s. With the build in place, `cluster` through `report` take about 15 seconds. |
| Cleanup | `bash explain_bloat_v17.sh clean` stops the server with `pg_ctl -m fast -w stop`, confirms no `postmaster.pid`, no process naming the data directory under `pgrep -f`, an empty socket directory and a free port, and only then deletes `$SANDBOX` after checking it lies under `$WIKI_ROOT/.wiki-runtime/tmp/`. `stop` does the teardown checks and keeps the files. `out/` is inside the sandbox, so copy it first if you need it. |

### Stages

| Stage | What it does | Needs first |
|---|---|---|
| `build` | checks the pin, configures out of tree under `$SANDBOX/build`, installs into `$SANDBOX/install`, installs `contrib/pgstattuple`; skipped when already installed | nothing |
| `check` | `make check` and the `pgstattuple` check; result lines into `out/checks.txt`; dies on failure | `build` |
| `cluster` | `initdb`, appends the four settings below, starts the server, creates database `lab` and `pgstattuple`, writes `out/platform.txt` including `EXPLAIN (SETTINGS) SELECT 1` | `build` |
| `harness` | creates the capture table `xr`, the `xp()` function that runs `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` twice and stores both texts, the `snap`, `maint`, `census` and `oracle` tables, and the report view | `cluster` |
| `controls` | builds `orders` (1,000,000 rows) and `customers` (50,000), `VACUUM (ANALYZE)`, then captures the controls `a` to `o`, the JSON plan, the plain `EXPLAIN`, the failed concurrent unique build and the index flags | `harness` |
| `fixtures` | phase 1: creates `drand`, `dseq`, `ffctl`, loads 1,000,000 rows each, flushes statistics, `ANALYZE`, creates the scored index | `harness` |
| `baseline` | phase 2: page classes into `snap`, then the five statements on each fixture | `fixtures` |
| `churn` | phase 3: rule 2's drain on `drand` and `dseq`, flush, `CHECKPOINT`, one bracketed `VACUUM (VERBOSE, ANALYZE)` per drained table with the proofs, page classes, rule 3's census; dies if any proof fails | `baseline` |
| `decide` | phase 4: the five statements on each fixture | `churn` |
| `oracle` | phase 5: per fixture, size, `REINDEX INDEX`, size again, page classes, the five statements. **Destructive**: rebuilds the scored indexes | `decide` |
| `coldread` | restarts the server and captures one statement with `track_io_timing` on | `oracle` |
| `report` | writes the result files listed above | any |
| `stop` | stops the server and confirms the teardown | `cluster` |
| `clean` | `stop`, then deletes the sandbox | nothing |

Each stage can be re-run on its own once the stages it needs have run; `fixtures` resets the fixture tables and their result rows, and `controls` resets the control tables and theirs. Re-running `decide` after `oracle` measures the rebuilt index, so to repeat the bloat result run `fixtures baseline churn decide oracle` again.

### Settings, fixtures and non-default behavior

Every setting the script changes, with its default, context, apply scope and the results that depend on it. Contexts come from the 17 GUC table; `postmaster` needs a restart, `sighup` a reload, and `user` and `superuser` apply per session or transaction.

| Setting or state | Value in the run | Default | Where set | Context and apply scope | What it changes, and which results depend on it |
|---|---|---|---|---|---|
| `listen_addresses`, `port`, `unix_socket_directories` | `''`, `55437`, a sandbox directory | `localhost`, `5432`, `/tmp` or the build's default | `postgresql.conf` before the first start | `postmaster`, restart | isolation only; no result depends on them |
| `autovacuum` | `off` | `on` ([guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)) | `postgresql.conf` before the first start | `sighup`, reload | no background worker vacuums or analyzes between phases, as the protocol requires. With the default, a worker could have vacuumed a fixture after its load, setting the visibility map before the baseline; the baseline's heap fetches and `drand`'s baseline Bitmap Heap Scan depend on it being off. `AutoVacuumingActive()` returns false when it is off ([autovacuum.c#AutoVacuumingActive](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3235-L3241)) |
| `statement_timeout`, `lock_timeout` in ordinary sessions | `30min`, `60s` | `0`, `0` ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)) | `PGOPTIONS` per connection | `user`, session | bound a runaway statement; no result depends on them |
| all four settable timeouts in maintenance sessions | `0` | `0` | `PGOPTIONS` per connection | `user`, session | the protocol forbids a timeout that could fire inside the maintenance `VACUUM`; an autovacuum worker forces the same values ([autovacuum.c#worker-timeouts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1462-L1470)) |
| `enable_*`, `random_page_cost`, `effective_cache_size`, `work_mem`, `max_parallel_workers_per_gather` in controls and probes | as printed in each plan's `Settings:` line | on, 4.0, 524,288 blocks, 4 MB, 2 ([costsize.c:134-137](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L134-L137), [cost.h:25](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25), [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34), [guc_tables.c#work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2457), [guc_tables.c#max_parallel_workers_per_gather](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3419-L3428)) | `set_config(name, value, true)` inside `xp()` | `user`, transaction | exactly the plans that print a `Settings:` line; every other plan ran at the defaults, as the absent line proves |
| `track_io_timing` in `coldread` | `on` | `off` ([guc_tables.c#track_io_timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1420-L1428)) | `set_config` inside `xp()` | `superuser`, transaction | the `I/O Timings` lines of the cold read; it never appears in `Settings:` because it is not `GUC_EXPLAIN` |
| `ffctl_k` fillfactor | 50 | 90 ([nbtree.h:200](../../../../raw/postgres-17/src/include/access/nbtree.h#L200)) | `CREATE INDEX ... WITH (fillfactor = 50)` | index storage parameter | the false-positive control; the leaf free-space target is `BLCKSZ * (100 - fillfactor) / 100` ([nbtree.h:1138-1145](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)) |
| build phase without `VACUUM` | visibility map empty at baseline | autovacuum would eventually vacuum a loaded table | the protocol's build phase: load, `ANALYZE`, `CREATE INDEX` | fixture state | baseline index-only scans need heap fetches (`allvisfrac = 0`); only baseline rows depend on it |
| rule 2 drain | 899,904 of 1,000,000 rows deleted per drained table | none | `churn` stage | fixture state | all bloat results |
| `CHECKPOINT` before the maintenance `VACUUM` | one forced checkpoint | checkpoints by time or WAL volume | `churn` stage | command | flushes the pages the drain dirtied so no background write pins a heap page while `VACUUM` wants its cleanup lock; the zero `tuples missed` proof depends on it |
| controls' `VACUUM (ANALYZE)` | map set on `orders`, `customers` | none | `controls` stage | fixture state | `Heap Fetches: 0` in the nested loop; controls are outside the bloat protocol |
| failed concurrent unique build | invalid `orders_amount_uq` | none | `controls` stage | fixture state | Case 3 |

`shared_buffers` is 128 MB, written by `initdb` into `postgresql.conf` and equal to the compiled-in 16,384 buffers ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270)). The ring-buffer reads on the fixtures' Seq Scans follow from it, because their 7,353 pages exceed a quarter of 16,384 ([heapam.c:433-459](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L459)).

### The script

```bash
#!/usr/bin/env bash
# explain_bloat_v17.sh
#
# Measurement script for the wiki page
#   wiki/v17/questions/observability/explain-analyze-buffers-settings-tutorial.md
# It builds PostgreSQL 17 from the pinned checkout, starts a disposable
# cluster, and captures every EXPLAIN (ANALYZE, BUFFERS, SETTINGS) output and
# every number the page quotes:
#   * controls outside the bloat protocol: a missing index, indexes the planner
#     cannot use, an index it chooses not to use, a nested loop, JSON output,
#     and an invalid index left by a failed CREATE INDEX CONCURRENTLY;
#   * three B-tree fixtures run through the five phases of the wiki's
#     Mandatory B-Tree Bloat Tests (build, baseline, churn with its maintenance
#     proofs, decide, REINDEX INDEX oracle), with rule 2's uniform drain on
#     two of them and the third kept fresh as a fillfactor control;
#   * one cold-cache read with track_io_timing on.
#
# Usage, from the repository root:
#   bash explain_bloat_v17.sh                 # every default stage, in order
#   bash explain_bloat_v17.sh decide report   # selected stages
#   bash explain_bloat_v17.sh stop            # stop the server, keep files
#   bash explain_bloat_v17.sh clean           # stop and delete the sandbox
#
# DISPOSABLE: every statement creates, drains, rebuilds or drops objects in a
# throwaway cluster under .wiki-runtime/tmp/.  Never point it at a database
# anyone cares about.

set -uo pipefail

PIN=786db8dcf168bd9df8f55047337525ac19118b1c
WIKI_ROOT=${WIKI_ROOT:-$PWD}
SRC=${SRC:-$WIKI_ROOT/raw/postgres-17}
SANDBOX=${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/exbt}
PORT=${PORT:-55437}
JOBS=${JOBS:-4}

BUILD=$SANDBOX/build
INST=$SANDBOX/install
DATA=$SANDBOX/data
SOCK=$SANDBOX/sock
OUT=$SANDBOX/out

DEFAULT_STAGES="build check cluster harness controls fixtures baseline churn decide oracle coldread report"

die() { printf 'FATAL: %s\n' "$*" >&2; exit 1; }
stamp() { printf '%s %s\n' "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" "$*" | tee -a "$OUT/timing.txt"; }

# Ordinary sessions: generous but finite timeouts, set per connection.
q() {
  PGOPTIONS='-c statement_timeout=30min -c lock_timeout=60s' \
    "$INST/bin/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -U postgres -d lab "$@"
}
# Maintenance sessions: all four settable timeouts at 0, as an autovacuum
# worker forces them, because the bloat protocol forbids a timeout that could
# fire inside the maintenance VACUUM.
qm() {
  PGOPTIONS='-c statement_timeout=0 -c lock_timeout=0 -c idle_in_transaction_session_timeout=0 -c transaction_timeout=0' \
    "$INST/bin/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -U postgres -d lab "$@"
}

check_pin() {
  head=$(git -C "$SRC" --no-optional-locks rev-parse HEAD) || die "cannot read $SRC"
  [ "$head" = "$PIN" ] || die "checkout is at $head, expected $PIN"
}

running() { [ -f "$DATA/postmaster.pid" ] && "$INST/bin/pg_isready" -q -h "$SOCK" -p "$PORT"; }

st_build() {
  check_pin
  mkdir -p "$BUILD" "$OUT" || die "mkdir"
  if [ -x "$INST/bin/postgres" ] && [ -f "$INST/share/extension/pgstattuple.control" ]; then
    stamp "build: already installed, skipped"
    return 0
  fi
  stamp "build: configure"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --without-icu --without-readline --without-zlib ) \
    > "$OUT/configure.log" 2>&1 || die "configure failed, see $OUT/configure.log"
  stamp "build: make"
  make -C "$BUILD" -j"$JOBS" -s > "$OUT/make.log" 2>&1 || die "make failed, see $OUT/make.log"
  make -C "$BUILD" -s install > "$OUT/install.log" 2>&1 || die "install failed"
  make -C "$BUILD/contrib/pgstattuple" -s install >> "$OUT/install.log" 2>&1 || die "pgstattuple install failed"
  stamp "build: done"
}

st_check() {
  stamp "check: make check"
  make -C "$BUILD" -s check > "$OUT/check.log" 2>&1
  rc=$?
  make -C "$BUILD/contrib/pgstattuple" -s check > "$OUT/check_pgstattuple.log" 2>&1
  rc2=$?
  {
    printf 'make check exit %s: ' "$rc"; grep -E '^# (All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests failed)' "$OUT/check.log"
    printf 'pgstattuple check exit %s: ' "$rc2"; grep -E '^# (All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests failed)' "$OUT/check_pgstattuple.log"
  } > "$OUT/checks.txt"
  cat "$OUT/checks.txt"
  [ "$rc" -eq 0 ] && [ "$rc2" -eq 0 ] || die "regression checks failed, see $OUT/check*.log"
}

st_cluster() {
  mkdir -p "$SOCK" "$OUT" || die "mkdir"
  if running; then stamp "cluster: already running"; return 0; fi
  if [ ! -d "$DATA" ]; then
    stamp "cluster: initdb"
    "$INST/bin/initdb" -D "$DATA" -U postgres --auth=trust --locale=C --encoding=UTF8 \
      > "$OUT/initdb.log" 2>&1 || die "initdb failed"
    printf "listen_addresses = ''\nport = %s\nunix_socket_directories = '%s'\nautovacuum = off\n" \
      "$PORT" "$SOCK" >> "$DATA/postgresql.conf"
  fi
  "$INST/bin/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w start > /dev/null || die "start failed"
  "$INST/bin/psql" -X -h "$SOCK" -p "$PORT" -U postgres -d postgres -At \
    -c "SELECT /* wiki_exbt_cluster */ 1 FROM pg_database WHERE datname = 'lab'" | grep -q 1 ||
    "$INST/bin/createdb" -h "$SOCK" -p "$PORT" -U postgres lab || die "createdb failed"
  q -q -c "CREATE /* wiki_exbt_cluster */ EXTENSION IF NOT EXISTS pgstattuple" || die "extension"
  {
    printf 'uname: %s\n' "$(uname -sm)"
    "$INST/bin/pg_controldata" "$DATA" | grep -E '^(Database block size|Maximum data alignment|Data page checksum version):'
    q -At -c "SELECT /* wiki_exbt_platform */ version()"
    q -At -c "SELECT /* wiki_exbt_platform */ name || ' = ' || setting || coalesce(' ' || unit, '') || ' (source ' || source || ')'
                FROM pg_settings
               WHERE name IN ('shared_buffers', 'effective_cache_size', 'random_page_cost', 'seq_page_cost',
                              'work_mem', 'max_parallel_workers_per_gather', 'autovacuum', 'track_io_timing',
                              'default_statistics_target', 'jit')
               ORDER BY name"
    printf 'EXPLAIN (SETTINGS) of SELECT 1, to show that no planner setting differs from its built-in default:\n'
    q -At -c "EXPLAIN /* wiki_exbt_platform */ (SETTINGS) SELECT 1"
  } > "$OUT/platform.txt"
  stamp "cluster: up on port $PORT"
}

st_harness() {
  q -q > /dev/null <<'SQL' || die "harness"
SET /* wiki_exbt_harness */ client_min_messages = warning;

-- One row per captured EXPLAIN: both runs of every statement are kept, so a
-- first (cold) run and a repeated (warm) run can be compared.
CREATE /* wiki_exbt_harness */ TABLE IF NOT EXISTS xr (
  fixture text, phase text, qid text, run int, settings text, plan text,
  PRIMARY KEY (fixture, phase, qid, run));

-- Index shape at each phase: size, catalog counts, and pgstatindex page classes.
CREATE /* wiki_exbt_harness */ TABLE IF NOT EXISTS snap (
  phase text, idx text, bytes bigint, blocks bigint, tbl_reltuples real, idx_reltuples real,
  tree_level int, internal_pages bigint, leaf_pages bigint, empty_pages bigint,
  deleted_pages bigint, avg_leaf_density float8, PRIMARY KEY (phase, idx));

-- The maintenance proofs the protocol's no-defeat rule asks for.
CREATE /* wiki_exbt_harness */ TABLE IF NOT EXISTS maint (
  tbl text PRIMARY KEY, started timestamptz, ended timestamptz, timeouts text,
  xmin_holders int, open_xacts int, slots int, slot_xmins int, prepared int, holders text,
  removed bigint, remain bigint, dead_not_removable bigint, missed bigint,
  mods_after bigint, dead_after bigint);

-- Rule 3's simulated auto-analyze, with the effective per-table values.
CREATE /* wiki_exbt_harness */ TABLE IF NOT EXISTS census (
  tbl text PRIMARY KEY, reltuples real, mods bigint, base_thresh float8, scale float8,
  threshold float8, needs_analyze bool, analyzed bool);

-- Phase 5: the measured REINDEX INDEX.
CREATE /* wiki_exbt_harness */ TABLE IF NOT EXISTS oracle (
  idx text PRIMARY KEY, before_bytes bigint, after_bytes bigint, actual_pct numeric);

-- Run one statement under EXPLAIN (ANALYZE, BUFFERS, SETTINGS) twice and keep
-- both texts.  p_set holds name=value pairs applied with set_config(..., true),
-- so they last only until the end of this one top-level statement.
CREATE /* wiki_exbt_harness */ OR REPLACE FUNCTION xp(p_fixture text, p_phase text, p_qid text,
                                                     p_sql text, p_set text[] DEFAULT '{}')
RETURNS void LANGUAGE plpgsql AS $f$
DECLARE r int; ln text; acc text; kv text;
BEGIN
  FOREACH kv IN ARRAY p_set LOOP
    PERFORM set_config(split_part(kv, '=', 1), split_part(kv, '=', 2), true);
  END LOOP;
  FOR r IN 1..2 LOOP
    acc := '';
    FOR ln IN EXECUTE 'EXPLAIN /* wiki_exbt_explain */ (ANALYZE, BUFFERS, SETTINGS) ' || p_sql LOOP
      acc := acc || ln || E'\n';
    END LOOP;
    INSERT /* wiki_exbt_xp */ INTO xr VALUES (p_fixture, p_phase, p_qid, r, array_to_string(p_set, ', '), acc)
    ON CONFLICT (fixture, phase, qid, run)
    DO UPDATE SET plan = EXCLUDED.plan, settings = EXCLUDED.settings;
  END LOOP;
END $f$;

CREATE /* wiki_exbt_harness */ OR REPLACE FUNCTION snap_take(p_phase text, p_idx text)
RETURNS void LANGUAGE sql AS $s$
  INSERT /* wiki_exbt_snap */ INTO snap
  SELECT p_phase, p_idx, pg_relation_size(ic.oid),
         pg_relation_size(ic.oid) / current_setting('block_size')::int,
         tc.reltuples, ic.reltuples, s.tree_level, s.internal_pages, s.leaf_pages,
         s.empty_pages, s.deleted_pages, s.avg_leaf_density
    FROM pg_class ic
    JOIN pg_index ix ON ix.indexrelid = ic.oid
    JOIN pg_class tc ON tc.oid = ix.indrelid
    CROSS JOIN LATERAL pgstatindex(ic.oid::regclass) s
   WHERE ic.relname = p_idx AND ic.relnamespace = 'public'::regnamespace
  ON CONFLICT (phase, idx) DO UPDATE
     SET bytes = EXCLUDED.bytes, blocks = EXCLUDED.blocks, tbl_reltuples = EXCLUDED.tbl_reltuples,
         idx_reltuples = EXCLUDED.idx_reltuples, tree_level = EXCLUDED.tree_level,
         internal_pages = EXCLUDED.internal_pages, leaf_pages = EXCLUDED.leaf_pages,
         empty_pages = EXCLUDED.empty_pages, deleted_pages = EXCLUDED.deleted_pages,
         avg_leaf_density = EXCLUDED.avg_leaf_density
$s$;

-- maint_begin publishes this session's pending statistics, records the
-- timeouts actually in force, and reads who could be holding the removal
-- horizon back: other backends with an xmin or an open transaction, slots,
-- and prepared transactions.  These are reads beside the VACUUM, not an
-- interlock around it.
CREATE /* wiki_exbt_harness */ OR REPLACE FUNCTION maint_begin(p_tbl text)
RETURNS void LANGUAGE sql AS $m$
  SELECT pg_stat_force_next_flush();
  DELETE /* wiki_exbt_maint */ FROM maint WHERE tbl = p_tbl;
  INSERT /* wiki_exbt_maint */ INTO maint (tbl, started, timeouts, xmin_holders, open_xacts,
                                          slots, slot_xmins, prepared, holders)
  SELECT p_tbl, clock_timestamp(),
         'statement_timeout=' || current_setting('statement_timeout') ||
         ' lock_timeout=' || current_setting('lock_timeout') ||
         ' idle_in_transaction_session_timeout=' || current_setting('idle_in_transaction_session_timeout') ||
         ' transaction_timeout=' || current_setting('transaction_timeout'),
         (SELECT count(*) FROM pg_stat_activity a
           WHERE a.pid <> pg_backend_pid() AND a.backend_xmin IS NOT NULL),
         (SELECT count(*) FROM pg_stat_activity a
           WHERE a.pid <> pg_backend_pid() AND a.xact_start IS NOT NULL),
         (SELECT count(*) FROM pg_replication_slots),
         (SELECT count(*) FROM pg_replication_slots s
           WHERE s.xmin IS NOT NULL OR s.catalog_xmin IS NOT NULL),
         (SELECT count(*) FROM pg_prepared_xacts),
         (SELECT coalesce(string_agg(a.pid || ':' || coalesce(a.backend_type, '?'), ', '), 'none')
            FROM pg_stat_activity a
           WHERE a.pid <> pg_backend_pid()
             AND (a.backend_xmin IS NOT NULL OR a.xact_start IS NOT NULL));
$m$;

CREATE /* wiki_exbt_harness */ OR REPLACE FUNCTION maint_end(p_tbl text)
RETURNS void LANGUAGE sql AS $m$
  SELECT pg_stat_clear_snapshot();
  UPDATE /* wiki_exbt_maint */ maint m
     SET ended = clock_timestamp(), mods_after = s.n_mod_since_analyze, dead_after = s.n_dead_tup
    FROM pg_stat_all_tables s
   WHERE s.schemaname = 'public' AND s.relname = m.tbl AND m.tbl = p_tbl;
$m$;

-- Text helpers for the report: the shared hit and read counts of one
-- "Buffers:" line, summed.
CREATE /* wiki_exbt_harness */ OR REPLACE FUNCTION bufsum(p text)
RETURNS bigint LANGUAGE sql IMMUTABLE AS $b$
  SELECT CASE WHEN p IS NULL THEN NULL ELSE
         coalesce((regexp_match(p, '^shared(?: hit=([0-9]+))?(?: read=([0-9]+))?'))[1]::bigint, 0)
       + coalesce((regexp_match(p, '^shared(?: hit=([0-9]+))?(?: read=([0-9]+))?'))[2]::bigint, 0) END
$b$;

-- One row per captured plan with the fields the page tabulates.  The scan
-- node is the first table-access node in the text; the first "Buffers:" line
-- belongs to the top node and therefore covers the whole plan; a Bitmap Index
-- Scan prints Index Cond and then its own Buffers line.
CREATE /* wiki_exbt_harness */ OR REPLACE VIEW xs AS
SELECT fixture, phase, qid, run, settings,
       (regexp_match(plan, '(Parallel Seq Scan|Seq Scan|Index Only Scan|Index Scan|Bitmap Heap Scan)'))[1] AS scan_node,
       m.a[2]::numeric AS scan_cost, m.a[3]::bigint AS scan_est_rows,
       m.a[4]::bigint AS scan_rows_per_loop, m.a[5]::bigint AS scan_loops,
       (regexp_match(plan, 'Buffers: ([^\n]*)'))[1] AS top_buffers,
       (regexp_match(plan, 'Planning:\n[ ]+Buffers: ([^\n]*)'))[1] AS planning_buffers,
       (regexp_match(plan, 'Bitmap Index Scan on [^ ]+  \(cost=[0-9.]+\.\.([0-9.]+)'))[1]::numeric AS bis_cost,
       (regexp_match(plan, 'Bitmap Index Scan on [^\n]*actual time=[0-9.]+\.\.[0-9.]+ rows=([0-9]+)'))[1]::bigint AS bis_rows,
       (regexp_match(plan, 'Bitmap Index Scan on [^\n]*\n[ ]+Index Cond: [^\n]*\n[ ]+Buffers: ([^\n]*)'))[1] AS bis_buffers,
       (regexp_match(plan, 'Heap Fetches: ([0-9]+)'))[1]::bigint AS heap_fetches,
       (regexp_match(plan, 'Heap Blocks: ([^\n]*)'))[1] AS heap_blocks,
       (regexp_match(plan, 'Workers Launched: ([0-9]+)'))[1]::int AS workers,
       (regexp_match(plan, '\nSettings: ([^\n]*)'))[1] AS settings_line,
       (regexp_match(plan, 'Execution Time: ([0-9.]+) ms'))[1]::numeric AS exec_ms
  FROM xr
  CROSS JOIN LATERAL (SELECT regexp_match(plan,
         '(Parallel Seq Scan|Seq Scan|Index Only Scan|Index Scan|Bitmap Heap Scan)[^\n]*\(cost=([0-9.]+)\.\.([0-9.]+) rows=([0-9]+) width=[0-9]+\) \(actual time=[0-9.]+\.\.[0-9.]+ rows=([0-9]+) loops=([0-9]+)\)') AS r) mm
  CROSS JOIN LATERAL (SELECT ARRAY[mm.r[1], mm.r[3], mm.r[4], mm.r[5], mm.r[6]] AS a) m;
SQL
  stamp "harness: installed"
}

# Fixture A and its variants: controls outside the bloat protocol.
st_controls() {
  stamp "controls: build"
  q -q > /dev/null <<'SQL' || die "controls build"
SET /* wiki_exbt_ctl */ client_min_messages = warning;
DROP /* wiki_exbt_ctl */ TABLE IF EXISTS orders, customers;
DELETE /* wiki_exbt_ctl */ FROM xr WHERE fixture = 'ctl';
CREATE /* wiki_exbt_ctl */ TABLE orders (
  id int NOT NULL, customer_id int NOT NULL, amount numeric(10,2) NOT NULL, note text NOT NULL);
INSERT /* wiki_exbt_ctl */ INTO orders
SELECT i, (i::bigint * 7919) % 50000 + 1, (i % 1000) / 10.0, 'order ' || i
  FROM generate_series(1, 1000000) i;
CREATE /* wiki_exbt_ctl */ TABLE customers (id int PRIMARY KEY, name text NOT NULL);
INSERT /* wiki_exbt_ctl */ INTO customers SELECT i, 'customer ' || i FROM generate_series(1, 50000) i;
VACUUM /* wiki_exbt_ctl */ (ANALYZE) orders, customers;
SQL
  stamp "controls: explain"
  q -q \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'a_no_index', 'lookup', 'SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242')" \
    -c "CREATE /* wiki_exbt_ctl */ INDEX orders_customer_id_idx ON orders (customer_id)" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'b_indexed', 'lookup', 'SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'c_cast_to_text', 'lookup', \$x\$SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id::text = '4242'\$x\$)" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'd_numeric_literal', 'lookup', 'SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242.0')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'e_methods_off', 'lookup', 'SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242', '{enable_indexscan=off,enable_bitmapscan=off}')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'f_random_page_cost_1000', 'lookup', 'SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242', '{random_page_cost=1000}')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'g_low_selectivity', 'range', 'SELECT /* wiki_exbt_ctl_range */ * FROM orders WHERE customer_id < 40000')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'h_nested_loop', 'join', 'SELECT /* wiki_exbt_ctl_join */ c.id, count(*) FROM customers c JOIN orders o ON o.customer_id = c.id WHERE c.id BETWEEN 100 AND 149 GROUP BY c.id')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'i_random_page_cost_1_1', 'lookup', 'SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242', '{random_page_cost=1.1,effective_cache_size=8GB}')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'l_sort_spill', 'sort', 'SELECT /* wiki_exbt_ctl_sort */ id, note FROM orders ORDER BY note', '{work_mem=1MB,max_parallel_workers_per_gather=0}')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'm_lossy_bitmap', 'range', 'SELECT /* wiki_exbt_ctl_lossy */ count(*) FROM orders WHERE customer_id BETWEEN 1000 AND 6000', '{work_mem=64kB,enable_seqscan=off,enable_indexscan=off,max_parallel_workers_per_gather=0}')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'o_parallel_lossy_bitmap', 'range', 'SELECT /* wiki_exbt_ctl_lossy */ count(*) FROM orders WHERE customer_id BETWEEN 1000 AND 6000', '{work_mem=64kB,enable_seqscan=off,enable_indexscan=off}')" \
    -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'n_never_executed', 'join', 'SELECT /* wiki_exbt_ctl_never */ * FROM customers c JOIN orders o ON o.customer_id = c.id WHERE c.id = 0')" \
    > /dev/null || die "controls explain"
  q -At -c "EXPLAIN /* wiki_exbt_ctl_json */ (ANALYZE, BUFFERS, SETTINGS, FORMAT JSON) SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242" \
    > "$OUT/ctl_json.txt" || die "json"
  # Without ANALYZE nothing runs: only estimates, and the planner's own buffer use.
  q -At -c "EXPLAIN /* wiki_exbt_ctl_plain */ (BUFFERS, SETTINGS) SELECT /* wiki_exbt_ctl_lookup */ * FROM orders WHERE customer_id = 4242" \
    > "$OUT/ctl_plain_explain.txt" || die "plain explain"
  # A unique concurrent build over duplicate keys must fail and leave an
  # invalid index behind; the planner then ignores it.
  if q -c "CREATE /* wiki_exbt_ctl_cic */ UNIQUE INDEX CONCURRENTLY orders_amount_uq ON orders (amount)" \
       > "$OUT/ctl_cic.txt" 2>&1; then
    die "the concurrent unique build was expected to fail"
  fi
  grep -q 'could not create unique index "orders_amount_uq"' "$OUT/ctl_cic.txt" || die "unexpected CIC failure, see $OUT/ctl_cic.txt"
  q -q -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'j_invalid_index', 'amount', 'SELECT /* wiki_exbt_ctl_amount */ * FROM orders WHERE amount = 42.0')" \
    > /dev/null || die "invalid index explain"
  q -c "SELECT /* wiki_exbt_ctl_flags */ indexrelid::regclass AS index, indisvalid, indisready, indislive
          FROM pg_index WHERE indrelid = 'orders'::regclass ORDER BY 1" > "$OUT/ctl_index_flags.txt" || die "flags"
  # The same query once a valid, non-unique index on the column exists.
  q -q -c "DROP /* wiki_exbt_ctl */ INDEX orders_amount_uq" \
       -c "CREATE /* wiki_exbt_ctl */ INDEX orders_amount_idx ON orders (amount)" \
       -c "SELECT /* wiki_exbt_ctl */ xp('ctl', 'k_valid_index', 'amount', 'SELECT /* wiki_exbt_ctl_amount */ * FROM orders WHERE amount = 42.0')" \
    > /dev/null || die "valid index explain"
  stamp "controls: done"
}

# Phase 1 of the protocol: create, load, ANALYZE, create the scored index.
st_fixtures() {
  stamp "fixtures: build"
  q -q > /dev/null <<'SQL' || die "fixtures"
SET /* wiki_exbt_fix */ client_min_messages = warning;
DROP /* wiki_exbt_fix */ TABLE IF EXISTS drand, dseq, ffctl;
DELETE /* wiki_exbt_fix */ FROM xr WHERE fixture IN ('drand', 'dseq', 'ffctl');
DELETE /* wiki_exbt_fix */ FROM snap; DELETE /* wiki_exbt_fix */ FROM maint;
DELETE /* wiki_exbt_fix */ FROM census; DELETE /* wiki_exbt_fix */ FROM oracle;

-- drand: key uncorrelated with heap order (rows loaded in hash order).
CREATE /* wiki_exbt_fix */ TABLE drand (id int NOT NULL, k int NOT NULL,
  f1 bigint NOT NULL, f2 bigint NOT NULL, f3 bigint NOT NULL);
INSERT /* wiki_exbt_fix */ INTO drand
SELECT i, i, i, i, i FROM generate_series(1, 1000000) i ORDER BY hashint4(i), i;
SELECT /* wiki_exbt_fix */ pg_stat_force_next_flush();
ANALYZE /* wiki_exbt_fix */ drand;
CREATE /* wiki_exbt_fix */ INDEX drand_k ON drand (k);

-- dseq: key perfectly correlated with heap order.
CREATE /* wiki_exbt_fix */ TABLE dseq (id int NOT NULL, k int NOT NULL,
  f1 bigint NOT NULL, f2 bigint NOT NULL, f3 bigint NOT NULL);
INSERT /* wiki_exbt_fix */ INTO dseq
SELECT i, i, i, i, i FROM generate_series(1, 1000000) i ORDER BY i;
SELECT /* wiki_exbt_fix */ pg_stat_force_next_flush();
ANALYZE /* wiki_exbt_fix */ dseq;
CREATE /* wiki_exbt_fix */ INDEX dseq_k ON dseq (k);

-- ffctl: the drain-exempt control, a fresh index built at fillfactor 50.
CREATE /* wiki_exbt_fix */ TABLE ffctl (id int NOT NULL, k int NOT NULL,
  f1 bigint NOT NULL, f2 bigint NOT NULL, f3 bigint NOT NULL);
INSERT /* wiki_exbt_fix */ INTO ffctl
SELECT i, i, i, i, i FROM generate_series(1, 1000000) i ORDER BY hashint4(i), i;
SELECT /* wiki_exbt_fix */ pg_stat_force_next_flush();
ANALYZE /* wiki_exbt_fix */ ffctl;
CREATE /* wiki_exbt_fix */ INDEX ffctl_k ON ffctl (k) WITH (fillfactor = 50);
SQL
  stamp "fixtures: done"
}

battery() {  # $1 = table, $2 = phase
  t=$1; ph=$2
  q -q \
    -c "SELECT /* wiki_exbt_battery */ xp('$t', '$ph', 'q1_narrow_rows', 'SELECT /* wiki_exbt_q1 */ * FROM $t WHERE k BETWEEN 100001 AND 110000')" \
    -c "SELECT /* wiki_exbt_battery */ xp('$t', '$ph', 'q2_mid_count', 'SELECT /* wiki_exbt_q2 */ count(*) FROM $t WHERE k BETWEEN 100001 AND 200000')" \
    -c "SELECT /* wiki_exbt_battery */ xp('$t', '$ph', 'q3_wide_count', 'SELECT /* wiki_exbt_q3 */ count(*) FROM $t WHERE k BETWEEN 1 AND 900000')" \
    -c "SELECT /* wiki_exbt_battery */ xp('$t', '$ph', 'p1_bitmap_probe', 'SELECT /* wiki_exbt_q2 */ count(*) FROM $t WHERE k BETWEEN 100001 AND 200000', '{enable_seqscan=off,enable_indexscan=off}')" \
    -c "SELECT /* wiki_exbt_battery */ xp('$t', '$ph', 'p2_ios_probe', 'SELECT /* wiki_exbt_q3 */ count(*) FROM $t WHERE k BETWEEN 1 AND 900000', '{enable_seqscan=off,enable_bitmapscan=off}')" \
    > /dev/null || die "battery $t $ph"
}

# Phase 2: record the as-built state, then run the published EXPLAINs.
st_baseline() {
  for t in drand dseq ffctl; do
    q -q -c "SELECT /* wiki_exbt_baseline */ snap_take('built', '${t}_k')" > /dev/null || die "snap"
    battery "$t" baseline
  done
  stamp "baseline: done"
}

maint_one() {  # $1 = table; one bracketed VACUUM (VERBOSE, ANALYZE)
  t=$1
  qm -q -c "SELECT /* wiki_exbt_maint_begin */ maint_begin('$t')" \
        -c "VACUUM /* wiki_exbt_maint */ (VERBOSE, ANALYZE) $t" \
        -c "SELECT /* wiki_exbt_maint_end */ maint_end('$t')" \
        > "$OUT/maint_$t.out" 2> "$OUT/maint_$t.log" || die "maintenance of $t failed, see $OUT/maint_$t.log"
  line=$(grep -E '^tuples: [0-9]+ removed, [0-9]+ remain, [0-9]+ are dead but not yet removable$' "$OUT/maint_$t.log")
  n=$(printf '%s\n' "$line" | grep -c 'tuples:')
  [ "$n" -eq 1 ] || die "expected one tuples line for $t, found $n"
  nums=$(printf '%s\n' "$line" | sed 's/^tuples: \([0-9]*\) removed, \([0-9]*\) remain, \([0-9]*\) are dead but not yet removable$/\1 \2 \3/')
  # Dead tuples on a page whose cleanup lock VACUUM could not get are counted
  # as missed and left in place, index entries included.
  missed=$(sed -n 's/^tuples missed: \([0-9]*\) dead from [0-9]* pages not removed due to cleanup lock contention$/\1/p' "$OUT/maint_$t.log")
  [ -n "$missed" ] || missed=0
  set -- $nums
  q -q -c "UPDATE /* wiki_exbt_maint_verbose */ maint SET removed = $1, remain = $2, dead_not_removable = $3, missed = $missed WHERE tbl = '$t'" || die "record"
  if grep -E "skipping (vacuum|analyze) of \"$t\"" "$OUT/maint_$t.log" "$OUT/server.log" > /dev/null; then
    die "a skip line names $t"
  fi
}

# Phase 3: rule 2's uniform drain on the two drained fixtures, the maintenance
# VACUUM ANALYZE with its proofs, the page classes, then rule 3's census.
st_churn() {
  stamp "churn: drain"
  q -q > /dev/null <<'SQL' || die "drain"
DELETE /* wiki_exbt_drain */ FROM drand WHERE ((ctid::text::point)[0])::int % 10 <> 0;
DELETE /* wiki_exbt_drain */ FROM dseq WHERE ((ctid::text::point)[0])::int % 10 <> 0;
SELECT /* wiki_exbt_drain */ pg_stat_force_next_flush();
SQL
  # Flush the pages the drain dirtied before VACUUM starts, so no background
  # write holds a pin on a heap page while VACUUM wants its cleanup lock.
  q -q -c "CHECKPOINT /* wiki_exbt_churn_checkpoint */" || die "checkpoint"
  stamp "churn: maintenance"
  for t in drand dseq; do maint_one "$t"; done
  for t in drand dseq ffctl; do
    q -q -c "SELECT /* wiki_exbt_churn */ snap_take('churned', '${t}_k')" > /dev/null || die "snap"
  done
  stamp "churn: census"
  q -q > /dev/null <<'SQL' || die "census"
SELECT /* wiki_exbt_census */ pg_stat_clear_snapshot();
INSERT /* wiki_exbt_census */ INTO census
SELECT c.relname, c.reltuples, s.n_mod_since_analyze, e.base, e.scale,
       e.base + e.scale * greatest(c.reltuples, 0),
       e.enabled AND s.n_mod_since_analyze > e.base + e.scale * greatest(c.reltuples, 0),
       false
  FROM pg_class c
  JOIN pg_stat_all_tables s ON s.relid = c.oid
  CROSS JOIN LATERAL (
    SELECT CASE WHEN o.thr >= 0 THEN o.thr ELSE current_setting('autovacuum_analyze_threshold')::float8 END AS base,
           CASE WHEN o.sf >= 0 THEN o.sf ELSE current_setting('autovacuum_analyze_scale_factor')::float8 END AS scale,
           coalesce(o.en, true) AS enabled
      FROM (SELECT (max(option_value) FILTER (WHERE option_name = 'autovacuum_analyze_threshold'))::float8 AS thr,
                   (max(option_value) FILTER (WHERE option_name = 'autovacuum_analyze_scale_factor'))::float8 AS sf,
                   (max(option_value) FILTER (WHERE option_name = 'autovacuum_enabled'))::bool AS en
              FROM pg_options_to_table(c.reloptions)) o) e
 WHERE c.relnamespace = 'public'::regnamespace AND c.relname IN ('drand', 'dseq', 'ffctl')
ON CONFLICT (tbl) DO UPDATE
   SET reltuples = EXCLUDED.reltuples, mods = EXCLUDED.mods, base_thresh = EXCLUDED.base_thresh,
       scale = EXCLUDED.scale, threshold = EXCLUDED.threshold,
       needs_analyze = EXCLUDED.needs_analyze, analyzed = false;
SELECT /* wiki_exbt_census */ format('ANALYZE /* wiki_exbt_census_analyze */ %I', tbl)
  FROM census WHERE needs_analyze ORDER BY tbl
\gexec
UPDATE /* wiki_exbt_census */ census SET analyzed = true WHERE needs_analyze;
SQL
  bad=$(q -At -c "SELECT /* wiki_exbt_proofs */ count(*) FROM maint
                   WHERE dead_not_removable IS DISTINCT FROM 0 OR missed IS DISTINCT FROM 0
                      OR xmin_holders <> 0 OR open_xacts <> 0
                      OR slot_xmins <> 0 OR prepared <> 0 OR ended IS NULL") || die "proofs"
  [ "$bad" = 0 ] || die "$bad maintenance statement(s) defeated or incomplete; the fixtures are not scored"
  nm=$(q -At -c "SELECT /* wiki_exbt_proofs */ count(*) FROM maint") || die "proofs"
  [ "$nm" = 2 ] || die "expected 2 maintenance rows, found $nm"
  stamp "churn: done, proofs clean"
}

# Phase 4: the published EXPLAIN statements, unmodified, on the churned state.
st_decide() {
  for t in drand dseq ffctl; do battery "$t" decide; done
  stamp "decide: done"
}

# Phase 5: REINDEX INDEX, the file measured again, and the same EXPLAINs.
st_oracle() {
  for t in drand dseq ffctl; do
    q -q -v idx="${t}_k" > /dev/null <<'SQL' || die "oracle $t"
SELECT /* wiki_exbt_oracle */ pg_relation_size(:'idx') AS b \gset
REINDEX /* wiki_exbt_oracle */ INDEX :"idx";
INSERT /* wiki_exbt_oracle */ INTO oracle
VALUES (:'idx', :b, pg_relation_size(:'idx'),
        round(100.0 * (1 - pg_relation_size(:'idx')::numeric / :b), 2))
ON CONFLICT (idx) DO UPDATE SET before_bytes = EXCLUDED.before_bytes,
   after_bytes = EXCLUDED.after_bytes, actual_pct = EXCLUDED.actual_pct;
SELECT /* wiki_exbt_oracle */ snap_take('rebuilt', :'idx');
SQL
    battery "$t" oracle
  done
  stamp "oracle: done"
}

# One cold read: restart empties shared buffers, the OS page cache stays warm.
st_coldread() {
  "$INST/bin/pg_ctl" -D "$DATA" -l "$OUT/server.log" -m fast -w restart > /dev/null || die "restart"
  q -q -c "SELECT /* wiki_exbt_cold */ xp('drand', 'coldread', 'q2_mid_count', 'SELECT /* wiki_exbt_q2 */ count(*) FROM drand WHERE k BETWEEN 100001 AND 200000', '{track_io_timing=on}')" \
    > /dev/null || die "coldread"
  stamp "coldread: done"
}

st_report() {
  q -At -c "SELECT /* wiki_exbt_report */ format(E'=== %s | %s | %s | run %s%s ===\n%s', fixture, phase, qid, run,
                   CASE WHEN settings <> '' THEN ' | set ' || settings ELSE '' END, plan)
              FROM xr
             ORDER BY CASE fixture WHEN 'ctl' THEN 0 WHEN 'drand' THEN 1 WHEN 'dseq' THEN 2 ELSE 3 END,
                      CASE phase WHEN 'baseline' THEN 0 WHEN 'decide' THEN 1 WHEN 'oracle' THEN 2 WHEN 'coldread' THEN 3 ELSE 4 END,
                      phase, qid, run" > "$OUT/plans.txt" || die "plans"
  q -c "SELECT /* wiki_exbt_report */ fixture, phase, qid, scan_node, scan_cost, scan_est_rows AS est_rows,
               scan_rows_per_loop * scan_loops AS act_rows, scan_loops AS loops, bufsum(top_buffers) AS buffers,
               bis_cost, bufsum(bis_buffers) AS bis_buffers, bis_rows, heap_fetches, heap_blocks, workers,
               settings_line
          FROM xs WHERE run = 2
         ORDER BY CASE fixture WHEN 'ctl' THEN 0 WHEN 'drand' THEN 1 WHEN 'dseq' THEN 2 ELSE 3 END,
                  CASE phase WHEN 'baseline' THEN 0 WHEN 'decide' THEN 1 WHEN 'oracle' THEN 2 WHEN 'coldread' THEN 3 ELSE 4 END,
                  phase, qid" > "$OUT/summary.txt" || die "summary"
  q -c "SELECT /* wiki_exbt_report */ fixture, phase, qid, top_buffers AS run1_buffers, planning_buffers AS run1_planning_buffers
          FROM xs WHERE run = 1
         ORDER BY fixture, phase, qid" > "$OUT/summary_run1.txt" || die "summary1"
  q -c "SELECT /* wiki_exbt_report */ * FROM snap
         ORDER BY idx, CASE phase WHEN 'built' THEN 0 WHEN 'churned' THEN 1 ELSE 2 END" > "$OUT/snap.txt" || die "snap"
  q -c "SELECT /* wiki_exbt_report */ o.*, s.leaf_pages, s.deleted_pages, s.avg_leaf_density
          FROM oracle o JOIN snap s ON s.idx = o.idx AND s.phase = 'rebuilt' ORDER BY o.idx" > "$OUT/oracle.txt" || die "oracle"
  q -c "SELECT /* wiki_exbt_report */ tbl, timeouts, xmin_holders, open_xacts, slots, slot_xmins, prepared, holders,
               removed, remain, dead_not_removable, missed, mods_after, dead_after, ended - started AS took
          FROM maint ORDER BY tbl" > "$OUT/maint_proofs.txt" || die "maint"
  q -c "SELECT /* wiki_exbt_report */ * FROM census ORDER BY tbl" > "$OUT/census.txt" || die "census"
  {
    printf 'skip or cancel lines in the server log: '
    grep -c -E 'skipping (vacuum|analyze) of|canceling statement due to' "$OUT/server.log"
    printf 'ERROR lines in the server log (the one expected is the failed concurrent unique build):\n'
    grep -E 'ERROR:' "$OUT/server.log"
  } > "$OUT/log_audit.txt"
  stamp "report: written to $OUT"
}

st_stop() {
  if [ -f "$DATA/postmaster.pid" ]; then
    "$INST/bin/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null || die "stop failed"
  fi
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid still present"
  if pgrep -f -- "$DATA" > /dev/null; then die "a process still names $DATA"; fi
  if [ -d "$SOCK" ] && [ -n "$(ls -A "$SOCK")" ]; then die "socket directory $SOCK is not empty"; fi
  if "$INST/bin/pg_isready" -q -h "$SOCK" -p "$PORT" 2> /dev/null; then die "something still answers on port $PORT"; fi
  echo "stop: no postmaster.pid, no process names $DATA, socket directory empty, port $PORT free"
}

st_clean() {
  st_stop
  case "$SANDBOX" in
    "$WIKI_ROOT"/.wiki-runtime/tmp/?*) ;;
    *) die "refusing to delete $SANDBOX: not under $WIKI_ROOT/.wiki-runtime/tmp/" ;;
  esac
  rm -rf "$SANDBOX" && echo "clean: deleted $SANDBOX"
}

mkdir -p "$OUT" 2> /dev/null
stages=${*:-$DEFAULT_STAGES}
for s in $stages; do
  case "$s" in
    build|check|cluster|harness|controls|fixtures|baseline|churn|decide|oracle|coldread|report|stop|clean)
      "st_$s" ;;
    *) die "unknown stage $s" ;;
  esac
done
```

### Last run

| Item | Value |
|---|---|
| Date | 2026-10-08, 15:25:49Z to 15:27:03Z |
| Server | PostgreSQL 17.11, `REL_17_STABLE` at `786db8dcf168bd9df8f55047337525ac19118b1c`, built by this script |
| Platform | Linux x86_64, kernel `6.18.40.1-microsoft-standard-WSL2`, 22 cores, gcc 13.3.0; block size 8192, maximum data alignment 8, data checksums off (from `pg_controldata`) |
| Checks | `make check`: `# All 225 tests passed.`; `pgstattuple`: `# All 1 tests passed.` |
| Cluster | `shared_buffers = 128MB` (16,384 buffers), every `GUC_EXPLAIN` setting at its boot value, `autovacuum = off` |
| Log audit | 0 skip or cancel lines; the only `ERROR` is the expected failed concurrent unique build, logged by two process IDs |
| Script | the text above is the text that produced every number on this page |

Values that move between runs: timings; the `hit`/`read` split; row estimates, because `ANALYZE` samples randomly ([analyze.c:1185-1187](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1187)); and the duplicate key named in the failed build's error. Between the filed run and the trial run before it, plan choices, page counts, sizes, the oracle's percentages and the buffer totals of the repeated serial runs were identical. The buffer totals of the parallel probes moved slightly, for example 11,467 and 11,437 for `dseq`'s baseline `p2`. A third run in between is not comparable, because its maintenance missed one `dseq` page, as described under [The bloat scenarios built with the mandatory B-tree bloat tests](#the-bloat-scenarios-built-with-the-mandatory-b-tree-bloat-tests).

## Context Reviewed

- EXPLAIN itself: `ExplainQuery()`'s option parsing and checks, `NewExplainState()`'s defaults, `standard_ExplainOneQuery()`'s planning-buffer snapshot, `ExplainOnePlan()`'s instrumentation options, execution and output order, `ExplainPrintPlan()`, `ExplainPrintSettings()`, `ExplainNode()`'s naming, cost line, actual line, per-node-type quals and counters, buffer and worker output, `show_instrumentation_count()`, `peek_buffer_usage()`, `show_buffer_usage()`, `show_wal_usage()`, `show_tidbitmap_info()`, `show_sort_info()` and `ExplainIndexScanDetails()`.
- Instrumentation: `Instrumentation` and `BufferUsage` in `instrument.h`; `InstrAlloc`, `InstrStartNode`, `InstrStopNode`, `InstrEndLoop`, `InstrAggNode` and `BufferUsageAccumDiff`; `ExecProcNodeFirst`/`ExecProcNodeInstr`; `MultiExecBitmapIndexScan`'s own instrumentation; `ExecReScan`'s loop counting; parallel aggregation in `execParallel.c`; the `InstrCount*` macros and their callers in `execScan.c` and `nodeIndexonlyscan.c`.
- Buffer accounting: `PinBufferForBlock()`, `WaitReadBuffers()`, `ReadRecentBuffer()`, `MarkBufferDirty()`, `MarkBufferDirtyHint()`, `FlushBuffer()` and `GetVictimBuffer()`, `ReleaseAndReadBuffer()`, the local-buffer and `buffile.c` counters, `pgstat_io.c`'s timing accumulation, the bulk-read ring in `heapam.c`, and `bulk_write.c`'s bypass of shared buffers.
- Settings: `GUC_EXPLAIN`, `get_explain_guc_options()`, and the 60 flagged entries in `guc_tables.c` (59 in a standard build, because `optimize_bounded_sort` needs `DEBUG_BOUNDED_SORT`), plus the contexts and boot values of every setting the script changes.
- Planner: `get_relation_info()` (invalid and `indcheckxmin` indexes, index pages, tuples and tree height), `estimate_rel_size()` and `table_block_relation_estimate_size()`, `cost_seqscan()`, `cost_gather()`, `get_parallel_divisor()`, `cost_index()`, `index_pages_fetched()`, `cost_bitmap_heap_scan()`, `compute_bitmap_pages()`, `cost_bitmap_tree_node()`, `genericcostestimate()`, `btcostestimate()`, `var_eq_const()` and `get_variable_numdistinct()`, `get_actual_variable_range()`/`get_actual_variable_endpoint()`, `match_clause_to_indexcol()`, `match_opclause_to_indexcol()`, `check_index_only()`, and `create_bitmap_subplan()`'s cost copy.
- B-tree: `_bt_search()`, `_bt_getroot()`, `_bt_getrootheight()`, `_bt_readnextpage()`, the README's page-deletion and recycling sections, the build's fillfactor target, and `pgstatindex_impl()`.
- Maintenance: `lazy_scan_heap()`'s cleanup-lock path and `lazy_scan_noprune()`, the VERBOSE report lines, `index_update_stats()` and `index_build()`'s two calls, `acquire_sample_rows()` and `BlockSampler_Init()`, `elog.c`'s rule that `INFO` always reaches the client.
- Docs and tests: `ref/explain.sgml`, `perform.sgml`'s EXPLAIN sections, `ref/create_index.sgml` on invalid indexes, `ref/reindex.sgml`, `maintenance.sgml`'s routine reindexing, `pgstattuple.sgml`, and `src/test/regress/sql/explain.sql`. No shipped test asserts a plan's buffer counts; the regression test filters `Buffers:` lines out.
- History: commit `78cb2466f75`, which disabled the bitmap heap scan's skip-fetch optimization in the 17 branch (first in 17.5).
- The concept page [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md), for the protocol only.

## Evidence Map

| Claim | Evidence |
|---|---|
| In 17 only `COSTS` defaults on; `TIMING` and `SUMMARY` follow `ANALYZE`; `BUFFERS` and `SETTINGS` default off | [explain.c#NewExplainState](../../../../raw/postgres-17/src/backend/commands/explain.c#L368-L382), [explain.c:284-285](../../../../raw/postgres-17/src/backend/commands/explain.c#L284-L285), [explain.c:305-306](../../../../raw/postgres-17/src/backend/commands/explain.c#L305-L306), [ref/explain.sgml:154-208](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L154-L208) |
| `ANALYZE` executes the statement; output rows are discarded | [explain.c:665-709](../../../../raw/postgres-17/src/backend/commands/explain.c#L665-L709), [ref/explain.sgml:90-109](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L90-L109) |
| Planning buffers are the delta around `pg_plan_query()` and print only when non-zero in text | [explain.c:485-506](../../../../raw/postgres-17/src/backend/commands/explain.c#L485-L506), [explain.c:723-745](../../../../raw/postgres-17/src/backend/commands/explain.c#L723-L745), [explain.c#peek_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3702-L3737) |
| Actual time and rows are divided by loops; buffers are not | [explain.c:1841-1858](../../../../raw/postgres-17/src/backend/commands/explain.c#L1841-L1858), [explain.c:2280-2282](../../../../raw/postgres-17/src/backend/commands/explain.c#L2280-L2282) |
| Removed-row counts are divided by loops; `Heap Fetches` is not | [explain.c#show_instrumentation_count](../../../../raw/postgres-17/src/backend/commands/explain.c#L3622-L3645), [explain.c:1992-1994](../../../../raw/postgres-17/src/backend/commands/explain.c#L1992-L1994) |
| Parent counters include children: snapshots around the node call | [instrument.c#InstrStartNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L66-L80), [instrument.c#InstrStopNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L82-L128), [execProcnode.c#ExecProcNodeInstr](../../../../raw/postgres-17/src/backend/executor/execProcnode.c#L468-L485) |
| A Bitmap Index Scan's buffers cover only the index | [nodeBitmapIndexscan.c#MultiExecBitmapIndexScan](../../../../raw/postgres-17/src/backend/executor/nodeBitmapIndexscan.c#L48-L121) |
| Worker instrumentation and buffers are added into the leader's | [execParallel.c:1039-1043](../../../../raw/postgres-17/src/backend/executor/execParallel.c#L1039-L1043), [execParallel.c:1167-1172](../../../../raw/postgres-17/src/backend/executor/execParallel.c#L1167-L1172), [instrument.c#InstrAggNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L167-L196) |
| `Heap Blocks` counters are per process | [execnodes.h:1803-1824](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L1803-L1824), [nodeBitmapHeapscan.c:250-255](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L250-L255) |
| Hit, read, dirtied, written, local and temp are incremented at these points | [bufmgr.c#PinBufferForBlock](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1148-L1160), [bufmgr.c:1452-1463](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1452-L1463), [bufmgr.c:2590-2596](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2590-L2596), [bufmgr.c:5174-5180](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5174-L5180), [bufmgr.c:3975](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3975), [buffile.c:477-483](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L477-L483) |
| Re-reading the pinned block does not count again | [bufmgr.c#ReleaseAndReadBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2599-L2645), [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L125-L140) |
| Large seq scans use a ring; B-tree builds bypass shared buffers | [heapam.c:433-459](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L459), [bulk_write.c:12-17](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L12-L17), [nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149) |
| `Settings:` lists `GUC_EXPLAIN` settings that differ from their boot values | [guc.c#get_explain_guc_options](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L5338-L5432), [explain.c#ExplainPrintSettings](../../../../raw/postgres-17/src/backend/commands/explain.c#L807-L863), [explain.sql:79-89](../../../../raw/postgres-17/src/test/regress/sql/explain.sql#L79-L89) |
| `track_io_timing` and `autovacuum` are not `GUC_EXPLAIN` | [guc_tables.c#track_io_timing](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1420-L1428), [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457) |
| The planner skips invalid indexes | [plancat.c:256-267](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267), [ref/create_index.sgml:646-661](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L661) |
| An index clause needs an opfamily operator on the indexed key | [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2392-L2460) |
| An expression without statistics gets `1 / 200` equality selectivity | [selfuncs.c:441-449](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L441-L449), [selfuncs.c:5957-5966](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5957-L5966), [selfuncs.h:52](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L52) |
| `enable_indexscan = off` adds `disable_cost`; `enable_indexonlyscan = off` removes the path | [costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130), [costsize.c:606-607](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L606-L607), [indxpath.c:1738-1740](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1738-L1740) |
| Index pages are the live physical count; index tuples are the table's estimate | [plancat.c:463-500](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L500), [tableam.c:711-747](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747) |
| Index I/O is prorated over all index pages and priced at `random_page_cost` | [selfuncs.c:6717-6787](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6787) |
| B-tree descent costs depend on the tree height | [selfuncs.c:7086-7106](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7086-L7106), [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145) |
| An index-only scan's heap I/O scales with `1 - allvisfrac` | [costsize.c:714-747](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L714-L747), [tableam.c:749-760](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L749-L760) |
| A Bitmap Index Scan prints the index path's own cost | [createplan.c:3476-3485](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3476-L3485) |
| B-tree scans descend once and walk live leaves; deleted pages are unlinked | [nbtsearch.c#_bt_search](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L95-L179), [nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2180-L2243), [nbtree/README:279-282](../../../../raw/postgres-17/src/backend/access/nbtree/README#L279-L282) |
| VACUUM deletes only completely empty B-tree pages | [nbtree/README:235-245](../../../../raw/postgres-17/src/backend/access/nbtree/README#L235-L245), [maintenance.sgml:1032-1039](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1039) |
| A missed cleanup lock leaves dead tuples counted as missed | [vacuumlazy.c:929-961](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L929-L961), [vacuumlazy.c:1762-1769](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769), [vacuumlazy.c:659-668](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L659-L668) |
| A build leaves `BLCKSZ * (100 - fillfactor) / 100` free on each leaf | [nbtree.h:1138-1145](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145), [nbtsort.c:661-665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665) |
| `REINDEX` refreshes both `pg_class` rows; `ANALYZE` counts exactly when it reads every block | [index.c:3126-3135](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135), [analyze.c:1285-1289](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1285-L1289), [sampling.c#BlockSampler_Init](../../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L39-L55) |
| The 17 branch always fetches heap pages in a bitmap heap scan | [nodeBitmapHeapscan.c:186-219](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L186-L219), commit `78cb2466f75` |
| Every measured number | the script under [The script](#the-script), run as recorded under [Last run](#last-run) |

## Open Questions

- **The method is not "tested" in the concept page's sense.** The bloat scenarios follow the protocol's phases, maintenance assumption, no-defeat proofs, rule 2 and rule 3, and use its `REINDEX INDEX` oracle, but they are three fixtures, not the 113 numbered tests, and no rebuild decision is scored, so the verdict bands, `expected_stage` and `want_stage` do not apply. A rule such as "rebuild when the probe reads more than N buffers per 1,000 entries" would have to be run against the whole suite before it could be called tested.
- **The missed-tuple proof is an addition to the protocol.** The concept page's no-defeat rule lists five forbidden states and four proofs; a cleanup-lock miss is none of them, yet it left 136 drained rows and their index entries behind on one run of this script. Whether the concept page should add `tuples missed` to its proofs, and whether `CHECKPOINT` before the maintenance step is the right mitigation, is a question for that page, which this filing did not change.
- **The pinning process was not identified.** The one missed page was most likely pinned by a background buffer flush of a page the drain had dirtied; the run recorded the symptom, not the holder.
- **Read latency was not measured against the device.** The cold-read timing of about 3.6 microseconds per block is consistent with OS page-cache copies, but the script does not drop the OS cache or measure the device, so the page makes no claim about device I/O.
- **One block size, one platform, no concurrency.** All numbers come from 8 kB blocks on Linux x86_64 with no concurrent sessions; the protocol names the same limits.
- **The two log lines of the failed build.** The server log holds the expected `could not create unique index` error twice, from two process IDs; the page does not establish which processes they were.
- **Deleted pages are not recycled in these fixtures.** No insert followed the drain, so `dseq_k`'s 1,721 deleted pages stayed in the file. A workload that inserts after the drain would recycle some of them and shrink the planner's overestimate; that path was not measured.

## Source References

- [pgstatindex.c:304-326](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L326)
- [maintenance.sgml:1032-1039](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1039)
- [perform.sgml:143-152](../../../../raw/postgres-17/doc/src/sgml/perform.sgml#L143-L152)
- [pgstattuple.sgml:226-272](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L226-L272)
- [ref/checkpoint.sgml:31-37](../../../../raw/postgres-17/doc/src/sgml/ref/checkpoint.sgml#L31-L37)
- [ref/create_index.sgml:646-661](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L661)
- [ref/explain.sgml:185-201](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L185-L201)
- [ref/reindex.sgml:24](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L24)
- [heapam.c:433-459](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L433-L459)
- [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L125-L140)
- [vacuumlazy.c:929-961](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L929-L961)
- [nbtree/README:235-245](../../../../raw/postgres-17/src/backend/access/nbtree/README#L235-L245)
- [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L704)
- [nbtsearch.c#_bt_search](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L95-L179)
- [nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1149)
- [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L666-L713)
- [index.c:3126-3135](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135)
- [analyze.c:1185-1187](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1187)
- [explain.c#ExplainNode](../../../../raw/postgres-17/src/backend/commands/explain.c#L1807-L1872)
- [execAmi.c#ExecReScan](../../../../raw/postgres-17/src/backend/executor/execAmi.c#L75-L80)
- [execParallel.c:1039-1043](../../../../raw/postgres-17/src/backend/executor/execParallel.c#L1039-L1043)
- [execProcnode.c:409-412](../../../../raw/postgres-17/src/backend/executor/execProcnode.c#L409-L412)
- [execScan.c:247-255](../../../../raw/postgres-17/src/backend/executor/execScan.c#L247-L255)
- [instrument.c#InstrStartNode](../../../../raw/postgres-17/src/backend/executor/instrument.c#L66-L80)
- [nodeBitmapHeapscan.c:250-255](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L250-L255)
- [nodeBitmapIndexscan.c#MultiExecBitmapIndexScan](../../../../raw/postgres-17/src/backend/executor/nodeBitmapIndexscan.c#L48-L121)
- [nodeIndexonlyscan.c:161-170](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L161-L170)
- [costsize.c:343-346](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L343-L346)
- [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2392-L2460)
- [createplan.c:3476-3485](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3476-L3485)
- [plancat.c:463-500](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L500)
- [autovacuum.c#AutoVacuumingActive](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3235-L3241)
- [bufmgr.c:1452-1463](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1452-L1463)
- [buffile.c:477-483](../../../../raw/postgres-17/src/backend/storage/file/buffile.c#L477-L483)
- [bulk_write.c:12-17](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L12-L17)
- [pgstat_io.c:125-147](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_io.c#L125-L147)
- [selfuncs.c:1099-1136](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1099-L1136)
- [guc.c#get_explain_guc_options](../../../../raw/postgres-17/src/backend/utils/misc/guc.c#L5332-L5432)
- [guc_tables.c:1686-1699](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1686-L1699)
- [sampling.c#BlockSampler_Init](../../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L39-L55)
- [nbtree.h:1138-1145](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)
- [pg_amop.dat:15-168](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L15-L168)
- [instrument.h#BufferUsage](../../../../raw/postgres-17/src/include/executor/instrument.h#L24-L42)
- [execnodes.h:1803-1824](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L1803-L1824)
- [cost.h:24-28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L24-L28)
- [guc.h:215](../../../../raw/postgres-17/src/include/utils/guc.h#L215)
- [selfuncs.h:52](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L52)
- [explain.sql:25-30](../../../../raw/postgres-17/src/test/regress/sql/explain.sql#L25-L30)

## Navigation

- [v17/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 17 (unverified)](../indexing/leaf-density-vs-fragmentation-index-scan-io.md)
- [How the PostgreSQL 17 Query Planner Works: A Comprehensive Tutorial (unverified)](../query-planning/query-planner-comprehensive-tutorial.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](../indexing/reindex-index-concurrently.md)
- [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)](../indexing/btree-bloat-with-pgstatindex.md)
