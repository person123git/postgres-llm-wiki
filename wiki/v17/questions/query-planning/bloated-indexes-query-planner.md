---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
  - [Reviews after filing](#reviews-after-filing)
- [Answer](#answer)
  - [Planner Mechanisms](#planner-mechanisms)
  - [What The Planner Does Not See](#what-the-planner-does-not-see)
  - [Types Of Bloated Indexes](#types-of-bloated-indexes)
  - [How Density And Fragmentation Affect Different Queries](#how-density-and-fragmentation-affect-different-queries)
  - [Exact-Pin Measurements](#exact-pin-measurements)
  - [What Changed Since PostgreSQL 12](#what-changed-since-postgresql-12)
  - [Settings That Move The Boundary](#settings-that-move-the-boundary)
  - [Practical Interpretation](#practical-interpretation)
  - [Key Data Structures](#key-data-structures)
  - [Caller And Callee Boundary](#caller-and-callee-boundary)
  - [Build, Generated-Header, And Extension Boundary](#build-generated-header-and-extension-boundary)
  - [Tests And Explicit Test Absence](#tests-and-explicit-test-absence)
  - [Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead](#follow-up-when-a-gin-index-is-discarded-and-a-b-tree-is-used-instead)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, are there mechanisms to penalize bloated indexes in the query planner? If there are, give a comprehensive explanation with examples of types of bloated indexes and how leaf fragmentation or density affects them, and what changed since PostgreSQL 12.

Follow-up:

When might a GIN index be discarded by the query planner and a B-tree used instead?

### Reviews after filing

Review prompt, 2026-09-19, corrected form: `Follow AGENTS.md. In PostgreSQL 17, review: Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified).` The request as written read `follow agents.md, in postgresql 17 , review : # Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)`; the defects were `agents.md` for AGENTS.md, lowercase `postgresql`, a space before the comma and before the colon, the lowercase sentence opening `follow`, a stray Markdown heading marker `#` carried in with the pasted title, and no terminal period. The asker chose **correct and restate** and **report only**, read the findings, and then asked for them to be fixed (`fix issues`). That pass corrected nine source readings, replaced the history method, re-measured every number on the 17.11 pin from the script now filed under [Measurement Script](#measurement-script), and is recorded in the first 2026-09-19 entry of [log](../../../log.md).

Second review prompt, 2026-09-19, corrected form: `Follow AGENTS.md. In PostgreSQL 17, for the question "Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)", fix these issues:` followed by a list of twelve findings, fourteen counting the three folded into the last one. The request as written read `follow agents.md, in postgresql 17 , for question : # Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified) fix these issues :`; the defects were `agents.md` for AGENTS.md, lowercase `postgresql`, the lowercase sentence openings `follow` and `fix`, a space before the comma in `17 ,` and before each of the two colons, a stray Markdown heading marker `#` carried in with the pasted title, and no terminal punctuation. The pasted finding list carried two defects of its own: the numbering stopped at 11 and the twelfth item began mid-sentence at `planner.md:148) omits potentially stale cached metadata`, its opening clause missing, and item 2's `fa predict` was unquoted so it did not read as a command. The asker chose **correct and restate**, and chose to **replace** fixture `l3` rather than keep its degenerate form. All fourteen sub-findings were confirmed against the pin and fixed; that pass is the second 2026-09-19 entry of [log](../../../log.md).

Third review prompt, 2026-09-20: `Follow AGENTS.md. In PostgreSQL 17, for the question "Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)", fix these issues:` followed by a list of ten findings, one of them a P1 in the published measurement script. All ten were confirmed against the pin and fixed; none was a false report. The asker chose to **rebuild fixture D on independent columns** rather than keep it as a labelled selectivity-error example, so every fixture-D number was re-measured. That pass is the 2026-09-20 entry of [log](../../../log.md).

Fourth review prompt, 2026-09-23: `Follow AGENTS.md. In PostgreSQL 17, review the question "Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)".` The review re-ran the filed script unchanged, reproduced every number, and found the prose wrong or stale in about fifty places, fifteen of them about PostgreSQL behavior. The asker chose to **fix everything and re-run**: every finding was fixed, the answer was moved under `## Answer` and the closing sections into the template's order, and the script was edited in place and re-run from an empty sandbox. That pass is the 2026-09-23 entry of [log](../../../log.md).

Fifth review prompt, 2026-09-25: `Follow AGENTS.md. In PostgreSQL 17, review the question "Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)".` The review re-ran the filed script unchanged and reproduced every number, then checked the page claim by claim. 138 findings survived an adversarial check; none of them changed a recorded measurement. The asker chose to **fix everything and re-run**, to **add two fixtures** for the mechanisms the review found missing, a partial index whose growth the planner does not see (fixture Q) and the planning-time probe of an index's end (fixture E), to **drop every measured number the current script does not produce**, and to **add seven glossary entries checked on PostgreSQL 17 only**. That pass is the 2026-09-25 entry of [log](../../../log.md).

## Answer

Yes, but every mechanism is indirect. PostgreSQL 17 stores no [planner](../../../glossary.md#planner) field called `bloat`, `avg_leaf_density`, or `leaf_fragmentation`. [`pg_class`](../../../glossary.md#pg_class) carries only three size statistics, [`relpages`](../../../glossary.md#reltuples-and-relpages), `reltuples` and `relallvisible` ([pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)), and [`IndexOptInfo`](../../../glossary.md#indexoptinfo) carries only three index-size fields, `pages`, `tuples` and `tree_height` ([pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)). For a non-partial [B-tree](../../../glossary.md#b-tree), [bloat](../../../glossary.md#bloat) is penalized only through the four [cost](../../../glossary.md#cost) inputs in the first four rows below. For a [partial index](../../../glossary.md#partial-index), `tuples` is also derived from the live [block](../../../glossary.md#block) count, which changes how its page charge follows bloat (consequence 3 in [How the planner obtains its index-size inputs](#how-the-planner-obtains-its-index-size-inputs)). The fifth row is not a cost input at all: it is a read of the ends of a plain B-tree that row estimation makes at planning time. [Hash](../../../glossary.md#hash-index), [GiST](../../../glossary.md#gist) and [SP-GiST](../../../glossary.md#sp-gist), which also route through `genericcostestimate()`, get a subset of the four cost inputs, set out in [Which access methods share these mechanisms](#which-access-methods-share-these-mechanisms):

| Planner input | Where it comes from | Penalizes bloat? |
|---|---|---|
| `IndexOptInfo.pages` | a stored field: the live block count, from the [storage manager](../../../glossary.md#storage-manager) directly (non-partial) or through `estimate_rel_size()` (partial) | Yes, this is the main channel. It is live for a non-partial index; for a partial index the pro-rata page share lags until the index's `pg_class` row is rewritten |
| `IndexOptInfo.tree_height` | a stored field: `_bt_getrootheight()`, B-tree only | Yes, one explicit charge per extra level |
| `index->pages` in the cache model, and the touched-pages estimate `numIndexPages` in [parallel-worker](../../../glossary.md#parallel-query) selection | computed at costing time from `pages`: whole for the cache model, prorated by [selectivity](../../../glossary.md#selectivity) for worker counts | Yes, second-order |
| `ceil(index->pages * 0.3333333)` descent clamp | computed at costing time from the same `pages` value | Yes, and this channel is new in v17 |
| the live endpoint of a range comparison, read by `get_actual_variable_range()` | read at planning time from the end of a non-partial B-tree whose first column is the compared column; nothing is stored | Not a cost input. Dead entries at that end cost planning time, since a probe reads and skips them until it gives up after 100 [heap](../../../glossary.md#heap) pages, and they can leave a stale row estimate in place; see [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree) |

Only the first two are fields. `get_relation_info()` fills them while it has the index open ([plancat.c#get_relation_info-index-block](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)); the third and fourth rows are expressions the cost code derives from `pages` later, inside `genericcostestimate()`, `cost_index()` and `btcostestimate()`. The fifth row prices nothing: `ineq_histogram_selectivity()` asks for the endpoint while it estimates a row count ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)). The [access method](../../../glossary.md#access-method)'s `amcostestimate` turns the page count, the height and the descent clamp into index cost ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)); `cost_index()` itself feeds `index->pages` into its heap-side cache model and picks the worker count ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)).

The table is a B-tree-family summary, not an all-index one. [GIN](../../../glossary.md#gin) and [BRIN](../../../glossary.md#brin) also read `IndexOptInfo.pages` ([selfuncs.c:7674](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7674), [selfuncs.c:8063](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8063)), but both reopen the index during costing and read [metapage](../../../glossary.md#metapage) values that no `IndexOptInfo` field carries. Only GIN's values track bloat:

- `gincostestimate()` calls `ginGetStats()` ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711)). It takes the live [pending-list](../../../glossary.md#pending-list) page count when that count is below `index->pages`, and zero otherwise, and scales the last-[`VACUUM`](../../../glossary.md#vacuum) entry-page, data-page and entry counts by `index->pages / nTotalPages`. When the entry-page or entry count is zero, or the last-`VACUUM` total exceeds `index->pages` or is no more than a quarter of it, it derives all three from `index->pages` instead: it raises that page count to at least 10, subtracts the pending pages, counts 90% of the rest, rounded down, as entry pages holding 100 entries each, and counts the remainder as data pages ([selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767)). So GIN's cost also moves with inputs this table does not list.
- `brincostestimate()` also reopens the index, but `brinGetStats()` returns only `pagesPerRange` and the revmap page count ([brin.c#brinGetStats](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1638-L1654), [selfuncs.c#brincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8086-L8101)). Both are structural values, not bloat counters. The first is the number of heap pages per range, which the estimator divides into the heap's page count to count ranges. The second grows as the heap adds ranges, because each revmap page maps a fixed number of ranges and the revmap is extended when a new range falls past its last page ([brin_revmap.c#HEAPBLK_TO_REVMAP_BLK](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L35-L43), [brin_revmap.c#revmap_extend_and_get_blkno](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L494-L514)); the estimator uses it to split the page charge. `brincostestimate()` charges every index page on every scan, the revmap pages at `spc_seq_page_cost` as startup cost and the other `index->pages - revmapNumPages` pages at `spc_random_page_cost` ([selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258)). BRIN bloat therefore reaches the cost only through `index->pages`, the table's first row.

GIN's extra channel is the subject of the [follow-up](#follow-up-when-a-gin-index-is-discarded-and-a-b-tree-is-used-instead); [Which access methods share these mechanisms](#which-access-methods-share-these-mechanisms) sets out which of the four cost rows apply to which access method.

The consequences are sharply uneven, and measurements on an isolated server built from this exact pin make that concrete:

- A **broad scan** is penalized in direct proportion to page count. A 1,000,000-row index at 90.06% leaf density (2,745 blocks) costs `28480.42` for a full [index-only scan](../../../glossary.md#index-only-scan); its byte-for-byte logical twin at 9.62% density (26,411 blocks) costs `123144.43`.
- A **single point lookup** is almost blind to bloat for as long as the index has no more pages than the table has rows. The same two indexes both cost exactly `4.44`. The only bloat signal available to a single-[leaf-page](../../../glossary.md#leaf-page) search is tree height, worth `(tree_height + 1) * 50 * cpu_operator_cost`, which is `0.125` per extra level at default settings. Two things end the blindness, and they are independent. Once pages outnumber rows, `ceil(pages / tuples)` exceeds one: a 2,745-block index over 1,000 surviving rows prices the same kind of lookup at `12.29`, and at `4.29` after a rebuild. And once the lookup *repeats* — as the inner side of a [nested loop](../../../glossary.md#nested-loop-join) — the cache model can price bloat even though each iteration still touches one page. With few loops over a large index, bloat barely changes that charge; once loops times touched pages reach twice the index's pages, for an index that fits its share of the cache, the charge is capped at the whole index spread over the loops ([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951), [selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)). Fixture A's twins sit at and just below that cap: 50,000 loops cost `0.66` against `2.50` per loop on the dense and [fillfactor](../../../glossary.md#fillfactor)-10 indexes (2,745 against 26,411 blocks).
- **A partial index hides its own growth until its statistics are rewritten.** After ten rounds of `UPDATE`s of the indexed column in one transaction, fixture Q's partial index grew from 57 to 331 blocks and a plain index on the same table from 551 to 825. Before any [`ANALYZE`](../../../glossary.md#statistics) or `VACUUM`, a [bitmap scan](../../../glossary.md#bitmap-scan) of the plain index was charged all 825 pages. The partial index was charged 113 pages: its estimated 40,000 matching rows, a tenth of the table's doubled 400,000-row estimate, divided by its old density of about 357 entries per page. It was charged all 331 only after `ANALYZE` rewrote its `pg_class` row ([plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160), [selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019), [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)). See [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten).
- **Dead entries at the end of a B-tree move a row estimate, not a cost term.** Fixture E deletes the upper half of a 200,000-row table and neither vacuums nor analyzes it before the plans. For four plans in a row, `id > 199990` kept its stale 10-row estimate, because each plan's endpoint probe gave up after 100 heap pages of [dead rows](../../../glossary.md#dead-tuple) and fell back to the [histogram](../../../glossary.md#most-common-values-and-histogram)'s last bound, 200000 ([selfuncs.c#get_actual_variable_endpoint-limit](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6436-L6457), [selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413)). Each of those probes also marked the entries it passed dead in the index (22,590, then 22,600 three times) ([indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652)). The fifth plan reached the live maximum, 100000, and estimated 1 row. See [Dead entries at the end of a B-tree: the endpoint probe](#dead-entries-at-the-end-of-a-b-tree-the-endpoint-probe).
- **Leaf fragmentation is worth exactly zero.** A 1,148-block index at 49.87% `leaf_fragmentation` and a 744-block index at 0% differ in cost by `1616.00`, which is precisely `(1148 - 744) * random_page_cost`. There is no residual for fragmentation.
- **`avg_leaf_density` and planner cost can point in opposite directions.** Two 2,745-block indexes, each left with 100,000 of the same 1,000,000 rows, cost an identical `12730.42`, while [`pgstatindex`](../../../glossary.md#pgstatindex) reports 9.27% density for one and a healthy-looking 89.18% for the other (the second hides 2,465 deleted pages).

The v17 manual describes B-tree bloat operationally as an index that "contains many empty or nearly-empty pages" and recommends [`REINDEX`](../../../glossary.md#reindex) ([ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)), notes that a page keeps its space when "all but a few index keys on a page have been deleted" ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)), and separately says a freshly built B-tree is slightly faster because logically adjacent pages are usually physically adjacent ([maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1048-L1054)). `leaf_fragmentation` is a one-sided proxy for that last effect rather than a measure of it: it counts only leaf pages whose right-[sibling link](../../../glossary.md#sibling-link) points at a *lower* block number ([pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323)). A leaf chain that always moves forward but in thousand-block jumps scores 0% while being no more physically adjacent than one that moves backwards. Whatever part of the effect the metric does capture, the planner ignores all of it.

Since PostgreSQL 12, the cost formulas themselves barely moved: `index_pages_fetched()` and `cost_index()` are byte-identical to their `REL_12_0` text, and the bloat-relevant core of `genericcostestimate()` is unchanged. The main changes are (a) one new v17 channel through `index->pages`, which makes a bloated B-tree cost more than its dense twin on `= ANY` by giving the dense one a discount, (b) a great deal of nbtree work in v13 and v14 that reduces how much bloat exists to be penalized in the first place, and (c) the 100-heap-page limit on the planning-time endpoint probe, first in PostgreSQL 16 and back-patched. [What Changed Since PostgreSQL 12](#what-changed-since-postgresql-12) sets out each of them and the smaller changes.

### Planner Mechanisms

#### How the planner obtains its index-size inputs

`get_relation_info()` opens each index, copies the AM's `amcostestimate` into the `IndexOptInfo` ([plancat.c#amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L331-L332)), then fills the size estimates. For a non-partial, non-partitioned index it takes the current block count directly from the storage manager and locks the tuple estimate to the parent table's:

```c
if (info->indpred == NIL)
{
    info->pages = RelationGetNumberOfBlocks(indexRelation);
    info->tuples = rel->tuples;
}
```

([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486))

Three consequences follow directly:

1. **For a non-partial index, bloat is visible immediately, without `ANALYZE`.** `pages` is a live `smgrnblocks()` answer, not `pg_class.relpages` ([bufmgr.c#RelationGetNumberOfBlocksInFork](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4002-L4020)). Measured below: forging `pg_class.relpages` to `1` leaves the cost unchanged, and fixture Q's plain index was charged all 825 of its live pages before any `ANALYZE`. A partial index reads the same live `pages`, but its page charge lags; see consequence 3.
2. **Removing index entries does not lower `tuples`.** For a non-partial index, `tuples` is the *table's* row estimate, so taking entries out of the index never lowers it on the index's account. The `pages / tuples` ratio rises as index pages accumulate, and it also rises when the table's estimated row count falls while the index keeps its pages. Fixture F shows this: its index keeps 2,745 blocks while the table's row estimate falls from 1,000,000 to 1,000, and its one-row lookup rises from `4.44` to `12.29`.
3. **Partial indexes take a different path, and it hides their growth.** `estimate_rel_size()` still reports live blocks as `*pages`, but sets `*tuples` to the density recorded in `pg_class`, `reltuples / (relpages - 1)`, times the live block count less the metapage ([plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)). It uses that recorded density only while `reltuples >= 0` and `relpages > 1`, so that the row counted at least one page besides the metapage. Otherwise it invents a density from the column widths, and the charge described below rests on that invented density instead ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)). A partial B-tree reaches that fallback, until `ANALYZE` or a `VACUUM` that writes the index's row replaces it, when it was created while no row matched its predicate, when a `REINDEX` or `TRUNCATE` rebuilt it with no matching row, or when [`pg_upgrade`](../../../glossary.md#pg_upgrade) created it. A new index's row starts at `reltuples = 0`, not `-1`, so a build with no entries still writes its counts ([index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024), [relcache.c#RelationBuildLocalRelation-rd_rel](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3656-L3659), [index.c#index_update_stats-write](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2923)), and an empty B-tree build is the metapage alone, so the row records `relpages = 1` ([nbtsort.c#_bt_load-first-page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1287-L1290), [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1063-L1128)). A `REINDEX` or `TRUNCATE` first resets the row to `relpages = 0` and `reltuples = -1`, and a rebuild with no entries then leaves those counts unwritten; a build during binary upgrade never writes them, so an index `pg_upgrade` created keeps `relpages = 0` ([relcache.c#RelationSetNewRelfilenumber-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3948-L3954), [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)); see [v14: reltuples turns negative for never-analyzed relations](#v14-reltuples-turns-negative-for-never-analyzed-relations). `get_relation_info()` then clamps the resulting `tuples`, recorded or invented, to the table's row estimate ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)). `btcostestimate()`, however, takes its tuple count from selectivity times the *table's* rows, not from `index->tuples` ([selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019)). In `ceil(numIndexTuples * pages / tuples)` ([selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)) the live page count therefore nearly cancels out, and a partial index is charged about its estimated matching rows divided by its last recorded density. Pages it gained since its `pg_class` row was last written reach that charge only in proportion to the table's own estimated growth; the table's row estimate is the heap's recorded density times its live block count ([tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747)). The charge catches up when `ANALYZE` ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)), a `VACUUM` that writes the index's row, or a rebuild refreshes that row, or once `tuples` reaches the table's row estimate and the clamp binds. Fixture Q measures it. Its partial index grew from 57 to 331 blocks while the heap doubled, and was charged `ceil(40000 * 331 / 117857) = 113` pages, then all 331 after `ANALYZE`. Here 40,000 is the predicate's tenth of the table's doubled 400,000-row estimate, and 117,857 is `rint(20000 / 56 * 330)`, the recorded 20,000 tuples over 56 pages scaled to 330, each count less the metapage; see [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten). The cache model, the descent clamp and the B-tree height still read live values, and because `tuples` grows with `pages`, so does the `ceil(log2(tuples))` comparison charge ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091)).

For B-trees only, the height comes from the metapage while the index is open; every other AM is left at `-1` ([plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500)). That read is not necessarily a fresh one — it can be answered from a per-backend cached copy of the metapage, which is the third caveat in [2. B-tree height carries an explicit anti-bloat charge](#2-b-tree-height-carries-an-explicit-anti-bloat-charge). [Partitioned indexes](../../../glossary.md#partitioned-index) have no storage, so `pages` and `tuples` are set to zero and `tree_height` to `-1` ([plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)).

#### 1. Physical page count enters index cost

`genericcostestimate()` estimates touched index pages as a pro-rata share of the whole index, and its comment states it counts only leaf pages, ignoring the metapage and upper levels:

```c
if (index->pages > 1 && index->tuples > 1)
    numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
else
    numIndexPages = 1.0;
```

([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732))

For a single scan it charges `spc_random_page_cost` per touched page ([selfuncs.c#genericcostestimate-single-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787)); for repeated scans it first runs the page count through the [Mackert-Lohman](../../../glossary.md#mackert-lohman-formula) cache model ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)).

This is the dominant penalty, and it scales linearly with bloat for anything that reads a meaningful fraction of the index; for a partial index, it does so once the index's statistics catch up (consequence 3 above). Two guards blunt it, and both are easy to state too loosely.

The first is the `index->pages > 1 && index->tuples > 1` test. It is usually read as a floor that stops a tiny index being charged a fraction of a page, and it is that, but it is also a ceiling, because it replaces the pro-rata formula with a flat `numIndexPages = 1.0` whenever *either* operand fails ([selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)). For a non-partial index `tuples` is the *table's* row estimate, not the index's entry count, so a table the planner estimates at one row or fewer prices a scan of an index of any size at a single page. [The pages-outnumber-rows guard at its limit](#the-pages-outnumber-rows-guard-at-its-limit) measures that on a 551-block index.

The second is `ceil()`. It collapses a lookup of `k` estimated rows to one page while `k * pages <= tuples`, so a one-row lookup stays at one page as long as the index has no more pages than the table has rows ([selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)). Past that point it fails: a one-row lookup is charged `ceil(pages / tuples)` pages, which is the shape a drained queue table leaves behind and is measured in [A mostly-empty index: the fast root drops and pages outnumber rows](#a-mostly-empty-index-the-fast-root-drops-and-pages-outnumber-rows). A lookup of more rows crosses sooner: fixture P's 200-row `tag = 7` bitmap scans over 200,000 rows are charged one page at 543 blocks and two at 1,173 blocks, `ceil(200 * 543 / 200000) = 1` against `ceil(200 * 1173 / 200000) = 2`, which costs `5.92` against `9.92`.

Neither guard survives repetition. When the scan runs more than once — a nested-loop inner scan or a [`ScalarArrayOpExpr`](../../../glossary.md#scalararrayopexpr) — `genericcostestimate()` takes the `num_scans > 1` branch, multiplies `numIndexPages` by the scan count and runs the product through the Mackert-Lohman model with `N = T = index->pages` ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)). The whole index enters the estimate as the page universe even when `numIndexPages` is still 1, which is why a repeated one-page lookup can be priced by bloat and a lone one is not. How much depends on the loop count. The formula's `2TNs / (2T + Ns)` is close to `Ns` while the pages fetched across all loops stay well below the index size `T`, so few loops over a large index are charged about one page each, whatever the index size. Once they reach the index size, the charge approaches the whole index spread over the loops. It is capped at `T` unless the query's table pages plus this index exceed [`effective_cache_size`](../../../glossary.md#effective_cache_size), in which case repeated reads can push it past `T` ([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)). At the default `effective_cache_size`, fixture A's 50,000-loop nested loop is the case where the loops reach the index size; the same join at `effective_cache_size = 64MB` is the past-`T` case. See [Index pages in the cache model](#index-pages-in-the-cache-model).

#### 2. B-tree height carries an explicit anti-bloat charge

`btcostestimate()` adds two CPU charges after delegating to `genericcostestimate()`. The first is roughly `log2(N)` comparisons ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091)). The second charges CPU for every page the descent passes through, and bloat is one of the two reasons the source gives for it:

```c
descentCost = (index->tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost;
```

([selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106))

The in-tree comment's first reason is that descending through upper pages costs some CPU even though it is charged no I/O; its second is explicit: "Moreover, if we had no such charge at all, bloated indexes would appear to have the same search cost as unbloated ones, at least in cases where only a single leaf page is expected to be visited." `DEFAULT_PAGE_CPU_MULTIPLIER` is `50.0` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), so each extra level costs `50 * cpu_operator_cost` = `0.125` at the default `cpu_operator_cost` of `0.0025` ([cost.h:28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L28)).

Three caveats matter.

First, this is a *level* charge, not a density charge: it changes only when the B-tree gains or loses a level.

Second, the planner's height is the **[fast-root](../../../glossary.md#fast-root)** level, not the true root level. `_bt_getrootheight()` returns `btm_fastlevel` ([nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)), and [page deletion](../../../glossary.md#b-tree-page-deletion) can lower `btm_fastlevel` in place ([nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)); the nbtree README explains the fast-root idea and states that tree height can never *decrease* by page deletion alone ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)). `pgstatindex` reports `btm_level` as `tree_level`: it reads `btm_level` from the metapage ([pgstatindex.c#metapage](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265)), returns it as the second output value ([pgstatindex.c:351](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L351)), and the default extension version 1.5 declares that column `tree_level` ([pgstattuple--1.4--1.5.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L24), [pgstattuple.control:3](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L3)). So `pgstatindex.tree_level` and the planner's `tree_height` are not guaranteed to be the same number. Fixture F below measures them apart: `tree_level` 2 against `fastlevel` 1.

Third, the number the planner gets can be stale, though a VACUUM that lowers the fast root normally ends the staleness at the next plan. `_bt_getrootheight()` answers from the copy of `BTMetaPageData` cached in `rel->rd_amcache` whenever one exists, and otherwise reads the metapage and caches it; an index with no root yet is never cached, so each call re-reads the metapage and returns `0` ([nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)). Scans and inserts fill the same per-backend cache through `_bt_getroot()` and `_bt_metaversion()` ([nbtpage.c#_bt_getroot-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L523-L528), [nbtpage.c#_bt_metaversion-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L773-L775)). The function's comment says the staleness is deliberate: "Since it's only an estimate, slightly-stale data is fine, hence we don't worry about updating previously cached data".

Three events discard the cached copy:

- An [invalidation](../../../glossary.md#invalidation-message) of the index's [relcache](../../../glossary.md#relcache) entry frees `rd_amcache`. `RelationCacheInvalidateEntry()` hands the entry to `RelationFlushRelation()` ([relcache.c#RelationCacheInvalidateEntry](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2975-L2985)), which for an index not created or given a new relfilenumber in the current transaction calls `RelationClearRelation()` ([relcache.c#RelationFlushRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2916-L2924)), and that frees the cache before it decides whether to rebuild or discard the entry ([relcache.c#RelationClearRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2600-L2603)).
- `_bt_getroot()` throws it away on the spot when the cached fast root fails its checks — the page must not be deleted or half-dead (`P_IGNORE`, [nbtree.h:225](../../../../raw/postgres-17/src/include/access/nbtree.h#L225)), must be at the cached level, and must be alone on that level — and then re-reads the metapage ([nbtpage.c#_bt_getroot-stale-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L380-L403)).
- `_bt_gettrueroot()` flushes it unconditionally before it starts, because "if we are here it suggests our cache is out-of-date anyway" ([nbtpage.c#_bt_gettrueroot-flush](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L592-L600)).

The first of them does reach a backend that only plans. When VACUUM cleans up an index whose counts it measured exactly, it writes the index's `relpages` and `reltuples` back to `pg_class` through `vac_update_relstats()` ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [vacuumlazy.c:512-513](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L512-L513)). `ANALYZE` calls the same function for every index of the table it analyzes ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)). Either write is an in-place update ([vacuum.c#vac_update_relstats-inplace](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1545-L1548)). An in-place update registers a relcache invalidation for the index's own entry before it locks the tuple ([heapam.c:6668](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6668), [inval.c#CacheInvalidateHeapTupleCommon-pg_class](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L1383-L1392), [inval.c:1447](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L1447)), sends it once the write completes ([heapam.c#heap_inplace_update_and_unlock-send](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6888-L6892)), and discards it if the caller cancels ([heapam.c#heap_inplace_unlock](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6914-L6920)). The planner opens every index under a lock ([plancat.c:253](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L253)), and taking a lock the transaction does not already hold absorbs pending invalidations first ([lmgr.c#LockRelationOid-accept](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L134-L138)). So the next plan after such a VACUUM or `ANALYZE` reads a fresh metapage, which is what the nbtree README expects: "we can expect a relcache flush will discard the cached metapage before long, since a VACUUM that's moved the fast root pointer can be expected to issue a statistics update for the index" ([README#metapage-cache](../../../../raw/postgres-17/src/backend/access/nbtree/README#L776-L790)).

The stale window is real where no invalidation arrives:

- A [root split](../../../glossary.md#page-split) raises the level by writing the metapage alone, with no [catalog](../../../glossary.md#catalog) change ([nbtinsert.c#_bt_newlevel-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2515-L2519)).
- Splitting the only page on a level below the true root, which may be the fast root, moves the fast root up the same way. Inserting the new downlink into the parent rewrites `btm_fastroot` and `btm_fastlevel` in the metapage alone ([nbtinsert.c#_bt_insertonpg-fastroot](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1250-L1295)). Single-page levels below the root are what massive deletions leave behind, and the README names this split as one of the two events that move the fast root ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)); fixture F's index is in that state.
- `vac_update_relstats()` writes nothing for an index whose `relpages`, `reltuples` and `relallvisible` are all unchanged ([vacuum.c#vac_update_relstats-dirty](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1446-L1461)), and it then discards the invalidation it registered ([vacuum.c#vac_update_relstats-inplace](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1545-L1548), [heapam.c#heap_inplace_unlock](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6914-L6920)).
- VACUUM skips the write when the index AM returned no statistics or only estimated counts, or when [index cleanup](../../../glossary.md#index_cleanup) was off ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [vacuumlazy.c:512-513](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L512-L513)). For a B-tree that covers every VACUUM that did not call `btbulkdelete()`, because `btvacuumcleanup()` then either returns no statistics or marks its counts as estimated ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)). That includes a VACUUM that collected no dead item identifiers, which never enters [index vacuuming](../../../glossary.md#index-vacuuming) ([vacuumlazy.c:1051-1052](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1051-L1052)), and one that bypassed index vacuuming ([vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)).

In those cases a backend keeps pricing from its old copy until something else invalidates it or a scan trips `_bt_getroot()`'s check, and two backends can price the same index differently. Every measurement below is planned in a `psql` session opened after the last change to the index it prices, so none of them reads a stale height.

#### 3. Index pages enter cache modeling and parallel worker counts

`index_pages_fetched()` prorates `effective_cache_size` across "all the tables in the query and the index currently under consideration":

```c
total_pages = root->total_table_pages + index_pages;
b = (double) effective_cache_size * T / total_pages;
```

([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951))

`root->total_table_pages` counts only non-dummy *table* pages ([allpaths.c#total_table_pages](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216)), so each call site on the B-tree-family path adds the index's own page count separately. `gincostestimate()` passes its entry-page or data-page count instead ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974)). `cost_index()` passes `index->pages` on each of its three `index_pages_fetched()` calls, two for repeated scans and one for the normal case; the normal case's perfectly-correlated estimate is computed without the cache model ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)). `genericcostestimate()` passes it for repeated index scans ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)), and `compute_bitmap_pages()` passes `get_indexpath_pages()` for repeated bitmap scans ([costsize.c#compute_bitmap_pages-repeated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476)). A bloated index therefore claims a larger notional share of cache for itself while shrinking the share available to the heap.

Parallel worker selection reads a different number with a similar name. The `index_pages` variable that `cost_index()` hands to `compute_parallel_worker()` is the access method's `*indexPages` output, which `btcostestimate()` sets to `numIndexPages`: the pages this scan is expected to touch, not the size of the index ([costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772), [selfuncs.c#btcostestimate-outputs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7206-L7210)). The source comment says workers are computed "based on number of index pages fetched". An index-only scan passes only that estimate; a plain [index scan](../../../glossary.md#index-scan) also passes its heap-page estimate, and the smaller of the two worker counts wins.

The page-based calculation is not unconditional. A [reloption](../../../glossary.md#storage-parameter) replaces it outright, and a second guard limits only its minimum-size rejection ([allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)):

- **A `parallel_workers` reloption on the table wins outright.** `compute_parallel_worker()` tests `rel->rel_parallel_workers != -1` first and, when it is set, uses that number and skips the whole `else` block — no size threshold, no tripling ramp, no page counts of any kind ([allpaths.c#compute_parallel_worker-reloption](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4208-L4213)). The only limit left is the caller's maximum, which `cost_index()` passes as `max_parallel_workers_per_gather` ([allpaths.c:4276](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4276), [costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)). The field is filled from the table's reloption in `get_relation_info()` ([plancat.c:205](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L205)), which `reloptions.c` declares on `RELOPT_KIND_HEAP` with a `-1` default and [`ShareUpdateExclusiveLock`](../../../glossary.md#shareupdateexclusivelock) ([reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382)). So on a table with `parallel_workers` set, index bloat moves no worker count at all.
- **The minimum-size rejection applies to plain base relations only.** The `return 0` for an index-page estimate below `min_parallel_index_scan_size`, or a heap-page estimate below `min_parallel_table_scan_size`, is guarded on `rel->reloptkind == RELOPT_BASEREL`, with the in-tree reason that an inheritance child should still get a parallel [path](../../../glossary.md#path) because "when combined with all of its inheritance siblings it may well pay off" ([allpaths.c#compute_parallel_worker-threshold](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227)). A partition small enough to be rejected on its own is therefore not rejected, though the tripling ramp below still sizes its workers.

When neither applies, the function rejects a parallel path whose index-page estimate is below `min_parallel_index_scan_size`, or whose heap-page estimate is below `min_parallel_table_scan_size`; a plain index scan passes a heap-page estimate and an index-only scan does not. Otherwise it adds one worker each time the estimate triples. Because `numIndexPages` is a pro-rata share of `index->pages`, bloat still raises the worker count, but in proportion to the scan's selectivity rather than to the whole index. Bloat can therefore change the *shape* of a plan, not only its price. Every fixture on this page is a plain, unpartitioned table with no `parallel_workers` reloption, so the measurements below are all of the unexempted path.

#### 4. v17 only: index pages cap the ScalarArrayOp descent estimate

`btcostestimate()` clamps the number of estimated array descents to one third of the index's physical pages:

```c
num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333));
num_sa_scans = Max(num_sa_scans, 1);
```

([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042))

`num_sa_scans` then multiplies both descent charges, and is handed to `genericcostestimate()` through `GenericCosts` ([selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073)), where it multiplies the page charge: `numIndexPages * num_sa_scans` pages go through the Mackert-Lohman formula ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)). The per-tuple CPU charge is not multiplied by the descent count, because `btcostestimate()` first divides the tuple count by the same number ([selfuncs.c:7064](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7064)) before `genericcostestimate()` multiplies it back ([selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810)). It still differs from the undivided charge in two ways. `rint()` rounding moves it up or down; a five-element array on a 3-descent clamp would be charged `rint(5 / 3) * 3 = 6` tuples, and fixture I's 8-block index, clamped to 3 descents, is charged `rint(10 / 3) * 3 = 9` tuples on the ten-element array and `rint(4 / 3) * 3 = 3` on the four-element one, which is why its `352.04` at three elements and `352.05` at four differ only by one heap row's `cpu_tuple_cost`. And a floor of one tuple per descent ([selfuncs.c#genericcostestimate-tuple-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6707-L6715)) makes the term `num_sa_scans * (cpu_index_tuple_cost + qual_op_cost)` whenever descents outnumber estimated rows, so there a bloated index's higher clamp raises it too. The clamp was introduced with v17's native `ScalarArrayOpExpr` execution and does not exist in `REL_12_0`; see [v17: index pages now cap ScalarArrayOp descents](#v17-index-pages-now-cap-scalararrayop-descents). Its effect on bloat is counter-intuitive but real: a bloated index has a *higher* cap, so it is charged for more descents than its dense twin on the identical query.

#### 5. Planning-time endpoint probes read the ends of a B-tree

This one is not a cost input, and no cost term charges it. It is a read that row estimation makes at planning time. Dead entries at either end of a plain B-tree make that read longer, and they can leave a stale row estimate in place, which then feeds every cost above it.

- **When it runs.** The range-comparison estimators `scalarltsel()`, `scalarlesel()`, `scalargtsel()` and `scalargesel()` pass a comparison against a constant to `scalarineqsel()` ([selfuncs.c#scalarineqsel_wrapper](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1397-L1503)), which calls `ineq_histogram_selectivity()` ([selfuncs.c:690](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L690)); the fixed-prefix estimate for a pattern match such as `LIKE 'foo%'` calls it too ([like_support.c#prefix_selectivity](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1213-L1270)). So does merge-join costing: `initial_cost_mergejoin()` reaches `mergejoinscansel()` through `cached_scansel()` ([costsize.c:3577](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L3577), [costsize.c:4016](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L4016)). `mergejoinscansel()` takes each side's minimum and maximum from `get_variable_range()`, which reads them from the column's `pg_statistic` histogram and most-common values and has its own call to the probe compiled out ([selfuncs.c#mergejoinscansel-ranges](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3141-L3156), [selfuncs.c#get_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5969-L6098), [selfuncs.c#get_variable_range-not-used](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5993-L6003)). It then calls `scalarineqsel()` four times, comparing each side's column against one of the other side's recorded extremes ([selfuncs.c#mergejoinscansel-scalarineqsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3164-L3203)). When the two columns span similar ranges, those constants fall in or beyond the end bins, so planning a merge join on an indexed column can run the probe too. When `ineq_histogram_selectivity()`'s binary search is about to compare against the first or last histogram bound, it first tries to replace the bound with the column's true current minimum or maximum. The source comment says this "ameliorates misestimates when the min or max is moving as a result of changes since the last ANALYZE" ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)). A constant in or beyond the first or last histogram bin reaches that bound; a two-bound histogram has both ends probed every time.
- **Which index it reads.** `get_actual_variable_range()` takes the first index on the table that is a B-tree, not partial, not hypothetical, and whose first column matches the compared expression, collation and sort operator; it gives up at once on a partitioned table's parent ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6157-L6331)). A partial index is never probed, and neither is a GIN, GiST, SP-GiST, hash or BRIN index.
- **What it reads.** `get_actual_variable_endpoint()` walks the index from that end with the index-only-scan machinery, under `SnapshotNonVacuumable`. An entry whose heap page is [all-visible](../../../glossary.md#visibility-map) is accepted without a heap visit; any other entry costs a heap visit. Rows that no [snapshot](../../../glossary.md#snapshot) can still see are skipped. Recently dead and uncommitted rows are accepted, so an extreme deleted while an older snapshot still needs it is still returned as the endpoint ([selfuncs.c#get_actual_variable_endpoint-snapshot](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6376-L6386), [selfuncs.c#get_actual_variable_endpoint-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6458)).
- **The bound.** Each dead entry that points at a different heap page from the previous one counts one visited page. Once the count passes `VISITED_PAGES_LIMIT`, 100, the probe gives up ([selfuncs.c#get_actual_variable_endpoint-limit](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6436-L6457)), and the caller keeps "whatever extremal value is recorded in pg_statistic" ([selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413)). The price is planning time and the row estimate, never a cost term.
- **Self-healing.** Every entry the probe skips because its whole [HOT](../../../glossary.md#hot) chain is dead is marked killed. `index_fetch_heap()` sets `kill_prior_tuple` ([indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652)); the B-tree records it and marks the entries dead before it leaves the leaf page or ends the scan ([nbtree.c#btgettuple-kill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L226-L245), [nbtree.c#btendscan-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L424-L426)). The next probe skips those entries without a heap visit, which the source comment calls a crucial point of that snapshot choice ([selfuncs.c#get_actual_variable_endpoint-killed](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6388-L6397)). Successive plans therefore each get up to 100 heap pages further, until one reaches a live entry and the estimate uses the true endpoint. A transaction started during recovery neither sets nor honors killed entries ([genam.c#RelationGetIndexScan-recovery](../../../../raw/postgres-17/src/backend/access/index/genam.c#L107-L119), [nbtsearch.c:1721](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L1721), [nbtsearch.c:1850](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L1850)), so on a [hot standby](../../../glossary.md#hot-standby) each probe walks the same dead entries again.
- **What it changes.** A probe that reaches a live entry moves the histogram's end and so corrects the row estimate; a probe that gives up leaves the stale estimate in place. Fixture E measures both: four plans kept a stale 10-row estimate while each marked 100 heap pages' worth of entries dead, and the fifth reached the live maximum and estimated 1 row; see [Dead entries at the end of a B-tree: the endpoint probe](#dead-entries-at-the-end-of-a-b-tree-the-endpoint-probe).
- **History.** The 100-page limit is commit `9c6ad5eaa9` (2022-11-22), first in PostgreSQL 16 and back-patched. Its message says the probes will "gradually whittle down the problem" by setting a few more killed bits in each planning attempt, "so eventually we'll reach a good state (barring further deletions), even in the absence of VACUUM". See [v16, back-patched: the endpoint probe gives up after 100 heap pages](#v16-back-patched-the-endpoint-probe-gives-up-after-100-heap-pages).

#### Which access methods share these mechanisms

Four core AMs route through `genericcostestimate()`: B-tree, hash, GiST and SP-GiST ([selfuncs.c:7073](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7073), [selfuncs.c:7221](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7221), [selfuncs.c:7265](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7265), [selfuncs.c:7320](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7320)). So does [contrib](../../../glossary.md#contrib) `bloom`, the only other caller in the tree, and it is the most exposed of the five: `blcostestimate()` sets `numIndexTuples = index->tuples` because "We have to visit all index tuples anyway", so `numIndexPages` comes out as all of `index->pages` and a single bloom scan is charged for every page of the index ([blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42), [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)). A repeated bloom scan, on the inner side of a nested loop, goes through the same cache model as every other `genericcostestimate()` caller: the `index->pages * loop_count` fetches run through the Mackert-Lohman formula with the index as the page universe, and the result is divided by the loop count ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)). With two or more loops over an index that fits its prorated cache share, that caps the fetches at the index's own page count, so the whole index is charged once and spread across the loops ([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)). GIN and BRIN have their own models and never call it; both source comments say their search behavior is "completely different from other index types" ([selfuncs.c#gincostestimate-header](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L7671), [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8052-L8061)).

The height charge is B-tree-only in the sense that only B-trees supply a *measured* height. GiST and SP-GiST fill the `-1` themselves by assuming a fanout of 100 and taking `log100(index->pages)`, then apply the same formula "calculated the same as for btrees" ([selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7256-L7308), [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7311-L7363)); their descent charge therefore tracks physical page count rather than real height. `hashcostestimate()` adds no descent charge at all ([selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7213-L7253)).

Mapped onto the four cost inputs in the answer's table:

| Input | B-tree | Hash | GiST, SP-GiST | contrib `bloom` |
|---|---|---|---|---|
| `pages` | yes | yes | yes | yes, the whole index on a single scan |
| `tree_height` | measured, fast root | none | derived from `pages` | none |
| Cache model on repeated scans | yes | yes | yes | yes |
| Worker count from `numIndexPages` | yes | no | no | no |
| Descent clamp | yes | no | no | no |

The worker row is B-tree's alone because every other AM here sets `amcanparallel = false` ([hash.c:75](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L75), [gist.c:77](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L77), [spgutils.c:62](../../../../raw/postgres-17/src/backend/access/spgist/spgutils.c#L62), [blutils.c:124](../../../../raw/postgres-17/contrib/bloom/blutils.c#L124)) and so never gets a [partial index path](../../../glossary.md#partial-path). The descent clamp exists only in `btcostestimate()` ([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042)). BRIN, which also sets `amcanparallel = false` ([brin.c:265](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L265)), charges every index page on every scan through its own model; see [Non-B-tree bloat](#non-b-tree-bloat). The planning-time endpoint probe in [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree) is not a cost input, and no AM but B-tree takes part in it: `get_actual_variable_range()` skips every index that is not a B-tree, is partial, or is hypothetical ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)).

Every `pgstatindex` metric discussed on this page is B-tree-only: the function reads B-tree page structures and rejects any other relation with `is not a btree index` ([pgstatindex.c#pgstatindex_impl-btree-check](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228)), which its [regression test](../../../glossary.md#regression-test) asserts for a GIN and a hash index ([sql/pgstattuple.sql#wrong-index-type](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L63)).

### What The Planner Does Not See

| Bloat signal | Planner-visible in v17? | Effect |
|---|---|---|
| More physical index blocks for the same useful keys | Yes | Raises `numIndexPages` through `index->pages / index->tuples` |
| An extra B-tree level | Yes | One `50 * cpu_operator_cost` charge per level, fast-root based |
| Deleted, half-dead, or empty pages | Yes, but as ordinary pages | They are inside `RelationGetNumberOfBlocks()`, so the planner charges them like live pages, whatever a scan does with them. A deleted page has been unlinked from its siblings, so no scan that starts after the unlink reaches it ([nbtpage.c#unlink-siblings](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2612)). A half-dead page is still in the sibling chain, so a scan that steps right reads it and skips it ([README#half-dead](../../../../raw/postgres-17/src/backend/access/nbtree/README#L247-L259), [nbtsearch.c#step-right](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2207-L2219)). An empty page that is not being deleted, such as the rightmost page of a level, which is never deleted ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)), is read like any other. The estimate's own guards can still collapse the charge to one page, which [The pages-outnumber-rows guard at its limit](#the-pages-outnumber-rows-guard-at-its-limit) measures |
| Low `avg_leaf_density` from `pgstatindex` | Not directly | Only matters when it produces extra pages or an extra level |
| High `leaf_fragmentation` from `pgstatindex` | No | Measured contribution below is exactly zero |
| Free space recorded in the index [FSM](../../../glossary.md#free-space-map) | No | Nothing on the cost path reads the FSM. The block count is the main [fork](../../../glossary.md#fork)'s alone ([bufmgr.h:280-281](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L280-L281)), so a page the FSM lists as free is still a block of the index and is charged like any other. That no optimizer file, `selfuncs.c` or contrib `bloom`'s `blcost.c` calls into the FSM is a grep result, recorded in the [Evidence Map](#evidence-map) |
| Index entries removed by VACUUM | No | For a non-partial index `tuples` stays pinned to the table estimate |
| Growth of a partial index since its `pg_class` row was last written | Only in proportion to the table's growth | `get_relation_info()` gives a partial index its live block count as `pages` and takes `tuples` from `estimate_rel_size()`, which multiplies the density recorded in `pg_class` by the live block count, less the metapage. `get_relation_info()` then caps that estimate at the table's rows ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)). The cost code sizes the scan from the table's row estimate ([selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019), [selfuncs.c:6695](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6695)), so in `ceil(numIndexTuples * pages / tuples)` the index's own growth nearly cancels out, because it raises `pages` and `tuples` together, and only the table's growth moves the charge, for as long as the tuple estimate stays below the cap. The growth arrives all at once when `ANALYZE` or `VACUUM` rewrites the index's `relpages` and `reltuples` ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)). Fixture Q's partial index at 331 live blocks was charged 113 pages, then 331 after `ANALYZE`; see [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten) |
| Where inside the index the useful entries sit | No | `numIndexPages` is a flat pro-rata share. The endpoint probe in the next row reads an index's two ends for a row estimate, not for a cost |
| Dead entries at either end of a B-tree | The probe is not priced; the pages holding the entries are, as ordinary pages (first row). At plan time the probe reads the entries and can change the row estimate | When a range comparison's binary search over the histogram reaches the first or last bound, `ineq_histogram_selectivity()` asks `get_actual_variable_range()` for the column's current minimum or maximum ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)). That function reads only a non-partial, non-hypothetical B-tree whose first column is the compared column ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)). It scans from that end and steps over entries whose heap rows are dead to every snapshot. It gives up once it has counted more than 100 heap-page visits that returned no usable row, where consecutive entries on one heap page count once ([selfuncs.c#VISITED_PAGES_LIMIT](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447-L6455)), and the estimate then uses the stale `pg_statistic` bound ([selfuncs.c#get_actual_variable_endpoint](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6349-L6501)). Each entry it steps over is marked dead in the index, except during recovery, so the next plan passes it without a heap visit ([indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652), [selfuncs.c#get_actual_variable_endpoint-killed](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6388-L6397)). The probe itself therefore costs planning time and adds no pages or levels to the charge. The leaf pages that hold those dead entries are blocks of the index, so `index->pages` charges them like any other page (first row); the probe's own effect reaches the index cost only through the row estimate. The selectivity sets `numIndexTuples` ([selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019)), which scales both the pro-rata page share and the per-entry CPU charge ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [selfuncs.c:6810](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6810)). In fixture E the same Index Only Scan fell from `0.42..4.59` to `0.42..4.44` when the estimate dropped from 10 rows to 1. Both totals rebuild from one page at `random_page_cost`, the unchanged `0.42` startup, and the row estimate: `4.0 + 0.42 + 10 * (0.005 + 0.0025) + 10 * 0.01 = 4.595` and `4.0 + 0.42 + 1 * (0.005 + 0.0025) + 1 * 0.01 = 4.4375`, printed as `4.59` and `4.44`. The `(0.005 + 0.0025)` term is the index's per-entry charge, `cpu_index_tuple_cost` plus one `cpu_operator_cost` for the single index [qual](../../../glossary.md#qual); the `0.01` term is the per-row `cpu_tuple_cost` that `cost_index()` adds above it ([selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810), [costsize.c#cost_index-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800), [cost.h#default-costs](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25-L28)). See [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree) and [Dead entries at the end of a B-tree: the endpoint probe](#dead-entries-at-the-end-of-a-b-tree-the-endpoint-probe) |

The two `pgstatindex` metrics people reach for are computed from live leaf pages only. Deleted and half-dead pages are counted separately and are excluded from the density and fragmentation arithmetic, though they *are* included in the reported `index_size`:

```c
if (P_ISDELETED(opaque))
    indexStat.deleted_pages++;
else if (P_IGNORE(opaque))
    indexStat.empty_pages++;   /* this is the "half dead" state */
else if (P_ISLEAF(opaque))
{
    ...
    if (opaque->btpo_next != P_NONE && opaque->btpo_next < blkno)
        indexStat.fragments++;
}
```

([pgstatindex.c#page-classification](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372))

So `avg_leaf_density` answers "how full are the leaf pages that still hold data", and `leaf_fragmentation` answers "how often does the right-sibling link point backwards". Neither answers the two questions the B-tree cost model asks about the index's physical state: how many blocks the planner is charged for ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)), and how many levels lie below the fast root ([selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)). The measurement section shows a case where `avg_leaf_density` reads 89.18% on an index that is ten times larger than it needs to be.

### Types Of Bloated Indexes

This page uses *bloat* for any index pages beyond what the useful entries need, because those are the pages the planner is charged for. That is broader than the manual's glossary, which defines bloat as space in data pages that holds no current row versions, "such as unused (free) space or outdated row versions" ([glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)). Two shapes below fit only the broad sense, because their extra pages come from how current entries are stored, not from dead ones. An index with [deduplication](../../../glossary.md#deduplication) off stores each duplicate as its own leaf tuple instead of merging the group into a [posting list](../../../glossary.md#posting-list) ([btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800)). A GIN pending list is deferred insert work: GIN adds new entries to an unsorted pending list and moves them into the main structure later, in bulk ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)). The wiki's v17 GIN measurement protocol does not score pending pages as waste ([Mandatory GIN Bloat Tests (unverified)](../../common-concepts/mandatory-gin-bloat-tests.md)), yet `gincostestimate()` charges a single scan for every pending page at `random_page_cost` ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)).

#### Low-density live leaf pages

The classic case the manual describes: scattered deletions leave every leaf page allocated but nearly empty ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)). Measured: deleting 90% of 1,000,000 rows with `id % 10 <> 0` and vacuuming twice left the index at **2,745 blocks with 2,733 live leaf pages, zero deleted pages, and 9.27% `avg_leaf_density`**. This is the one bloat shape that both `avg_leaf_density` and the planner agree on, because low density here means high `pages / tuples`.

Split policy sets how full a page is left, not how full it can get. An index build fills each leaf page to the index fillfactor, and a split leaves its left page at a target that depends on the split ([nbtree.h#fillfactor-comment](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L197)):

- A rightmost leaf split uses the index fillfactor.
- A leaf split that the "split after new item" optimization recognizes as a localized ascending insertion either uses the fillfactor too or splits exactly after the new item.
- A rightmost internal split uses `BTREE_NONLEAF_FILLFACTOR`.
- Every other internal or leaf split aims to divide the data equally: it ranks candidate split points by how evenly they balance free space between the halves ([nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtsplitloc.c#_bt_deltasortsplits](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L561-L588)), then takes the lowest-penalty point among those close to the best balance ([nbtsplitloc.c#_bt_defaultinterval](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L852-L920), [nbtsplitloc.c#_bt_bestsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L770-L812)). On a leaf page the penalty favors a point that lets suffix truncation drop more attributes from the new high key; on an internal page it favors the smallest new pivot ([nbtsplitloc.c#_bt_split_penalty](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1119-L1153)). The result is near-equal, not exact.
- A leaf page with many duplicates, but not entirely one value, may instead split at either side of the group of duplicates that encloses the balanced split point, which can leave the halves far from equal ([nbtsplitloc.c#many-duplicates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L374-L405), [nbtsplitloc.c#_bt_strategy-many-duplicates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L985-L1009)).
- A leaf page full of one value, when it is the rightmost page holding that value, is split at `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L389-L416), [nbtsplitloc.c#single-value-condition](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1011-L1041)).

Those constants are `90`, `70`, and `96` ([nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202)). Between splits, inserts keep filling a page toward 100%. That is how fixture P's 1,000-value `tag` index, churned without a held snapshot, reads 98.01% after five update rounds, though it was built at the default fillfactor of 90 and measured 91.47% after its build. The per-index `fillfactor` reloption takes `ShareUpdateExclusiveLock` because it only affects later inserts ([reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)).

#### Deleted and half-dead pages

When a leaf page becomes completely empty, VACUUM can delete it, but the page stays in the relation as a tombstone with its own sibling links intact ([nbtpage.c#side-links](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2600)), labelled with a `safexid` ([nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659)). It becomes recyclable through the FSM only once no scan can still hold a reference ([README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)). nbtree never shortens its own file: nothing under `src/backend/access/nbtree/` calls `RelationTruncate()` or `smgrtruncate()`, and recycling goes through the FSM only. VACUUM records a recyclable page with `RecordFreeIndexPage()` ([nbtree.c:1168](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1168), [nbtpage.c:3050](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3050)), and `_bt_allocbuf()` asks `GetFreeIndexPage()` for such a page before it extends the file ([nbtpage.c#_bt_allocbuf-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L905)). Those blocks therefore remain inside `RelationGetNumberOfBlocks()` and remain charged, recycled or not.

Measured: deleting a *contiguous* 90% (`id > 100000`) let VACUUM empty whole pages, giving **2,745 blocks, of which 276 are live leaf pages and 2,465 are deleted pages** (the rest are 3 internal pages and the metapage), and `avg_leaf_density` of **89.18%**. A second and a third VACUUM did not shrink the fork. `REINDEX` cut it to 276 blocks. This is the shape where `avg_leaf_density` is actively misleading about size. The planner sees the size, but it overcharges fixture M's scans. VACUUM unlinked each of the 2,465 deleted pages from its level's sibling chain ([nbtpage.c#unlink-siblings](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2612)), so a whole-index scan walks only the 276 live leaves. The planner still prices that scan at `12730.42`, the same as fixture B, whose scan walks 2,733 live leaves.

#### Extra tree levels

Because a taller tree means more descent pages, the level charge fires. Measured: 50,000 rows produced a 139-block index at `fastlevel = 1`, while the same 50,000 rows with `fillfactor = 10` produced a 1,323-block index at `fastlevel = 2`. Point-lookup cost rose from `8.31` to `8.43`, and the gap before rounding is exactly the `0.125` level charge, which [The tree-height charge, isolated to the cent](#the-tree-height-charge-isolated-to-the-cent) reads as `50.00` at `cpu_operator_cost = 1`. The README's rule that the height of the tree cannot decrease is about the true root, `btm_level`. The planner charges the fast-root level, and page deletion does lower that one: in fixture F, deleting all but the top 1,000 of 1,000,000 keys and vacuuming twice, with no rebuild, left `tree_level` at 2 and moved `fastlevel` from 2 to 1, which removed one level charge from every scan of that index.

#### Physically fragmented leaf chains

`leaf_fragmentation` counts leaf pages whose right sibling lives at a lower block number. It rises when pages split in the middle of the key space. The split gets its new right half from `_bt_allocbuf()` ([nbtinsert.c:1720](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1720)), which takes a recycled page from the FSM if one is there and otherwise extends the file ([nbtpage.c#_bt_allocbuf](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L868-L988)). An appended right half then links right to an older page at a lower block number. Only VACUUM records recyclable pages in the FSM ([nbtree.c:1168](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1168), [nbtpage.c:3050](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3050)), so an index filled by random-order inserts with no page deletion behind it, like fixture G's, appends every split. After deletions a split's new page can land at a low block number instead, and its links point whichever way the recycled block happens to lie. Only *backward* links count ([pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323)), so the metric under-reports: a right-sibling link that jumps a thousand blocks forward is no more physically adjacent than one that points backwards, and it scores nothing. The manual ties physical adjacency to real runtime cost ([maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1048-L1054)). The planner never reads the metric at all. Measured contribution to cost: exactly `0.00`.

#### Version-churn duplicates from non-HOT UPDATEs

An `UPDATE` that modifies a column used by a *hot-blocking* index cannot be HOT, so it writes a new index entry in every index that accepts the new row, including indexes whose own columns did not change ([btree.sgml#version-churn](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L638-L655)). Those entries are logically unchanged duplicates.

One case spares every non-summarizing index, and one spares each partial index whose predicate the new row fails:

- **Only [summarizing-index](../../../glossary.md#summarizing-index) columns changed, *and* the new version fits on the old page.** The whole HOT decision sits inside `if (newbuf == buffer)`; when the new tuple has to go to a different page, `heap_update()` skips that branch entirely and only hints the old page as full ([heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166)). Inside the branch it tests the modified columns against the hot-blocking set and the summarizing set separately: an update that misses the hot-blocking set takes the HOT path and sets `summarized_update` if it touched the summarizing set, which becomes `TU_Summarizing` — "Only summarized columns were updated, TID is unchanged" ([heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429), [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127)). `ExecUpdateEpilogue()` passes that through as `onlySummarizing` ([nodeModifyTable.c#ExecUpdateEpilogue-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L2162-L2166)), and `ExecInsertIndexTuples()` then skips every non-summarizing index ([execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366)). A BRIN-indexed column can therefore be churned without adding one B-tree entry — but only while the updated rows keep fitting on their own pages. The moment one does not, `use_hot_update` stays false and `heap_update()` reports `TU_All` ([heapam.c:4429](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4429)), so every index that accepts the new row gets an entry, the unrelated B-trees included. Changing only summarizing-index columns is therefore a necessary condition for the exception, not a sufficient one: a table with no free space on its pages churns every index anyway.
- **A partial index whose predicate the new row fails.** `ExecInsertIndexTuples()` evaluates each index's `ii_Predicate` against the new tuple and skips the insert when it is not satisfied ([execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L387)), so a non-HOT update whose new row leaves a partial index's predicate adds nothing to that index. Every other index that accepts the new row, the unrelated non-partial B-trees included, still gets an entry, because `heap_update()` reports `TU_All` and the insert loop skips only the partial indexes whose predicates the new row fails ([heapam.c:4429](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4429), [execIndexing.c#ExecInsertIndexTuples-loop](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L343-L387)).

Since v14 ([README#bottom-up-added-in-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L981)), nbtree attacks the duplicates it does get with [bottom-up index deletion](../../../glossary.md#bottom-up-index-deletion) passes triggered when a version-churn page split is anticipated ([btree.sgml#bottom-up-deletion](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L678), [nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L280-L320), [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782)).

Measured on the pin: five whole-table non-HOT `UPDATE` rounds over 200,000 rows grew an unrelated index on a 1,000-value column from 169 to **543 blocks**. Repeating the identical workload with a long-lived [`REPEATABLE READ`](../../../glossary.md#isolation-level) snapshot open in another session, which is the condition the README names as keeping deletion from freeing tuples ([README#deduplication-and-bottom-up-note](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)), grew it to **1,173 blocks** instead. The benefit depends on the shape of the index: the same workload over a 100-value column grew its index from 180 to **1,020 blocks in both runs**, so there the unheld [horizon](../../../glossary.md#xmin-horizon) bought nothing.

#### Duplicate-heavy indexes with deduplication disabled

Deduplication merges duplicate leaf tuples into posting lists in two places. An index build merges each group of duplicates before it adds them to the leaf page it is filling ([btree.sgml#deduplication-at-build](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L786-L797)). An insert runs a deduplication pass lazily, at the point a page would otherwise split ([btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800), [README#notes-about-deduplication](../../../../raw/postgres-17/src/backend/access/nbtree/README#L904-L948), [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L33-L58)). It is on by default and can be turned off per index with the `deduplicate_items` reloption ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)).

The reloption is not the only gate. Both paths also require the index to be [`allequalimage`](../../../glossary.md#allequalimage), which `CREATE INDEX` or `REINDEX` decides once from each key column's [operator class](../../../glossary.md#operator-class) and collation and stores in the metapage ([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183), [nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564)). The build checks it beside the reloption, and skips deduplication for a unique index as well ([nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)); the insert-time pass checks it beside the reloption ([nbtinsert.c#dedup-gate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2778-L2781)). The manual lists what never qualifies: `text`, `varchar` and `char` under a nondeterministic collation, `numeric`, `jsonb`, `float4` and `float8`, container types such as composites, arrays and ranges, and every `INCLUDE` index ([btree.sgml#deduplication-restrictions](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L908)). A duplicate-heavy index of one of those kinds keeps every duplicate as its own tuple whatever `deduplicate_items` says.

Measured: 1,000,000 rows over 100 distinct keys built **852 blocks** with `deduplicate_items = on` and **2,749 blocks** with it off, a 3.2x difference in the planner's `pages` input for identical logical content. Both indexes were created after the rows were loaded, so this difference is the build path's.

#### Bloat that VACUUM deliberately skipped

Two v14-era escape hatches let VACUUM skip index vacuuming, the step that deletes dead index entries; [v14: VACUUM can skip index vacuuming on its own](#v14-vacuum-can-skip-index-vacuuming-on-its-own) gives their commits:

- The 2% bypass. `lazy_vacuum()` skips index *vacuuming* when fewer than `BYPASS_THRESHOLD_PAGES` (2% of `rel_pages`) hold [`LP_DEAD`](../../../glossary.md#line-pointer) items and the [TID](../../../glossary.md#tid) store is under 32MB ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1949)). It is considered only when `INDEX_CLEANUP` is left at its `AUTO` default, because `ON` switches it off ([vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402)), and only when no earlier round of index vacuuming ran in the same VACUUM, because a round forced by a full dead-TID store switches it off too ([vacuumlazy.c#bypass-off-after-round](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L894-L896)).
- The [wraparound failsafe](../../../glossary.md#vacuum-failsafe), which makes the ongoing VACUUM bypass all further index vacuuming ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2347)).

The 2% bypass is not a skip of all index maintenance, and reading it that way overstates how much bloat it can hide. It clears only `do_index_vacuuming`, and the branch says so in as many words — "bypass index vacuuming, but do index cleanup" ([vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)) — so `amvacuumcleanup` still runs. What that buys differs by access method:

| AM | Under the 2% bypass |
|---|---|
| B-tree | No entry is deleted, but `btvacuumcleanup()` is still called with `stats == NULL` and asks `_bt_vacuum_needs_cleanup()`; when that says yes it runs a full `btvacuumscan()`, which can place previously deleted pages in the FSM ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)) |
| GIN | `ginvacuumcleanup()` flushes the pending list whenever `ginbulkdelete()` was not called ([ginvacuum.c#ginvacuumcleanup-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L719-L729)). That covers the bypass, and also every VACUUM that collected no dead items, because `lazy_vacuum()` and with it every `ambulkdelete` run only when dead items exist, while index cleanup runs whenever `do_index_cleanup` is set ([vacuumlazy.c#lazy_vacuum-call](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1066)). When `ginbulkdelete()` does run, it flushes the list itself on its first call ([ginvacuum.c#ginbulkdelete-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)) |

The wraparound failsafe is the stronger escape hatch, because it clears `do_index_cleanup` as well as `do_index_vacuuming` ([vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326)), so it does stop `amvacuumcleanup` and with it the GIN pending-list flush.

`vacuum_index_cleanup = off` has the same effect by request: it clears both switches before the scan starts ([vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402)), and the manual warns it "may also lead to severely bloated indexes if table modifications are frequent" ([ref/create_table.sgml#vacuum_index_cleanup](../../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1558-L1575)).

#### Non-B-tree bloat

The manual states plainly that "the potential for bloat in non-B-tree indexes has not been well researched" and recommends monitoring physical size ([maintenance.sgml#non-btree-bloat](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)). Hash, GiST and SP-GiST get the page-count penalty through `genericcostestimate()`. GIN and BRIN price pages through their own models.

BRIN charges every index page on every scan. `brincostestimate()` puts the range-map (revmap) pages into the startup cost at the [tablespace](../../../glossary.md#tablespace)'s sequential page cost, `spc_seq_page_cost * revmapNumPages * loop_count`, and adds the remaining `index->pages - revmapNumPages` pages at `spc_random_page_cost * (numPages - revmapNumPages) * loop_count`, with `numPages = index->pages` ([selfuncs.c:8063](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8063), [selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258)). There is no cache model: instead of running repeated fetches through `index_pages_fetched()`, both terms are multiplied by `loop_count`. The metapage read, `brinGetStats()`, returns only `pagesPerRange` and the revmap page count ([brin.c#brinGetStats](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1638-L1654), [brin.h#BrinStatsData](../../../../raw/postgres-17/src/include/access/brin.h#L32-L36)). Those are structural values, not bloat counters, so BRIN bloat reaches the cost only through `index->pages`, and a bloated BRIN index is charged in proportion to its size on every query.

GIN is different. `gincostestimate()` reads the current pending-page count plus the entry-page, data-page and entry counts that the last build or `VACUUM` wrote to the metapage ([gininsert.c:406](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L406), [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789)), and scales the last three by `index->pages / nTotalPages`, or invents all three from `index->pages` when those counters cannot be trusted: when the entry-page or entry count is zero, or the index is now smaller than the recorded total or at least four times larger ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767)). Its pending list is charged like bloat, though it is deferred insert work rather than waste (see the start of this section). Its cost consequences are covered in [Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead](#follow-up-when-a-gin-index-is-discarded-and-a-b-tree-is-used-instead) below.

### How Density And Fragmentation Affect Different Queries

| Query shape | Sensitivity to extra pages | Sensitivity to extra levels | Sensitivity to fragmentation |
|---|---|---|---|
| Equality lookup returning `k` rows | None while `k * pages <= tuples`, which for a unique key means `pages <= tuples`: `ceil()` keeps `numIndexPages` at 1. Past that, `ceil(k * pages / tuples)` pages | Full: the only signal below that threshold | None |
| Range scan / broad index-only scan | Linear in `pages / tuples` | One charge per level | None |
| Bitmap index scan feeding a [`BitmapAnd`](../../../glossary.md#bitmapand) | Linear, and can get the index dropped by `choose_bitmap_and()` ([indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489)) | One charge per level | None |
| Nested-loop inner index scan | From none to linear. While loops times touched pages stay well below the index's page count, the cache model prices nearly every fetch as new and the index's size barely moves the charge. Once they reach twice the page count, for an index that fits its prorated cache share, the fetches are capped at the whole index and spread across the loops, `index->pages * random_page_cost / loop_count` per loop, which is linear in pages ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)). Measured on fixture A's dense and fillfactor-10 twins over 50,000 loops: `0.66` against `2.50` per loop. Their page charges are `2745 * 4.0 / 50000 = 0.22`, capped at the whole index, and `ceil(2 * 26411 * 50000 / (2 * 26411 + 50000)) * 4.0 / 50000 = 25687 * 4.0 / 50000 = 2.05`, just short of the cap: 9.36x the charge for 9.62x the pages | Charged once per loop | None |
| `= ANY (array)` on a B-tree | Linear, and the v17 descent clamp gives a bloated index a higher cap on estimated descents than its dense twin | Charged once per estimated descent | None |
| Parallel index or index-only scan | Threshold and worker count both move with the pages the scan is expected to touch | One charge per level | None |
| Any scan of a partial B-tree | Pages the index had when `ANALYZE` or `VACUUM` last wrote its `pg_class` row: in full. Pages gained since then: in proportion to the table's growth until the next such write, then in full: fixture Q's partial index, at 331 live blocks, was charged 113 pages before `ANALYZE` and 331 after; see [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten) | One charge per level | None |
| GIN | Its own model. A single scan pays for the whole pending list ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886)); for `rint(pow(numEntryPages, 0.15))` entry pages per search entry plus the partial-match fraction of all entry pages ([selfuncs.c#gincostestimate-entrypages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7888-L7914)); and for the partial-match fraction of the data pages plus the exact-match entries' share of them, raised to a floor set by the selectivity ([selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7982-L8006)). Every one of those pages costs `random_page_cost` ([selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980), [selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029)). The entry-page and data-page counts are the counters of the last build or `VACUUM`, scaled to `index->pages`, or invented from it when they cannot be trusted ([selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767)). Repeated scans, from a nested loop or an array qual, run the entry pages, pending pages included, and the data pages through the cache model, with the entry-page and data-page counts as the page universes, so the pending list is not charged in full on each repetition ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974), [selfuncs.c#gincostestimate-datapages-cache](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8017-L8025)) | Derived from pages, as for GiST: `rint(pow(numEntryPages, 0.15))` entry pages per search entry ([selfuncs.c:7895](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7895)), each charged `50 * cpu_operator_cost` of CPU and `random_page_cost` of I/O ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)) | None |
| BRIN | Every index page on every scan, multiplied by `loop_count`: range-map pages at `seq_page_cost`, the rest at `random_page_cost` ([selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258)) | None | None |

The practical shape of this: bloat mostly hurts plans that were already reading a lot of the index, and it barely registers on the selective lookups that dominate OLTP traffic. That is why a badly bloated index can keep serving primary-key lookups with an almost unchanged plan and price while quietly wrecking reporting queries over the same table. There are two exceptions. One is an index with more pages than its table has rows, which is what a drained queue table leaves behind: there a one-row lookup is charged several pages, measured at `12.29` against `4.29` for the same 1,000 rows after a rebuild. The other is repetition: on the inner side of a nested loop the cache model can price up to the whole index, spread across the loops, measured at `0.66` against `2.50` per loop. Apart from the page and height charges, dead entries at either end of a plain B-tree reach planning itself: a range comparison near the end of the histogram makes the planner read them, and after more than 100 heap-page visits that return no usable row it falls back to the stale histogram bound. That can hold the row estimate at a stale value, and the estimate moves the price: fixture E's Index Only Scan cost `4.59` at the stale 10-row estimate and `4.44` once a probe reached the live maximum and the estimate fell to 1 row ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136), [selfuncs.c#VISITED_PAGES_LIMIT](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447-L6455)); see [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree).

### Exact-Pin Measurements

#### Fixtures and method

All numbers below come from one isolated server built from the pinned checkout by the script filed under [Measurement Script](#measurement-script), last run on 2026-09-25 (`PostgreSQL 17.11 on aarch64-apple-darwin27.0.0`, pin `786db8dcf168bd9df8f55047337525ac19118b1c`, `block_size` 8192, maximum data alignment 8), with `autovacuum = off`, `shared_buffers = 256MB`, and default planner cost settings (`random_page_cost = 4`, `seq_page_cost = 1`, `cpu_tuple_cost = 0.01`, `cpu_index_tuple_cost = 0.005`, `cpu_operator_cost = 0.0025`, `effective_cache_size = 4GB`). [`pgstattuple`](../../../glossary.md#pgstattuple) supplied `pgstatindex`. [`pageinspect`](../../../glossary.md#pageinspect) supplied `bt_metap()`, so the planner's fast-root height could be read directly, and `bt_multi_page_stats()`, whose `dead_items` counts the line pointers on each page that are marked dead ([btreefuncs.c#GetBTPageStatistics-items](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L172-L187)). `pg_class.relallvisible` against `relpages` confirmed visibility-map state.

Measurement provenance: the first filing of this page measured its fixtures on the previous pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on Linux x86_64, with scripts that were never published and no longer exist. This page no longer quotes any number from those runs. The 2026-09-19 run rebuilt the first filing's fixtures from its prose on the current pin, on a different operating system and architecture. The numbers that depend only on an index and on exact statistics agreed with the first filing; the ones that moved depended on `ANALYZE`'s random sample or on fixture details the first filing did not record. That agreement is consistent with the range between the two pins, which changes no cost function and none of the planner's index-size inputs. `git log 54eeefaedbee..786db8dcf168` over the cost path (`src/backend/optimizer/`, `src/backend/utils/adt/selfuncs.c`, `src/backend/access/nbtree/`, `src/backend/access/index/`, `src/backend/access/gin/`, `src/backend/access/brin/` and `src/backend/commands/analyze.c`) returns nine commits, and over the other files this page cites it returns twenty-three more:

| Files | Commits | What they change |
|---|---|---|
| `selfuncs.c` | `0ebf896f44` | a datatype check on the `ctid` special case in `scalarineqsel()` ([selfuncs.c#scalarineqsel-ctid-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L599-L601)) |
| `nbtsearch.c` | `8434c93859` | an empty-index recheck under `SERIALIZABLE` in `_bt_endpoint()` ([nbtsearch.c#_bt_endpoint-empty-index](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2590-L2606)) |
| `clauses.c`, `subselect.c`, `pg_operator.dat` | `26d6b7dc9e`, `7fbbe75875`, `19152e3c29` | `ScalarArrayOpExpr` nullability and strictness logic, a compiler-warning fix, and hashability checks for container-type equality |
| `allpaths.c` | `c3f1db2b88` | the unsafe-flag test in `check_output_expressions()` |
| `prepjointree.c` | `b308eb3661`, `cdcec567da` | join-tree preprocessing: a skipped `get_relids_in_jointree()` call and an outer-join edge case in `remove_useless_result_rtes()` |
| `ginvacuum.c` | `ba5e463295` | a restored `vacuum_delay_point()` in posting-tree leaf vacuum |
| `heapam.c`, `bufmgr.c`, `rel.h` | `c0d9864f5c`, `7edec8b578`, `885dc83380`, `4e49f68b72`, `4dfae59a1d`, `40927d458f` | WAL registration of visibility-map blocks, macros for WAL block reference IDs, a macro rename, two checks on access to other sessions' temporary tables, and tests for that access |
| `heap.c`, `index.c`, `indexcmds.c`, `tablecmds.c` | `d1c8aa0b09`, `28269fed66`, `1d6c654c81`, `c1588f92a9`, `6713596055`, `75a03c569c`, `18006c1bd6` | a `USAGE` check on types used by stored expressions, the `indimmediate` flag in `index_create_copy()`, restoring partitions with exclusion constraints, no dependency on a dropped column's type, a `USAGE` check in `ALTER TABLE OF`, the owner of [extended statistics](../../../glossary.md#extended-statistics) rebuilt by `ALTER TABLE`, and `DROP EXPRESSION` with subpartitions |
| `relcache.c`, `plancache.c` | `2d1ed2c1dd`, `1974acf23c` | a `polpermissive` check when comparing policies, and plan-cache invalidation after role changes |
| `vacuum.c`, `maintenance.sgml` | `a287fd85d2`, `9484169ba0`, `e90251176d` | MultiXact wraparound hint text and two documentation fixes |
| `guc_tables.c`, `config.sgml` | `01992176e0`, `786db8dcf1`, `dec60e8ada`, `39a9c4079c`, `92b1299450` | one new logical-decoding [GUC](../../../glossary.md#guc), `output_plugin_libraries`, and four changes to `config.sgml` text |

Method notes that matter for reading the numbers:

- Most comparisons use **two separate tables with identical contents**, so every query has exactly one candidate index and no index-choice tie-breaking is involved. Fixtures B and M differ on purpose: each keeps 100,000 of the same 1,000,000 rows, B every tenth row and M the first tenth. Fixtures B, M, F, F-one, L3, P, Q, D and T also compare one index with itself before and after a change, and fixture E follows one index through six consecutive plans. Where one table carries two indexes that are priced separately (fixtures N, Q and S), each is priced with the other dropped inside a rolled-back subtransaction, so the statistics are literally the same. Fixture L3 keeps both of its indexes, and fixture D keeps both except in its B-tree-only plan, because how the planner combines them is what they measure.
- Most comparisons use `Index Only Scan` on tables vacuum-frozen to 100% all-visible, so the heap I/O term of `cost_index()` is zero ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)). The reported total is still not a pure index cost: `cost_index()` charges `cpu_tuple_cost` for every row it expects to fetch, index-only or not ([costsize.c#cost_index-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800)), so each total carries `rows * cpu_tuple_cost`. That amount is the same on both sides of every comparison whose row estimates match; fixture F-one's before and after estimates differ, 200,000 rows against 1, and its section separates that term. The exceptions are deliberate:
  - H and N select a non-indexed column, and I is analyzed but never vacuumed. Each pays the same heap fetches on both sides of its comparison.
  - P and Q price the `Bitmap Index Scan` node. That node's cost is the index access method's own total, with no heap term ([costsize.c#cost_index-save-indextotalcost](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L623-L629), [createplan.c#create_bitmap_subplan-indexpath](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3480-L3485)), so it does not matter that their churn left the tables not all-visible.
  - E reads row estimates and dead-entry counts. Its two `Index Only Scan` costs carry no heap term only because `pg_class` still counted every heap page all-visible after the delete, although the delete had cleared the visibility-map bits of the pages it touched; see [Dead entries at the end of a B-tree: the endpoint probe](#dead-entries-at-the-end-of-a-b-tree-the-endpoint-probe).
  - The plan-choice tests select a non-indexed column too, because a heap fetch is what makes the [sequential scan](../../../glossary.md#sequential-scan) competitive.
- `enable_seqscan` and `enable_bitmapscan` were turned off where an index or index-only scan had to be priced. Fixture P's bitmap scans turn off `enable_seqscan` and `enable_indexscan` instead, and fixture Q's also turn off `enable_indexonlyscan`. The nested-loop test turns off `enable_hashjoin` and `enable_mergejoin` and sets `max_parallel_workers_per_gather = 0`. All of these are [`PGC_USERSET`](../../../glossary.md#guc-context) ([guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792), [guc_tables.c#enable_indexscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L793-L802), [guc_tables.c#enable_indexonlyscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L803-L812), [guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822), [guc_tables.c#enable_mergejoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L893-L902), [guc_tables.c#enable_hashjoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L903-L912), [guc_tables.c#max_parallel_workers_per_gather](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3419-L3428)), and the script sets them transaction-locally for one statement. The plan-choice tests, fixture L3 and fixture E run at the defaults.
- **Statistics are exact, so the numbers are reproducible.** Every session runs with `default_statistics_target = 10000` (`PGC_USERSET`, session scope, [guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2070-L2079)). `ANALYZE` samples 300 rows per unit of that target ([analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1876-L1894)), which at 3,000,000 is more rows than any fixture holds, so it reads every row and the statistics carry no sampling noise. The setting does not enter the cost model. Every run of the filed script on this pin has reproduced every recorded value; the latest runs are described under [Last run](#last-run).
- **Exact statistics are not exact row estimates.** Exhaustive sampling makes the statistics, and therefore every cost on this page, the same on every run. It does not make a row estimate equal the true row count, for two reasons. First, the selectivity model applies its own assumptions on top of exact statistics: `clauselist_selectivity_ext()` multiplies the per-clause selectivities of clauses it cannot pair into a range query, which assumes they are independent ([clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)). Four measurements here put two clauses on two columns: fixture L3's `a = 5 AND c = 7`, fixture D's `tsv @@ … AND cat = 7`, fixture P-gin's `id <= 5000 AND n = 42`, and fixture Q's `id <= 20000 AND v > 0`, where `v > 0` holds for every row. The first two record each estimate beside its true count, in [Bloat changes plans](#bloat-changes-plans) and [A GIN index with a large pending list loses to a B-tree](#a-gin-index-with-a-large-pending-list-loses-to-a-b-tree). Both are built with independent columns, so neither violates the assumption, and no fixture on this page bounds the error it can cause. Second, statistics describe the table as it was at the last `ANALYZE`. Fixtures Q and E change their tables after it on purpose, and their estimates depart from the counts their construction fixes: Q estimates 40,000 rows for 20,000 live ones, and E estimates 10 for none.
- The script records each plan as `EXPLAIN (FORMAT JSON)` and keeps every node's type, index, costs, rows, width, workers, and index, recheck and filter conditions. The three `text` excerpts below set those recorded values in [`EXPLAIN`](../../../glossary.md#explain)'s text layout; the relation names come from each fixture's query.

| Fixture | Contents | Dense index | Bloated twin |
|---|---|---|---|
| A | 1,000,000 rows, `int` key | `a_dense_idx`, 2,745 blocks, 90.06% density, `fastlevel` 2 | `a_sparse_idx`, `fillfactor = 10`, 26,411 blocks, 9.62% density, `fastlevel` 2 |
| B | 1,000,000 rows, then `DELETE` of 90% by `id % 10 <> 0` | after `REINDEX`: 276 blocks, 89.83%, `fastlevel` 1 | before: 2,745 blocks, 9.27%, `fastlevel` 2 |
| F | 1,000,000 rows, then `DELETE` of all but the top 1,000 keys, two VACUUMs | after `REINDEX`: 5 blocks, `fastlevel` 1 | 2,745 blocks: 4 live leaves, 2 internal pages, 2,738 deleted pages and the metapage; `tree_level` 2, **`fastlevel` 1** |
| F-one | 200,000 rows, then `DELETE` of all but one row, two VACUUMs | before: 551 blocks, 547 live leaves, 90.00% density, `fastlevel` 2 | after: the same 551 blocks, now 1 live leaf, 2 internal pages, **547 deleted pages** and the metapage; 0.29% density, `tree_level` 2, **`fastlevel` 0**, `reltuples` 1 |
| G | 300,000 rows | `g_seq_idx`, `fillfactor = 100`, 744 blocks, 99.89% density, 0% fragmentation | `g_frag_idx` filled by `setseed(0.42)` random-order inserts, 1,148 blocks, 64.69% density, **49.87% fragmentation** |
| H | 50,000 rows | `h_l1_idx`, 139 blocks, `fastlevel` 1 | `h_l2_idx`, `fillfactor = 10`, 1,323 blocks, `fastlevel` 2 |
| I | 2,000 rows, analyzed but not vacuumed | `i_small_idx`, 8 blocks | `i_big_idx`, `fillfactor = 10`, 55 blocks |
| M | 1,000,000 rows, then `DELETE` of a contiguous 90% | after `REINDEX`: 276 blocks | 2,745 blocks: 276 live leaves, 3 internal pages, **2,465 deleted pages** and the metapage; 89.18% density |
| N | 1,000,000 rows over 100 distinct keys | `deduplicate_items = on`, 852 blocks | `deduplicate_items = off`, 2,749 blocks |
| P | 200,000 rows, `tag = id % 1000`, five whole-table non-HOT `UPDATE` rounds | no blocking snapshot: 169 -> 543 blocks | `REPEATABLE READ` snapshot held: 169 -> 1,173 blocks |
| P-100 | the same with `tag = id % 100` | no blocking snapshot: 180 -> 1,020 blocks | snapshot held: 180 -> 1,020 blocks |
| L3 | 500,000 rows, `a = g % 2000` plus an independent seeded `c` over 0..19; `l3_a` is 449 blocks at 85.47% throughout | `l3_c` at the default fillfactor: 427 blocks, 89.81% density | `l3_c` rebuilt at `fillfactor = 10`: 3,801 blocks, 10.37% density |
| Q | 200,000 rows, `v = id`, heap `fillfactor = 100`; then ten `UPDATE ... SET v = v + 1` rounds over the 20,000 rows with `id <= 20000`, in one transaction, with no `VACUUM` or `ANALYZE` until the last step | built: partial `q_part` (`WHERE id <= 20000`) 57 blocks, plain `q_full` 551 blocks | churned: `q_part` 331 blocks at 77.56% density, `q_full` 825 blocks at 85.14%; `pg_class` still records 57 and 551 |
| E | 200,000 rows, primary key `e_t_pkey`; then `DELETE` of `id > 100000`, with no `VACUUM` or `ANALYZE` until a final plain `VACUUM` | before the delete: 885 heap blocks, `id` histogram ending at 200000 | after it: 100,000 dead rows at the index's upper end, the same histogram |

Fixture G's random order is seeded, so its block count is reproducible; the first filing's build was unseeded. Fixture P records its tag column's cardinality, which the first filing did not. Fixture L3 was rebuilt on the second 2026-09-19 pass because its old `c` column made the predicate impossible: `c` used to be `g % 20`, and because 20 divides 2,000, `a = 5` forced `c = 5`, so `a = 5 AND c = 7` matched no row at all. `c` is now an independent seeded draw over the same twenty values.

#### Index cost is a closed form in pages, tuples and tree height

Fixture A, whole-index scan (`WHERE id > 0`, selectivity exactly 1 so both row estimates are 1,000,000):

| Index | Blocks | Density | `fastlevel` | Observed total cost | Predicted from `(pages, tuples, fastlevel)` |
|---|---:|---:|---:|---:|---:|
| `a_dense_idx` | 2,745 | 90.06% | 2 | `28480.42` | `28480.42` |
| `a_sparse_idx` | 26,411 | 9.62% | 2 | `123144.43` | `123144.43` |

The prediction is `pages * random_page_cost + tuples * (cpu_index_tuple_cost + qual_op_cost) + ceil(log2(tuples)) * cpu_operator_cost + (fastlevel + 1) * 50 * cpu_operator_cost + rows * cpu_tuple_cost`, where `qual_op_cost` is one `cpu_operator_cost` for the single index qual. `genericcostestimate()` adds one more term, `qual_arg_cost`, which is zero when the comparison value is a constant ([selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810)). The script's `predict()` evaluates that expression in `float8`, in the planner's order of operations, from three inputs only: the index's live block count, the planner's row estimate for the table, and `bt_metap()`'s `fastlevel`. The row estimate is `reltuples` scaled from `relpages` to the live heap size, as the planner scales it ([tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747)); on these freshly vacuumed tables the scaling leaves `reltuples` unchanged. It equals the `EXPLAIN` total on 7 of 7 whole-index scans: both fixture-A indexes, fixtures B and M before the rebuild, fixture B after it, and both fixture-G indexes. Both fixture-A totals are exactly `x.425` in decimal; the planner's doubles print one as `.42` and the other as `.43`, and the `float8` recomputation lands on the same side both times. The first filing's formula carried a standalone `+ qual_op_cost` term that the source does not have; this form drops it. A 9.62x larger index costs 4.32x more in total. Without the `1000000 * 0.01 = 10000.00` of `cpu_tuple_cost` that both totals carry, it costs 6.12x more on the index alone: `113144.43` against `18480.42`.

Fixture A, 10% range scan (`id BETWEEN 1 AND 100000`, 100,000 rows estimated by both): `3100.43` dense against `12568.42` bloated.

#### The point lookup is nearly blind to bloat

Fixture A, `WHERE id = 42`, both tables 100% all-visible:

```text
Index Only Scan using a_dense_idx on a_dense  (cost=0.42..4.44 rows=1 width=4)
  Index Cond: (id = 42)
Index Only Scan using a_sparse_idx on a_sparse  (cost=0.42..4.44 rows=1 width=4)
  Index Cond: (id = 42)
```

Identical to the cent, across a 9.62x size difference and an 80-point density difference, because `ceil(1 * pages / tuples)` is `1` for both ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)) and both trees have `fastlevel = 2`.

Fixture B shows the same thing where the bloat is real rather than synthetic. Before `REINDEX` the index is 2,745 blocks at 9.27% density; after, 276 blocks at 89.83%. The whole-index scan drops from `12730.42` to `2854.29` (4.46x), but the point lookup drops only from `4.44` to `4.31`, and that entire `0.125` difference is the level the rebuild removed.

#### The tree-height charge, isolated to the cent

Fixture H holds row count constant at 50,000. The `fillfactor = 10` twin has 9.5x the blocks and one more level, and a one-row lookup is charged one page either way, so only the level reaches its cost:

| Index | Blocks | `fastlevel` | `WHERE id = 25000` | Same with `cpu_operator_cost = 1` |
|---|---:|---:|---|---|
| `h_l1_idx` | 139 | 1 | `cost=0.29..8.31` | `cost=116.00..125.02` |
| `h_l2_idx` | 1,323 | 2 | `cost=0.41..8.43` | `cost=166.00..175.01` |

With `cpu_operator_cost = 1` the startup costs are `116.00` and `166.00`. Both decompose exactly: `ceil(log2(50000)) = 16` comparison charges ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091)) plus `(fastlevel + 1) * 50` ([selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)). The gap is `50.00`, exactly `DEFAULT_PAGE_CPU_MULTIPLIER` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)). At default settings the same gap is `0.125`, so a 9.5x larger index raises a point-lookup cost by about 1.5%. Fixture H selects a non-indexed column, so all four costs include one heap page at `4.0`.

#### A mostly-empty index: the fast root drops and pages outnumber rows

Fixture F builds the same 2,745-block index as fixture A, deletes every row except the top 1,000 keys (`id > 999000`), and vacuums twice. The index is not rebuilt until the last row of the table below.

| State | Blocks | Live leaf pages | Deleted pages | `tree_level` | `fastlevel` | `WHERE id = 999950` | Whole-index scan |
|---|---:|---:|---:|---:|---:|---|---:|
| built, 1,000,000 rows | 2,745 | 2,733 | 0 | 2 | 2 | `cost=0.42..4.44` | |
| after the delete and two VACUUMs, 1,000 rows | 2,745 | 4 | 2,738 | 2 | **1** | `cost=0.28..12.29` | `10997.77` |
| after `REINDEX` | 5 | 3 | 0 | 1 | 1 | `cost=0.28..4.29` | |

Two things happen here that no fixture before it reaches.

**The planner's height fell without a rebuild.** Page deletion left the true root where it was, at block 290 and level 2, and moved the fast root to block 2570 at level 1, which is the adjustment the README describes for deleting the next-to-last page on a level ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381), [nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)). `pgstatindex` still reports `tree_level` 2, while the startup cost shows the planner charging for one level fewer: `0.28` is `ceil(log2(1000)) * 0.0025 + (1 + 1) * 0.125 = 0.275`.

**The point lookup stopped being cheap.** With 1,000 rows and 2,745 pages, `numIndexPages = ceil(1 * 2745 / 1000) = 3`, so the one-row lookup is charged three random pages: `3 * 4.0 + 0.0075 + 0.025 + 0.25 + 0.01 = 12.2925`, printed `12.29`. The rebuilt index prices it at `4.29`. A whole-index scan of the 1,000 rows costs `10997.77`, of which `10980.00` is the 2,745 pages. This is the shape a drained queue table leaves behind, and it is the one case on this page where bloat reaches the *page* charge of a lone single-row lookup. Elsewhere bloat reaches such a lookup only through the `0.125` per-level charge, as fixtures B and H show. A lookup that returns `k` rows is charged `ceil(k * pages / tuples)` pages, so it feels bloat sooner: fixture P's `tag = 7` bitmap scans, 200 rows each, cost `5.92` against `9.92` for one page against two. The other shape that reaches OLTP is repetition rather than size, and it is measured in [Index pages in the cache model](#index-pages-in-the-cache-model). Drain the table one step further and the charge disappears again; that is the next section.

#### The pages-outnumber-rows guard at its limit

Take the same draining one step further and the penalty inverts. `genericcostestimate()` computes the pro-rata share only when `index->pages > 1` **and** `index->tuples > 1`; when either operand fails it writes a flat `numIndexPages = 1.0` ([selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)). For a non-partial index `tuples` is the table's row estimate, so the second operand fails on a table the planner believes holds one row, whatever the index's size.

Fixture F-one is a 200,000-row table whose index is then stripped of all but one entry. Nothing is rebuilt.

| State | Blocks | Live leaf pages | Deleted pages | `tree_level` | `fastlevel` | `pg_class` `relpages` / `reltuples` | `WHERE id > 0` | `WHERE id = 200000` |
|---|---:|---:|---:|---:|---:|---|---:|---:|
| built, 200,000 rows | 551 | 547 | 0 | 2 | 2 | 885 / 200000 | `5704.42` | |
| after deleting 199,999 rows and two VACUUMs | 551 | 1 | **547** | 2 | **0** | 885 / **1** | **`4.14`** | **`4.14`** |

The index still occupies all 551 blocks; `pgstatindex` reports 0.29% `avg_leaf_density` and 547 deleted pages. The planner charges it for **one** page. The whole-index scan and the single-row lookup come out at the identical `4.14`, because with one tuple there is nothing left for either estimate to differ about:

- The page term fell from `551 * 4.0 = 2204.00` to `1 * 4.0 = 4.00`. That fall is the guard alone, since `index->pages` did not change.
- The per-tuple term fell from `200000 * (0.005 + 0.0025) = 1500.00` to `0.0075`, and `cpu_tuple_cost` from `2000.00` to `0.01`. That fall is the row count, not the guard.
- The `ceil(log2(tuples))` comparison charge disappeared entirely, because that term is guarded on `index->tuples > 1` too ([selfuncs.c#btcostestimate-log2-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7086-L7091)).
- The level charge fell from `(2 + 1) * 0.125 = 0.375` to `(0 + 1) * 0.125 = 0.125`, because page deletion pulled the fast root down to a leaf, `fastlevel` 0 against an unchanged `tree_level` 2 — the same fast-root effect fixture F shows, taken to the bottom of the tree.

Adding those back: `4.00 + 0.0075 + 0.125 + 0.01 = 4.1425`, printed `4.14`, with a startup cost of `0.125` printed `0.12`.

So the two guards in [1. Physical page count enters index cost](#1-physical-page-count-enters-index-cost) bracket the page charge from both ends. With at least as many rows as index pages, a one-row lookup is charged one page, so its page charge is blind to bloat; only the level charge still sees an extra level. With more than one row but fewer rows than pages, it is charged `ceil(pages / tuples)` pages, which is fixture F. At one row or fewer, an index of any size is charged one page again — so the cheapest whole-index scan on this page belongs to its emptiest index, and a monitoring rule that reads plan cost as a bloat signal reads this index as healthy.

#### Leaf fragmentation contributes exactly zero

Fixture G, both tables frozen to 100% all-visible so the index-only scan pays no heap cost:

| Index | Blocks | Density | Fragmentation | `fastlevel` | Observed | Predicted |
|---|---:|---:|---:|---:|---:|---:|
| `g_seq_idx` | 744 | 99.89% | **0.00%** | 2 | `8226.42` | `8226.42` |
| `g_frag_idx` | 1,148 | 64.69% | **49.87%** | 2 | `9842.42` | `9842.42` |

The observed gap is `9842.42 - 8226.42 = 1616.00`. The page-count gap is `1148 - 744 = 404`, and `404 * random_page_cost = 404 * 4.0 = 1616.00`. The residual attributable to a 49.87-point fragmentation difference is `0.00`, and the closed-form prediction, which has no fragmentation term, reproduces both costs.

#### Two indexes with the same cost and opposite avg_leaf_density

Fixtures B and M both end at 2,745 blocks over 100,000 surviving rows, and both price a whole-index scan at exactly `12730.42`, dropping to `2854.29` after `REINDEX`. Their `pgstatindex` output could hardly be more different:

| Fixture | Deletion pattern | Blocks | Live leaf pages | Deleted pages | `avg_leaf_density` | Whole-index cost |
|---|---|---:|---:|---:|---:|---:|
| B | scattered (`id % 10 <> 0`) | 2,745 | 2,733 | 0 | **9.27%** | `12730.42` |
| M | contiguous (`id > 100000`) | 2,745 | 276 | **2,465** | **89.18%** | `12730.42` |

A monitoring rule that flags low `avg_leaf_density` catches fixture B and completely misses fixture M, even though the planner is charged the same 2,745 pages in both and a rebuild recovers 10x in both. For planner cost, blocks against the rows the planner expects are what count, and on that reading the two indexes are identical. That reading prices plans; it does not predict what a rebuild returns. `REINDEX` builds a new file for the same index relation ([index.c#reindex_index-rebuild](../../../../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789)), and the build reapplies the index's reloptions: it sets each leaf page's free-space target from the `fillfactor` reloption ([nbtsort.c:665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L665)) and deduplicates only when `deduplicate_items` is on ([nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152), [nbtree.h#BTGetFillFactor-BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1150)). So fixture N's `deduplicate_items = off` index, 2,749 blocks against 852 for the same rows at about 90% density, and fixture A's `fillfactor = 10` twin, 26,411 blocks against 2,745, look bloated by blocks per row, yet by the build code a rebuild would return them at the same size; this page did not rebuild either one. Fixture L3 shows the fillfactor half directly: its `REINDEX` at `fillfactor = 10` turned `l3_c`'s 427 blocks into 3,801. A partial index's population is not its table's row count either: fixture Q's `q_part` holds 20,000 of 200,000 rows by its predicate. This page's size-against-rows reading is a statement about planner cost. It has not been scored against [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md), which governs claims about whether an index should be rebuilt.

#### The planner reads the live block count, not pg_class.relpages

A 200,000-row table with a `fillfactor = 10` index occupying 5,285 blocks priced its whole-index scan at `24640.42`. Forging the catalog on the scratch server, which is a disposable-fixture step and must never be run against a database anyone cares about:

```text
UPDATE /* wiki_bloatplan_fixture_catalog_forgery */ pg_class SET relpages = 1 WHERE relname = 'b_stale_idx';
-- pg_class: relpages = 1, reltuples = 200000; live size 5285 blocks
Index Only Scan using b_stale_idx on b_stale  (cost=0.42..24640.42 rows=200000 width=4)
  Index Cond: (id > 0)
```

The cost did not move, confirming that `get_relation_info()` used `RelationGetNumberOfBlocks()` ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)). A **partial** index behaves differently, because `estimate_rel_size()` derives its tuple count from `pg_class` density ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)): forging `reltuples` from 200,000 to 20 on an otherwise identical partial index (`WHERE id > 0`) moved the cost from `24140.42` to `23140.49` and the startup cost from `0.42` to `0.39`. With 20 claimed tuples, `genericcostestimate()` caps the entries read at the index's `tuples` ([selfuncs.c#genericcostestimate-tuple-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6707-L6715)), so the scan is charged `20 * cpu_index_tuple_cost` instead of `200000 *`, and `ceil(log2(20)) = 5` comparisons instead of 18 ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091)), while the page charge stays at all 5,285 pages. [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten) shows the same path reached by real growth, with no forgery.

The partial index starts `500.00` below the plain one for a reason that has nothing to do with size. Its predicate implies the query's only clause, so `check_index_predicates()` drops that clause from the list the index is matched against, the scan carries no index qual, and its 200,000 tuples pay no `qual_op_cost` ([indxpath.c#check_index_predicates-indrestrictinfo](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3318-L3378), [indxpath.c#match_restriction_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1968-L1974)). The script vacuums every fixture table when it builds it, except fixture I's, which are analyzed but never vacuumed on purpose. Stage `fstale` prices both of its tables straight after that vacuum, so both are all-visible (`relallvisible` 885 of 885), and neither index-only scan pays a heap fetch ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)).

#### A partial index hides its growth until its statistics are rewritten

A partial index is charged for its growth only as fast as the table's row estimate grows, until `ANALYZE`, or a `VACUUM` that counts the index's entries exactly, rewrites the index's `pg_class` row. Then the whole growth arrives at once. A plain index is charged its live size at once. Fixture Q measures both on one table. After the churn the partial index had grown from 57 to 331 blocks and was charged 113 pages; the plain index had grown from 551 to 825 blocks and was charged all 825. After `ANALYZE` the partial index was charged all 331.

Fixture Q: table `q_t (id int, v int, pad text)` with `fillfactor = 100`, 200,000 rows with `v = id`, a partial index `q_part ON q_t (v) WHERE id <= 20000` and a plain index `q_full ON q_t (v)`, then `VACUUM (FREEZE, ANALYZE)`. The churn is ten `UPDATE q_t SET v = v + 1 WHERE id <= 20000` rounds inside one `DO` block. Each round changes an indexed column, so none can be HOT ([heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166)), and each adds an entry to both indexes. The rounds share one transaction, so no replaced version is dead to anyone before the commit ([heapam_visibility.c#HeapTupleSatisfiesVacuumHorizon-same-xact](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1253-L1266), [heapam_visibility.c#HeapTupleSatisfiesVacuumHorizon-xmax-in-progress](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1383-L1386)); the heap doubled exactly, from 1,471 to 2,942 blocks. Nothing writes `pg_class` until the final `ANALYZE q_t`. Each state prices two forced bitmap scans, `SELECT v FROM q_t WHERE id <= 20000 AND v > 0` on the partial index and `SELECT v FROM q_t WHERE v > 0` on the plain one, each with the other index hidden in a rolled-back subtransaction. The charged pages are the `Bitmap Index Scan` node's cost difference between `random_page_cost` 4 and 1, divided by 3: on a single scan `genericcostestimate()` charges `numIndexPages * spc_random_page_cost`, and no other term of the node's cost depends on that setting ([selfuncs.c#genericcostestimate-single-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787)).

| State | `q_t` `relpages` / `reltuples` / live blocks | `q_part` `relpages` / `reltuples` / live blocks | `q_full` `relpages` / `reltuples` / live blocks |
|---|---|---|---|
| built, after `VACUUM (FREEZE, ANALYZE)` | 1,471 / 200,000 / 1,471 | 57 / 20,000 / 57 | 551 / 200,000 / 551 |
| churned, `pg_class` untouched | 1,471 / 200,000 / **2,942** | 57 / 20,000 / **331** | 551 / 200,000 / **825** |
| after `ANALYZE q_t` | 2,942 / 200,000 / 2,942 | 331 / 20,000 / 331 | 825 / 200,000 / 825 |

| State | Partial query: rows, `Bitmap Index Scan` at `random_page_cost` 4 / 1, pages charged | Plain query: rows, `Bitmap Index Scan` at 4 / 1, pages charged |
|---|---|---|
| built | 20,000, `378.29` / `207.29`, **57** | 200,000, `3704.42` / `2051.42`, **551** |
| churned | 40,000, `752.29` / `413.29`, **113** | 400,000, `6300.42` / `3825.42`, **825** |
| after `ANALYZE` | 20,000, `1474.29` / `481.29`, **331** | 200,000, `4800.42` / `2325.42`, **825** |

After the churn `q_part` is 331 blocks and `q_full` 825 by [`pg_relation_size()`](../../../glossary.md#relation-size-functions); `pgstatindex` reports 329 leaves at 77.56% density and 49.85% fragmentation for `q_part` and 820 leaves at 85.14% and 20% for `q_full`, and `bt_metap()` gives their fast-root levels as 1 and 2.

The churned partial charge, step by step from the pinned code:

- The table's row estimate scales `reltuples` by the live heap size ([plancat.c#get_relation_info-table-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L196-L202), [tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747)): `rint(200000 / 1471 * 2942) = 400000`. The dead versions fill heap pages that the recorded density reads as live rows, which is why both row estimates doubled.
- The partial index's `pages` is its live block count, and its `tuples` comes from `estimate_rel_size()`, clamped to the table's rows ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)). `estimate_rel_size()` reports the live block count as `pages`, then scales the recorded density, both counts less the metapage, by the live blocks ([plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)): `rint(20000 / (57 - 1) * (331 - 1)) = 117857`, below `400000`, so no clamp.
- `btcostestimate()` estimates the entries read as the selectivity of the index predicate ANDed with the index qual, times the **table's** rows ([selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019)): `0.1 * 400000 = 40000`, because `id <= 20000` is 10% of the rows and `v > 0` holds for all of them.
- `genericcostestimate()` charges `ceil(40000 * 331 / 117857) = 113` pages ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)). The node's total checks out: `113 * 4.0 + 40000 * (0.005 + 0.0025) + ceil(log2(117857)) * 0.0025 + (1 + 1) * 0.125 = 452 + 300 + 0.0425 + 0.25 = 752.2925`, printed `752.29`.
- The plain index's `tuples` is the table's `400000`, so it is charged `ceil(400000 * 825 / 400000) = 825` pages, its whole live size.
- `ANALYZE` rewrites every index's `relpages` and `reltuples` ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)). Afterwards the partial charge is `ceil(20000 * 331 / 20000) = 331`. The plain index keeps its 825 pages, and its total falls by `1500.00` as printed: `200000 * (0.005 + 0.0025)` for the halved row estimate, plus one fewer `log2` comparison at `0.0025` that the rounding hides.

The partial index's growth cancels out because its live block count appears in both `pages` and `tuples`. `numIndexPages` therefore comes to about `selectivity * table rows / recorded density`, which does not depend on the index's live size. The charge rose from 57 to 113 because the table's row estimate doubled with its heap, not because the index grew 5.8x. Two conditions bound this. It holds while the scaled `tuples` stays below the table's row estimate; past that, `get_relation_info()` clamps `tuples` to the table's rows, and the charge becomes `selectivity * pages`, like a plain index's. And it ends when something rewrites the index's `pg_class` row: `ANALYZE`, measured here; a `VACUUM` whose counts for the index are exact ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)); or a build or rebuild, which writes the index's own counts ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842)). Fixture Q runs neither of the last two. This is the partial-index path of [How the planner obtains its index-size inputs](#how-the-planner-obtains-its-index-size-inputs), reached by real growth.

#### Bloat changes plans

Fixture A with all `enable_*` settings at their defaults, selecting a non-indexed column so a heap fetch is required:

| Selectivity | Dense index | Bloated twin |
|---|---|---|
| 25% (`id BETWEEN 1 AND 250000`) | `Index Scan using a_dense_idx` at `9590.42` | `Seq Scan` at `22353.00`; the index path is rejected |
| 12% (`id BETWEEN 1 AND 120000`) | `Index Scan` at `4606.43` | `Index Scan` at `15966.42` |

The sequential scan is the same `22353.00` on both tables: 7,353 heap pages plus 1,000,000 rows at `cpu_tuple_cost` and two operator evaluations each ([costsize.c#cost_seqscan](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L315-L322)). With exact statistics the row estimates are 250,000 and 120,000 rows.

Bloat also removes an index from a `BitmapAnd`. Fixture L3 is a 500,000-row table with `a = g % 2000` and an independent `c` drawn from a seeded PRNG over 0..19, so the two columns are independent of each other and `a = 5 AND c = 7` selects real rows. With both indexes healthy — `l3_a` at 449 blocks and 85.47% density, `l3_c` at 427 and 89.81% — the two-clause predicate produced a `BitmapAnd` over both under a `Bitmap Heap Scan` whose plan total is `328.51`. The `BitmapAnd` node itself costs `282.26`, of which the `c` bitmap is `275.70` and the `a` bitmap `6.30`. Bloating only the `c` index to 3,801 blocks at 10.37% density, by rebuilding it at `fillfactor = 10`, made the planner drop it and demote `c = 7` to a `Filter`:

```text
Bitmap Heap Scan on l3  (cost=6.30..806.01 rows=12 width=25)
  Recheck Cond: (a = 5)
  Filter: (c = 7)
  ->  Bitmap Index Scan on l3_a  (cost=0.00..6.30 rows=250 width=0)
        Index Cond: (a = 5)
```

Fixture L3 is also one of two fixtures on the page that check the method note's claim that exhaustive statistics make every cost reproducible without thereby making every estimate true; fixture D in the GIN follow-up is the other. Both columns are analyzed exhaustively, and the conjunction is still estimated by multiplying two per-clause selectivities, which assumes independence ([clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)). Recording each estimate beside the true count:

| Predicate | Estimated rows | Actual rows |
|---|---:|---:|
| `a = 5` | 250 | 250 |
| `c = 7` | 24,971 | 24,971 |
| `a = 5 AND c = 7` | 12 | 12 |

All three match, but not for the same reason. The two single-column estimates are exact because `default_statistics_target = 10000` buys each column an MCV list long enough to hold all of its values at their true frequencies: when every sampled value repeats, `ANALYZE` takes the sample's distinct count as the column's, and when every value fits the target it keeps them all ([analyze.c#compute_scalar_stats-stadistinct](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2550-L2560), [analyze.c#compute_scalar_stats-complete-mcv](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2614-L2636)). The conjunction matches by luck of the seed. Independence makes the product right only on average: the estimate is `500000 * (250 / 500000) * (24971 / 500000) = 12.49`, printed 12, while the true count of `c = 7` among the 250 rows with `a = 5` is itself a random draw, with a standard deviation of about 3.4. That it came out at exactly 12 is a property of `setseed(0.42)`, not of the model. Fixture D is built so that its conjunction is exact by construction; see [A GIN index with a large pending list loses to a B-tree](#a-gin-index-with-a-large-pending-list-loses-to-a-b-tree). Either way, agreement is not proof: the multiplication is an assumption, and no fixture on this page violates it, so nothing here bounds the error it can cause. That limit is filed under [Open Questions](#open-questions).

#### Bloat changes parallel worker counts

With `max_parallel_workers_per_gather = 8`, `max_parallel_workers = 8`, `enable_seqscan = off` and `enable_bitmapscan = off`, fixture A's parallel index-only scans differed only in the index. The script also sets `min_parallel_table_scan_size = 0`, which an index-only scan never consults, because `cost_index()` passes it no heap-page estimate ([costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)):

| Index | Blocks | Workers, whole index | Workers, 50% of the keys | Workers, 20% of the keys |
|---|---:|---:|---:|---:|
| `a_dense_idx` | 2,745 | 4 | 3 | 2 |
| `a_sparse_idx` | 26,411 | 6 | 5 | 5 |

Every count follows `compute_parallel_worker()`'s powers-of-three ramp ([allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)) from `min_parallel_index_scan_size` (default 512kB, that is 64 blocks, [guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540): one worker from 64 pages, two from 192, three from 576, four from 1,728, five from 5,184, six from 15,552), applied to the pages the scan is expected to touch. The whole-index scans touch every page, so they read as if the index size set the count. The partial scans show that it does not: the bloated index plans 5 workers for a 20% scan because 20% of 26,411 pages is still 5,283 pages, where the dense index's 549 pages earn 2. If `index->pages` chose the count, every cell in the bloated row would read 6. The two partial-scan columns were planned twice, once at the default `parallel_setup_cost` and `parallel_tuple_cost` and once with both set to `0` (both `PGC_USERSET`, [guc_tables.c#parallel_tuple_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3730-L3740), [guc_tables.c#parallel_setup_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3741-L3751)). The planner chose the same parallel plan with the same worker counts both times, as it did for the whole-index column: each plan is a partial `Aggregate` under `Gather`, so each worker returns one row and the tuple-transfer charge is negligible. Bloat bought two or three extra workers for no extra useful data.

#### Index pages in the cache model

Fixture A again, this time as the inner side of a 50,000-iteration nested loop: `a_outer` holds 50,000 keys spread evenly over the 1,000,000, hash and merge joins are disabled, and `max_parallel_workers_per_gather = 0`. The index-only scan is now repeated, so `genericcostestimate()` prices its pages through `index_pages_fetched()` ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)):

| Inner index | Blocks | Inner scan, per loop | Nested Loop total | Inner scan at `effective_cache_size = 64MB` | Nested Loop total at 64MB |
|---|---:|---|---:|---|---:|
| `a_dense_idx` | 2,745 | `cost=0.42..0.66` | `34327.00` | `cost=0.42..1.38` | `70323.00` |
| `a_sparse_idx` | 26,411 | `cost=0.42..2.50` | `126095.00` | `cost=0.42..3.55` | `178623.00` |

The `count(*)` `Aggregate` above each join adds the same `125.01` in every cell, so the query totals are `34452.01`, `126220.01`, `70448.01` and `178748.01`.

Each loop touches one index page, so with no cache model every loop would pay `4.0`. At the default 4GB the Mackert-Lohman formula caps the dense index's 50,000 fetches at its 2,745 pages, `2745 * 4.0 / 50000 = 0.22` per loop, while the bloated twin's 50,000 fetches are spread over 26,411 pages and come to about 25,700 distinct ones, `2.05` per loop. The remaining `0.44` is the same CPU and descent charge on both. Every table is all-visible, so the heap contributes nothing and the whole difference is the index. Fifty thousand one-page loops reach both indexes' sizes, so the model prices about the whole index spread over the loops, and the sensitivity to bloat here is close to linear. With few loops over a large index the formula counts about one page per loop, and bloat barely registers. Shrinking `effective_cache_size` to 64MB (`PGC_USERSET`, set for the one statement, [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)) makes the prorated cache share smaller than either index, and both costs rise. A bloated index is therefore charged more for *repeated* lookups even though each one touches a single page, which is the part of the penalty a lone point lookup never shows.

#### The v17 ScalarArrayOp descent clamp

Fixture I, with `cpu_operator_cost = 1` so that each additional estimated descent adds about 116 cost units and the clamp is directly readable: 111 of descent CPU (`ceil(log2(2000)) = 11` comparisons plus `(1 + 1) * 50` for two pages), one more index page at `4.0`, and about `1.0` of per-tuple CPU. The tables are analyzed but never vacuumed, so the index-only scan is charged heap access as a plain index scan would be: one random heap page (`4.0`) at every array length, because the heap is in key order, plus `cpu_tuple_cost` per row, identically for both indexes ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-correlation-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L781-L800)). The clamp is `ceil(pages * 0.3333333)`: `3` for the 8-block index, `19` for the 55-block index.

| Array length | `i_small_idx` (8 blocks, clamp 3) | `i_big_idx` (55 blocks, clamp 19) |
|---:|---:|---:|
| 1 | `120.02` | `120.02` |
| 2 | `236.03` | `236.03` |
| 3 | `352.04` | `352.04` |
| 4 | `352.05` | `468.06` |
| 6 | `355.09` | `700.09` |
| 10 | `358.14` | `1164.15` |

The dense index plateaus at exactly three descents and stops charging for longer arrays; the bloated index keeps paying. At ten elements the bloated index is charged `1164.15` against `358.14` for the identical query and identical row estimates, purely because its page count raised the cap. Residual growth on the plateaued rows is the per-tuple term, not descents.

#### Version churn with and without a held snapshot

Fixture P: a 200,000-row table `(id, payload, tag)` with indexes on `payload` and `tag`, where `UPDATE ... SET payload = payload + 1` is non-HOT and therefore writes a logically unchanged duplicate into the `tag` index on every round. Each of the five rounds is its own transaction, and no VACUUM ran. Fixture P sets `tag = id % 1000`; fixture P-100 sets `tag = id % 100` and changes nothing else.

| Fixture | Run | `tag` index blocks after 5 rounds | Density | Fragmentation | `fastlevel` | Bitmap index scan cost, `tag = 7` |
|---|---|---:|---:|---:|---:|---:|
| P, 1,000 tag values | no blocking snapshot | 543 (from 169) | 98.01% | 64.31% | 1 -> 2 | `5.92` |
| P, 1,000 tag values | `REPEATABLE READ` snapshot held elsewhere | 1,173 (from 169) | 82.52% | 42.88% | 1 -> 2 | `9.92` |
| P-100, 100 tag values | no blocking snapshot | 1,020 (from 180) | 91.16% | 18.48% | 1 -> 2 | `59.42` |
| P-100, 100 tag values | `REPEATABLE READ` snapshot held elsewhere | 1,020 (from 180) | 91.11% | 18.48% | 1 -> 2 | `59.42` |

Six row versions exist per logical row after five rounds. In fixture P the blocked run grew 6.9x, close to storing every version, and the unblocked run grew 3.2x. The difference is the work bottom-up index deletion was able to do, and the README names an old snapshot holding up cleanup as exactly the condition that defeats it ([README#deduplication-and-bottom-up-note](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). The `4.00` gap between the two bitmap index scan costs is one page: `ceil(200 * 1173 / 200000) = 2` against `ceil(200 * 543 / 200000) = 1`. Note also that the unblocked run's density reads 98.01%, above the 91.47% its build left at the default leaf fillfactor of 90 ([nbtree.h:200](../../../../raw/postgres-17/src/include/access/nbtree.h#L200)): fillfactor sets how full a build or a split leaves a page, not how full later inserts can make it (see [Low-density live leaf pages](#low-density-live-leaf-pages)). Which of deduplication, bottom-up deletion and the single-value split strategy did the packing here was not traced; it is filed under [Open Questions](#open-questions).

Fixture P-100 is the control that keeps this result from being over-read. With 2,000 rows per tag value instead of 200, the same five rounds grew the index 5.7x to the same 1,020 blocks whether or not a snapshot was held, so there the free horizon bought nothing. Why was not traced; it is filed under [Open Questions](#open-questions).

#### Dead entries at the end of a B-tree: the endpoint probe

Dead entries at the end of a B-tree cost planning work and can hold a row estimate at a stale value. They add no page charge of their own: the probe reads no page the cost model counts, and in fixture E the index-only scan's cost moves only with the row estimate, `4.59` at `rows=10` against `4.44` at `rows=1`. When the histogram search for a range predicate reaches a column's first or last histogram bound, the planner reads the column's true minimum or maximum from a B-tree. Each such read, one probe, walks past dead entries on at most 100 heap pages and gives up on the 101st; the mechanism is [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree). Fixture E measures it. In each of four consecutive plans the probe passed exactly 100 heap pages of dead rows, gave up on the 101st, and left a stale 10-row estimate. Each also marked the entries it passed as dead, so the fifth plan reached a live row and the estimate fell to 1.

Fixture E: `e_t (id int, v int)`, 200,000 rows, primary key `e_t_pkey`, then `VACUUM (FREEZE, ANALYZE)`, which leaves 885 heap blocks and an `id` histogram of 10,001 bounds ending at 200000. `DELETE FROM e_t WHERE id > 100000` follows, with no `VACUUM` and no `ANALYZE` before the six plans; a plain `VACUUM` runs after them. Each plan is an `EXPLAIN` of `SELECT count(*) FROM e_t WHERE id > 199990` at default settings, followed by `sum(dead_items)` over `bt_multi_page_stats('e_t_pkey', 1, -1)`:

| Moment | Index Only Scan estimate | Plan total | Index entries marked dead |
|---|---:|---:|---:|
| before the delete | `rows=10` | `4.63` | not read |
| after the delete, before any plan | | | 0 |
| plan 1 | `rows=10` | `4.63` | 22,590 |
| plan 2 | `rows=10` | `4.63` | 45,190 |
| plan 3 | `rows=10` | `4.63` | 67,790 |
| plan 4 | `rows=10` | `4.63` | 90,390 |
| plan 5 | `rows=1` | `4.45` | 100,000 |
| plan 6 | `rows=1` | `4.45` | 100,000 |
| after a plain `VACUUM` | `rows=1` | `4.45` | 0 |

The plan total is the `count(*)` `Aggregate`; the `Index Only Scan` under it costs `0.42..4.59` at `rows=10` and `0.42..4.44` at `rows=1`. Nothing here is timed. Every plan is `EXPLAIN` alone and `bt_multi_page_stats()` only reads pages, so the planner's probe is the only thing that can have marked the entries dead.

The steps, from the pinned source:

1. `id > 199990` falls in the histogram's last bin, so `ineq_histogram_selectivity()`'s binary search reaches the last bound. Before comparing with it, the function asks `get_actual_variable_range()` for the column's current maximum ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)).
2. `get_actual_variable_range()` uses only a B-tree that is neither partial nor hypothetical and whose first column is the variable, with a matching collation and sort operator ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)). `e_t_pkey` qualifies.
3. `get_actual_variable_endpoint()` reads the index from that end as an index-only scan under `SnapshotNonVacuumable`. An entry whose heap page is not all-visible costs a heap fetch. When the fetch finds no acceptable row, the probe counts that heap page if it differs from the last page it counted, and moves to the next entry. The fetch runs before the count, so the probe stops on the fetch that takes the count past `VISITED_PAGES_LIMIT` (100): the 101st page it reads ([selfuncs.c#get_actual_variable_endpoint-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6458)). The caller then keeps the bound stored in `pg_statistic` ([selfuncs.c#get_actual_variable_endpoint-comment](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6371-L6414)).
4. The source comment says that an entry `SnapshotNonVacuumable` rejects is one the index scan will mark dead, so that "the next get_actual_variable_endpoint() call will not have to re-consider that index entry" ([selfuncs.c#get_actual_variable_endpoint-comment](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6371-L6414)). The marking runs through the ordinary index-scan path: `index_fetch_heap()` sets `kill_prior_tuple` when the whole chain is dead ([indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652)), `btgettuple()` records the item on its next call ([nbtree.c#btgettuple-kill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L226-L245)), and the recorded items are marked dead when the scan leaves the leaf page or ends ([nbtsearch.c#_bt_steppage-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2049-L2051), [nbtree.c#btendscan-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L424-L426)).

The counts follow from the loop:

- 885 heap blocks hold 200,000 rows at 226 to a full block, so the last block holds `200000 - 884 * 226 = 216`. Plan 1 marked the dead entries of that block and 99 full ones, `216 + 99 * 226 = 22590`, exactly 100 heap pages. The fetch on the 101st page trips the limit, and its entry is never recorded, because no further `btgettuple()` call follows.
- Plans 2 to 4 each passed 100 more full blocks, `100 * 226 = 22600` entries each.
- Plan 5 found `100000 - 90390 = 9610` dead entries left: 118 on block 442, which holds `id` 99,893 to 100,118, and `42 * 226 = 9492` on blocks 443 to 484. That is 43 heap pages, under the limit, and the next entry, `id` 100,000 on block 442, is live.
- While plans 1 to 4 gave up, the stale bound 200000 kept `id > 199990` at half of the last 20-row bin: `200000 * (1 - 9999.5 / 10000) = 10` rows. Plan 5's new maximum, 100000, put `199990` above every bound, so the selectivity became 0 and the estimate fell to the one-row floor ([selfuncs.c#ineq_histogram_selectivity-above-last-bound](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1162-L1168), [selfuncs.c#ineq_histogram_selectivity-flip-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1319-L1344), [costsize.c#clamp_row_est](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L202-L218)).
- The plain `VACUUM` removed every dead entry. The histogram still ends at 200000, but the probe now reads 100000 at once, so the estimate stays at 1.

No cost term saw the dead entries. The two `Index Only Scan` totals, `4.59` and `4.44`, differ only by nine estimated rows at `0.005 + 0.0025 + 0.01 = 0.0175` each, `0.1575` before rounding; the page, descent and level terms are the same in both, because one page covers either estimate ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810), [costsize.c#cost_index-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800)). Neither total has a heap term either, although the delete cleared the visibility-map bits of every page it touched ([heapam.c#heap_delete-clear-vm](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3120-L3146)) and step 3 relies on those cleared bits. The planner does not read the map for this. It takes the table's all-visible fraction from `pg_class.relallvisible` ([plancat.c#get_relation_info-table-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L196-L202), [tableam.c#table_block_relation_estimate_size-allvisfrac](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L749-L760)), which `VACUUM` and `ANALYZE` refresh from the map ([vacuumlazy.c#heap_vacuum_rel-relstats](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575), [analyze.c#do_analyze_rel-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645)), and the stage runs neither between the delete and plan 6. The totals show the fraction was still 1. Below 1, the `rows=10` plans would fetch at least one heap page, `ceil(ceil(10 / 200000 * 885) * (1 - fraction)) = 1`, and add at least one `random_page_cost`, 4.00 ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-correlation-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L781-L800)). The index terms and `cpu_tuple_cost` already account for the whole total: `0.42 + 1 * 4.0 + 10 * (0.005 + 0.0025) + 10 * 0.01 = 4.595`, printed `4.59`.

What the dead entries cost is planning work, which no plan cost prices. One probe reads up to 101 heap pages: 100 that yield nothing, and the 101st, whose fetch trips the limit ([selfuncs.c#get_actual_variable_endpoint-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6458)). The limit binds each probe, not each plan. `ineq_histogram_selectivity()` calls `get_actual_variable_range()` whenever a clause's binary search reaches the first or last bound, and for both ends when the histogram has only two bounds ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)). That function runs a separate `get_actual_variable_endpoint()` call for the minimum and for the maximum ([selfuncs.c#get_actual_variable_range-endpoint-calls](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6282-L6314)), and each call starts its own page count at zero ([selfuncs.c:6365](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6365)). So a plan pays up to 101 heap pages for every probe it makes, and a plan whose range clauses reach several histogram ends makes several probes. Fixture E's single clause made one probe per plan, which is what its 22,590 and 22,600 counts show. Later plans pay again, each starting past the entries earlier probes marked dead, until a probe reaches a live entry within its limit or a `VACUUM` removes the dead entries, as plans 1 to 5 and the final `VACUUM` show. Fixture E ran with no other session, so every deleted row was dead to all transactions. Under a snapshot old enough to still see the deleted rows, `SnapshotNonVacuumable` accepts them as recently dead ([selfuncs.c#get_actual_variable_endpoint-comment](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6371-L6414)), so the probe would stop at the first one and return the deleted maximum without marking anything; fixture E does not measure that case. The 100-page limit has its own history; see [v16, back-patched: the endpoint probe gives up after 100 heap pages](#v16-back-patched-the-endpoint-probe-gives-up-after-100-heap-pages).

#### Deduplication

Fixture N, 1,000,000 rows over 100 distinct keys, two indexes on the same column:

| Index | Blocks | Density | `WHERE k = 5` | `WHERE k > 0` |
|---|---:|---:|---:|---:|
| `deduplicate_items = on` | 852 | 89.70% | `24021.01` | `50111.17` |
| `deduplicate_items = off` | 2,749 | 90.16% | `24097.01` | `57623.17` |

Each index is priced with the other one dropped inside a rolled-back subtransaction. These are plain index scans on `SELECT *`, so every cost also contains the same heap fetches in both rows; only the gaps belong to the index, and both gaps are pure page arithmetic. For `k > 0` (990,000 rows estimated), touched pages are `ceil(990000 * 852 / 1000000) = 844` against `ceil(990000 * 2749 / 1000000) = 2722`, and `(2722 - 844) * 4.0 = 7512.00`, exactly the observed `57623.17 - 50111.17`. For `k = 5` (10,000 rows), `(28 - 9) * 4.0 = 76.00`, exactly the observed gap. Note that `avg_leaf_density` is ~90% in both cases: it cannot see that one index stores 3.2x more pages for the same information.

### What Changed Since PostgreSQL 12

Every attribution below is anchored to the pinned v17 checkout's own commit history, and the method needs no tags, because this checkout carries none: it is a single-branch clone of `REL_17_STABLE`. A commit on the main line is assigned to the first major version whose branch point follows it, and the branch points are the `Stamp HEAD as NNdevel.` commits that open each development cycle: `615cebc94b` (13devel, 2019-07-01), `d10b19e224` (14devel, 2020-06-07), `596b5af1d3` (15devel, 2021-06-28), `d31d30973a` (16devel, 2022-06-30) and `5bcc7e6dc8` (17devel, 2023-06-29). A commit that has `d10b19e224` as an ancestor but not `596b5af1d3` first shipped in PostgreSQL 14, which is what "first in `REL_14_0`" means below. The rule dates the main-line commit only. A separate back-patch can carry all or part of a change into an older branch's minor releases: `9c6ad5eaa9` below reached 12.14, and `9f3665fbfc`'s message says a version of it went to 13, where, per `effdd3f3b6`'s message, it disabled `vacuum_cleanup_index_scale_factor` rather than removing it. For the same reason the range `615cebc94b..HEAD` can list a main-line commit whose back-patch `REL_12_0` already contains: `d3751adcf1` (2019-07-12) appears in the `get_actual_variable_endpoint()` history, but its `REL_12_STABLE` twin `cee976c4e8` is an ancestor of `REL_12_0`. The commit counts below are not affected: every commit they count is dated 2020 to 2024, after `REL_12_0` (2019-09-30). Within 17.x, a commit is assigned to the first `Stamp 17.N.` commit that contains it. "Since `REL_12_0`" in a `git log -L` claim means the range `615cebc94b..HEAD`. The `REL_12_0` release commit lives on the `REL_12_STABLE` branch and is not an ancestor of `REL_17_STABLE`, so without tags this checkout cannot show its text. The byte-for-byte comparisons below therefore read it from the v12 checkout's own `REL_12_0` tag (`git -C raw/postgres-12 show REL_12_0:<path>`), which is the matching-version evidence a cross-version claim needs.

#### The cost code that did not change at all

Comparing the `REL_12_0` file text with the pin:

- `index_pages_fetched()` is byte-identical.
- `cost_index()` is byte-identical.
- The bloat-relevant core of `genericcostestimate()` is unchanged: the `numIndexPages` prorating formula, the `index->pages > 1 && index->tuples > 1` guard, the Mackert-Lohman call, and the single-scan `numIndexPages * spc_random_page_cost` charge. The function's whole diff against `REL_12_0` is one hunk from two v17 commits: `5bf748b86bc` made `num_sa_scans` a caller-supplied input, and `9391f71523b` changed the `estimate_array_length()` call to pass `root` and to return a `double`.
- Exactly five commits touched `btcostestimate()` since `REL_12_0`, and only two are functional: `5bf748b86bc` and `9391f71523b`. The remaining three are a typo fix (`950d4a2cb1d`), a `MemSet`-to-struct-initializer change (`9fd45870c14`), and the macro extraction (`eb5c4e953bb`).

So a reader who knows the v12 model already knows most of the v17 model.

#### v16: the 50x page charge became a macro

`eb5c4e953bb` "Extract the multiplier for CPU process cost of index page into a macro" (2023-01-08, first in `REL_16_0`) replaced three literal `50.0` occurrences with `DEFAULT_PAGE_CPU_MULTIPLIER`. `REL_12_0` already charged `(index->tree_height + 1) * 50.0 * cpu_operator_cost` in `btcostestimate`, `gistcostestimate` and `spgcostestimate`. The value and the behavior are unchanged; only the spelling moved ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).

The companion commit `cd9479af2af` "Improve GIN cost estimation" (also first in `REL_16_0`) newly applied the same per-page CPU multiplier inside `gincostestimate()` ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955)), which changes how GIN bloat is priced but does not touch the B-tree path.

#### v13: GIN's whole-index estimate narrowed

`4b754d6c16e` "Avoid full scan of GIN indexes when possible" (2020-01-18, first in `REL_13_0`) is the other commit that touched `gincostestimate()` since `REL_12_0`; it also changed GIN's executor. In `REL_12_0`, any `GIN_SEARCH_MODE_ALL` key set a single `haveFullScan` flag, and `gincostestimate()` then priced the scan as if every entry in the index had been listed in the query (`if (counts.haveFullScan || indexQuals == NIL)` in the `REL_12_0` text). Since v13 the flags are per column. A match-all key sets `attHasFullScan` and a default or include-empty key sets `attHasNormalScan` ([selfuncs.c#gincost_pattern-searchmode](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7454-L7489)), and the whole-index branch fires only when some column has a full-scan key and no normal key ([selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877)). A query that pairs a match-all key with an ordinary key on the same column is therefore no longer charged for every entry of a possibly bloated GIN index. No v12 server was run to compare the two.

#### v17: index pages now cap ScalarArrayOp descents

`5bf748b86bc` "Enhance nbtree ScalarArrayOp execution." (2024-04-06, first in `REL_17_0`) is the only commit since v12 that added a new cost input, a page-count term, through which a bloated B-tree is charged more than its dense twin. `9c6ad5eaa9` (v16) can raise a bloated B-tree's charge only indirectly, through a stale row estimate; see [v16, back-patched: the endpoint probe gives up after 100 heap pages](#v16-back-patched-the-endpoint-probe-gives-up-after-100-heap-pages). It introduced the clamp `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))` ([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042)). It made `GenericCosts.num_sa_scans`, which `REL_12_0`'s `selfuncs.h` already declared as an output ("# indexscans from ScalarArrayOps"), an input as well ([selfuncs.h#GenericCosts](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L108-L138)): `btcostestimate()` hands its own estimate over ([selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073)), and `genericcostestimate()` counts the arrays itself only when it is given less than one ([selfuncs.c#genericcostestimate-num_sa_scans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6657-L6679)). And it reworded the descent comments from "per SA scan" to "per estimated SA index descent" ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091), [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)).

In `REL_12_0` there was no clamp: `num_sa_scans` was simply the product of array lengths, computed inside `genericcostestimate()`. The measured consequence in v17 is that `index->pages` now sets the ceiling on descent charges, so the dense and bloated twins in fixture I diverge by 3.25x on a ten-element `= ANY` where v12's formula had no page-count input at that point at all.

How v17's descent count compares with v12's depends on the array and on where the `= ANY` sits:

- **A constant list in a boundary qual.** A `Const` array or an `ARRAY[...]` list has the same length in both versions ([selfuncs.c#estimate_array_length](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2131-L2207)), so the clamp can only lower v12's count. v17 charges fewer descents exactly when `ceil(index pages / 3)` is below the product of the boundary arrays' lengths ([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042)). Here a bloated index keeps more of v12's descent charge than its rebuilt twin does; it is not charged more than v12 charged it. Fixture I is this case.
- **An `= ANY` outside the boundary quals.** `btcostestimate()` counts descents only from the boundary quals, which stop at the first index column that has no `=` qual ([selfuncs.c#btcostestimate-bound-saop](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6904-L6989)), and since `5bf748b86bc` that count is the one `genericcostestimate()` uses. `REL_12_0`'s `genericcostestimate()` multiplied the lengths of every `ScalarArrayOpExpr` among the index quals, and `btcostestimate()` charged its descent costs by that product. So on an index on `(a, b, c)`, a path that carries `c = ANY('{...}')` behind `a = 1` was charged one descent per element in v12 and is charged one descent in v17, whatever the index size.
- **A non-constant array.** `9391f71523b` (below) sizes such an array from the average distinct-element count in its statistics, and falls back to v12's flat 10 only when it finds none ([selfuncs.c#estimate_array_length](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2131-L2207)). A parameterized `t.id = ANY(o.ids)` is a valid index clause, because the array side only has to avoid the index's own table and volatile functions ([indxpath.c#match_saopclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2651-L2669)). If the statistics report 40 elements, v17 charges `Min(40, ceil(pages / 3))` descents where v12 charged 10. An index of more than 30 pages is then charged more descents than in v12, and a bloated index more than its dense twin while the twin's cap is below 40.

No v12 server was run for this page, and no fixture here builds the second or third case.

The related commit `9391f71523b` "Teach estimate_array_length() to use statistics where available." (2024-01-04, also `REL_17_0`) is the change behind the third case: it changed how the *unclamped* array length is estimated, which feeds the same variable.

#### v16: partitioned indexes are zeroed out

`3c569049b7b` "Allow left join removals and unique joins on partitioned tables" (2023-01-09, first in `REL_16_0`) stopped skipping partitioned indexes in `get_relation_info()`, so that their uniqueness can prove joins, and added the `RELKIND_PARTITIONED_INDEX` guard ([plancat.c:471](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471)) that lists them with `pages = 0`, `tuples = 0.0` and `tree_height = -1` ([plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)). `REL_12_0`'s `get_relation_info()` skipped them before sizing, with the comment "Ignore partitioned indexes, since they are not usable for queries", so neither version gives a partitioned parent index a size and bloat pricing is unchanged. The guard is the only structural change to `get_relation_info()`'s index-size block since `REL_12_0`.

#### v14: reltuples turns negative for never-analyzed relations

`3d351d916b2` "Redefine pg_class.reltuples to be -1 before the first VACUUM or ANALYZE." (2020-08-30, first in `REL_14_0`) changed `estimate_rel_size()`'s index branch from `if (relpages > 0)` to `if (reltuples >= 0 && relpages > 0)` ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)), and the catalog header now documents `-1` as "unknown" ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). For an index, "never analyzed" is not the condition that matters, and a new index does not start at `-1`:

- `index_create()` inserts the zero-filled row that `RelationBuildLocalRelation()` built, so a new index starts at `relpages = 0` and `reltuples = 0` ([index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024), [relcache.c#RelationBuildLocalRelation-rd_rel](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3656-L3659)). Only a new table's row gets `-1` ([heap.c#AddNewRelationTuple-empty](../../../../raw/postgres-17/src/backend/catalog/heap.c#L1006-L1009)).
- An index build then writes `relpages` and `reltuples` itself, analyzed or not. It skips the write in two cases: when it found no entries and the row already held `-1`, a hack `3d351d916b2` itself added, and always during binary upgrade, where the index is built before its data is moved into place. The second skip is `71b66171d0` "CREATE INDEX: do not update stats during binary upgrade." (2024-04-03, first in `REL_17_0`) ([index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)). So an index that `pg_upgrade` creates keeps `relpages = 0` and `reltuples = 0` until ANALYZE, or a VACUUM whose index pass reports an exact count, writes its row ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)); a cleanup-only B-tree pass reports an estimate and does not write it ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)).
- A rebuild through `reindex_index()` first resets the row to `relpages = 0` and `reltuples = -1` and then builds the index ([relcache.c#RelationSetNewRelfilenumber-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3948-L3954), [index.c#reindex_index-rebuild](../../../../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789)). A plain `REINDEX` takes that path, and so does every command that [rewrites the table](../../../glossary.md#table-rewrite). `TRUNCATE` calls `reindex_relation()` directly ([tablecmds.c#ExecuteTruncateGuts-rewrite](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2167-L2189)). The exception is a table created or given new storage in the current subtransaction: `TRUNCATE` empties it in place and rebuilds each index without the reset ([tablecmds.c#ExecuteTruncateGuts-in-place](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2133-L2145), [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3083-L3087)), so that path can keep an earlier `-1` but never sets one. [`VACUUM FULL`](../../../glossary.md#vacuum-full), [`CLUSTER`](../../../glossary.md#cluster), a table-rewriting `ALTER TABLE` and a non-concurrent `REFRESH MATERIALIZED VIEW` end in `finish_heap_swap()` ([vacuum.c:2260](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2260), [cluster.c:670](../../../../raw/postgres-17/src/backend/commands/cluster.c#L670), [tablecmds.c:5873](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L5873), [matview.c:890](../../../../raw/postgres-17/src/backend/commands/matview.c#L890)), which calls `reindex_relation()` too ([cluster.c:1508](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1508)). `reindex_relation()` rebuilds each index with `reindex_index()` ([index.c:4048](../../../../raw/postgres-17/src/backend/catalog/index.c#L4048)). [`REINDEX CONCURRENTLY`](../../../glossary.md#concurrently) does not take this path: it builds a new index with `index_create()` ([index.c:1459](../../../../raw/postgres-17/src/backend/catalog/index.c#L1459)), whose row starts at `0` and `0` like any new index.

So an index holds `-1` only after a rebuild through `reindex_index()` produced no entries. `index_build()` passes the build's own entry count to `index_update_stats()`, and the hack keeps the reset's `-1` when that count is zero ([index.c#index_build-index-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3133-L3135), [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)). The hack tests the index's count, not the table's. A B-tree build counts each row the heap scan hands it ([nbtsort.c:599](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L599), [nbtsort.c:338](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L338)), and the scan hands over no row that is dead to every transaction and no row that fails a partial index's predicate ([heapam_handler.c#index-build-dead](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1419-L1423), [heapam_handler.c#index-build-predicate](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1636-L1644)). So a rebuild produces no entries when the table is empty, when it holds only such dead rows, or when no row satisfies a partial index's predicate, however many other rows the table holds. The index keeps `-1` while later inserts fill it, until ANALYZE or a VACUUM with an exact count writes the row ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)). Its catalog `relpages` stays `0` all that time, so the new half of the test changes nothing for it. A non-partial index never reads its own `reltuples` at all: its tuple count is the table's estimate ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)).

`REL_12_0` had no `-1` state for an index: its rewrite reset the row to `relpages = 0` and `reltuples = 0`, and every build then wrote its block count and entry count, because the hack did not exist yet (`REL_12_0` `relcache.c` and `index.c`). Take a partial B-tree whose rebuild produced no entries, so that the build wrote the metapage alone, and which later inserts then filled. Both versions take `estimate_rel_size()`'s width-based fallback for it. They differ in one page. `REL_12_0` held `relpages = 1` and discounted the metapage; v17 holds `relpages = 0`, skips the discount ([plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)), and so estimates one page's worth of fallback density more before `get_relation_info()` clamps the result to the table's rows.

#### v16, back-patched: the endpoint probe gives up after 100 heap pages

`9c6ad5eaa9` "YA attempt at taming worst-case behavior of get_actual_variable_range." (2022-11-22, first in `REL_16_0`) changed how dead index entries at the end of a B-tree reach the planner. It does not touch a cost formula; it changes a row estimate.

When a range comparison's histogram search reaches the first or last bound, `ineq_histogram_selectivity()` asks `get_actual_variable_range()` for the column's current minimum or maximum instead ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)). That function uses only a B-tree that is not partial and not hypothetical and whose first column matches the compared expression, its collation and its operator ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)). `get_actual_variable_endpoint()` then reads the index from that end under `SnapshotNonVacuumable`, which rejects rows already dead to every snapshot and accepts recently dead and uncommitted ones ([selfuncs.c#get_actual_variable_endpoint-snapshot](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6376-L6386), [selfuncs.c#get_actual_variable_endpoint-horizon](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6416)). Since this commit it counts a heap page each time a rejected entry points to a different heap page than the one before, and gives up when that count passes 100 ([selfuncs.c#VISITED_PAGES_LIMIT](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447-L6455)). The caller then keeps whatever bound `pg_statistic` recorded ([selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413)). Every entry the probe steps past is marked killed by the index scan, so the next plan starts past it; the entry whose heap fetch trips the limit is rejected but not marked ([selfuncs.c#get_actual_variable_endpoint-killed](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6388-L6397), [indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652)). That holds only outside recovery. A transaction that started during recovery neither marks entries killed nor skips entries already marked ([indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652), [genam.c#RelationGetIndexScan-recovery](../../../../raw/postgres-17/src/backend/access/index/genam.c#L107-L119)). So on a hot standby every plan rejects the same entries again, and a run of dead entries that spans more than 100 heap pages keeps the estimate on the stale bound until the standby replays the removal of those entries, such as a VACUUM on the primary, or replays an `ANALYZE` that replaces the stale bound in `pg_statistic` ([nbtxlog.c#btree_xlog_vacuum](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L598-L634)).

`REL_12_0`'s `get_actual_variable_endpoint()` has no page limit, so it walked the whole run of dead entries. That cost planning time, but dead entries sent the estimate back to the histogram bound only when no live entry remained at all. Since the limit, when a delete at one end of the index leaves dead entries spanning more than 100 heap pages, the next plans fall back to the stale bound and can estimate a different row count. On a primary that lasts until killed entries let a plan reach an accepted row, `VACUUM` removes the entries, or `ANALYZE` rebuilds the histogram from live rows; on a hot standby only a replayed removal or a replayed `ANALYZE` ends it. Fixture E, run on a single primary server, measures the primary case: four plans at `rows=10`, each marking another 100 heap pages' worth of entries dead (22,590 after the first plan, 90,390 after the fourth), then a fifth plan at `rows=1`; see [Dead entries at the end of a B-tree: the endpoint probe](#dead-entries-at-the-end-of-a-b-tree-the-endpoint-probe). The same probe is described as a planning-time mechanism in [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree).

The commit message says "Back-patch to all supported branches". In the v12 checkout its `REL_12_STABLE` twin `ec10b6139c` is contained in `REL_12_14` and in neither `REL_12_0` nor the wiki's v12 pin `45b88269` (`REL_12_2`), so 12.14 and later 12.x releases behave like v17 here and the wiki's v12 pin does not. An earlier commit on the same function, `dc7420c2c9` (2020-08-12, first in `REL_14_0`), replaced the snapshot's horizon, `RecentGlobalXmin` in `REL_12_0`, with `GlobalVisTestFor(heapRel)` ([selfuncs.c#get_actual_variable_endpoint-horizon](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6416)).

#### v13: deduplication

`0d861bbb702` "Add deduplication to nbtree." (2020-02-26, first in `REL_13_0`) introduced posting-list tuples, deduplication during index builds in `nbtsort.c`, the lazy pre-split deduplication pass in the new `nbtdedup.c`, and the `deduplicate_items` reloption, which does not exist in `REL_12_0`'s `reloptions.c`. It does not change any cost formula; it changes how many pages a duplicate-heavy index needs, and therefore what the unchanged formula is fed. Fixture N measures 852 blocks against 2,749 for the same data with the feature disabled.

An index built on PostgreSQL 12 and carried into v17 by `pg_upgrade` does not deduplicate until it is rebuilt. Deduplication needs the metapage's `btm_allequalimage` flag, only `CREATE INDEX` or `REINDEX` sets it, and "pg_upgrade hasn't been taught to set the metapage field" ([nbtree.h#allequalimage-upgrade](../../../../raw/postgres-17/src/include/access/nbtree.h#L135-L141), [btree.sgml#deduplication-safety](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L838)). An `INCLUDE` index, or one with an operator class that lacks a `BTEQUALIMAGE_PROC` returning true, never deduplicates ([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)). Bottom-up deletion, below, has no such gate: `_bt_delete_or_dedup_one_page()` runs it before the `allequalimage` test that guards deduplication ([nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782)). So an upgraded v12 index gets bottom-up deletion at once but deduplication only after a `REINDEX`.

#### v14: bottom-up index deletion

`d168b666823` "Enhance nbtree index tuple deletion." (2021-01-13, first in `REL_14_0`) added `_bt_bottomupdel_pass()`. The v17 documentation states the boundary directly: "Prior to PostgreSQL 14, the only category of B-Tree deletion was simple deletion" ([btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703)), and the README says bottom-up index deletion "was added to PostgreSQL 14" ([README#deduplication-and-bottom-up-note](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). The docs claim it is possible for such an index's on-disk size to "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)); fixture P measures a 3.2x growth under a deliberately harsh five-round whole-table churn with no VACUUM, against 6.9x when an old snapshot blocks deletion, and fixture P-100 measures no difference at all on a column with ten times fewer distinct values.

#### v16: updates of only summarizing-index columns can stay HOT

`19d8e2308b` "Ignore BRIN indexes when checking for HOT updates" (2023-03-20, first in `REL_16_0`) removed one source of version-churn B-tree entries. Its message says it re-applies `5753d4ee32`, which `e3fcca0d0d` reverted; both of those fall in the 15 development cycle, before 15 branched, so they net to nothing.

In `REL_12_0`, `heap_update()` tested the modified columns against `INDEX_ATTR_BITMAP_ALL`, the columns of every index on the table, BRIN included, and the table AM told the executor only whether to insert into every index or into none (`REL_12_0` `heapam.c` and `heapam_handler.c`). So an `UPDATE` that changed only a BRIN-indexed column was never HOT and wrote a new entry into every B-tree on the table. In v17 the relcache keeps separate column sets for hot-blocking and for summarizing indexes, chosen by the AM's `amsummarizing` flag ([relcache.c#summarizing-split](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5390-L5398)), which BRIN sets ([brin.c:269](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L269)). `heap_update()` tests HOT against the hot-blocking set only ([heapam.c#heap_update-attr-bitmaps](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3434-L3437), [heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166)) and reports `TU_Summarizing` when the update touched only summarizing columns ([heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429), [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127)). `ExecInsertIndexTuples()` then skips every non-summarizing index ([execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366)). The saving holds only while the new version fits on the old page; [Version-churn duplicates from non-HOT UPDATEs](#version-churn-duplicates-from-non-hot-updates) describes the v17 rule and its limit. No fixture here measures it.

#### v14: faster recycling of deleted pages

`9dd963ae253` "Recycle nbtree pages deleted during same VACUUM." (2021-03-21, first in `REL_14_0`). The README describes the change in its own words: before v14 VACUUM placed only *previously* deleted pages in the FSM, and "PostgreSQL 14 added the ability for VACUUM to consider if it's possible to recycle newly deleted pages at the end of the full index scan where the page deletion took place" ([README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424)). This shortens the window in which deleted pages are dead weight. Recycling still only records a deleted page in the FSM for reuse ([README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)), and nothing under `src/backend/access/nbtree/` calls `RelationTruncate()` or `smgrtruncate()`, so the file never shrinks. The planner reads the live block count ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)) and keeps paying for those pages until a rebuild. Fixture M measures 2,465 deleted pages still inside a 2,745-block index after three VACUUMs.

Two earlier `REL_14_0` commits changed the same recycling path. `e5d8a99903` "Use full 64-bit XIDs in deleted nbtree pages." (2021-02-24) stores a 64-bit `safexid` in each deleted page ([nbtree.h#BTDeletedPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L230-L236)), which its message says removes the risk of "leaking" deleted pages "by making them non-recyclable indefinitely". It also replaced the metapage's oldest-[XID](../../../glossary.md#transaction-id) field, `btm_oldest_btpo_xact` in `REL_12_0`, with `btm_last_cleanup_num_delpages` ([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)). `_bt_vacuum_needs_cleanup()` uses that count to decide whether a cleanup-only VACUUM must scan the index: it asks for a scan when the count exceeds 5% of the index's blocks, or when the metapage is too old to hold the count ([nbtpage.c#_bt_vacuum_needs_cleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L172-L223)).

`9f3665fbfc` "Don't consider newly inserted tuples in nbtree VACUUM." (2021-03-10) removed the `vacuum_cleanup_index_scale_factor` setting, so inserted tuples no longer trigger that scan. `effdd3f3b6` "Add back vacuum_cleanup_index_scale_factor parameter." (2021-03-11, also `REL_14_0`) restored the B-tree storage parameter, though not the GUC, so that dumps from older versions still load; it is marked "Deprecated B-Tree parameter." and no code reads its value ([reloptions.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L462-L470), [nbtree.h:1134](../../../../raw/postgres-17/src/include/access/nbtree.h#L1134)). `9f3665fbfc` also marks the tuple count of a cleanup-only B-tree scan as an estimate ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)), and VACUUM writes an index's `relpages` and `reltuples` only from an exact count ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)). So a VACUUM that only cleans up a B-tree no longer refreshes its `pg_class` row. `REL_12_0` did: its `btvacuumcleanup()` left the count marked exact and its `lazy_cleanup_index()` wrote the row. The planner's size estimate reads that row only for a partial index, whose tuple count comes from `pg_class` density ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)); [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten) measures what a stale row costs. Neither commit changes a cost formula.

#### v14: VACUUM can skip index vacuuming on its own

Two `REL_14_0` commits made VACUUM skip index vacuuming by itself: `5100010ee4d` "Teach VACUUM to bypass unnecessary index vacuuming." (the 2% `BYPASS_THRESHOLD_PAGES` rule) and `1e55e7d1755` "Add wraparound failsafe to VACUUM.". `REL_12_0` skipped index vacuuming only on request: `VACUUM (INDEX_CLEANUP false)` or the boolean `vacuum_index_cleanup = false` reloption skipped index vacuuming and index cleanup together (`REL_12_0` `vacuum.c` and `vacuumlazy.c`). A third v14 commit, `3499df0dee8` "Support disabling index bypassing by VACUUM." (2021-06-18), made the setting tri-valued and added `auto`. `auto` is now the reloption's default, and a VACUUM that names no `INDEX_CLEANUP` takes the reloption's value ([reloptions.c#vacuum_index_cleanup](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L510-L520), [vacuum.c#index_cleanup-default](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179)). `INDEX_CLEANUP ON` / `vacuum_index_cleanup = on` now forces index vacuuming by switching the 2% bypass off, though it cannot switch off the failsafe ([vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402)). The 2% bypass clears only `do_index_vacuuming`, so it never skipped index cleanup ([vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)); the failsafe clears `do_index_cleanup` as well ([vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326)).

#### Since-v12 summary table

| Change | First release | Effect on planner-visible bloat |
|---|---|---|
| `0d861bbb702` deduplication | 13 | Fewer pages for duplicate-heavy indexes built or rebuilt on v13 or later; an index `pg_upgrade` carried over from v12 needs a `REINDEX` first; measured 3.2x |
| `4b754d6c16e` per-column GIN full-scan test | 13 | Fewer GIN scans charged for every entry of a possibly bloated index: only a column with a match-all key and no normal key triggers the whole-index estimate |
| `3d351d916b2` `reltuples = -1` | 14 | Documents `-1` as unknown; an index holds it only after a rebuild through `reindex_index()` (a plain `REINDEX`, `TRUNCATE`, `VACUUM FULL`, `CLUSTER`, a table-rewriting `ALTER TABLE` or a non-concurrent `REFRESH MATERIALIZED VIEW`) produced no entries, because the table was empty, held only rows dead to every transaction, or had no row that satisfied a partial index's predicate; its catalog `relpages` is `0` then, so the new test never changes an index's branch |
| `d168b666823` bottom-up index deletion | 14 | Fewer pages under non-HOT `UPDATE` churn; measured 2.2x on a 1,000-value column and nothing on a 100-value one |
| `e5d8a99903` 64-bit XIDs in deleted pages | 14 | Deleted pages can no longer become unrecyclable; fork still not shortened |
| `9f3665fbfc` inserts no longer trigger cleanup | 14; its message says a version went to 13, where the setting was only disabled; not in 12 | Cleanup-only index scans no longer driven by inserted tuples, and a cleanup-only B-tree VACUUM no longer rewrites the index's `relpages` and `reltuples`, which only a partial index's estimate reads |
| `9dd963ae253` recycle newly deleted pages | 14 | Deleted pages reusable sooner; fork still not shortened |
| `5100010ee4d` 2% index-vacuum bypass | 14 | New automatic way for dead index entries to stay, while index cleanup still runs ([vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)) |
| `1e55e7d1755` wraparound failsafe | 14 | Skips index vacuuming and cleanup under wraparound pressure ([vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326)) |
| `3499df0dee8` `INDEX_CLEANUP` auto | 14 | Lets an operator force index vacuuming: switches off the 2% bypass, not the failsafe |
| `eb5c4e953bb` `DEFAULT_PAGE_CPU_MULTIPLIER` | 16 | Cosmetic; value stays 50.0 |
| `cd9479af2af` GIN page CPU charges | 16 | Changes GIN bloat pricing, not B-tree |
| `3c569049b7b` partitioned-index zeroing | 16 | Lists partitioned indexes with zero size; v12 skipped them, so pricing is unchanged |
| `9c6ad5eaa9` endpoint-probe page limit | 16; back-patched, in 12.14 and later but not in the wiki's 12.2 pin | Not a cost input: dead entries at the end of a B-tree can leave a range estimate on the stale histogram bound for several plans; measured `rows=10` for four plans, then `rows=1` |
| `19d8e2308b` summarizing-only updates stay HOT | 16 | An update of only BRIN-indexed columns adds no B-tree entry while the new version fits on its page; in v12 it added one to every index |
| `5bf748b86bc` SAOP descent clamp | 17 | **New** `index->pages` input for `= ANY`: for a constant list in a boundary qual, a cap that discounts small or dense indexes relative to v12. It also charges an `= ANY` outside the boundary quals one descent instead of v12's one per element |
| `9391f71523b` `estimate_array_length()` statistics | 17 | Sizes a non-constant array from its element statistics instead of v12's fixed 10, which can raise or lower the descent count before the clamp |
| `71b66171d0` no index statistics during binary upgrade | 17 | An index created by `pg_upgrade` keeps `relpages = 0` and `reltuples = 0` until ANALYZE, or a VACUUM whose index pass reports an exact count, writes its row (a cleanup-only B-tree pass reports an estimate and writes nothing); only a partial index's estimate reads them |
| `index_pages_fetched()`, `cost_index()` | unchanged | Byte-identical to `REL_12_0` |
| `numIndexPages` prorating, height charge | unchanged | Same formulas and same `50.0` constant as v12 |

### Settings That Move The Boundary

Every setting below is `PGC_USERSET`, so each takes effect at session or transaction scope with no reload or restart. None of them is a bloat control; they change how heavily the existing page count is weighted.

| Setting | v17 default | Role in bloat pricing | Apply scope |
|---|---|---|---|
| `random_page_cost` | `DEFAULT_RANDOM_PAGE_COST`, 4.0 ([cost.h:25](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25)) | Multiplies each index page the estimate counts as fetched: the pro-rata `numIndexPages` on a single scan, or the Mackert-Lohman result on a repeated one ([selfuncs.c#genericcostestimate-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6786)). B-tree upper levels are charged CPU only ([selfuncs.c#btcostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7106)), and BRIN charges its range-map pages at `seq_page_cost` ([selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258)). A tablespace setting replaces it for every index stored there ([spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196)) | session/transaction ([guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)) |
| `cpu_operator_cost` | `DEFAULT_CPU_OPERATOR_COST`, 0.0025 ([cost.h:28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L28)) | Scales the B-tree `log2(N)` comparison charge and the per-level height charge `(tree_height + 1) * 50 * cpu_operator_cost`, which is the one the source names as its guard against bloat ([selfuncs.c#btcostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7106)) | session/transaction ([guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729)) |
| `effective_cache_size` | `DEFAULT_EFFECTIVE_CACHE_SIZE`, 524288 pages, 4GB at 8kB blocks ([cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34)) | Sets the cache the Mackert-Lohman model shares out: `index_pages_fetched()` gives the relation being priced `effective_cache_size * T / (total_table_pages + index_pages)`, so a larger index takes a larger share for its own repeated scans and leaves a smaller one for the heap ([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)) | session/transaction ([guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)) |
| `min_parallel_index_scan_size` | `(512 * 1024) / BLCKSZ`, 64 blocks or 512kB at 8kB blocks ([guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540)) | The smallest touched-page estimate at which an index path can go parallel on a plain base relation, and the base of the powers-of-three worker ramp; both read the scan's `numIndexPages`, not `index->pages`. The index threshold is necessary, not sufficient: a plain index scan must also have a heap-page estimate of at least `min_parallel_table_scan_size`, and it gets the smaller of the heap and index worker counts; only an index-only scan skips the heap test ([costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)). A table's `parallel_workers` reloption bypasses all of it ([allpaths.c#compute_parallel_worker-reloption](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4208-L4213), [allpaths.c#compute_parallel_worker-threshold](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227), [allpaths.c#compute_parallel_worker-index-ramp](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4253-L4272)) | session/transaction ([guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540)) |

One override is not a setting at all: a tablespace's own `random_page_cost`, set with `ALTER TABLESPACE … SET`, replaces the GUC for every index stored in it, because `genericcostestimate()` looks the page cost up by the index's tablespace ([selfuncs.c#genericcostestimate-tablespace](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6734-L6737), [spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196)).

Two per-index storage parameters change the physical layout rather than its price, and both take `ShareUpdateExclusiveLock` with the in-tree reason "since it applies only to later inserts": `fillfactor` ([reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)) and `deduplicate_items` ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)).

Neither is inert on the pages that already exist, so "no effect until a rebuild" is too strong. Both are read from the relation at the moment a leaf page is about to split:

- `_bt_findsplitloc()` reads `leaffillfactor = BTGetFillFactor(rel)` on every split ([nbtsplitloc.c#leaffillfactor](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L170-L176)). A rightmost leaf split targets that fillfactor. A split that the "split after new item" optimization recognizes as a localized ascending insertion either targets it too or splits exactly after the new item. Every other leaf split starts from a 50:50 target ([nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334)). `_bt_strategy()` can then override any other leaf split whose default split interval has no split point that avoids adding a heap TID to the new high key, which happens on a page crowded with duplicates; an exact split after the new item returns before this step ([nbtsplitloc.c:317](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L317), [nbtsplitloc.c:364](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L364), [nbtsplitloc.c#split-strategies](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L364-L416), [nbtsplitloc.c#_bt_strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L985-L1041)). A page that is not entirely one value splits beside its group of duplicates (`SPLIT_MANY_DUPLICATES`). A page that is entirely one value and is the last page holding it, meaning the rightmost leaf or a page whose high key differs from the new item, splits at `BTREE_SINGLEVAL_FILLFACTOR`, 96% ([nbtree.h:202](../../../../raw/postgres-17/src/include/access/nbtree.h#L202)), whatever the index fillfactor. A page entirely of one value that is not the last page of that value keeps the default split. A changed `fillfactor` therefore governs the next rightmost or split-after-new-item leaf split, including one of a page built under the old value, unless a duplicate strategy overrides it.
- `_bt_delete_or_dedup_one_page()` runs `_bt_dedup_pass()` over the existing leaf page before splitting it ([nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782)), gated on `BTGetDeduplicateItems(rel)` and on `allequalimage`, a flag fixed into the metapage when the index is built and not affected by the reloption ([nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564)). The pass runs only after three earlier exits have not fired: [simple deletion](../../../glossary.md#simple-index-deletion) of `LP_DEAD` entries frees room for the new item; the caller asked for simple deletion only, or is a unique-index insert that found no duplicate; or a bottom-up deletion pass frees enough space ([nbtinsert.c#early-returns](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2721-L2776)). `allequalimage` is false for any index with `INCLUDE` columns, and otherwise depends on each key column's opclass and collation ([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)); the v17 source also relies on it being zero on an index pg_upgraded from PostgreSQL 12 ([nbtpage.c#_bt_metaversion-comment](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L720-L737)). Turning `deduplicate_items` back on therefore compacts existing leaves in place as inserts reach them, for an `allequalimage` index and only on inserts where none of those three exits fired. Only the first and third exits avoid the split. The second returns before the deduplication pass without having made room for the new item, so its page can still split undeduplicated.

What a rebuild adds is reach and immediacy: `REINDEX` applies the new layout to every page at once instead of page by page as traffic happens to touch them, which is one of the scenarios it documents ([ref/reindex.sgml#storage-parameter](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L66-L71)). A rebuild applies the index's reloptions as they stand, not a healthy default: the build fills each leaf page to the index fillfactor ([nbtsort.c#_bt_pagestate-leaf-fill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665)), and it deduplicates only when `deduplicate_items` is on, the index is `allequalimage`, and the index is not unique ([nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)). The `l3_c` bloating step in [Bloat changes plans](#bloat-changes-plans) uses `ALTER INDEX … SET (fillfactor = 10)` followed by `REINDEX` for exactly that reason. Fixture N instead builds its two variants directly, one of them with `CREATE INDEX … WITH (deduplicate_items = off)`.

### Practical Interpretation

- **Read bloat's planner cost from blocks, not from `avg_leaf_density`, and read neither as a rebuild verdict.** The planner is charged for blocks, and fixture M shows a 10x-oversized index reporting 89.18% density. Blocks per row does not predict what `REINDEX` returns, because a rebuild reapplies the index's reloptions (see [Settings That Move The Boundary](#settings-that-move-the-boundary)). Fixture N's `n_off` holds 2,749 blocks against `n_on`'s 852 over the same 1,000,000 rows, and fixture A's `a_sparse_idx` 26,411 against `a_dense_idx`'s 2,745. A rebuild of either larger index would come back at about the same size, because the build skips deduplication while `deduplicate_items` is off ([nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)) and fills leaves only to the fillfactor ([nbtsort.c#_bt_pagestate-leaf-fill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665)). That is a source reading: this page's script rebuilds neither index. A partial index's population is not the table's row count either, since `ANALYZE` counts only the sampled rows that pass the predicate ([analyze.c#partial-index-population](../../../../raw/postgres-17/src/backend/commands/analyze.c#L902-L908), [analyze.c#tupleFract](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)). This page's advice is about planner pricing. It has not been scored against [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md), the wiki's acceptance suite for rebuild decisions.
- **Do not expect a bloated index to be abandoned by OLTP queries.** While the index has no more pages than the table has rows, a *single* point lookup that touches one leaf page is charged only the height difference: `0.125` per level at default settings, about 1.5% of fixture H's heap-fetching lookup and 2.8% of fixture A's index-only one. The height charge still sees bloat: fixture B's lookup drops from `4.44` to `4.31` when a rebuild removes a level, and fixture H's two lookups cost `8.31` and `8.43`. Only the page charge is blind.
- **Two independent things end that blindness, and both run through the index's page count.** The first needs pages to outnumber rows; the second only needs the lookup to repeat. An index with more pages than its table has rows — a drained queue table is the usual way to get one — charges a one-row lookup `ceil(pages / tuples)` random pages ([selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)); more generally, a lookup of `k` estimated rows is charged one page only while `k * pages <= tuples`. Fixture F measures `12.29` for its 2,745-block index over 1,000 rows, against `4.29` after a rebuild. A *repeated* lookup, on the inner side of a nested loop, is priced through the cache model with the whole index as the page universe, even while the per-iteration estimate stays at one page ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)). How much bloat moves that price ranges from not at all, for a few loops over a large index, to linearly in the index's pages. The linear end arrives, while the index fits its share of `effective_cache_size`, once loops times touched pages reach twice the index size: the model then caps the fetches at the whole index, spread over the loops ([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)). Fixture A measures `2.50` per loop for the 26,411-block `a_sparse_idx` against `0.66` for the 2,745-block `a_dense_idx`, over 50,000 loops. Only `a_dense_idx` is at that cap: 50,000 ≥ 2 × 2,745, so the model charges all 2,745 pages, `2745 * 4.0 / 50000 = 0.22` per loop. `a_sparse_idx` is just short of it, 50,000 < 2 × 26,411 = 52,822, so the model charges `ceil(2 * 26411 * 50000 / (2 * 26411 + 50000)) = 25,687` of its 26,411 pages, `25687 * 4.0 / 50000 = 2.05` per loop; capped, it would pay `2.11`. Both per-loop totals add the same `0.44` of descent and tuple CPU. A rebuild changes what OLTP lookups cost in both shapes; the join case is the one people miss, because the lookup it repeats looks free when priced alone.
- **`ceil(pages / tuples)` is not the whole rule either.** It runs only while `index->pages > 1` *and* `index->tuples > 1`; otherwise the estimate is a flat one page whatever the index's size ([selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)). A non-partial index inherits the table's row estimate, so a table down to one row prices a scan of a large index at a single page, measured in [The pages-outnumber-rows guard at its limit](#the-pages-outnumber-rows-guard-at-its-limit) on 551 blocks. Do not read a cheap plan on a nearly empty table as evidence that its index is healthy.
- **Do not read the planner's height from `pgstatindex`.** `tree_level` is the true root level; the planner charges the fast-root level, which VACUUM's page deletion can lower without a rebuild. `bt_metap()` shows both. Even `bt_metap()` only shows what is on disk now. A backend can be costing from its own cached copy of an older metapage until a relcache invalidation reaches it ([relcache.c#RelationCacheInvalidateEntry](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2975-L2985), [relcache.c#RelationFlushRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2916-L2924), [relcache.c#RelationClearRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2600-L2603)) or a scan in that backend trips `_bt_getroot()`'s stale-fast-root check ([nbtpage.c#_bt_getroot-stale-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L380-L403)). The `pg_class` write that VACUUM or ANALYZE makes for the index sends that invalidation when its page or tuple count changed ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [vacuum.c#vac_update_relstats-dirty](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1446-L1461), [heapam.c:6668](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6668)). A root split alone sends none. It writes index pages only, among them a new root page carrying the new level ([nbtinsert.c#_bt_newlevel-root-level](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2507-L2513)), and the metapage's root and fast-root fields ([nbtinsert.c#_bt_newlevel-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2515-L2519)). The index's `pg_class` row is not touched. So between a root split and the next such write, a backend that has only planned against the index can price it at a different height from one that has scanned it.
- **Expect plan changes on the analytical side.** Fixture A flipped to a sequential scan at 25% selectivity, and fixture L3's bloated index was dropped from a `BitmapAnd` entirely.
- **Treat `leaf_fragmentation` as a partial runtime signal only.** The manual ties physical adjacency to I/O; the metric sees only backward sibling links, and the cost model reads neither.
- **On v17, expect `= ANY (...)` over a constant list to get cheaper on a rebuilt B-tree, not dearer on a bloated one.** For a `Const` array or an `ARRAY[...]` list, `estimate_array_length()` returns the literal element count ([selfuncs.c#estimate_array_length](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2131-L2207)), and the clamp `Min(num_sa_scans, ceil(index->pages * 0.3333333))` can only lower that count ([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042)). A bloated index, whose cap is higher, keeps more of it. Two other v17 rules move the count whatever the page count. Only arrays in the boundary quals add descents ([selfuncs.c#btcostestimate-boundary-saop](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6950-L6961)). A non-constant array expression with statistics is counted as the average number of distinct elements that its distinct-element-count histogram (`STATISTIC_KIND_DECHIST`) records, and only one without such statistics gets the flat default of 10 ([selfuncs.c#estimate_array_length-statistics](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2162-L2206)); that average can exceed 10, and a bloated index's higher cap lets it keep the larger count. [v17: index pages now cap ScalarArrayOp descents](#v17-index-pages-now-cap-scalararrayop-descents) sets each rule against v12.
- **A bloated partial index is priced from its recorded density, not its live size.** `get_relation_info()` sets a partial index's `tuples` to the density recorded in its `pg_class` row times its live block count less the metapage, capped at the table's row estimate ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)). `btcostestimate()` takes the rows a scan reads from selectivity times the *table's* row estimate ([selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019)), and the table's estimate is its own recorded density times its live block count ([tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747)). The charged page count is therefore about those rows divided by the index's recorded density. Pages the index gains after its `pg_class` row was last written reach the charge only in proportion to the table's growth, until the scaled `tuples` reaches the table's row estimate and the cap takes over. They arrive all at once when that row is rewritten: `ANALYZE` rewrites every index's row ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)), a `VACUUM` with index cleanup on rewrites the rows whose counts it measured exactly ([vacuumlazy.c:512-513](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L512-L513), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)), and a build or rebuild writes its own counts ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842)). Fixture Q measures it. Ten rounds of non-HOT updates in one transaction grew the partial `q_part` from 57 to 331 blocks, and its charge rose only from 57 to 113 pages, because the heap doubled; after `ANALYZE` it was charged all 331. The plain `q_full` on the same table was charged its 825 live blocks at once. See [A partial index hides its growth until its statistics are rewritten](#a-partial-index-hides-its-growth-until-its-statistics-are-rewritten).
- **A range predicate near either end of a B-tree can make planning read dead entries.** This is not a cost term; it moves a row estimate and costs planning time. When the histogram search in `ineq_histogram_selectivity()` reaches the first or last bound, it replaces that bound with the column's current minimum or maximum from `get_actual_variable_range()` ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)). That function reads the first suitable index: a B-tree, not partial, not hypothetical, whose first column is the compared column with a matching collation and sort order ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)). `get_actual_variable_endpoint()` walks in from that end of the index under `SnapshotNonVacuumable`, which still counts rows deleted so recently that some snapshot may see them ([selfuncs.c#get_actual_variable_endpoint-snapshot](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6376-L6386)). It gives up after 100 heap pages without a visible row, and the estimate then keeps the histogram's recorded bound ([selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413), [selfuncs.c#get_actual_variable_endpoint-limit](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6436-L6457)). The walk heals itself: the index entries it steps past are marked dead, so the next plan starts further in ([selfuncs.c#get_actual_variable_endpoint-killed](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6388-L6397), [indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652)). Fixture E measures both effects. After the top half of a 200,000-row table was deleted, four plans in a row estimated `id > 199990` at 10 rows from the stale bound. Each marked 100 heap pages' worth of entries dead: 22,590 on the first plan, whose first heap page was the table's partly filled last one, and 22,600 on each of the next three. The fifth plan reached a live entry and estimated 1 row. See [5. Planning-time endpoint probes read the ends of a B-tree](#5-planning-time-endpoint-probes-read-the-ends-of-a-b-tree) and [Dead entries at the end of a B-tree: the endpoint probe](#dead-entries-at-the-end-of-a-b-tree-the-endpoint-probe).

### Key Data Structures

| Structure | Field | Role |
|---|---|---|
| `IndexOptInfo` | `pages`, `tuples`, `tree_height` | The size inputs `get_relation_info()` fills for every AM ([pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)). `genericcostestimate()` reads `pages` and `tuples`, plus the table's `rel->tuples` and the index's tablespace ([selfuncs.c:6695](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6695), [selfuncs.c#genericcostestimate-tablespace](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6734-L6737)); `tree_height` is read by `btcostestimate()`, and `gistcostestimate()` and `spgcostestimate()` fill it themselves. Not the complete set for every AM: `gincostestimate()` and `brincostestimate()` each reopen the index and read their own metapage counters on top ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [selfuncs.c#brincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8086-L8101)) |
| [`RelOptInfo`](../../../glossary.md#reloptinfo) | `pages`, `tuples`, `allvisfrac` | Parent-table estimates ([pathnodes.h#RelOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L941-L944)), filled by `estimate_rel_size()` ([plancat.c:201](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L201)); `tuples` is copied into a non-partial index's `tuples` ([plancat.c:476](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L476)) and caps a partial index's ([plancat.c:484](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L484)) |
| `PlannerInfo` | `total_table_pages` | Table-only page total used to prorate `effective_cache_size` ([pathnodes.h#total_table_pages](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L483-L484)) |
| `GenericCosts` | `numIndexPages`, `numIndexTuples`, `num_sa_scans` | Shared cost scratchpad; `numIndexPages` is what every caller returns as `*indexPages`, and `num_sa_scans` became an input in v17 ([selfuncs.h#GenericCosts](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L108-L138)) |
| `BTMetaPageData` | `btm_level`, `btm_fastlevel` | True root level versus the fast-root level the planner uses ([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)) |
| `AttStatsSlot` | `values`, `nvalues` | The histogram bounds `ineq_histogram_selectivity()` searches ([lsyscache.h#AttStatsSlot](../../../../raw/postgres-17/src/include/utils/lsyscache.h#L46-L62)). It overwrites the first or last bound in its copy with the value the endpoint probe reads, and keeps `pg_statistic`'s bound when the probe fails ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)) |
| `LVRelState` | `consider_bypass_optimization`, `do_index_vacuuming`, `do_index_cleanup` | `consider_bypass_optimization` gates the 2% bypass ([vacuumlazy.c:1900](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1900)). `do_index_vacuuming` and `do_index_cleanup` are two separate switches: the bypass clears only `do_index_vacuuming`, so `amvacuumcleanup` still runs, and the wraparound failsafe clears both ([vacuumlazy.c#LVRelState](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L148-L156), [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949), [vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326)) |
| `pg_class` | `relpages`, `reltuples`, `relallvisible` | The only physical-size catalog columns; no density or fragmentation column exists ([pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)) |

### Caller And Callee Boundary

```text
query_planner                                        planmain.c
  ├─ add_base_rels_to_query -> build_simple_rel      initsplan.c, relnode.c
  │    └─ get_relation_info                          plancat.c
  │         ├─ RelationGetNumberOfBlocks             -> smgrnblocks (non-partial index)
  │         ├─ estimate_rel_size                     -> pg_class density (partial index)
  │         ├─ _bt_getrootheight                     -> btm_fastlevel (B-tree only)
  │         └─ get_relation_info_hook                a plugin may rewrite pages, tuples, tree_height
  └─ make_one_rel                                    allpaths.c
       ├─ set_base_rel_sizes -> set_rel_size -> set_plain_rel_size
       │    └─ set_baserel_size_estimates -> clauselist_selectivity    (row estimate; endpoint probe below)
       ├─ root->total_table_pages is summed
       └─ set_base_rel_pathlists -> set_rel_pathlist
            ├─ set_plain_rel_pathlist -> create_index_paths              indxpath.c
            │    ├─ get_index_paths -> build_index_paths -> create_index_path -> cost_index
            │    │    ├─ amcostestimate  ==  btcostestimate / hash / gist / spgist / gin / brin
            │    │    │    ├─ clauselist_selectivity  (btcostestimate: boundary quals)
            │    │    │    ├─ genericcostestimate    (bt, hash, gist, spgist, contrib bloom)
            │    │    │    │    ├─ clauselist_selectivity  (all index quals)
            │    │    │    │    └─ index_pages_fetched     (repeated scans, T = index->pages)
            │    │    │    └─ index_pages_fetched    (gincostestimate: entry and data pages scaled from index->pages)
            │    │    ├─ index_pages_fetched         (heap fetches, 3 call sites, index->pages)
            │    │    └─ compute_parallel_worker     allpaths.c (numIndexPages, not index->pages)
            │    ├─ add_path -> compare_path_costs_fuzzily         pathnode.c
            │    └─ choose_bitmap_and                              indxpath.c
            │         └─ bitmap_scan_cost_est -> cost_bitmap_heap_scan -> compute_bitmap_pages
            │              └─ index_pages_fetched    (repeated bitmap scans, get_indexpath_pages)
            └─ set_rel_pathlist_hook                 a plugin may delete or modify costed paths

clauselist_selectivity -> clauselist_selectivity_ext -> clause_selectivity_ext   clausesel.c
  └─ restriction_selectivity -> the operator's oprrest, e.g. scalargtsel        plancat.c
       └─ scalarineqsel_wrapper -> scalarineqsel -> ineq_histogram_selectivity  selfuncs.c
            └─ get_actual_variable_range         first or last histogram bound; plain B-tree only
                 └─ get_actual_variable_endpoint index-only walk, SnapshotNonVacuumable, 100 heap pages
```

Call sites, in tree order: [planmain.c:170](../../../../raw/postgres-17/src/backend/optimizer/plan/planmain.c#L170), [initsplan.c:165](../../../../raw/postgres-17/src/backend/optimizer/plan/initsplan.c#L165), [relnode.c:340](../../../../raw/postgres-17/src/backend/optimizer/util/relnode.c#L340), [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486), [plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500), [plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576), [planmain.c:280](../../../../raw/postgres-17/src/backend/optimizer/plan/planmain.c#L280), [allpaths.c#total_table_pages](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216), [allpaths.c:322](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L322), [allpaths.c:411](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L411), [allpaths.c:581](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L581), [costsize.c:5264](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L5264), [allpaths.c:221](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L221), [allpaths.c:351](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L351), [allpaths.c:499](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L499), [allpaths.c:783](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L783), [indxpath.c:279](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L279), [indxpath.c:722](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L722), [indxpath.c:963](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L963), [pathnode.c:1024](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L1024), [costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621), [selfuncs.c:7015](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7015), [selfuncs.c:7073](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7073), [selfuncs.c:6682](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6682), [selfuncs.c:6767](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6767), [selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974), [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c:765](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L765), [indxpath.c#choose_bitmap_and-call](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L340-L343), [pathnode.c:452](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L452), [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526-L1553), [costsize.c:1044](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044), [costsize.c#compute_bitmap_pages-repeated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476), [allpaths.c#set_rel_pathlist_hook](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L532-L539).

Endpoint-probe chain: [clausesel.c#clauselist_selectivity](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L100-L108), [clausesel.c:136](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L136), [clausesel.c:183](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L183), [clausesel.c:848](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L848), [plancat.c#restriction_selectivity](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1951-L1979), [pg_operator.dat#int4gt](../../../../raw/postgres-17/src/include/catalog/pg_operator.dat#L453-L456), [selfuncs.c#scalargtsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1490-L1494), [selfuncs.c:1461](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1461), [selfuncs.c:690](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L690), [selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6157-L6331), [selfuncs.c#get_actual_variable_endpoint](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6349-L6501). The same probe is reached from `prefix_selectivity()` for `LIKE` prefixes ([like_support.c:1245](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1245), [like_support.c:1266](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1266)) and from `mergejoinscansel()` through `scalarineqsel()` ([selfuncs.c#mergejoinscansel-scalarineqsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3164-L3203)). `get_variable_range()`'s own call to it is compiled out with `#ifdef NOT_USED` ([selfuncs.c#get_variable_range-not-used](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5993-L6003)).

Definitions: [costsize.c#cost_index](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L821), [selfuncs.c#genericcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6626-L6828), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951), [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279), [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L163-L212).

`CLUSTER` reuses the same costing outside query planning. For a B-tree clustering index, `CLUSTER` asks `plan_cluster_use_sort()` whether to scan the index or to sort a sequential scan ([cluster.c#plan_cluster_use_sort-call](../../../../raw/postgres-17/src/backend/commands/cluster.c#L948-L951)). That function builds a minimal planner state, costs a whole-index `create_index_path()`, and picks the sort when it is cheaper ([planner.c#plan_cluster_use_sort](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6761-L6874), [planner.c#plan_cluster_use_sort-compare](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6861-L6873)). A whole-index scan is charged every page. With no quals, `btcostestimate()` estimates the table's full row count ([selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019)). For a non-partial index, the only kind `CLUSTER` accepts ([cluster.c#check_index_is_clusterable-partial](../../../../raw/postgres-17/src/backend/commands/cluster.c#L525-L534)), `genericcostestimate()` turns that count into all `index->pages`, each at the random-page cost, unless the table's row estimate is one or less ([selfuncs.c#genericcostestimate-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6786)). A bloated clustering index therefore raises the index side of that comparison and pushes `CLUSTER` toward the sort. It does not decide the choice alone, because the index side also carries heap fetches priced by [correlation](../../../glossary.md#correlation) ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)). Two conditions force the sort regardless of bloat: `enable_indexscan = off`, and an index missing from the planner's index list, for example one not yet past its `indcheckxmin` horizon ([planner.c#plan_cluster_use_sort-short-circuits](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6788-L6841)).

On the write side, nbtree drives how many pages exist, but it leans on two services outside it. It asks the index free space map for a recyclable page before extending the fork ([nbtpage.c:903](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L903), [nbtpage.c:978](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L978)), and it asks the table AM which entries simple and bottom-up deletion may remove ([nbtpage.c:1526](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1526)). Inside nbtree, [nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334) chooses where a page splits, [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782) tries bottom-up deletion and then deduplication before splitting, and [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659) marks pages deleted and can lower the fast-root level.

### Build, Generated-Header, And Extension Boundary

- The cost path compiles against two generated catalog headers. `get_relation_info()` tests `RELKIND_PARTITIONED_INDEX` and `BTREE_AM_OID` ([plancat.c:471](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471), [plancat.c:488](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488)), and the endpoint probe tests the same `BTREE_AM_OID` to pick its index ([selfuncs.c:6197](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6197)). Both symbols reach that code only through headers the build generates. The `RELKIND_*` letters sit in `pg_class.h`'s `EXPOSE_TO_CLIENT_CODE` block ([pg_class.h#relkinds](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L162-L173)), which `Catalog.pm` collects ([Catalog.pm#client_code](../../../../raw/postgres-17/src/backend/catalog/Catalog.pm#L61-L66), [Catalog.pm#client_code-push](../../../../raw/postgres-17/src/backend/catalog/Catalog.pm#L197-L205)) and `genbki.pl` copies into the generated `pg_class_d.h` ([genbki.pl#client_code](../../../../raw/postgres-17/src/backend/catalog/genbki.pl#L563-L567)). `BTREE_AM_OID` is an `oid_symbol` in `pg_am.dat` ([pg_am.dat:18](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18)) that `genbki.pl` turns into a `#define` in `pg_am_d.h` ([genbki.pl#oid_symbol](../../../../raw/postgres-17/src/backend/catalog/genbki.pl#L675-L687)). Renumbering either is a catalog change that needs a rebuild and an `initdb`. No generated parser artifact is involved. `pg_class.relpages` / `reltuples` / `relallvisible` come from the hand-written catalog header `src/include/catalog/pg_class.h`, which documents `-1` as "unknown" ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). Its `BKI_DEFAULT(-1)` on `reltuples` applies only to the rows describing bootstrap catalogs, as the header's own note says ([pg_class.h#CATALOG](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L32)); a table created later gets its `-1` from `AddNewRelationTuple()` ([heap.c#AddNewRelationTuple-empty](../../../../raw/postgres-17/src/backend/catalog/heap.c#L1006-L1009)), while a new index's row starts at `relpages = 0` and `reltuples = 0` ([index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024), [relcache.c#RelationBuildLocalRelation-rd_rel](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3656-L3659)). The `FormData_pg_class` struct that `estimate_rel_size()` reads is not generated by `genbki.pl`: it is the `CATALOG()` macro in that same hand-written header expanding through cpp, `#define CATALOG(name,oid,oidmacro) typedef struct CppConcat(FormData_,name)` ([genbki.h:23](../../../../raw/postgres-17/src/include/catalog/genbki.h#L23)), which the header's own comment states — "cpp turns this into typedef struct FormData_pg_class" ([pg_class.h#CATALOG](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L32)). What `genbki.pl` does with the header, by way of `Catalog.pm` reading it ([pg_class.h#NOTES](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L11-L16)), is emit the derived `pg_class_d.h` and the bootstrap [`.bki`](../../../glossary.md#bki) data that `initdb` consumes. So changing one of these columns' declarations is an initdb-visible catalog change and not only a recompile, but the struct the planner reads is plain cpp output.
- `DEFAULT_PAGE_CPU_MULTIPLIER` is a private `#define` inside `selfuncs.c` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), not an exported symbol and not a GUC, so the multiplier itself is fixed at compile time. What it multiplies is not: an extension can change `cpu_operator_cost`, and it can rewrite `tree_height` through `get_relation_info_hook`, below. `VISITED_PAGES_LIMIT`, the endpoint probe's 100-page limit, is likewise a local `#define` inside `get_actual_variable_endpoint()` ([selfuncs.c:6447](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447)), with no GUC to change it.
- `pathnodes.h` deliberately types `IndexOptInfo.amcostestimate` weakly to avoid including `amapi.h`, so `cost_index()` casts it before calling ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). A custom index AM therefore participates in bloat pricing mainly through its own `amcostestimate`, and gets `tree_height = -1` unless it is a B-tree; `cost_index()` still feeds its `index->pages` into the heap-side cache model, as for every AM ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)).
- Four [hooks](../../../glossary.md#hook) sit on or beside this path. `get_relation_info_hook` runs at the end of `get_relation_info()`, after every `IndexOptInfo` is filled, so that a plugin can "editorialize on the info we obtained from the catalogs. Actions might include altering the assumed relation size, removing an index, or adding a hypothetical index to the indexlist" ([plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576), [plancat.h#get_relation_info_hook_type](../../../../raw/postgres-17/src/include/optimizer/plancat.h#L20-L25)). A plugin there can substitute `pages`, `tuples`, `tree_height` or the `amcostestimate` pointer itself before any path is costed, and the cost code expects it: `gincostestimate()` skips the metapage read for an index marked `hypothetical` ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711)), and the endpoint probe skips a hypothetical index rather than reading it ([selfuncs.c#get_actual_variable_range-skips](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6196-L6212)). `get_relation_stats_hook` and `get_index_stats_hook` supply statistics wherever the planner looks them up. `examine_variable()` and `examine_simple_variable()` consult them for every selectivity estimate, which sets `numIndexTuples` and through it `numIndexPages` ([selfuncs.c:5203-5204](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5203-L5204), [selfuncs.c:5375-5376](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5375-L5376)); `btcostestimate()` consults them for correlation ([selfuncs.c#btcostestimate-stats-hooks](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7116-L7171)), and so does `brincostestimate()` ([selfuncs.c:8135-8136](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8135-L8136), [selfuncs.c:8166-8167](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8166-L8167)). They cannot substitute a page count or a tree height, but they can move the page charge through selectivity. `set_rel_pathlist_hook` runs after the core code has built and costed a relation's paths, and a plugin there "could also delete or modify paths added by the core code" ([allpaths.c#set_rel_pathlist_hook](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L532-L539)).
- Every density and fragmentation metric on this page comes from `contrib`, not core: `pgstatindex` in `pgstattuple` and `bt_metap()` in `pageinspect`. Core SQL exposes physical size (`pg_relation_size`) and the catalog columns, and nothing else about index page structure.
- Within the 17.x series, `036decbba2a` "pgstattuple: Improve reports generated for indexes (hash, gist, btree)" (first contained in the `Stamp 17.7.` commit; its message says `Backpatch-through: 13`) added a `BTPageOpaqueData` size check to `pgstattuple`'s B-tree page handling ([pgstattuple.c#pgstat_btree_page-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L419-L425)); its rule that an all-zero page counts as free space was new for GiST and hash and, in the commit message's words, "already applied to btree". The pin contains it. It touches only `pgstattuple.c`, so it changes `pgstattuple()` output, not `pgstatindex()` density or fragmentation.

### Tests And Explicit Test Absence

- **No test asserts the bloat charge.** Every B-tree plan in the regression suites runs the height charge, but `src/test` contains no reference to `tree_height`, `btcostestimate` or `genericcostestimate`, and no regression test asserts that a bloated index is costed higher than an unbloated one.
- **No test asserts the endpoint probe or its limit.** Nothing under `src/test`, `contrib` or `doc` names `get_actual_variable_range`, `get_actual_variable_endpoint`, `VISITED_PAGES_LIMIT` or `SnapshotNonVacuumable`. None of the three commits that bounded how long the probe runs added a test. `fccebe421` and `3ca930fc3` changed the snapshot it reads under, `9c6ad5eaa9` added the 100-page limit, and each touched only source and header files. A regression-test plan reaches the probe only when all of these hold: it estimates an inequality against a constant on a column with a histogram; the column leads a B-tree that is not partial or hypothetical and matches the comparison's collation and sort operator; and the histogram search reaches the first or last bound, or the histogram has only two bounds ([selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136), [selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)). No test asserts the value it returns, its fallback after 100 heap pages, or the entries it marks dead; this page's fixture E measures them, and no in-tree test does.
- **Every successful in-tree `pgstatindex` call runs against an empty index**, so none asserts anything about density or fragmentation. The test creates `test (a int primary key, b int[])` with no rows and expects `avg_leaf_density` and `leaf_fragmentation` to be `NaN` through four spellings of the call ([sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)), and a fifth call on the empty index of an empty partition expects `(4,0,8192,0,0,0,0,0,NaN,NaN)` ([expected/pgstattuple.out#partition-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L264-L268)). The remaining calls are error paths: GIN, hash, a partitioned table, a view, a foreign table, a partition and a sequence are each rejected as `not a btree index`. The `NaN` comes from the `max_avail > 0` and `leaf_pages > 0` guards ([pgstatindex.c#density-guards](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
- The nbtree features that limit bloat do have tests, but they test correctness, not size: deduplication and deletion are covered through `src/test/regress/sql/btree_index.sql` and [`contrib/amcheck`](../../../glossary.md#amcheck), neither of which asserts page counts. `btree_index.sql` does build a fast root by deleting most of an 80,000-row index and vacuuming, then splits it, but as a correctness and WAL-coverage test ([btree_index.sql#multilevel-page-deletion](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L252-L270)).
- All measurements on this page were therefore produced by this page's own script on an isolated exact-pin server, not by any in-tree test.

### Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead

#### Short answer

Once an index is valid, is usable by the planning transaction and, if partial, has a predicate the query implies, PostgreSQL 17 discards a GIN index at three separate gates, and only the third one is about cost. The checks that come before them are not GIN-specific; [Gate 0](#gate-0-the-index-is-not-usable-for-this-query) lists them. Gates 1 and 2 are absolute in the sense that no setting moves them: they are catalog and access-method properties. Gate 1 is absolute about one *clause*, not about the index, since a partial GIN index with a useful predicate still yields a clauseless path. Gate 3 is a comparison of computed costs, not a rule, and what it compares depends on which clauses each index serves:

- **Both indexes serve the identical clause set**, as for one predicate on one column. `choose_bitmap_and()` keeps only the cheaper-to-scan path of each such group ([indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1399)). Its measure is the index's own `amcostestimate` total plus `0.1 * cpu_operator_cost` per row for the bitmap ([costsize.c:628](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L628), [costsize.c#cost_bitmap_tree_node-indexpath](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1114-L1128)), and the row term is the same for both, because an unparameterized index path carries the relation's estimated output row count, `baserel->rows`, which is the same for every such path on the table ([costsize.c#cost_index-rows](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L589-L604)). That is not the `rows` figure `EXPLAIN` prints on the `Bitmap Index Scan` node, which is `clamp_row_est(indexselectivity * tuples)`, the index's own selectivity times the table's estimated tuple count ([createplan.c#create_bitmap_subplan-indexpath](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3480-L3485)). Here the GIN index drops out of the bitmap whenever its own estimate comes out higher; on a tie the path met first stays. Winning that comparison does not win the plan, because the B-tree's plain index scan paths still compete with the bitmap heap scan in `add_path()` ([indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751)).
- **The indexes serve different clauses.** Then the GIN index's own estimate does not decide. `choose_bitmap_and()` adds an index to an AND group only when the whole bitmap-heap-scan estimate drops, and returns the cheapest group ([indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489), [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526-L1553), [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571)). `add_path()` then compares whole paths on fuzzy cost, pathkeys, parameterization, rows and parallel safety ([pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L622)). In fixture D at 736 pending pages the GIN index estimate is `3141.12` against the B-tree's `168.80`, and the GIN index still stays in the chosen `BitmapAnd`, because that plan (`3321.93`) is cheaper than the B-tree-only plan (`3486.80`); see [A GIN index with a large pending list loses to a B-tree](#a-gin-index-with-a-large-pending-list-loses-to-a-b-tree).

For ordinary comparison predicates on the same column the GIN estimate does come out higher, and the reason is a model asymmetry rather than a size difference: `gincostestimate()` charges `random_page_cost` for every pending, entry and data page it expects to touch and adds a `50 * cpu_operator_cost` charge per page on top ([selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L8050)), while `btcostestimate()` charges the B-tree only a pro-rata share of `index->pages` through `genericcostestimate()` ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073)), plus a cheap CPU descent charge that it adds itself after `genericcostestimate()` returns ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091), [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)). That is the outcome on every comparison predicate measured below, and it is what [`contrib/btree_gin`](../../../glossary.md#btree_gin-and-btree_gist)'s own documentation says in general terms: "In general, these operator classes will not outperform the equivalent standard B-tree index methods" ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)). Nothing in the source makes it a guarantee, and [Where GIN still wins](#where-gin-still-wins) measures a case where the GIN path is the cheaper one.

| Gate | Where | What makes GIN lose | Recovery |
|---|---|---|---|
| 0. Index usability (not GIN-specific) | `get_relation_info()`, `create_index_paths()` | The index is not [`indisvalid`](../../../glossary.md#invalid-index); or its `indcheckxmin` flag is set and its `pg_index` row is not yet older than the transaction's `TransactionXmin`; or it is a partial index whose predicate the query does not imply ([plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281), [indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266)). See [Gate 0](#gate-0-the-index-is-not-usable-for-this-query) | Rebuild an invalid index; wait for older transactions to finish; write a `WHERE` clause that implies the predicate |
| 1. Clause matching | `match_clause_to_indexcol()` | The query operator is not in the GIN index's operator family and no [planner support function](../../../glossary.md#planner-support-function) rewrites it, so the clause yields no `IndexClause` and can never become an `Index Cond` on that index ([indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269), [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2386-L2503)). It rejects the clause, not necessarily the index: see below | Use a matching operator, or add the operators with `contrib/btree_gin` |
| 2. Plan shape | `build_index_paths()` / `get_index_paths()` | GIN has no `amgettuple`, no ordering, no `amcanreturn`, no null search, no native array search and no `amcanparallel`, so it cannot produce a plain `Index Scan`, satisfy `ORDER BY` pathkeys, feed an `Index Only Scan`, serve `IS NULL`, or contribute a *partial* index path of its own ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89), [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767)). It can still sit under a parallel plan; see [Gate 2](#gate-2-the-required-plan-shape-rules-gin-out) | None. These are AM properties, not costs |
| 3. Cost | `gincostestimate()` versus `btcostestimate()`, then `choose_bitmap_and()` / `add_path()` | The index path, or the plan that uses it, is estimated dearer than the alternative, by the two rules above. GIN's page charges are all at `random_page_cost` and include the whole pending list, as startup cost ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)) | Drain the pending list with `gin_clean_pending_list()`, `VACUUM`, or an `ANALYZE` run by an autovacuum worker. That `ANALYZE` flushes the list in its final index-cleanup step, after sampling, and only as far as the tail page recorded when that flush begins (`full_clean = false`); a manual `ANALYZE` leaves the list alone ([ginvacuum.c#ginvacuumcleanup-analyze-only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717), [analyze.c#do_analyze_rel-index-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L694-L721), [analyze.c:527](../../../../raw/postgres-17/src/backend/commands/analyze.c#L527), [ginfast.c:847](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L847), [ginfast.c#ginInsertCleanup-stop-at-tail](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L881-L888), [gin.sgml#fast-update-flush](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515)). Turning `fastupdate` off is not enough on its own: it stops future entries from joining the pending list but "does not in itself flush previous entries" ([ref/create_index.sgml#fastupdate-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L524-L532)), so it still needs one of those three afterward |

Measured at this pin on one table carrying both indexes over the same 300,000 rows, so the two `EXPLAIN` runs saw literally identical statistics: the same `n = 42` predicate cost **`12.97` through a `btree_gin` GIN index and `4.52` through a B-tree**, and the planner chose the B-tree. The GIN index was 279 blocks and the B-tree 280, so GIN lost while being the physically smaller index.

#### Gate 0: the index is not usable for this query

Three checks run before any clause is matched. None is GIN-specific, and each can drop a GIN index while an older B-tree on the same table stays usable:

- **Invalid index.** `get_relation_info()` leaves out every index whose `pg_index.indisvalid` is false, so the planner never builds an `IndexOptInfo` for it ([plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281)). A `CREATE INDEX CONCURRENTLY` that fails leaves such an index behind; the manual recommends dropping it and building it again, or `REINDEX INDEX CONCURRENTLY` ([ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667)).
- **Index not yet usable by this transaction.** `get_relation_info()` also leaves out an index whose `indcheckxmin` flag is set while the `xmin` of its `pg_index` row does not precede the transaction's `TransactionXmin`, and marks the plan transient ([plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281)). A cached transient plan records the `TransactionXmin` it was built under and is invalidated when a later use sees a different one, so it is planned again ([plancache.c#BuildCachedPlan-transient](../../../../raw/postgres-17/src/backend/utils/cache/plancache.c#L1022-L1033), [plancache.c#CheckCachedPlan-transient](../../../../raw/postgres-17/src/backend/utils/cache/plancache.c#L866-L873)). `index_build()` sets `indcheckxmin` only for a non-concurrent `CREATE INDEX` that met broken HOT chains, never for a concurrent build or a `REINDEX` ([index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3099-L3101)). `reindex_index()` handles `REINDEX` itself: it clears the flag when the rebuild met no broken HOT chains; when the rebuild did meet them, it leaves the flag as it was on a valid index and sets it on one that was invalid, not ready or dead ([index.c#reindex_index-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3800-L3850)). The `xmin` of the index's `pg_index` row records its usability horizon; for a new index it is the XID of the `CREATE INDEX` transaction. A transaction may use a flagged index only once that `xmin` is below its `TransactionXmin` horizon, and the creating transaction cannot use it while it holds old snapshots ([README.HOT#indcheckxmin](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L341-L353)).
- **Partial index whose predicate is not proven.** `check_index_predicates()` sets `predOK` only when the rel's restriction clauses, together with join clauses movable to the rel and equivalence-derived join clauses, imply the index predicate ([indxpath.c#check_index_predicates-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3272-L3350)), and `create_index_paths()` skips a partial index without `predOK` before it matches any clause. The comment at that skip leaves such an index to `generate_bitmap_or_paths()`, which may still use it for an arm of an `OR` ([indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266)).

#### Gate 1: the clause never matches the GIN index

`create_index_paths()` walks `rel->indexlist` and calls `match_restriction_clauses_to_index()` for each index before any cost model runs ([indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L413)). For an `OpExpr`, `match_opclause_to_indexcol()` accepts the clause only when the index column's collation matches and `op_in_opfamily(expr_op, opfamily)` is true; otherwise it falls through to `get_index_clause_from_support()`, the planner-support-function escape hatch ([indxpath.c#match_opclause_to_indexcol-op_in_opfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2433-L2459), [indxpath.c#get_index_clause_from_support](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2557-L2615)).

The core GIN operator families are bootstrap catalog data, and none of them lists `<`, `<=`, `>=`, or `>`:

| GIN opfamily | Operators declared | Evidence |
|---|---|---|
| `gin/array_ops` | `&&`, `@>`, `<@`, `=` (whole-array equality, GIN strategy 4) | [pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244) |
| `gin/tsvector_ops` | `@@`, `@@@` | [pg_amop.dat#gin-tsvector_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1290-L1296) |
| `gin/jsonb_ops` | `@>`, `?`, `?|`, `?&`, `@?`, `@@` | [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611) |
| `gin/jsonb_path_ops` | `@>`, `@?`, `@@` | [pg_amop.dat#gin-jsonb_path_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1613-L1622) |

So `WHERE jb = '{"k": 42}'::jsonb` cannot use a `jsonb_ops` GIN index at all: `jsonb`'s `=` lives in the B-tree and hash families, not the GIN one ([pg_amop.dat#btree-hash-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1571-L1591)). The manual states the rule directly: "each column must be used with operators appropriate to the index type; clauses that involve other operators will not be considered" ([indices.sgml#other-operators](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L505-L508)). `contrib/pg_trgm` says the same about its own classes: "Inequality operators are not supported. Note that those indexes may not be as efficient as regular B-tree indexes for equality operator." ([pgtrgm.sgml#index-support](../../../../raw/postgres-17/doc/src/sgml/pgtrgm.sgml#L413-L425)).

`contrib/btree_gin` closes gate 1 deliberately. Each of its operator classes declares exactly strategies 1 through 5 — `<`, `<=`, `=`, `>=`, `>` — with a comparison function as GIN support function 1, which for `int4` is the type's B-tree comparison proc ([btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69)). Its own documentation states the conclusion this follow-up asks about: "In general, these operator classes will not outperform the equivalent standard B-tree index methods, and they lack one major feature of the standard B-tree code: the ability to enforce uniqueness." ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)).

**A rejected clause is not a rejected index.** Gate 1 decides whether one clause can become an `Index Cond`; it does not decide whether a path over the index exists. `build_index_paths()` generates a path when *any* of four things is true — there is at least one index clause, the index's ordering is useful, an index-only scan is possible, **or the index has a useful predicate** ([indxpath.c#build_index_paths-generate](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L954-L962)). The only clause-related hard stop is on the first index column of an `amoptionalkey = false` AM, and GIN sets `amoptionalkey = true` ([indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897), [ginutil.c:49](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L49)). Two other stops do not depend on clause matching: `build_index_paths()` returns `NIL` when the AM lacks the requested scan type ([indxpath.c#build_index_paths-scantype](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L826-L842)), which [Gate 2](#gate-2-the-required-plan-shape-rules-gin-out) covers, and `create_index_paths()` skips a partial index whose predicate is not proven before it matches any clause ([indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266)), which [Gate 0](#gate-0-the-index-is-not-usable-for-this-query) covers. So a partial GIN index whose predicate the query implies still produces a clauseless path that `gincostestimate()` prices as a whole-index scan, even when not one query operator is in any of its operator families. [The keyless full-index path on a partial GIN index](#the-keyless-full-index-path-on-a-partial-gin-index) measures exactly that. Read gate 1 as "this clause will not be an `Index Cond`", and expect the index to disappear only when no other reason to build a path survives either.

One case that looks like a gate-1 rejection but is not: a boolean column. `WHERE i = true` is simplified to a bare boolean `Var` during constant folding ([clauses.c#simplify_boolean_equality](../../../../raw/postgres-17/src/backend/optimizer/util/clauses.c#L3990-L4045)), so no `OpExpr` survives, but v17 still matches it. `IsBooleanOpfamily()` accepts any opfamily containing `BooleanEqualOperator`, falling back to a catcache lookup for non-built-in opfamilies ([indxpath.c#IsBooleanOpfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2271-L2286), [pg_opfamily.h#IsBuiltinBooleanOpfamily](../../../../raw/postgres-17/src/include/catalog/pg_opfamily.h#L59-L65)), and `match_boolean_index_clause()` rewrites the bare `Var` back into `indexkey = true` ([indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384)). The upstream expected output shows the resulting `Index Cond: (i = true)` on a `btree_gin` bool index ([bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)), and the measurement below reproduces it for `i`, `i = true` and `i IS TRUE` alike.

#### Gate 2: the required plan shape rules GIN out

`get_relation_info()` copies a fixed set of AM capability flags into each `IndexOptInfo`, deriving `amhasgettuple` from whether the AM supplies `amgettuple`, and `amhasgetbitmap` from `amgetbitmap` together with the table AM's `scan_bitmap_next_block` ([plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335)). GIN and B-tree differ on almost every one:

| `IndexAmRoutine` field | GIN | B-tree | Planner consequence |
|---|---|---|---|
| `amgettuple` | `NULL` ([ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79)) | `btgettuple` ([nbtree.c:143](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L143)) | `get_index_paths()` submits a path to `add_path()` only when `index->amhasgettuple`; a GIN path can only be collected into `*bitindexpaths` ([indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751)), and `build_index_paths()` would also return `NIL` for `ST_INDEXSCAN`, though no caller in the pinned tree asks for that scan type ([indxpath.c#build_index_paths-scantype](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L826-L842)) |
| `amcanorder` / `amcanorderbyop` | `false` / `false` ([ginutil.c:44-45](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L44-L45)) | `true` / `false` ([nbtree.c:108-109](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L108-L109)) | `get_relation_info()` fills `sortopfamily` only for `BTREE_AM_OID` or another `amcanorder` AM ([plancat.c#get_relation_info-sortopfamily](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L340-L422)), so `index_is_ordered` is false for GIN and `useful_pathkeys` stays `NIL` ([indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944)) |
| `amcanreturn` | `NULL` ([ginutil.c:70](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L70)) | `btcanreturn` ([nbtree.c:134](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L134)) | `index_can_return()` returns false when `amcanreturn` is `NULL` ([indexam.c#index_can_return](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L780-L797)), so every `canreturn[i]` is false ([plancat.c#get_relation_info-canreturn](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L296-L301)) and `check_index_only()` fails for any query that needs a column; one that needs none, such as a bare `count(*)`, passes it trivially, because an empty set is a subset of any set ([indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800)) |
| `amsearchnulls` | `false` ([ginutil.c:51](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L51)) | `true` ([nbtree.c:115](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L115)) | `match_clause_to_indexcol()` accepts a `NullTest` only when `index->amsearchnulls`, so `IS NULL` never reaches GIN ([indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266)) |
| `amsearcharray` | `false` ([ginutil.c:50](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L50)) | `true` ([nbtree.c:114](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L114)) | A `ScalarArrayOpExpr` is omitted from plain paths and re-offered only as a bitmap path ([indxpath.c#build_index_paths-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L862-L885), [indxpath.c#get_index_paths-nonnative-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L753-L766)), and `counts.arrayScans` multiplies the GIN estimate ([selfuncs.c#gincost_scalararrayopexpr](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7550-L7660)) |
| `amcanparallel` | `false` ([ginutil.c:55](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L55)) | `true` ([nbtree.c:119](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L119)) | No partial GIN *index* path: `build_index_paths()` gates parallel paths on `index->amcanparallel` ([indxpath.c#build_index_paths-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L975-L1002)). It does not keep GIN out of a parallel plan, because the bitmap heap scan above it can still be partial |

**GIN is not shut out of parallel plans, though, and "no parallelism" overstates the flag.** What `amcanparallel = false` removes is a partial *index* path. The bitmap heap scan built on top of a GIN bitmap can still be parallel: `create_index_paths()` hands whatever `choose_bitmap_and()` produced straight to `create_partial_bitmap_paths()` ([indxpath.c:347](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L347)), which sizes workers from the heap pages alone — it passes `index_pages = -1` to `compute_parallel_worker()` — and adds a `Parallel Bitmap Heap Scan` over the unchanged bitmapqual ([allpaths.c#create_partial_bitmap_paths](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4162-L4185)). The bitmap is built once and shared: whichever participating process reaches it first builds it, and the executor calls that process the leader for the parallel bitmap scan ([nodeBitmapHeapscan.c#BitmapShouldInitializeSharedState](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L774-L806), [nodeBitmapHeapscan.c#shared-bitmap-build](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L130-L141)). Every participating process then attaches to one shared iterator over that bitmap, and `tbm_shared_iterate()` hands out the next heap page under a lock on each call, so the heap fetches are divided among the processes ([nodeBitmapHeapscan.c#shared-iterator](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L143-L170), [nodeBitmapHeapscan.c:241](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L241), [tidbitmap.c#tbm_shared_iterate](../../../../raw/postgres-17/src/backend/nodes/tidbitmap.c#L1044-L1136)). So a GIN index cannot *drive* parallelism, and it does not prevent it either.

Three consequences follow, and none can be reversed by tuning:

- **No plain index scan.** The AM developer documentation explains why: `amgetbitmap` returns tuples in a bitmap that "doesn't have any specific ordering", "Ordering operators will never be supplied for such a scan", and "there is no provision for index-only scans with `amgetbitmap`, since there is no way to return the contents of index tuples" ([indexam.sgml#amgetbitmap](../../../../raw/postgres-17/doc/src/sgml/indexam.sgml#L991-L1010)). Two upstream test comments say the same operationally: "GIN currently supports only bitmap scans, not plain indexscans" and "GIN only supports bitmapscan, so no need to test plain indexscan" ([create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268), [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)).
- **No sorted output.** "Of the index types currently supported by PostgreSQL, only B-tree can produce sorted output — the other index types return matching rows in an unspecified, implementation-dependent order." ([indices.sgml#ordering](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L530-L538)). Even a bitmap plan built from a B-tree loses order, because the bitmap is laid out in physical order ([indices.sgml#bitmap-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L643-L656)).
- **No index-only scan.** "As a counterexample, GIN indexes cannot support index-only scans because each index entry typically holds only part of the original data value" ([indices.sgml#index-only-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L1125-L1136)), matching the `amcanreturn` contract.

The upstream `amutils` regression test asserts exactly this property matrix for `gin` versus `btree`: `orderable`, `returnable`, `search_array` and `search_nulls` are all `f` for GIN while `bitmap_scan` is `t` and `index_scan` is `f`, and `can_order`, `can_unique`, `can_exclude`, `can_include` are all `f` ([amutils.out#column-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L96-L108), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129), [amutils.out#am-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L152-L157)).

#### Gate 3: cost, and why GIN loses on the same column

`cost_index()` calls the AM's `amcostestimate` through the `IndexOptInfo` function pointer, so GIN and B-tree paths for the same clause are priced by different code ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). `gincostestimate()` builds its estimate like this:

1. Read the metapage counters with `ginGetStats()`. Only `nPendingPages` is current; the rest are as of the last `VACUUM` or index build ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [selfuncs.c#gincostestimate-trust](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7727), [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)). Scale or invent the other counters as [Stale GIN metapage statistics](#stale-gin-metapage-statistics) describes.
2. Seed the startup page count with the **entire pending list**: `entryPagesFetched = numPendingPages` ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886)).
3. Add `ceil(counts.searchEntries * rint(pow(numEntryPages, 0.15)))` entry pages, plus a proportional share of entry and data pages for partial-match keys ([selfuncs.c#gincostestimate-entrypages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7888-L7914)).
4. Charge about `log2(numEntries)` comparisons per search entry for the entry-tree descent ([selfuncs.c#gincostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7919-L7935)).
5. Charge `DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost`, that is `50 * cpu_operator_cost`, for every entry page and every partial-match data page ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)). The descent charge and this entry-page charge are each added to both the startup and the total estimate, and step 8 adds the startup estimate to the total again, so at `arrayScans = 1` each is counted twice in the total.
6. Charge the pending, entry and partial-match data pages from steps 2 and 3 at `random_page_cost` as **startup** cost, "because logically-close pages could be far apart on disk". When `outer_scans > 1` or `counts.arrayScans > 1`, first multiply both page counts by `outer_scans * counts.arrayScans`, cap each with `index_pages_fetched()` over `numEntryPages` or `numDataPages`, and divide by `outer_scans` ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)).
7. Count the scan-time data pages, taking the larger of the per-entry estimate and a selectivity-derived floor of `ceil(indexSelectivity * numTuples / (BLCKSZ / 3))` ([selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7982-L8006)). Add one more `50 * cpu_operator_cost` per search entry to the startup cost, and one per scan-time data page to the total ([selfuncs.c#gincostestimate-search-entry-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8008-L8015)).
8. Add the startup estimate, and the scan-time data pages at `random_page_cost`, to the total ([selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029)); then add per-qual and per-tuple CPU, with no descent-height charge and no ordering support ([selfuncs.c#gincostestimate-qualcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8031-L8048)).

`btcostestimate()` instead calls `genericcostestimate()`, which prorates `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)` ([selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073), [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)). `btcostestimate()` then adds a `log2(index->tuples)` comparison charge and the `(tree_height + 1) * 50 * cpu_operator_cost` descent charge itself ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091), [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)); see [2. B-tree height carries an explicit anti-bloat charge](#2-b-tree-height-carries-an-explicit-anti-bloat-charge).

That asymmetry is the whole story for a selective equality lookup, and it survives GIN being the physically smaller index. Reconciling both closed forms by hand from the pinned code, with the block counts and metapage counters the script records, matches `EXPLAIN` to the cent for `n = 42` (30 matching rows out of 300,000, `numEntryPages` 278, `numEntries` 10,000, `nDataPages` 0):

- GIN: 2 entry pages plus 1 data page at `random_page_cost` (`12.00`); the entry-tree descent, `ceil(log2(10000)) * 0.0025 = 0.035`, and the two entry pages' `0.25` of page CPU, each counted twice (`0.57`); one search entry's and one data page's page CPU (`0.25`); and `1 * 0.0025 + 30 * 0.005 = 0.1525` of qual and tuple CPU. Predicted total `12.9725`, printed `12.97`.
- B-tree: `ceil(30 * 280 / 300000) = 1` page at `4.0`, plus `ceil(log2(300000)) = 19` comparisons, plus `(1 + 1) * 50 * 0.0025 = 0.25`, plus `30 * (0.005 + 0.0025)`, predicted total `4.5225`, printed `4.52`.

The losing path is then dropped by ordinary path pruning. `add_path()` compares candidates with `compare_path_costs_fuzzily()` at `STD_FUZZ_FACTOR = 1.01` and refuses or removes a dominated path ([pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L622), [pathnode.c#STD_FUZZ_FACTOR](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L42-L47)). Because a GIN path is only ever a bitmap input, the decisive filter is usually `choose_bitmap_and()`: it first keeps only the cheapest path in each group of paths using identical clause sets, sorts the survivors by index access cost, and then adds a further index to the AND group only when `bitmap_and_cost_est()` reports a lower total, returning the cheapest group ([indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489), [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571)). A GIN index whose own scan cost exceeds the saving it produces is therefore dropped from the bitmap tree entirely, and its predicate reappears as a `Filter` above the surviving B-tree bitmap scan.

#### A GIN index with a large pending list loses to a B-tree

This is the case that connects the follow-up back to this page's subject. A large `fastupdate` pending list is charged to the planner in full and immediately, because `nPendingPages` is the one metapage counter `gincostestimate()` treats as current ([selfuncs.c#gincostestimate-trust](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7727), [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)). The pending list is insert work that GIN has postponed, not wasted space ([gin.sgml#fast-update-flush](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515)); it counts as bloat here only in this page's broad sense of excess pages the planner is charged for (see [Types Of Bloated Indexes](#types-of-bloated-indexes)). The manual states the runtime consequence: "searches must scan the list of pending entries in addition to searching the regular index, and so a large list of pending entries will slow searches significantly" ([gin.sgml#fast-update-searches](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L521-L529)).

Measured at the pin on fixture D: a `docs` table with a `tsvector` GIN index (`fastupdate = on`) and a B-tree index on an `int` category column, querying `tsv @@ to_tsquery('simple','zebracorn') AND cat = 7`. The table starts at 200,000 vacuumed rows and takes two further 100,000-row inserts, with `gin_pending_list_limit` raised for the inserting sessions so that nothing drains the list.

The two clauses have to be able to hold together, or every cost below prices a plan for a predicate that matches nothing. `cat` is `i % 20` and the rare lexeme lands on every 4,999th row. 4,999 is prime, so it is coprime with 20 and the two populations are independent: the lexeme rows walk all twenty residues of `i % 20` in turn, and exactly one in twenty of them carries each category. At 400,000 rows that is 80 lexeme rows, 20,000 rows with `cat = 7`, and 4 rows with both. In the final state, after `VACUUM` (400,000 rows; the last row of the table below), the planner estimates 80, 19,999 and 4; in the first, 200,000-row vacuumed state it estimated 40, 10,000 and 2. The `cat = 7` estimate is a row short because the plain `VACUUM` extrapolated `reltuples` from the pages it scanned ([vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1299-L1366)) and left it at 399,985, which `0.05` turns into 19,999. Building the multicolumn index below rewrote `reltuples` to exactly 400,000, because `ginbuild()` returns the heap tuple count of its build scan and `index_build()` writes that count into the table's `pg_class` row ([gininsert.c#ginbuild-scan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L382-L384), [gininsert.c:424](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L424), [index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3131), [index.c#index_update_stats-write](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2923)), and the same clause is then estimated at 20,000. The conjunction is exact by construction: 4 rows exist, and the product of the two selectivities gives 4. That is agreement rather than proof, for the reason [Bloat changes plans](#bloat-changes-plans) gives: the conjunction is estimated by multiplying two per-clause selectivities, which assumes independence, and this fixture is built to satisfy it.

| State | Pending pages (`pgstatginindex`) | GIN blocks | GIN scan cost for `tsv @@ …` alone | Two-clause plan using the GIN index | Same query with the GIN index hidden | Plan chosen |
|---|---:|---:|---:|---:|---:|---|
| vacuumed, 200,000 rows | 0 | 302 | `13.01` | `132.40` | `2323.30` | `BitmapAnd` of GIN and B-tree |
| first insert, 300,000 rows | 736 | 1,038 | `3141.12` | `3321.93` | `3486.80` | `BitmapAnd` still, at 25 times the vacuumed price |
| second insert, 400,000 rows | 1,471 | 1,773 | `6264.98` | not chosen | `4646.42` | B-tree bitmap scan only; `tsv @@ …` demoted to `Filter` |
| after `gin_clean_pending_list()` | 0 | 2,073 | `17.49` | `255.85` | `4646.42` | `BitmapAnd` of GIN and B-tree |
| after `VACUUM` | 0 | 2,073 | `17.46` | `255.81` | `4646.40` | `BitmapAnd` of GIN and B-tree |

The GIN index leaves the `BitmapAnd` at the point where its own scan costs more than it saves. `choose_bitmap_and()` keeps it for as long as the two-index plan is cheaper than the B-tree-only plan, and on this fixture that comparison flips between 736 and 1,471 pending pages: `3321.93` against `3486.80` keeps it, and a GIN scan of `6264.98` against a whole alternative plan of `4646.42` drops it. At 736 pages the planner goes on using the GIN index at 25 times the vacuumed plan's cost. The boundary therefore depends on what the alternative costs and not only on the pending list; the B-tree-only plan is expensive here because `cat = 7` alone selects one row in twenty. The GIN-scan column is a separate `EXPLAIN` of the GIN qual on its own, and the hidden-index column drops the GIN index inside a rolled-back subtransaction.

`gin_clean_pending_list()` returned exactly `1471`, matching `pgstatginindex` ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091), [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)).

Note the direction of the sign: unlike the B-tree page-count penalty in [1. Physical page count enters index cost](#1-physical-page-count-enters-index-cost), this is not a mild pro-rata increase. On a single scan every pending page is charged in full: `random_page_cost` plus the `50 * cpu_operator_cost` page charge counted twice, `4.25` per page at default settings ([selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029)). `gincostestimate()` books all of that charge except one `50 * cpu_operator_cost` term as startup cost, `4.125` of the `4.25`, but the distinction never reaches a plan: a GIN path is only ever a bitmap input, and `cost_bitmap_heap_scan()` takes the index's total cost as its own startup cost whatever the split was ([costsize.c#cost_bitmap_heap_scan-startup](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044-L1048)).

"In full on every scan" needs one qualification, because the two halves of the charge behave differently once the scan repeats. The pending pages are seeded into `entryPagesFetched` ([selfuncs.c:7886](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7886)), and when there is more than one iteration — a nested-loop inner scan with `loop_count > 1`, or an array qual with `counts.arrayScans > 1` — `gincostestimate()` runs `entryPagesFetched` and `dataPagesFetched` through `index_pages_fetched()` and divides by `outer_scans`, much as `genericcostestimate()` does for the B-tree, except that the page universe is `numEntryPages` or `numDataPages`, which does not include the pending pages, so a repeated scan can be charged fewer entry-side pages than the pending list alone holds ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974)):

| Half of the charge | Repeated scans |
|---|---|
| The `random_page_cost` I/O charge | **Amortized.** It is applied after the cache adjustment, so the Mackert-Lohman cap covers the pending pages too ([selfuncs.c:7980](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7980)) |
| The `50 * cpu_operator_cost` per-page charge | **Not amortized.** It is computed from the pre-adjustment page count, and the source says why in as many words: "This is not amortized over a loop" ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955)) |

One GIN measurement on this page does cross that boundary, so "every measurement is the unamortized case" would be wrong. `loop_count` is 1 for every GIN query here, so `outer_scans` is 1 throughout and nothing is pro-rated by it. But fixture S's `n IN (1,2,3)` is a `ScalarArrayOpExpr` over three satisfiable constants, and `gincost_scalararrayopexpr()` ends by multiplying `counts.arrayScans` by the number of them ([selfuncs.c#gincost_scalararrayopexpr-arrayScans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7648-L7658)). That row is therefore priced at `arrayScans = 3`, which satisfies `outer_scans > 1 || counts.arrayScans > 1` and takes the cache adjustment ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974)), while the `50 * cpu_operator_cost` per-page charge is multiplied by `arrayScans` before that and is not amortized. The branch does not depend on `amsearcharray`: `gincostestimate()` hands any `ScalarArrayOpExpr` among a path's index clauses to `gincost_scalararrayopexpr()` ([selfuncs.c#gincostestimate-saop-clause](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7823-L7833)). What `amsearcharray = false` decides is how the array clause reaches the estimate: it is left out of the first `build_index_paths()` call and comes back only through the `ST_BITMAPSCAN` retry ([indxpath.c#build_index_paths-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L862-L885), [indxpath.c#get_index_paths-nonnative-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L753-L766)); see [Gate 2](#gate-2-the-required-plan-shape-rules-gin-out). Every other GIN row on this page prices a single scan with `arrayScans = 1`, where the adjustment does not fire and the charge is unamortized.

#### Stale GIN metapage statistics

`gincostestimate()` trusts the last-`VACUUM` counters only when the index has not grown too much. It requires `numPages > 0`, `nTotalPages <= numPages`, `nTotalPages > numPages / 4`, `nEntryPages > 0` and `nEntries > 0`. Inside that window it does not use the counters as stored: it scales `nEntryPages`, `nDataPages` and `nEntries` by `numPages / nTotalPages`, rounding up, then clamps the entry pages to the non-pending pages and the data pages to what remains ([selfuncs.c#gincostestimate-stats-branch](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7729-L7747)). Fixture D at 736 pending pages is priced this way: `302 <= 1038 < 302 * 4`, the entry pages scale to `ceil(273 * 1038 / 302) = 939` and are clamped to the `1038 - 736 = 302` non-pending pages, and the data pages, `ceil(28 * 1038 / 302) = 97`, are clamped to the `0` pages left. Outside the window it invents statistics from the live block count. After clamping the page count to at least 10, it takes 90% of the non-pending pages as entry pages and the rest of the non-pending pages as data pages, and assumes 100 entries per entry page ([selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767)). The source comment names the 4X cutoff and calls the 100-entries figure "rather bogus".

Two details matter operationally. First, `numPages` comes from `index->pages`, which `get_relation_info()` reads live with `RelationGetNumberOfBlocks()`, so index growth reaches the cost model before any `ANALYZE` ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)). Second, `numPendingPages` is discarded when it is not smaller than `numPages`, which is a sanity guard, not a cost reduction ([selfuncs.c#gincostestimate-pending-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7724-L7727)).

Measured on fixture T: a 2,000-row table's GIN index (`fastupdate = off`, 100 distinct keys) sat at 2 blocks with metapage counters `(nTotalPages, nEntryPages, nDataPages, nEntries) = (2, 1, 0, 100)` and priced `n = 42` at `8.64`. Adding 398,000 rows over 200,000 distinct keys, with an `ANALYZE` but no `VACUUM`, grew it to 1,369 live blocks while the metapage stayed at 2, so `1369 > 2 * 4` put the estimate on the invented branch (1,232 entry pages, 137 data pages, 123,200 entries) and the cost rose to `17.19`. A later `VACUUM` rewrote the metapage to `(1369, 1368, 0, 200000)` and the cost moved by one cent, to `17.20`.

The same staleness is visible after a manual pending-list drain, which is a trap worth naming: `gin_clean_pending_list()` moves entries into the tree and grows the fork but does **not** refresh `nTotalPages`, because only an index build and `VACUUM`'s cleanup call `ginUpdateStats()` ([gininsert.c:406](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L406), [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789)). In fixture D, the freshly vacuumed index read `(302, 273, 28, 1779)` at 302 live blocks, and after the drain it was 2,073 live blocks with the metapage still reading `302`, so `2073 > 302 * 4` kept the cost model on invented statistics until the next `VACUUM` wrote `(2073, 547, 54, 1779)`. The flush that an autovacuum `ANALYZE` performs leaves the counters stale in the same way, because `ginvacuumcleanup()` returns right after that flush, before the `ginUpdateStats()` call ([ginvacuum.c#ginvacuumcleanup-analyze-only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717), [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789)). No fixture on this page exercises that path.

#### The keyless full-index path on a partial GIN index

GIN sets `amoptionalkey = true` ([ginutil.c:49](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L49)), so `build_index_paths()` does not bail out when no clause matches the first index column ([indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897)), and a partial index whose predicate is proven still yields a path through `useful_predicate` ([indxpath.c#build_index_paths-generate](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L954-L962)). `gincostestimate()` then prices that clauseless path as a whole-index scan: when `fullIndexScan` is set or `indexQuals == NIL`, it sets `searchEntries = numEntries`, "as if every key in the index had been listed in the query" ([selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877)). The same branch fires when an attribute has a full scan but no normal scan, which is how `GIN_SEARCH_MODE_ALL` reaches the estimate ([gin.h#GIN_SEARCH_MODE](../../../../raw/postgres-17/src/include/access/gin.h#L34-L37), [selfuncs.c#gincost_pattern-searchmode](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7454-L7489)).

Measured: a **10-block** partial GIN index with 1,000 entries priced `WHERE id <= 5000` (its own predicate, no GIN-indexable clause) at `4430.38`, and the two-`random_page_cost` probe recovered exactly `1001.00` charged pages — 100 times the index's physical size, because `ceil(1000 * rint(pow(9, 0.15))) = 1000`. Adding `AND n = 42` dropped the same index's cost to `8.55`.

#### Jobs no GIN index can be created for

Before any planner gate, four things simply cannot be built on GIN in v17, each rejected because the AM lacks the capability: `amcanunique`, `amcaninclude` and `amclusterable` are false, and an exclusion constraint needs `amgettuple`, which GIN does not supply ([indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879), [cluster.c#check_index_is_clusterable-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522)). All four messages were reproduced verbatim at the pin:

| Attempt | v17 error |
|---|---|
| `CREATE UNIQUE INDEX … USING gin` | `access method "gin" does not support unique indexes` |
| `CREATE INDEX … USING gin (…) INCLUDE (…)` | `access method "gin" does not support included columns` |
| `EXCLUDE USING gin (… WITH =)` | `access method "gin" does not support exclusion constraints` |
| `CLUSTER … USING <gin index>` | `cannot cluster on index "…" because access method does not support clustering` |

#### Where GIN still wins

- **Operators only GIN has.** Gate 1 runs in both directions: `@@`, `@>`, `?`, `&&` and the `jsonpath` operators are in GIN families and not in B-tree ones ([pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244), [pg_amop.dat#gin-tsvector_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1290-L1296), [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611)), so for those predicates there is no B-tree candidate to lose to.
- **One multicolumn GIN instead of a `BitmapAnd`.** The `btree_gin` documentation says that "for queries that test both a GIN-indexable column and a B-tree-indexable column, it might be more efficient to create a multicolumn GIN index that uses one of these operator classes than to create two separate indexes that would have to be combined via bitmap ANDing" ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)). Measured on fixture D after its `VACUUM`: a `gin (tsv, cat)` index priced the two-column predicate at `21.51` and won the plan at `37.20`, against `240.13` for the `BitmapAnd` of the separate GIN and B-tree indexes at a plan cost of `255.82`. That is one cent above the `255.81` the same `BitmapAnd` cost in the table above, because building the multicolumn index rewrote the table's `reltuples` from VACUUM's 399,985 to 400,000 ([index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3131), [index.c#index_update_stats-write](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2923)).
- **Write amortization.** The pending list exists to make GIN insertion cheap, at the documented cost of slower searches ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)).

#### GIN exact-pin measurements

All numbers in this follow-up come from the same run of the same script as the rest of the page: `PostgreSQL 17.11` built from the pin, `autovacuum = off`, `shared_buffers = 256MB`, default planner cost settings (`random_page_cost = 4`, `cpu_operator_cost = 0.0025`, `cpu_index_tuple_cost = 0.005`), and exact statistics; see [Fixtures and method](#fixtures-and-method). `btree_gin` supplied the int4 and bool GIN opclasses, `pgstattuple` supplied `pgstatginindex`, and `pageinspect` supplied `gin_metapage_info()` and `bt_metap()`.

| Fixture | Contents | GIN index | B-tree index |
|---|---|---|---|
| S | `t_both`, 300,000 rows, `n = i % 10000` | `t_both_n_gin`, `btree_gin` int4 opclass, **279 blocks**, `nEntryPages` 278, `nEntries` 10,000, `nDataPages` 0 | `t_both_n_bt`, **280 blocks**, `fastlevel` 1, 90.31% `avg_leaf_density` |
| D | `docs`, 200,000 then 300,000 then 400,000 rows; each `tsvector` is `filler`, `w` plus `i % 1000`, `v` plus `i % 777`, and `zebracorn` on every 4,999th row | `docs_tsv_gin` on `tsvector`, `fastupdate = on`, 302 blocks when vacuumed | `docs_cat_bt` on `cat = i % 20`, 171 blocks |
| T | `t_stale`, 2,000 rows with `n = i % 100`, then 398,000 more with `n = i % 200000` | `t_stale_gin`, `fastupdate = off`, vacuumed at 2,000 rows and not again until the last step | none |
| P-gin | `t_part`, 100,000 rows, `n = i % 1000` | `t_part_gin`, partial `WHERE id <= 5000`, 10 blocks, 1,000 entries | none |
| B-gin | `t_bool`, 100,000 boolean rows, 1% true | `t_bool_gin`, `btree_gin` bool opclass | none |

##### Same table, same statistics, both indexes

Each row below is two `EXPLAIN` runs on fixture S, with the *other* index dropped inside a rolled-back subtransaction, so `pg_statistic`, `reltuples` and every selectivity estimate are identical. `enable_seqscan` was off so that a missing index path is visible as a `disable_cost`-priced sequential scan, and the B-tree runs of the first four rows also had `enable_indexscan` off, so those four rows compare the two `Bitmap Index Scan` nodes. The last three rows give each plan's top node.

| Predicate | GIN | B-tree | Row estimate (both) |
|---|---:|---:|---:|
| `n = 42` | `12.97` | `4.52` | 30 |
| `n BETWEEN 100 AND 200` | `66.30` | `42.60` | 3,030 |
| `n < 20` | `28.57` | `8.80` | 600 |
| `n IN (1,2,3)` | `30.10` | `13.57` | 90 |
| `ORDER BY n LIMIT 10` | no index path: `Sort` over `Seq Scan`, `Limit` at `10000010810.92` | `Limit` at `0.66` | 10 |
| `SELECT n WHERE n = 42` | no index-only scan: `Bitmap Heap Scan` at `119.83` | `Index Only Scan` at `4.82` | 30 |
| `n IS NULL` | no index path: `Seq Scan` at `10000004328.00` | `Index Scan` at `8.31` | 1 |

The last three rows are gate-2 outcomes rather than cost losses, and they are not the same outcome. Two of them, `ORDER BY n LIMIT 10` and `n IS NULL`, have **no GIN path at all**, so the planner falls back to the forced-off sequential scan. `enable_seqscan = off` adds `disable_cost = 1.0e10` ([costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130)) to that scan's startup cost and then prices it normally ([costsize.c#cost_seqscan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L304-L305)). So each figure is `1.0e10` plus the plan's normal cost: the `Seq Scan` at `10000004328.00` is `1.0e10 + 4328.00`, and the `Limit` at `10000010810.92` is `1.0e10 + 10810.92`, the sort over that scan plus its first 10 rows. The other, `SELECT n WHERE n = 42`, still **uses the GIN index**: it loses only the index-*only* scan and falls back to a `Bitmap Heap Scan` at `119.83`, against the B-tree's `Index Only Scan` at `4.82`. Two `NULL` callbacks cause that, and neither removes the bitmap path. GIN's `NULL` `amcanreturn` ([ginutil.c:70](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L70)) fails `check_index_only()`. Its `NULL` `amgettuple` ([ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79)) leaves `amhasgettuple` false ([plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335)), and `get_index_paths()` then keeps every plain or index-only `IndexPath` out of `add_path()` and passes it on only as a bitmap candidate ([indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751)). So GIN cannot get an index-only scan even for a query that needs no columns. That is why only two of the seven rows carry a `disable_cost` figure. With every `enable_*` setting at its default, the planner chose the B-tree for `n = 42`, `n BETWEEN 100 AND 200` and `n < 20` alike.

##### The page charge, isolated

Running the same query at `random_page_cost = 4` and `random_page_cost = 1` isolates the page count, because every other GIN charge is a CPU charge that does not scale with it:

| Case | Cost at `rpc = 4` | Cost at `rpc = 1` | Difference / 3 | Reconciliation |
|---|---:|---:|---:|---|
| Fixture S, `n = 42` | `12.97` | `3.97` | `3.00` | trusted stats (scale `279 / 279 = 1.0`): `rint(278^0.15) = 2` entry pages + 1 data page from the selectivity floor, since the metapage records 0 data pages |
| Fixture D, vacuumed, `nTotalPages` 302 = 302 live blocks | `13.01` | `4.01` | `3.00` | trusted stats (scale `302 / 302 = 1.0`): `rint(273^0.15) = 2` entry pages + 1 data page |
| Fixture D, 736 pending pages, 1,038 live blocks | `3141.12` | `924.12` | `739.00` | scaled stats (`302 > 1038 / 4`, scale `1038 / 302 = 3.44`): entry pages scale to `ceil(273 * 1038 / 302) = ceil(938.33) = 939` but clamp to `1038 - 736 = 302`, so `rint(302^0.15) = 2`; data pages clamp to `1038 - 736 - 302 = 0`, so the 1 data page is the selectivity floor: 736 pending + 2 entry + 1 data page |
| Fixture D, 1,471 pending pages, 1,773 live blocks | `6264.98` | `1842.98` | `1474.00` | invented stats (`302 <= 1773 / 4`): `floor((1773 - 1471) * 0.9) = 271` entry pages, `rint(271^0.15) = 2`: 1,471 pending + 2 entry + 1 data page |
| Fixture D, drained, 2,073 live blocks, metapage still 302 | `17.49` | `5.49` | `4.00` | invented stats (`302 <= 2073 / 4`): `floor(2073 * 0.9) = 1865` entry pages, `rint(1865^0.15) = 3`: 3 entry pages + 1 data page |
| Fixture T, stale metapage, 1,369 live blocks | `17.19` | `5.19` | `4.00` | invented stats (`2 <= 1369 / 4`): `floor(1369 * 0.9) = 1232` entry pages, `rint(1232^0.15) = 3`: 3 entry pages + 1 data page |
| Fixture T after `VACUUM`, metapage `(1369, 1368, 0, 200000)` | `17.20` | `5.20` | `4.00` | trusted stats (scale `1369 / 1369 = 1.0`): `rint(1368^0.15) = 3` entry pages + 1 data page from the selectivity floor |
| Fixture P-gin, keyless partial index, 10 live blocks | `4430.38` | `1427.38` | `1001.00` | trusted stats (scale `10 / 10 = 1.0`); no index qual, so a full-index scan with `searchEntries = numEntries = 1000`, each `rint(9^0.15) = 1` entry page: 1,000 entry pages + 1 data page from the selectivity floor |

`gincostestimate()` has two branches, and the labels in the last column name them. It scales the last-`VACUUM` counters by `numPages / nTotalPages` when the index has at least one page, the recorded total, entry-page and entry counts are nonzero (the data-page count may be zero, as it is for fixtures S, T after `VACUUM` and P-gin), and the recorded total is no larger than the index but larger than a quarter of it, and then clamps entry and data pages to the pages the pending list leaves; "trusted" marks a scale of 1.0 ([selfuncs.c#gincostestimate-stats-branch](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7729-L7747)). Otherwise it invents the counters from the live block count: 90% of the non-pending pages are entry pages, the rest data pages, and each entry page holds 100 entries ([selfuncs.c#gincostestimate-invented](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7748-L7767)). The scan is then charged every pending page plus `rint(numEntryPages^0.15)` entry pages per search entry ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886), [selfuncs.c#gincostestimate-entrypages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7888-L7914)). A scan with no index qual counts every entry as a search entry ([selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877)). The data-page count has a floor of `ceil(selectivity * tuples / (BLCKSZ / 3))`, so a scan whose prorated data-page count is 0 is still charged at least one data page whenever its selectivity and the index's tuple estimate, `index->tuples`, are both above zero ([selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7982-L8006), [selfuncs.c:7675](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7675)).

The two pending-list rows are the sharpest result: 736 and 1,471 pages that hold no tree structure at all are charged page for page, `739.00` and `1474.00`, on a scan that is charged 4 pages once the list is gone. They also land on different branches. At 1,038 live blocks the last-`VACUUM` counters are still within the 4X window (`302 > 1038 / 4`), so they are scaled; at 1,773 blocks they are not, and the estimate is invented from the block count. The plan consequences are in [A GIN index with a large pending list loses to a B-tree](#a-gin-index-with-a-large-pending-list-loses-to-a-b-tree).

##### Boolean column

Fixture B-gin priced `WHERE i`, `WHERE i = true` and `WHERE i IS TRUE` identically at `38.26` for the GIN index scan, each with `Index Cond: (i = true)`. So a bare boolean `Var` is not a gate-1 rejection in v17. `i IS TRUE` additionally left `Filter: (i IS TRUE)` on the heap node while still using the index.

##### Live property matrix

Queried on fixture S, `pg_index_has_property` and `pg_index_column_has_property` returned exactly the values the `amutils` expected output asserts: `index_scan`, `clusterable` and `backward_scan` false for GIN and true for B-tree; `bitmap_scan` true for both; `orderable`, `returnable`, `search_array` and `search_nulls` false for GIN and true for B-tree ([amutils.out#column-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L96-L108), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129)).

##### A diagnostic pair for a live server

The script's `diag` stage reads both blocks out of this page and runs them at the pin against objects literally named `my_table`, `my_col` and `my_gin_index`. Both blocks below are exactly what that stage ran in the two final runs on 2026-09-25, after the script text and both blocks were final. The first reports how much of one named GIN index is pending list. The planner charges a single scan one page read for every pending page ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886)), although those pages hold deferred insert work rather than wasted space ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)). Drop the `c.relname` condition to list every valid, non-partitioned GIN index in the database other than another session's temporary ones. The inner filters keep `pgstatginindex()` away from three kinds of GIN index that it rejects with an `ERROR`, which would abort the whole query. The first is a partitioned index: it carries the GIN access method in `relam` like any other index, but its `relkind` is `'I'`, and `pgstatginindex()` accepts only `relkind` `'i'` ([index.c:1015](../../../../raw/postgres-17/src/backend/catalog/index.c#L1015), [pg_class.h#relkinds](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L162-L173), [pgstatindex.c#IS_INDEX](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70-L72)). The second is an invalid index, such as a failed `CREATE INDEX CONCURRENTLY` leaves behind ([create_index.sgml#concurrently-invalid](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L651)). The third is another session's temporary index, whose `relpersistence` is `'t'` ([pg_class.h:177](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L177), [pgstatindex.c#pgstatginindex_internal-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L520-L543), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L667-L669)). The `OFFSET 0` fence makes sure those filters run before the call: a subquery with `OFFSET` is never pulled up into the outer query, so it is planned and filtered on its own before its rows reach the `LATERAL` call ([prepjointree.c#is_simple_subquery](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1689-L1701)).

```sql
SET /* wiki_gin_pending_list_share */ statement_timeout = '30s';
SET /* wiki_gin_pending_list_share */ lock_timeout = '5s';

SELECT /* wiki_gin_pending_list_share */
       gi.relname                                       AS gin_index,
       pg_relation_size(gi.oid) / current_setting('block_size')::int AS live_blocks,
       g.pending_pages,
       g.pending_tuples,
       round(100.0 * g.pending_pages
             / greatest(pg_relation_size(gi.oid)
                        / current_setting('block_size')::int, 1), 2)
                                                        AS pending_pct_of_index
  FROM (SELECT c.oid, c.relname
          FROM pg_class c
          JOIN pg_index i ON i.indexrelid = c.oid
          JOIN pg_am    a ON a.oid = c.relam
         WHERE a.amname = 'gin'
           AND c.relkind = 'i'
           AND i.indisvalid
           AND (c.relpersistence <> 't'
                OR c.relnamespace = pg_my_temp_schema())
           AND c.relname = 'my_gin_index'
        OFFSET 0) AS gi  -- fence: filter before pgstatginindex() runs
  CROSS JOIN LATERAL pgstatginindex(gi.oid) AS g
 ORDER BY g.pending_pages DESC;

RESET /* wiki_gin_pending_list_share */ statement_timeout;
RESET /* wiki_gin_pending_list_share */ lock_timeout;
```

It reported `live_blocks = 340`, `pending_pages = 246`, `pending_tuples = 50000`, `pending_pct_of_index = 72.35`, against a `my_table` of 100,000 vacuumed rows with 50,000 more waiting in the pending list. `pgstatginindex` comes from `pgstattuple`. Since extension version 1.5 the SQL function binds to a C entry point with no superuser check of its own and relies on `EXECUTE`, which the upgrade script revokes from `PUBLIC` and grants to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57), [pgstatindex.c#pgstatginindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L497-L504)). It takes `AccessShareLock` on the index and reads only the metapage ([pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)).

The second recovers the number of pages the planner is charging, without any contrib module:

```sql
SET /* wiki_gin_page_charge_probe */ statement_timeout = '30s';
SET /* wiki_gin_page_charge_probe */ lock_timeout = '5s';
SET /* wiki_gin_page_charge_probe */ enable_seqscan = off;

-- Run this first.  get_tablespace_page_costs() returns the tablespace's own
-- random_page_cost whenever that reloption is set, and ignores the session
-- GUC entirely.  If spc_random_page_cost below is not NULL, the two EXPLAIN
-- runs price the index's pages identically and the subtraction reports zero
-- charged pages no matter how many the planner is really charging for.
SELECT /* wiki_gin_page_charge_tablespace_check */
       c.relname                                       AS gin_index,
       t.spcname                                       AS tablespace,
       (SELECT o.option_value
          FROM pg_options_to_table(t.spcoptions) AS o
         WHERE o.option_name = 'random_page_cost')     AS spc_random_page_cost
  FROM pg_class c
  JOIN pg_tablespace t
    ON t.oid = CASE WHEN c.reltablespace <> 0 THEN c.reltablespace
                    ELSE (SELECT d.dattablespace FROM pg_database d
                           WHERE d.datname = current_database()) END
 WHERE c.relname = 'my_gin_index';

SET /* wiki_gin_page_charge_probe */ random_page_cost = 4;
EXPLAIN /* wiki_gin_page_charge_probe_high */ (COSTS ON)
SELECT count(*) FROM my_table
 WHERE my_col @@ to_tsquery('simple', 'filler');

SET /* wiki_gin_page_charge_probe */ random_page_cost = 1;
EXPLAIN /* wiki_gin_page_charge_probe_low */ (COSTS ON)
SELECT count(*) FROM my_table
 WHERE my_col @@ to_tsquery('simple', 'filler');

RESET /* wiki_gin_page_charge_probe */ random_page_cost;
RESET /* wiki_gin_page_charge_probe */ enable_seqscan;
RESET /* wiki_gin_page_charge_probe */ statement_timeout;
RESET /* wiki_gin_page_charge_probe */ lock_timeout;
```

The `Bitmap Index Scan` costs were `2030.46` and `1121.46`, so `(2030.46 - 1121.46) / 3 = 303.00` pages, of which 246 were pending list. Divide by three because the two runs differ by exactly `3.0` per charged page. Both blocks change settings for their own session only and reset them: [`statement_timeout`](../../../glossary.md#statement_timeout-and-lock_timeout), `lock_timeout`, `enable_seqscan` and `random_page_cost` are all `PGC_USERSET`, so none needs a reload or a restart ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792), [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)). On the measurement server the check returns `pg_default` and a NULL `spc_random_page_cost`, so the session GUC is what `gincostestimate()` sees.

Two conditions have to hold before the subtraction means anything, and neither is visible in the arithmetic:

- **The index's tablespace must not override `random_page_cost`.** `gincostestimate()` prices its pages with `get_tablespace_page_costs(index->reltablespace, ...)`, and that function returns the tablespace's `random_page_cost` reloption whenever it is set to a non-negative value, falling back to the GUC only when there is no reloption ([spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196), [selfuncs.c#gincostestimate-tablespace-costs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7786-L7789)). An index on such a tablespace prices identically at `random_page_cost = 4` and `= 1`, and the probe then reports `0.00` charged pages for an index that may be charging thousands. The query above is what tells the two cases apart; it reads the catalog only and takes no lock on the index.
- **Both plans must name the same index.** Changing `random_page_cost` changes relative costs, so the cheaper setting can select a different index, a different bitmap combination, or a different node shape. The difference is then two unrelated plans subtracted from each other. Read the `Bitmap Index Scan` line in both outputs and check it names the index you are probing before dividing by three.

#### GIN settings that move the boundary

| Setting | v17 default | Role in the GIN-versus-B-tree decision | Apply scope |
|---|---|---|---|
| `random_page_cost` | 4.0 ([cost.h:25](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25)) | Multiplies every pending, entry and data page GIN expects to touch, unless the index's tablespace sets its own `random_page_cost` ([selfuncs.c#gincostestimate-tablespace-costs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7786-L7789), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980), [selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029), [spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196)) | session/transaction (`PGC_USERSET`, [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)) |
| `cpu_operator_cost` | 0.0025 ([cost.h:28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L28)) | Scales GIN's entry-tree descent and its `50 *` per-page CPU charges ([selfuncs.c#gincostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7919-L7935), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c#gincostestimate-search-entry-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8008-L8015), [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), and the B-tree's height charge ([selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)) | session/transaction (`PGC_USERSET`, [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729)) |
| `gin_pending_list_limit` | 4MB (`4096` kB, [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)) | The size at which an insert *asks* for a cleanup. Not a ceiling: see below ([ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471)) | session/transaction (`PGC_USERSET`, [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)) |
| `enable_bitmapscan` | on ([guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822)) | Turning it off does not remove GIN's plan shape; it adds `disable_cost` to it. See below ([costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042)) | session/transaction (`PGC_USERSET`, [guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822)) |

Two of those four need the fine print spelled out, because the obvious reading of each is wrong.

**`gin_pending_list_limit` triggers a cleanup; it does not cap the list.** `ginHeapTupleFastInsert()` writes the new entries first and only then compares the resulting size against the limit, setting `needCleanup` after the fact, so the list is already over the limit when the test fires ([ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471)). The cleanup that follows is deliberately not forced: `ginInsertCleanup()` called from a regular insert takes the metapage lock only conditionally and, if another process holds it, returns at once "in hope that concurrent process will clean up pending list" ([ginfast.c#ginInsertCleanup-conditional-lock](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828)). The GIN chapter frames it the same way, as a condition under which entries are moved, and notes that "the overhead work can be done by a background process" ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)). The setting's own reference entry and its `guc_tables.c` description, however, call it the "maximum size" of the pending list ([config.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9870-L9890), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)), which the source does not enforce; that discrepancy is filed under [Open Questions](#open-questions). So the setting bounds the *typical* startup penalty, not its maximum — fixture D reaches 1,471 pending pages precisely by raising the limit so that no insert ever asks.

**`enable_bitmapscan = off` does not remove the path.** It is a cost penalty, not a veto: `cost_bitmap_heap_scan()` adds `disable_cost` to the startup cost and then prices the path normally ([costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042)). A GIN plan therefore still wins when every alternative is also disabled or more expensive — which is exactly how the `disable_cost`-priced sequential scans in the table above were produced, with `enable_seqscan = off`. What turning it off reliably does is make any other viable plan win.

Two per-index storage parameters change the physical shape rather than its price, and both take [`AccessExclusiveLock`](../../../glossary.md#accessexclusivelock): `fastupdate` (default on) and a per-index `gin_pending_list_limit` override ([reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L130), [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)). `gin_clean_pending_list()` drains the list on demand and takes `RowExclusiveLock` on the index ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091)).

#### GIN key data structures

| Structure | Field | Role |
|---|---|---|
| `GinQualCounts` | `partialEntries`, `exactEntries`, `searchEntries`, `arrayScans`, `attHasFullScan`, `attHasNormalScan` | The whole per-qual working set `gincostestimate()` derives from the index quals ([selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7370-L7378)) |
| `GinStatsData` | `nPendingPages`, `nTotalPages`, `nEntryPages`, `nDataPages`, `nEntries` | The metapage counters ([gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L40-L50)); only `nPendingPages` and `ginVersion` are current ([ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)) |
| `GinMetaPageData` | same counters, on disk | Where those numbers live, and why `VACUUM` is what refreshes them ([ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101)) |
| `FormData_pg_index` | `indisvalid`, `indcheckxmin` | Gate-0 inputs, read before any `IndexOptInfo` exists ([pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L43)). `get_relation_info()` skips an invalid index, and skips an `indcheckxmin` index whose `pg_index` row is not yet older than `TransactionXmin`, marking the plan transient ([plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281)) |
| `IndexOptInfo` | `indpred`, `predOK` | The gate-0 partial-index test ([pathnodes.h:1168](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1168), [pathnodes.h:1181](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1181)). `check_index_predicates()` sets `predOK` when the query's restrictions imply the predicate ([indxpath.c#check_index_predicates-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3272-L3350)), and `create_index_paths()` skips a partial index without it ([indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266)) |
| `IndexOptInfo` | `amhasgettuple`, `amhasgetbitmap`, `amcanparallel`, `amsearcharray`, `amsearchnulls`, `amoptionalkey`, `sortopfamily`, `canreturn[]` | The gate-2 inputs, all filled once in `get_relation_info()`: the flags ([plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335)), `canreturn[]` ([plancat.c#get_relation_info-canreturn](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L296-L301)) and `sortopfamily` ([plancat.c#get_relation_info-sortopfamily](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L340-L422)) |

#### GIN caller and callee boundary

```text
build_simple_rel                                relnode.c
  └─ get_relation_info                          plancat.c
       ├─ skips an index that is not indisvalid                      GATE 0
       ├─ skips an indcheckxmin index not yet usable (plan transient) GATE 0
       └─ fills IndexOptInfo: AM flags, canreturn[] via index_can_return, sortopfamily

set_plain_rel_pathlist                          allpaths.c
  └─ create_index_paths                         indxpath.c
       ├─ skips a partial index that is not predOK                   GATE 0
       ├─ match_restriction_clauses_to_index
       │    └─ match_clauses_to_index -> match_clause_to_index
       │         └─ match_clause_to_indexcol
       │              ├─ match_boolean_index_clause  (IsBooleanOpfamily opfamilies)
       │              ├─ match_opclause_to_indexcol  -> op_in_opfamily   GATE 1
       │              │    └─ get_index_clause_from_support
       │              ├─ match_saopclause_to_indexcol
       │              └─ NullTest branch             needs amsearchnulls GATE 2
       ├─ get_index_paths
       │    ├─ build_index_paths(ST_ANYSCAN)    amoptionalkey / pathkeys
       │    │    ├─ check_index_only            reads canreturn[]    GATE 2
       │    │    └─ create_index_path -> cost_index
       │    │         └─ amcostestimate == gincostestimate   GATE 3 input
       │    │              ├─ ginGetStats       -> GIN metapage
       │    │              ├─ gincost_opexpr / gincost_scalararrayopexpr
       │    │              │    └─ gincost_pattern   -> extractQuery support proc
       │    │              └─ index_pages_fetched    (nestloop / array scans)
       │    ├─ add_path                         amhasgettuple only   GATE 2
       │    └─ build_index_paths(ST_BITMAPSCAN)  non-native SAOP retry
       ├─ choose_bitmap_and                     GATE 3
       │    ├─ cost_bitmap_tree_node            same clause set: keeps the cheaper index (GIN vs B-tree)
       │    ├─ bitmap_scan_cost_est -> cost_bitmap_heap_scan   prices each AND-group leader
       │    └─ bitmap_and_cost_est -> bitmap_scan_cost_est     adds an index only if the whole scan gets cheaper
       ├─ create_bitmap_heap_path -> cost_bitmap_heap_scan     enable_bitmapscan adds disable_cost
       └─ add_path                              whole paths compete  GATE 3
```

`GATE 3` marks where cost decides. `gincostestimate()` only supplies the index estimate. For one clause matched by both a GIN and a B-tree index, as on fixture S, `choose_bitmap_and()` keeps whichever index path `cost_bitmap_tree_node()` prices lower: its index estimate plus a small per-row bitmap charge ([indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1399), [costsize.c#cost_bitmap_tree_node](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1114-L1144)). For different clauses it adds an index to an AND group only when the whole bitmap-heap-scan estimate drops ([indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489)). The resulting bitmap heap path then competes with every other path for the relation in `add_path()` ([indxpath.c:343](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L343), [pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L622)).

Symbols: [relnode.c:340](../../../../raw/postgres-17/src/backend/optimizer/util/relnode.c#L340), [plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281), [plancat.c#get_relation_info-canreturn](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L296-L301), [allpaths.c:783](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L783), [indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L413), [indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266), [indxpath.c#match_restriction_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1968-L1974), [indxpath.c#match_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2051-L2064), [indxpath.c#match_clause_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2084-L2136), [indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269), [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767), [indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751), [indxpath.c#build_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L804-L1057), [indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800), [pathnode.c:1024](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L1024), [costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621), [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L8050), [selfuncs.c#gincost_pattern](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7380-L7492), [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526-L1553), [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571), [indxpath.c:341](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L341), [pathnode.c#create_bitmap_heap_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L1042-L1068), [costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042), [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89).

#### GIN tests and explicit test absence

- **No in-tree test compares a GIN plan against a B-tree plan on the same column**, and none asserts a `gincostestimate()` cost, though every GIN plan in the suites runs it. `src/test` contains no reference to `gincostestimate`.
- The `btree_gin` regression suite sets `enable_seqscan = off` and uses `EXPLAIN (COSTS OFF)` throughout, so it asserts plan *shape* only, never cost ([bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9), [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)).
- Gate 0's `indcheckxmin` skip has no regression test: no file under `src/test` mentions `indcheckxmin`, and neither does any `contrib` test (its only `contrib` mention is in `amcheck`'s `verify_nbtree.c`).
- What *is* covered is the gate-2 property matrix, by `amutils`: its column-level half (`orderable`, `returnable`, `search_array`, `search_nulls`) and its index-level half (`clusterable`, `index_scan`, `bitmap_scan`, `backward_scan`) ([amutils.out#column-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L96-L108), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129)), and the bitmap-only restriction, by comments and expected plans in `create_index` and `tsearch` ([create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268), [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)).
- Every measurement in this follow-up comes from this page's own script on an isolated exact-pin server, not from an in-tree test.

## Measurement Script

### Usage

| Item | Detail |
|---|---|
| Purpose | produces every measured number on this page, in the main answer and in the GIN follow-up: block counts, `pgstatindex` density and fragmentation, `bt_metap()` fast-root levels, GIN metapage counters and pending pages, every `EXPLAIN` cost, row estimate, plan width and worker count, the closed-form predictions, fixture Q's charged index pages and catalog rows, fixture E's dead-entry counts, the catalog, metapage-root, estimate-against-truth and AM-property values the page quotes, the four GIN rejection messages, and the verbatim run of the two filed diagnostic blocks |
| Invocation | `bash .wiki-runtime/tmp/bloatplan.sh` from the repository root, with the script saved at that path. Any path works: it resolves everything from `WIKI_ROOT`, which defaults to `$PWD` |
| Stages | `build check cluster fa fb ff fg fh fi fn fstale fl3 fp fq fe gs gd gt gp gb grej diag predict summary stop` in that default order, plus `clean` on request. Select stages as arguments: `bash .wiki-runtime/tmp/bloatplan.sh fa predict`. `build` checks that the checkout is at the pin with no tracked file changed, configures out of tree, installs core plus `pgstattuple`, `pageinspect` and `btree_gin`, and records the pin beside the binaries last. It skips the build when that record exists and names the pin; an install with no record, left by an interrupted build, is removed and built again. `check` repeats the pin checks and runs `make check` and the three contrib suites. `cluster` runs `initdb` once, starts the server, and installs the recording helpers `xp()`, `nodes()`, `idxcost()`, `ixstat()`, `ginstat()`, `predict()` and `fact()`. `fa` `fb` `ff` `fg` `fh` `fi` `fn` `fstale` `fl3` `fp` `fq` `fe` build the B-tree fixtures A, B with M, F with F-one, G, H, I, N, the forged catalog rows, the `BitmapAnd` pair L3, P with P-100, the partial-index pair Q, and the endpoint-probe table E. `gs` `gd` `gt` `gp` `gb` build the GIN fixtures S, D, T, P-gin and B-gin. `grej` sends the four statements that must fail. `diag` reads the two `sql` blocks out of this page and runs them verbatim. `predict` compares the closed form with `EXPLAIN`. `summary` writes the result file. `stop` stops the server and asserts the teardown. Every fixture stage drops and rebuilds its own tables and replaces its own result rows, so any stage can be re-run alone; `grej` needs `gs` and refuses to run without it, and `predict` needs `fa`, `fb` and `fg` for its seven rows. Every SQL stage starts the server itself if it is not up, so a selected re-run works after a default run has already stopped the cluster; `cluster` starts it by definition, and `build`, `check`, `stop` and `clean` never start one |
| Failure handling | Every stage name is checked before any stage runs, so a misspelt stage stops the run before anything is built or started. Every `psql` call that fails ends the run with a non-zero status: `pg()`, `pgq()` and `pgopt()` end in a `die`, and each `psql` call made inside `$(...)`, where `die` can only end the subshell, is followed by its own `\|\| die`. Stage exit status is checked on top of that, so a failed `configure`, `make`, `make check` or `psql` cannot be reported as a pass. There are two deliberate exceptions. One is `pgerr()`, used only by `grej`, whose four statements are meant to fail. The other is the background snapshot holder, which the script ends itself with `pg_terminate_backend()`; its exit status is discarded, and the script checks instead that the holder took a snapshot and that exactly one live holder was ended after the churn. `grej` therefore first checks that fixture S's GIN index exists, then requires exactly four `ERROR` lines, each containing one of the four expected rejection texts, so four failures for any other reason, such as a missing table, stop the run. An `EXIT` trap stops the server and the snapshot-holding session on every path out, including a `die` in the middle of a fixture and an interrupt, and leaves the sandbox for inspection; it touches a server only in a sandbox the run claimed as its own |
| Environment | `WIKI_ROOT` (`$PWD`), `SRC` (`$WIKI_ROOT/raw/postgres-17`), `SANDBOX` (`$WIKI_ROOT/.wiki-runtime/tmp/bloatplan`, and it must resolve to a directory directly inside `.wiki-runtime/tmp`), `PAGE` (this page under `$WIKI_ROOT`), `PORT` (`55437`), `JOBS` (`8`), `STATS_TARGET` (`10000`). The pin is a constant in the script, not a variable. A `PGOPTIONS` in the caller's environment reaches no measurement session, because every session gets its options passed explicitly. The script also clears a caller's `PGSERVICE`, `PGSERVICEFILE` and `PGHOSTADDR`: libpq fills options from a service entry before it reads the environment ([fe-connect.c#conninfo_add_defaults](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L6201-L6245)), so a service would override the exported `PGHOST`, `PGPORT`, `PGDATABASE` and `PGUSER`, and a host address would send sessions over TCP instead of to the sandbox socket ([fe-connect.c#pqConnectOptions2-host-type](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L1189-L1204), [fe-connect.c#PQconnectPoll-host-address](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L2754-L2764)) |
| Prerequisites | `bash` 3.2 or later (both macOS `/bin/bash` 3.2.57 and bash 5.3.15 ran the final text; see [Last run](#last-run)), `git`, a C toolchain, GNU `make` 3.81 or newer ([installation.sgml:40](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L40)), `flex`, `bison`, `perl` for the build, zlib headers, and `pgrep`, without which `stop` and `clean` refuse to report a clean teardown. The tree is configured `--without-icu --without-readline`, so neither library is needed. `initdb` runs with `--locale=C --encoding=UTF8`. `$SANDBOX` must contain no single quote, because the socket directory is passed to the server inside single quotes, and `$SANDBOX/sock/.s.PGSQL.$PORT` must be shorter than the platform's Unix-socket path buffer, which the server checks against `UNIXSOCK_PATH_BUFLEN` ([pqcomm.h:60](../../../../raw/postgres-17/src/include/libpq/pqcomm.h#L60), [pqcomm.c:453](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L453)). About 3 GiB of disk for the build, install and data directories |
| Output | `$SANDBOX/out/summary.txt`: the B-tree index table, the GIN table, the prediction table, each fixture table's size and visibility state, the `fx` table of every other value the page quotes, one line per recorded plan with every node's index, costs, rows, width, workers, index condition, recheck condition and filter, then the platform facts, the four error messages and the two diagnostic outputs. Read it first. `server.log`, the build and test logs, the snapshot holders' logs, and the extracted `diag1.sql` and `diag2.sql` sit beside it |
| Runtime | under 2 minutes for a full run on 10 cores, of which roughly 1 minute 25 seconds is the build and the four test suites; a re-run of every fixture stage on a built tree takes about 30 seconds |
| Cleanup | `bash .wiki-runtime/tmp/bloatplan.sh clean` stops the server and deletes `$SANDBOX`, and refuses a directory that does not carry the script's `.bloatplan-sandbox` mark. The `stop` stage, which runs by default, stops the server and asserts that `pg_ctl` finds no server and that no `postmaster.pid`, no process from the data directory and no socket on `$PORT` is left; it refuses to assert anything on a host without `pgrep`. Neither is the only safety net: the `EXIT` trap stops the server on any exit path, so an aborted run leaves no postmaster behind either |

Isolation: the pinned checkout is read only and checked against the pin before it is built, the build is out of tree, the cluster has its own data directory, socket directory and port `55437`, the libpq variables that could redirect a session elsewhere are cleared, and every fixture table is disposable. Stage `fstale` forges two `pg_class` rows on purpose, tagged `wiki_bloatplan_fixture_catalog_forgery`; they belong to a throwaway fixture and must never be pointed at a database anyone cares about. GUC apply scopes are named in the script's own header, from the pinned definitions: `shared_buffers`, `port`, `listen_addresses` and `unix_socket_directories` are `PGC_POSTMASTER`, so they need a restart and are set once on the postmaster command line ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270), [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2393-L2401), [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4436-L4445), [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4425-L4434)); `autovacuum` is `PGC_SIGHUP`, a reload setting, also set once there ([guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)); every other setting the script touches is `PGC_USERSET`, session or transaction scope: `default_statistics_target`, the six `enable_*` switches, `effective_cache_size`, `random_page_cost`, `cpu_operator_cost`, the five parallel settings, `gin_pending_list_limit`, `statement_timeout`, `lock_timeout` and `application_name` ([guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2070-L2079), [guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792), [guc_tables.c#enable_indexscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L793-L802), [guc_tables.c#enable_indexonlyscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L803-L812), [guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822), [guc_tables.c#enable_mergejoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L893-L902), [guc_tables.c#enable_hashjoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L903-L912), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518), [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696), [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729), [guc_tables.c#max_parallel_workers_per_gather](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3419-L3428), [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439), [guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529), [guc_tables.c#parallel_tuple_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3730-L3740), [guc_tables.c#parallel_setup_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3741-L3751), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585), [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#application_name](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4644-L4653)). Every `psql` call runs `-X` with session-scoped `statement_timeout` and `lock_timeout` and with `ON_ERROR_STOP`, except `grej`'s, whose four statements are meant to fail and which asserts that exactly four errors came back. The snapshot holder runs with the same options in the background; its 20-minute `statement_timeout` outlasts its 900-second sleep, and the script ends it after the churn and checks that exactly one live holder was ended. Every statement the script writes carries a `/* wiki_bloatplan_... */` tag after its leading verb, statements inside the recording helpers included; the two filed diagnostic blocks carry their own `/* wiki_gin_... */` tags.

### Last run

| Item | Value |
|---|---|
| Date | 2026-09-25, two full runs of the script text filed below, each from an empty sandbox of its own, started together: one under Homebrew bash 5.3.15 (16:20:37Z to 16:22:49Z, port 55437) and one under macOS `/bin/bash` 3.2.57 (16:20:39Z to 16:22:46Z, `SANDBOX=.wiki-runtime/tmp/bloatplan32`, `PORT=55438`). Both ran after the script text and the two diagnostic blocks were final. The same text had run twice earlier that day, once under each bash, before the page's prose was revised, and the unchanged 2026-09-23 text ran twice before that, as the review's baseline |
| Server | `PostgreSQL 17.11 on aarch64-apple-darwin27.0.0, compiled by Apple clang version 21.0.0 (clang-2100.3.34.2), 64-bit`, built from pin `786db8dcf168bd9df8f55047337525ac19118b1c`, configured `--without-icu --without-readline`. The script's pin checks passed, and its platform record names the same commit for the pin, the install and the checkout |
| Platform | Darwin arm64, bash 5.3.15 and bash 3.2.57, `block_size` 8192, maximum data alignment 8, `initdb --locale=C --encoding=UTF8` |
| Test suites | in both runs, `make check` All 225 tests passed; `pgstattuple` All 1, `pageinspect` All 8, `btree_gin` All 30 |
| Agreement | the two runs wrote a byte-identical `summary.txt` apart from its run timestamps and the bash version line: 37 B-tree index rows, 11 GIN rows, 7 of 7 closed-form predictions equal to `EXPLAIN`, 29 fixture-table rows, 50 recorded facts, 146 plans, four error messages and both diagnostic outputs. Every row the 2026-09-23 text recorded came back unchanged, both in the review's unchanged re-run and in these runs; the new rows are fixtures Q and E, and each plan line now also carries its width and recheck condition |
| Runtime | 2 minutes 12 seconds and 2 minutes 7 seconds for the two runs side by side, of which about 1 minute 30 seconds is the two builds and test suites competing for 10 cores; the fixture stages took 40 seconds. A single run on an idle machine took 1 minute 35 seconds earlier the same day |
| Failure handling, checked | `cluster bogus` is refused before anything is built or started. A `SANDBOX` outside `.wiki-runtime/tmp`, one that leaves it through `..` or through a symbolic link, and an existing non-empty unmarked directory are refused; an empty unmarked directory is claimed and marked, and `stop` and `clean` refuse any unmarked directory and leave it intact. `clean` treats a missing sandbox as nothing to do. A `SRC` that is not a git checkout stops `build`. Those checks date from the 2026-09-23 text and were not repeated. Checked on 2026-09-25 against the new code: `grej` run where fixture S's table had been dropped stopped with "grej needs fixture S: run the gs stage first" and status 1, and `gs grej stop` then passed; `build` run on an install whose build record had been deleted removed the install, rebuilt it and wrote the record, status 0; and `stop` run with no `pgrep` on `PATH` stopped with "pgrep is needed to assert that no postgres process is left" and status 1 |
| Teardown | after each run the `stop` stage asserted that `pg_ctl` found no server and that no `postmaster.pid`, no process from the data directory and no socket on the run's port was left; the `clean` stage then stopped the server and deleted each sandbox |

The published text was extracted from this page and compared byte for byte with the script text that ran; its md5 is `1a33c8e4a82da25dde6ed462f6495abf`.

### The script

```bash
#!/usr/bin/env bash
# Measurements for the wiki page
#   wiki/v17/questions/query-planning/bloated-indexes-query-planner.md
#
# What this measures, and why it is safe to run
# ---------------------------------------------
# Every number that page reports about a running v17 server comes from this
# script.  It builds the pinned PostgreSQL 17 checkout out of tree, starts one
# isolated cluster, builds pairs of indexes that hold the same logical content
# in different physical shapes, and records what the planner charges for them:
# index block counts, pgstatindex density and fragmentation, the B-tree
# fast-root level, GIN metapage counters, and EXPLAIN costs.  Nothing is timed.
#
# The pinned checkout under raw/postgres-17 is read only, and the build and
# check stages refuse to run unless it is at $PIN with no tracked file changed;
# build records $PIN beside the binaries, and every later stage refuses an
# install built from anything else.  Everything this script writes lives under
# $SANDBOX, which must resolve to a directory directly inside this
# repository's .wiki-runtime/tmp.  The script marks the sandbox when it creates
# it, and stop and clean touch nothing that does not carry that mark.  The
# cluster has its own data directory, its own socket directory and a
# non-default port; it is never a cluster anyone else named.  Every fixture is
# disposable: the script creates and drops its own tables in its own database.
# Two statements forge pg_class rows on purpose (stage fstale); they are part
# of a disposable fixture and must never be pointed at a database anyone
# cares about.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/bloatplan.sh                # every stage, in order
#   bash .wiki-runtime/tmp/bloatplan.sh fa predict     # selected stages
#   bash .wiki-runtime/tmp/bloatplan.sh clean          # stop, delete the sandbox
#
# Stages, in default order:
#   build check cluster fa fb ff fg fh fi fn fstale fl3 fp fq fe
#   gs gd gt gp gb grej diag predict summary stop
# and, on request only: clean
#
# Results.  Index rows, GIN rows, predictions and plans go to the tables ix,
# gx, pr and r; every other value the page reports (catalog probes, metapage
# roots, estimate-against-truth counts, AM properties, the pending-list drain,
# the snapshot holder, the page charges of fixture Q, the dead entries of
# fixture E) goes to the table fx through fact().  The summary stage writes all
# five, with the platform facts, to $SANDBOX/out/summary.txt.
#
# Failure and teardown.  Every psql call that fails ends the run with a
# non-zero status.  pg(), pgq() and pgopt() call die themselves.  A psql call
# inside $(...) runs in a subshell, where die can only end the subshell, so
# every such call is followed by its own || die.  Two pipelines carry psql
# output: the platform record in cluster, which is checked the same way, and
# grej's, whose statements are meant to fail, so grej first checks that
# fixture S exists and then matches the four error texts instead of trusting
# the pipeline's status.  Stage exit status is checked on top of
# that, and every stage name is checked before any stage runs, so a failed
# configure, make, make check or psql, or a misspelt stage, can never be
# reported as a pass.  An EXIT trap stops the server and the snapshot-holding
# session on every path out of the script, including a failure in the middle
# of a fixture and an interrupt, so no postmaster is left behind; the sandbox
# is kept for inspection unless the clean stage ran.  Every SQL stage starts
# the server if it is not up, so a selected re-run works after a default run
# has stopped the cluster; cluster starts it itself, and build, check, stop
# and clean never start one.
#
# Environment: WIKI_ROOT SRC SANDBOX PAGE PORT JOBS STATS_TARGET.  The script
# also clears three libpq variables a caller may have set, PGSERVICE,
# PGSERVICEFILE and PGHOSTADDR, because a service entry's values win over the
# exported PGHOST, PGPORT, PGDATABASE and PGUSER, and a host address sends a
# session over TCP instead of to the sandbox socket.  PGOPTIONS is never
# exported; every session gets its options passed explicitly.
#
# GUC apply scopes, from the pinned v17 guc_tables.c:
#   shared_buffers, port, listen_addresses, unix_socket_directories
#       -> PGC_POSTMASTER, restart.  Set once on the postmaster command line.
#   autovacuum -> PGC_SIGHUP, reload.  Set once on the postmaster command line.
#   default_statistics_target, enable_seqscan, enable_bitmapscan,
#   enable_indexscan, enable_indexonlyscan, enable_hashjoin, enable_mergejoin,
#   effective_cache_size,
#   random_page_cost, cpu_operator_cost,
#   max_parallel_workers_per_gather, max_parallel_workers,
#   min_parallel_table_scan_size, parallel_setup_cost, parallel_tuple_cost,
#   gin_pending_list_limit, statement_timeout, lock_timeout, application_name
#       -> PGC_USERSET, session or transaction scope, no reload and no restart.
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-17}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/bloatplan}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/query-planning/bloated-indexes-query-planner.md}"
PORT="${PORT:-55437}"
JOBS="${JOBS:-8}"
# ANALYZE samples 300 * default_statistics_target rows.  At 10000 that is
# 3,000,000 rows, more than any fixture holds, so every ANALYZE reads every row
# and the statistics, and with them every row estimate and cost, are the same
# on every run.  The setting does not enter the cost model.
STATS_TARGET="${STATS_TARGET:-10000}"

# The commit the page pins.  build and check refuse a checkout at any other
# commit, and every later stage refuses binaries built from any other.
PIN=786db8dcf168bd9df8f55047337525ac19118b1c
# MARKER is the file claim_sandbox() writes into a sandbox this script created;
# PINFILE is the file build writes beside the binaries it installed.
MARKER=.bloatplan-sandbox
PINFILE=.bloatplan-pin
SANDBOX_OK=0

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# resolve_sandbox: turn $SANDBOX into its real path, and refuse one that is not
# a directory directly inside this repository's .wiki-runtime/tmp once .. and
# symbolic links are resolved.  It runs before any stage, and every path below
# is derived from its result, so pg_ctl, pgrep and rm all see the resolved path.
resolve_sandbox() {
  local tmp parent base real
  mkdir -p "$WIKI_ROOT/.wiki-runtime/tmp" || die "cannot create $WIKI_ROOT/.wiki-runtime/tmp"
  tmp=$(cd "$WIKI_ROOT/.wiki-runtime/tmp" && pwd -P) || die "cannot resolve $WIKI_ROOT/.wiki-runtime/tmp"
  base=$(basename "$SANDBOX")
  case "$base" in ''|.|..|/) die "refusing sandbox $SANDBOX: it names no directory of its own" ;; esac
  if [ -e "$SANDBOX" ] || [ -L "$SANDBOX" ]; then
    real=$(cd "$SANDBOX" 2>/dev/null && pwd -P) || die "refusing sandbox $SANDBOX: it is not a directory"
  else
    parent=$(cd "$(dirname "$SANDBOX")" 2>/dev/null && pwd -P) \
      || die "refusing sandbox $SANDBOX: its parent directory does not exist"
    real="$parent/$base"
  fi
  [ "$(dirname "$real")" = "$tmp" ] || die "refusing sandbox $SANDBOX: $real is not directly inside $tmp"
  SANDBOX="$real"
}
resolve_sandbox

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; DATA="$SANDBOX/data"
SOCK="$SANDBOX/sock"; OUT="$SANDBOX/out"; BIN="$INST/bin"; DB=bloatplan
# libpq applies a service entry before it reads PGHOST, PGPORT, PGDATABASE and
# PGUSER, and a PGHOSTADDR sends the session over TCP to that address.  Either
# one inherited from the caller could point every fixture statement, the
# catalog forgery included, at a server nobody named, so both are cleared.
unset PGSERVICE PGSERVICEFILE PGHOSTADDR
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE="$DB" PGUSER=postgres

# check_pin: the build and the regression suites must come from exactly the
# commit the page records, with no tracked file changed.  Untracked files, such
# as a Finder .DS_Store, are not part of the build and are not checked.
check_pin() {
  local head dirty
  head=$(git -C "$SRC" rev-parse HEAD 2>/dev/null) || die "$SRC is not a git checkout"
  [ "$head" = "$PIN" ] || die "$SRC is at $head, not at the pinned $PIN"
  dirty=$(git -C "$SRC" status --porcelain --untracked-files=no 2>/dev/null) \
    || die "git status failed in $SRC"
  [ -z "$dirty" ] || die "$SRC has changed tracked files, so a build would not be the pin"
}

# check_install_pin: the binaries under $INST were built from $PIN.
check_install_pin() {
  local built
  built=$(cat "$INST/$PINFILE" 2>/dev/null) || die "$INST has no build record: run the build stage"
  [ "$built" = "$PIN" ] || die "$INST was built from $built, not the pinned $PIN: run clean, then build"
}

# claim_sandbox: create $SANDBOX, or accept one this script created before, and
# mark it as this script's.  An existing directory that is not empty and
# carries no mark is refused.
claim_sandbox() {
  if [ -e "$SANDBOX" ] && [ ! -f "$SANDBOX/$MARKER" ] \
     && [ -n "$(ls -A "$SANDBOX" 2>/dev/null)" ]; then
    die "refusing sandbox $SANDBOX: it exists, is not empty, and this script did not create it"
  fi
  mkdir -p "$SANDBOX" && : > "$SANDBOX/$MARKER" || die "cannot create $SANDBOX"
  SANDBOX_OK=1
}

# own_sandbox: the check stop and clean make before they touch anything.  It
# returns 1 when there is no sandbox at all, and refuses one without the mark.
own_sandbox() {
  [ -d "$SANDBOX" ] || { note "no sandbox at $SANDBOX"; return 1; }
  [ -f "$SANDBOX/$MARKER" ] || die "refusing: $SANDBOX carries no $MARKER, so this script did not create it"
  SANDBOX_OK=1
}

# -X ignores ~/.psqlrc so a stray file cannot change a result; ON_ERROR_STOP
# means no failed statement passes silently.  The three settings are
# PGC_USERSET, so passing them through libpq applies them at session scope.
#
# pg() and pgq() abort the whole run when psql fails, rather than returning a
# status to a caller that may not look at it.  Without that, only the LAST
# command of a stage function set the stage's exit status, so a failed fixture
# build or a failed UPDATE round in the middle of a stage was invisible: the
# stage returned 0, run_stage saw 0, and the run reported a pass over numbers
# that were never produced.  Every psql call in this script is essential in
# exactly that sense.  Three calls do not go through these helpers: pgerr(),
# for the grej stage, which expects its statements to fail and counts the
# errors itself, and so runs without ON_ERROR_STOP; the snapshot holder in
# hold_snapshot(), which runs in the background; and the two diagnostic blocks
# in stage_diag.  The last two carry the same -X, ON_ERROR_STOP and
# SESSION_OPTS.  The options are passed per call and never exported, so a
# PGOPTIONS in the caller's environment reaches no measurement session.
SESSION_OPTS="-c statement_timeout=20min -c lock_timeout=60s -c default_statistics_target=$STATS_TARGET"
CURRENT_STAGE=""
pg()    { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -v ON_ERROR_STOP=1 "$@" \
            || die "psql failed in stage ${CURRENT_STAGE:-<none>}: psql $*"; }
pgq()   { pg -At "$@"; }
# A psql invocation that has to override PGOPTIONS goes through this, so it
# aborts the run the same way pg() does.
pgopt() { local o="$1"; shift; PGOPTIONS="$SESSION_OPTS $o" "$BIN/psql" -X -q -v ON_ERROR_STOP=1 "$@" \
            || die "psql failed in stage ${CURRENT_STAGE:-<none>} with PGOPTIONS $o: psql $*"; }
# grej sends four statements that are meant to fail, so it runs without
# ON_ERROR_STOP and keeps the error text.
pgerr() { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X "$@"; }

running() { [ -x "$BIN/pg_ctl" ] && "$BIN/pg_ctl" -D "$DATA" status >/dev/null 2>&1; }
stop_server() { running && "$BIN/pg_ctl" -D "$DATA" -m fast -w stop >/dev/null 2>&1; return 0; }

# Teardown on every path out of the script, not only on the stop stage: a die
# in the middle of a fixture, a failed psql, or an interrupt all land here, so
# no measurement postmaster and no snapshot holder survives the run.  The
# sandbox is left alone; only the clean stage deletes it.  The trap stops a
# server only in a sandbox this run claimed or verified as its own
# (SANDBOX_OK=1), so an exit caused by a refused sandbox stops nothing.
HOLDER_PID=""
on_exit() {
  local st=$?
  trap - EXIT INT TERM
  if [ "$SANDBOX_OK" = 1 ] && [ "$st" -ne 0 ] && running; then
    printf '!! exit status %s with the server still up; stopping it\n' "$st" >&2
  fi
  # Stopping the server first drops the holder session's connection, so its
  # pg_sleep ends and the background subshell exits on its own; the kill is only
  # there for the case where the shutdown could not reach it.
  [ "$SANDBOX_OK" = 1 ] && stop_server
  [ -n "$HOLDER_PID" ] && kill "$HOLDER_PID" 2>/dev/null
  wait 2>/dev/null
  if [ "$st" -ne 0 ]; then
    if [ "$SANDBOX_OK" = 1 ] && [ -d "$SANDBOX" ]; then
      printf '!! failed with status %s; %s was kept\n' "$st" "$SANDBOX" >&2
    else
      printf '!! failed with status %s\n' "$st" >&2
    fi
  fi
  exit "$st"
}
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

# Any stage that talks SQL needs a server.  stage_cluster is idempotent, so
# calling it again after a default run's stop stage just restarts the same data
# directory with the same recorded rows still in it.
need_server() {
  running && return 0
  [ -x "$BIN/pg_ctl" ] || die "no built server at $BIN: run the build stage first"
  stage_cluster || die "could not start the measurement server"
}

stage_build() {
  say "build: configure the pinned checkout out of tree, install core and three contrib modules"
  check_pin
  mkdir -p "$BUILD" "$OUT" "$SOCK" || die "cannot create the build directories under $SANDBOX"
  # The build record is written last, so it, not a postgres binary, is what
  # marks a finished build.  A build interrupted after the core install but
  # before the contrib installs or the record leaves binaries behind with no
  # record; that install is incomplete, so it is removed and built again.
  if [ -f "$INST/$PINFILE" ]; then
    check_install_pin
    [ -x "$BIN/postgres" ] || die "$INST has a build record but no postgres binary: run clean, then build"
    note "already built from $PIN: $("$BIN/postgres" --version)"; return 0
  fi
  if [ -e "$INST" ]; then
    note "removing the incomplete install at $INST, which has no build record"
    rm -rf "$INST" || die "cannot remove the incomplete install at $INST"
  fi
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --without-icu --without-readline \
      > "$OUT/configure.log" 2>&1 ) || die "configure failed, see $OUT/configure.log"
  ( cd "$BUILD" && make -s -j "$JOBS" > "$OUT/make.log" 2>&1 \
      && make -s install > "$OUT/install.log" 2>&1 \
      && for m in pgstattuple pageinspect btree_gin; do
           make -s -C contrib/$m install >> "$OUT/install.log" 2>&1 || exit 1
         done ) || die "make failed, see $OUT/make.log"
  printf '%s\n' "$PIN" > "$INST/$PINFILE" || die "cannot record the build's pin in $INST"
  note "$("$BIN/postgres" --version), built from $PIN"
}

stage_check() {
  say "check: core regression suite, then the three contrib suites, on the built tree"
  check_pin
  check_install_pin
  # A failing suite is a failed stage.  Without the || die the subshell's status
  # is discarded and a run with four broken suites reports a pass.
  ( cd "$BUILD" && make -s check > "$OUT/check.log" 2>&1 ) \
    || die "make check failed, see $OUT/check.log"
  note "core: $(grep -E 'tests passed|tests failed|failed' "$OUT/check.log" | tail -1)"
  local m
  for m in pgstattuple pageinspect btree_gin; do
    ( cd "$BUILD" && make -s -C contrib/$m check > "$OUT/check_$m.log" 2>&1 ) \
      || die "contrib/$m check failed, see $OUT/check_$m.log"
    note "$m: $(grep -E 'tests passed|tests failed|failed' "$OUT/check_$m.log" | tail -1)"
  done
}

stage_cluster() {
  say "cluster: initdb, start with autovacuum off, install contrib and the recording helpers"
  check_install_pin
  mkdir -p "$OUT" "$SOCK" || die "cannot create $OUT and $SOCK"
  [ -f "$DATA/PG_VERSION" ] || "$BIN/initdb" -D "$DATA" -U postgres --locale=C --encoding=UTF8 \
    > "$OUT/initdb.log" 2>&1 || die "initdb failed, see $OUT/initdb.log"
  running || "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w \
    -o "-p $PORT -k '$SOCK' -c listen_addresses='' -c autovacuum=off -c shared_buffers=256MB" start \
    >/dev/null 2>&1 || die "server did not start, see $OUT/server.log"
  local have
  have=$(PGDATABASE=postgres pgq -c "SELECT /* wiki_bloatplan_dbcheck */ count(*) FROM pg_database WHERE datname = '$DB'") \
    || die "could not read pg_database"
  [ "$have" = "1" ] || PGDATABASE=postgres pg -q -c "CREATE /* wiki_bloatplan_fixture */ DATABASE $DB"
  pg -q <<'SQL' || die "could not install the helpers"
CREATE /* wiki_bloatplan_fixture */ EXTENSION IF NOT EXISTS pgstattuple;
CREATE /* wiki_bloatplan_fixture */ EXTENSION IF NOT EXISTS pageinspect;
CREATE /* wiki_bloatplan_fixture */ EXTENSION IF NOT EXISTS btree_gin;
CREATE /* wiki_bloatplan_fixture */ TABLE IF NOT EXISTS r
  (seq serial, label text, top_node text, startup numeric, total numeric, plan_rows numeric, plan jsonb);
CREATE /* wiki_bloatplan_fixture */ TABLE IF NOT EXISTS ix
  (seq serial, label text, idx text, blocks bigint, tree_level int, fastlevel int, leaf_pages bigint,
   internal_pages bigint, empty_pages bigint, deleted_pages bigint, density float8, frag float8);
CREATE /* wiki_bloatplan_fixture */ TABLE IF NOT EXISTS gx
  (seq serial, label text, idx text, blocks bigint, pending_pages bigint, n_total bigint,
   n_entry bigint, n_data bigint, n_entries bigint);
CREATE /* wiki_bloatplan_fixture */ TABLE IF NOT EXISTS pr
  (seq serial, label text, pages float8, tuples float8, fastlevel float8, predicted text, observed text);
CREATE /* wiki_bloatplan_fixture */ TABLE IF NOT EXISTS fx
  (seq serial, label text, value text);

-- fact(): keep one value the page reports that is not an index row, a GIN
-- row, a prediction or a plan.  A later call under the same label replaces
-- the earlier one, so a re-run of a stage replaces its own facts.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION fact(p_label text, p_value text)
RETURNS text LANGUAGE plpgsql AS $fn$
BEGIN
  DELETE /* wiki_bloatplan_record */ FROM fx WHERE label = p_label;
  INSERT /* wiki_bloatplan_record */ INTO fx(label, value) VALUES (p_label, p_value);
  RETURN p_label || ': ' || p_value;
END $fn$;

-- xp(): EXPLAIN one statement and keep the plan.  p_hide names an index that
-- is dropped inside a subtransaction which is always rolled back, so two
-- indexes on one table can each be priced alone on literally identical
-- statistics.  p_set entries are 'name=value' pairs of PGC_USERSET settings,
-- applied transaction-locally, so they end with the calling statement.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION xp
  (p_label text, p_q text, p_hide text DEFAULT NULL, VARIADIC p_set text[] DEFAULT '{}'::text[])
RETURNS text LANGUAGE plpgsql AS $fn$
DECLARE j json; s text;
BEGIN
  FOREACH s IN ARRAY p_set LOOP
    PERFORM /* wiki_bloatplan_explain_setting */ set_config(split_part(s, '=', 1), split_part(s, '=', 2), true);
  END LOOP;
  IF p_hide IS NULL THEN
    EXECUTE 'EXPLAIN /* wiki_bloatplan_explain */ (FORMAT JSON) ' || p_q INTO j;
  ELSE
    BEGIN
      EXECUTE 'DROP /* wiki_bloatplan_fixture_rolled_back */ INDEX ' || p_hide;
      EXECUTE 'EXPLAIN /* wiki_bloatplan_explain */ (FORMAT JSON) ' || p_q INTO j;
      RAISE EXCEPTION 'undo' USING ERRCODE = 'P0099';
    EXCEPTION WHEN SQLSTATE 'P0099' THEN NULL;
    END;
  END IF;
  DELETE /* wiki_bloatplan_record */ FROM r WHERE label = p_label;
  INSERT /* wiki_bloatplan_record */ INTO r(label, top_node, startup, total, plan_rows, plan)
  VALUES (p_label, j->0->'Plan'->>'Node Type', (j->0->'Plan'->>'Startup Cost')::numeric,
          (j->0->'Plan'->>'Total Cost')::numeric, (j->0->'Plan'->>'Plan Rows')::numeric, j::jsonb);
  RETURN format('%s | %s', p_label, nodes(p_label));
END $fn$;

-- nodes(): one line per recorded plan, every node with its index, costs, rows,
-- width, workers, index condition, recheck condition and filter.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION nodes(p_label text)
RETURNS text LANGUAGE sql AS $fn$
  SELECT /* wiki_bloatplan_record */ string_agg(
           (n->>'Node Type')
           || coalesce(' ' || (n->>'Index Name'), '')
           || ' ' || (n->>'Startup Cost') || '..' || (n->>'Total Cost')
           || ' rows=' || (n->>'Plan Rows')
           || ' width=' || (n->>'Plan Width')
           || coalesce(' workers=' || (n->>'Workers Planned'), '')
           || coalesce(' cond=' || (n->>'Index Cond'), '')
           || coalesce(' recheck=' || (n->>'Recheck Cond'), '')
           || coalesce(' filter=' || (n->>'Filter'), ''), ' ; ')
    FROM r, LATERAL jsonb_path_query(plan, 'strict $.** ? (exists(@."Node Type"))') AS n
   WHERE label = p_label
$fn$;

-- idxcost(): the total cost of the one Bitmap Index Scan node in a recorded
-- plan.  That node carries the index's own amcostestimate total and nothing
-- from the heap, so two plans at random_page_cost 4 and 1 differ there by
-- exactly 3.0 per index page the planner charged.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION idxcost(p_label text)
RETURNS numeric LANGUAGE sql AS $fn$
  SELECT /* wiki_bloatplan_record */ (jsonb_path_query_first(plan,
           'strict $.** ? (@."Node Type" == "Bitmap Index Scan")') ->> 'Total Cost')::numeric
    FROM r WHERE label = p_label
$fn$;

-- ixstat(): block count, pgstatindex figures and the fast-root level of one B-tree.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION ixstat(p_label text, p_idx regclass)
RETURNS text LANGUAGE plpgsql AS $fn$
DECLARE s record; m record; b bigint;
BEGIN
  SELECT /* wiki_bloatplan_record */ * INTO s FROM pgstatindex(p_idx);
  SELECT /* wiki_bloatplan_record */ * INTO m FROM bt_metap(p_idx::text);
  b := pg_relation_size(p_idx) / current_setting('block_size')::int;
  DELETE /* wiki_bloatplan_record */ FROM ix WHERE label = p_label;
  INSERT /* wiki_bloatplan_record */ INTO ix(label, idx, blocks, tree_level, fastlevel, leaf_pages, internal_pages, empty_pages,
                 deleted_pages, density, frag)
  VALUES (p_label, p_idx::text, b, s.tree_level, m.fastlevel, s.leaf_pages, s.internal_pages,
          s.empty_pages, s.deleted_pages, s.avg_leaf_density, s.leaf_fragmentation);
  RETURN format('%s | %s blocks=%s tree_level=%s fastlevel=%s leaf=%s internal=%s deleted=%s density=%s frag=%s',
                p_label, p_idx, b, s.tree_level, m.fastlevel, s.leaf_pages, s.internal_pages,
                s.deleted_pages, s.avg_leaf_density, s.leaf_fragmentation);
END $fn$;

-- ginstat(): block count, pending pages and the metapage counters of one GIN index.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION ginstat(p_label text, p_idx regclass)
RETURNS text LANGUAGE plpgsql AS $fn$
DECLARE m record; g record; b bigint;
BEGIN
  SELECT /* wiki_bloatplan_record */ * INTO m FROM gin_metapage_info(get_raw_page(p_idx::text, 0));
  SELECT /* wiki_bloatplan_record */ * INTO g FROM pgstatginindex(p_idx);
  b := pg_relation_size(p_idx) / current_setting('block_size')::int;
  DELETE /* wiki_bloatplan_record */ FROM gx WHERE label = p_label;
  INSERT /* wiki_bloatplan_record */ INTO gx(label, idx, blocks, pending_pages, n_total, n_entry, n_data, n_entries)
  VALUES (p_label, p_idx::text, b, g.pending_pages, m.n_total_pages, m.n_entry_pages, m.n_data_pages, m.n_entries);
  RETURN format('%s | %s blocks=%s pending=%s meta(total,entry,data,entries)=(%s,%s,%s,%s)',
                p_label, p_idx, b, g.pending_pages, m.n_total_pages, m.n_entry_pages, m.n_data_pages, m.n_entries);
END $fn$;

-- predict(): the whole-index-scan cost of one B-tree, recomputed in float8 in
-- the planner's own order of operations from three inputs only: the index's
-- live block count, the planner's row estimate for the table, and the
-- fast-root level.  It is compared with the EXPLAIN total recorded under
-- p_label.  The index-only scan pays no heap cost because every fixture it is
-- used on is 100% all-visible.  The row estimate is the one
-- table_block_relation_estimate_size() makes: reltuples scaled from relpages
-- to the live heap block count, rounded with rint(), which is what SQL
-- round() does to a float8.  On every fixture predict() is used on, the table
-- was vacuumed just before, so the scaling changes nothing.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION predict(p_label text, p_idx regclass, p_tbl regclass)
RETURNS text LANGUAGE plpgsql AS $fn$
DECLARE
  pages float8 := pg_relation_size(p_idx) / current_setting('block_size')::int;
  relp float8 := (SELECT /* wiki_bloatplan_record */ relpages FROM pg_class WHERE oid = p_tbl);
  relt float8 := (SELECT /* wiki_bloatplan_record */ reltuples FROM pg_class WHERE oid = p_tbl);
  curp float8 := pg_relation_size(p_tbl) / current_setting('block_size')::int;
  tuples float8;
  fl float8 := (SELECT /* wiki_bloatplan_record */ fastlevel FROM bt_metap(p_idx::text));
  rpc float8 := current_setting('random_page_cost')::float8;
  citc float8 := current_setting('cpu_index_tuple_cost')::float8;
  coc float8 := current_setting('cpu_operator_cost')::float8;
  ctc float8 := current_setting('cpu_tuple_cost')::float8;
  generic float8; d1 float8; d2 float8; startup float8; idx_total float8; total float8; obs text;
BEGIN
  tuples := CASE WHEN relp > 0 AND relt >= 0 THEN round(relt / relp * curp) ELSE relt END;
  generic := (pages * rpc) + tuples * (citc + coc * 1);   -- genericcostestimate, one index qual
  d1 := ceil(ln(tuples) / ln(2.0::float8)) * coc;         -- btcostestimate, log2(N) comparisons
  d2 := (fl + 1) * 50.0 * coc;                            -- btcostestimate, per-page descent charge
  startup := d1 + d2;
  idx_total := generic + d1 + d2;
  total := startup + ((idx_total - startup) + ctc * tuples);   -- cost_index
  SELECT /* wiki_bloatplan_record */ to_char(r.total, 'FM999999990.00') INTO obs FROM r WHERE label = p_label;
  DELETE /* wiki_bloatplan_record */ FROM pr WHERE label = p_label;
  INSERT /* wiki_bloatplan_record */ INTO pr(label, pages, tuples, fastlevel, predicted, observed)
  VALUES (p_label, pages, tuples, fl, to_char(total, 'FM999999990.00'), obs);
  RETURN format('%s | pages=%s tuples=%s fastlevel=%s predicted=%s observed=%s',
                p_label, pages, tuples, fl, to_char(total, 'FM999999990.00'), obs);
END $fn$;
SQL
  local head built
  head=$(git -C "$SRC" rev-parse HEAD 2>/dev/null) || die "cannot read HEAD from $SRC"
  built=$(cat "$INST/$PINFILE") || die "cannot read the build record in $INST"
  {
    date -u '+run started %Y-%m-%dT%H:%M:%SZ'
    uname -sm
    echo "bash $BASH_VERSION"
    pgq -c "SELECT /* wiki_bloatplan_version */ version()"
    echo "pin $PIN; install built from $built; checkout HEAD $head"
    "$BIN/pg_controldata" "$DATA" | grep -E 'Database block size|Maximum data alignment' \
      || die "pg_controldata did not report the block size and the maximum alignment"
    pgq -c "SELECT /* wiki_bloatplan_settings */ name || ' = ' || setting FROM pg_settings WHERE name IN ('autovacuum','shared_buffers','random_page_cost','seq_page_cost','cpu_tuple_cost','cpu_index_tuple_cost','cpu_operator_cost','effective_cache_size','min_parallel_index_scan_size','default_statistics_target','gin_pending_list_limit') ORDER BY 1"
  } | tee "$OUT/platform.txt" || die "could not record the platform facts"
}

# Settings that force an index or index-only scan so that one named index is
# priced, not chosen.  Both are PGC_USERSET and last for one statement.
IOS="'enable_seqscan=off','enable_bitmapscan=off'"
PAR="'max_parallel_workers_per_gather=8','max_parallel_workers=8','min_parallel_table_scan_size=0','enable_seqscan=off','enable_bitmapscan=off'"
PAR0="$PAR,'parallel_setup_cost=0','parallel_tuple_cost=0'"
# A nested loop with the fixture index on the inner side, so that the scan is
# repeated and index_pages_fetched() prices the index pages through the cache model.
NL="'enable_hashjoin=off','enable_mergejoin=off','max_parallel_workers_per_gather=0'"
# A bitmap scan, so that the Bitmap Index Scan node carries the index's own
# cost with no heap component (fixture Q).  All three are PGC_USERSET.
BMP="'enable_seqscan=off','enable_indexscan=off','enable_indexonlyscan=off'"

stage_fa() {
  say "fa: fixture A, 1,000,000 rows, a default index against a fillfactor = 10 twin"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS a_dense, a_sparse, a_outer;
CREATE /* wiki_bloatplan_fixture */ TABLE a_outer  (id int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO a_outer SELECT g * 20 FROM generate_series(1, 50000) g;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) a_outer;
CREATE /* wiki_bloatplan_fixture */ TABLE a_dense  (id int NOT NULL, pad text NOT NULL);
CREATE /* wiki_bloatplan_fixture */ TABLE a_sparse (id int NOT NULL, pad text NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO a_dense  SELECT g, repeat('x', 24) FROM generate_series(1, 1000000) g;
INSERT /* wiki_bloatplan_fixture */ INTO a_sparse SELECT g, repeat('x', 24) FROM generate_series(1, 1000000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX a_dense_idx  ON a_dense  (id);
CREATE /* wiki_bloatplan_fixture */ INDEX a_sparse_idx ON a_sparse (id) WITH (fillfactor = 10);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) a_dense;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) a_sparse;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_heap */ 'heap ' || relname || ' pages=' || relpages || ' allvisible=' || relallvisible FROM pg_class WHERE relname IN ('a_dense','a_sparse') ORDER BY 1;
SELECT /* wiki_bloatplan_record */ ixstat('A dense', 'a_dense_idx');
SELECT /* wiki_bloatplan_record */ ixstat('A sparse', 'a_sparse_idx');
SELECT /* wiki_bloatplan_record */ xp('A dense full',   'SELECT id FROM a_dense  WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('A sparse full',  'SELECT id FROM a_sparse WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('A dense point',  'SELECT id FROM a_dense  WHERE id = 42', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('A sparse point', 'SELECT id FROM a_sparse WHERE id = 42', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('A dense 10pct',  'SELECT id FROM a_dense  WHERE id BETWEEN 1 AND 100000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('A sparse 10pct', 'SELECT id FROM a_sparse WHERE id BETWEEN 1 AND 100000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('A dense flip25',  'SELECT pad FROM a_dense  WHERE id BETWEEN 1 AND 250000');
SELECT /* wiki_bloatplan_record */ xp('A sparse flip25', 'SELECT pad FROM a_sparse WHERE id BETWEEN 1 AND 250000');
SELECT /* wiki_bloatplan_record */ xp('A dense flip12',  'SELECT pad FROM a_dense  WHERE id BETWEEN 1 AND 120000');
SELECT /* wiki_bloatplan_record */ xp('A sparse flip12', 'SELECT pad FROM a_sparse WHERE id BETWEEN 1 AND 120000');
SELECT /* wiki_bloatplan_record */ xp('A dense parallel',  'SELECT count(id) FROM a_dense  WHERE id > 0', NULL, $PAR);
SELECT /* wiki_bloatplan_record */ xp('A sparse parallel', 'SELECT count(id) FROM a_sparse WHERE id > 0', NULL, $PAR);
SELECT /* wiki_bloatplan_record */ xp('A dense parallel 50pct',  'SELECT count(id) FROM a_dense  WHERE id BETWEEN 1 AND 500000', NULL, $PAR);
SELECT /* wiki_bloatplan_record */ xp('A dense parallel 20pct',  'SELECT count(id) FROM a_dense  WHERE id BETWEEN 1 AND 200000', NULL, $PAR);
SELECT /* wiki_bloatplan_record */ xp('A sparse parallel 50pct', 'SELECT count(id) FROM a_sparse WHERE id BETWEEN 1 AND 500000', NULL, $PAR);
SELECT /* wiki_bloatplan_record */ xp('A sparse parallel 20pct', 'SELECT count(id) FROM a_sparse WHERE id BETWEEN 1 AND 200000', NULL, $PAR);
SELECT /* wiki_bloatplan_record */ xp('A dense par0 100pct',  'SELECT count(id) FROM a_dense  WHERE id > 0', NULL, $PAR0);
SELECT /* wiki_bloatplan_record */ xp('A dense par0 50pct',   'SELECT count(id) FROM a_dense  WHERE id BETWEEN 1 AND 500000', NULL, $PAR0);
SELECT /* wiki_bloatplan_record */ xp('A dense par0 20pct',   'SELECT count(id) FROM a_dense  WHERE id BETWEEN 1 AND 200000', NULL, $PAR0);
SELECT /* wiki_bloatplan_record */ xp('A sparse par0 100pct', 'SELECT count(id) FROM a_sparse WHERE id > 0', NULL, $PAR0);
SELECT /* wiki_bloatplan_record */ xp('A sparse par0 50pct',  'SELECT count(id) FROM a_sparse WHERE id BETWEEN 1 AND 500000', NULL, $PAR0);
SELECT /* wiki_bloatplan_record */ xp('A sparse par0 20pct',  'SELECT count(id) FROM a_sparse WHERE id BETWEEN 1 AND 200000', NULL, $PAR0);
SELECT /* wiki_bloatplan_record */ xp('A dense nestloop',        'SELECT count(*) FROM a_outer o JOIN a_dense  d ON d.id = o.id', NULL, $NL);
SELECT /* wiki_bloatplan_record */ xp('A sparse nestloop',       'SELECT count(*) FROM a_outer o JOIN a_sparse d ON d.id = o.id', NULL, $NL);
SELECT /* wiki_bloatplan_record */ xp('A dense nestloop ecs64',  'SELECT count(*) FROM a_outer o JOIN a_dense  d ON d.id = o.id', NULL, $NL, 'effective_cache_size=64MB');
SELECT /* wiki_bloatplan_record */ xp('A sparse nestloop ecs64', 'SELECT count(*) FROM a_outer o JOIN a_sparse d ON d.id = o.id', NULL, $NL, 'effective_cache_size=64MB');
SQL
}

stage_fb() {
  say "fb: fixtures B (scattered 90% delete) and M (contiguous 90% delete)"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS b_scat, m_cont;
CREATE /* wiki_bloatplan_fixture */ TABLE b_scat (id int NOT NULL, pad text NOT NULL);
CREATE /* wiki_bloatplan_fixture */ TABLE m_cont (id int NOT NULL, pad text NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO b_scat SELECT g, repeat('x', 24) FROM generate_series(1, 1000000) g;
INSERT /* wiki_bloatplan_fixture */ INTO m_cont SELECT g, repeat('x', 24) FROM generate_series(1, 1000000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX b_scat_idx ON b_scat (id);
CREATE /* wiki_bloatplan_fixture */ INDEX m_cont_idx ON m_cont (id);
DELETE /* wiki_bloatplan_fixture */ FROM b_scat WHERE id % 10 <> 0;
DELETE /* wiki_bloatplan_fixture */ FROM m_cont WHERE id > 100000;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) b_scat;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) b_scat;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) m_cont;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) m_cont;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('B before', 'b_scat_idx');
SELECT /* wiki_bloatplan_record */ ixstat('M before', 'm_cont_idx');
SELECT /* wiki_bloatplan_record */ xp('B before full',  'SELECT id FROM b_scat WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('M before full',  'SELECT id FROM m_cont WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('B before point', 'SELECT id FROM b_scat WHERE id = 50000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('M before point', 'SELECT id FROM m_cont WHERE id = 50000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ predict('B before full', 'b_scat_idx', 'b_scat');
SELECT /* wiki_bloatplan_record */ predict('M before full', 'm_cont_idx', 'm_cont');
SQL
  pg -q -c "VACUUM /* wiki_bloatplan_fixture */ m_cont"
  pgq -c "SELECT /* wiki_bloatplan_record */ ixstat('M third vacuum', 'm_cont_idx')"
  pg -q -c "REINDEX /* wiki_bloatplan_fixture */ INDEX b_scat_idx" -c "REINDEX /* wiki_bloatplan_fixture */ INDEX m_cont_idx"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('B after', 'b_scat_idx');
SELECT /* wiki_bloatplan_record */ ixstat('M after', 'm_cont_idx');
SELECT /* wiki_bloatplan_record */ xp('B after full',  'SELECT id FROM b_scat WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('M after full',  'SELECT id FROM m_cont WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('B after point', 'SELECT id FROM b_scat WHERE id = 50000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ predict('B after full', 'b_scat_idx', 'b_scat');
SQL
}

stage_ff() {
  say "ff: fixture F, the fast root moves down without a rebuild, and pages outnumber rows"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS f_root;
CREATE /* wiki_bloatplan_fixture */ TABLE f_root (id int NOT NULL, pad text NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO f_root SELECT g, repeat('x', 24) FROM generate_series(1, 1000000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX f_root_idx ON f_root (id);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) f_root;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('F built', 'f_root_idx');
SELECT /* wiki_bloatplan_record */ xp('F built point', 'SELECT id FROM f_root WHERE id = 999950', NULL, $IOS);
SQL
  pg -q <<'SQL'
DELETE /* wiki_bloatplan_fixture */ FROM f_root WHERE id <= 999000;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) f_root;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) f_root;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('F deleted', 'f_root_idx');
SELECT /* wiki_bloatplan_record */ fact('F deleted metapage', 'root=' || root || ' level=' || level || ' fastroot=' || fastroot || ' fastlevel=' || fastlevel) FROM bt_metap('f_root_idx');
SELECT /* wiki_bloatplan_record */ xp('F deleted point', 'SELECT id FROM f_root WHERE id = 999950', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('F deleted full',  'SELECT id FROM f_root WHERE id > 0', NULL, $IOS);
SQL
  pg -q -c "REINDEX /* wiki_bloatplan_fixture */ INDEX f_root_idx"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('F rebuilt', 'f_root_idx');
SELECT /* wiki_bloatplan_record */ xp('F rebuilt point', 'SELECT id FROM f_root WHERE id = 999950', NULL, $IOS);
SQL
  # The other end of the same guard.  genericcostestimate() prorates pages only
  # when index->pages > 1 AND index->tuples > 1; otherwise the estimate is a
  # flat one page.  For a non-partial index tuples is the TABLE's row estimate,
  # so a table down to one surviving row prices a scan of an index of any size
  # at a single page.  f_one keeps its 200,000-row index and loses all but one
  # row.
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS f_one;
CREATE /* wiki_bloatplan_fixture */ TABLE f_one (id int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO f_one SELECT g FROM generate_series(1, 200000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX f_one_idx ON f_one (id);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) f_one;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('F one before', 'f_one_idx');
SELECT /* wiki_bloatplan_record */ fact('F one catalog before', 'relpages=' || relpages || ' reltuples=' || reltuples) FROM pg_class WHERE relname = 'f_one';
SELECT /* wiki_bloatplan_record */ xp('F one before full', 'SELECT id FROM f_one WHERE id > 0', NULL, $IOS);
SQL
  pg -q <<'SQL'
DELETE /* wiki_bloatplan_fixture */ FROM f_one WHERE id < 200000;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) f_one;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) f_one;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('F one after', 'f_one_idx');
SELECT /* wiki_bloatplan_record */ fact('F one catalog after', 'relpages=' || relpages || ' reltuples=' || reltuples) FROM pg_class WHERE relname = 'f_one';
SELECT /* wiki_bloatplan_record */ xp('F one after full',  'SELECT id FROM f_one WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('F one after point', 'SELECT id FROM f_one WHERE id = 200000', NULL, $IOS);
SQL
}

stage_fg() {
  say "fg: fixture G, a fillfactor = 100 build against seeded random-order inserts"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS g_seq, g_frag;
CREATE /* wiki_bloatplan_fixture */ TABLE g_seq  (id int NOT NULL);
CREATE /* wiki_bloatplan_fixture */ TABLE g_frag (id int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO g_seq SELECT g FROM generate_series(1, 300000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX g_seq_idx  ON g_seq  (id) WITH (fillfactor = 100);
CREATE /* wiki_bloatplan_fixture */ INDEX g_frag_idx ON g_frag (id);
SELECT /* wiki_bloatplan_fixture */ setseed(0.42);
INSERT /* wiki_bloatplan_fixture */ INTO g_frag SELECT g FROM generate_series(1, 300000) g ORDER BY random();
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) g_seq;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) g_frag;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('G seq', 'g_seq_idx');
SELECT /* wiki_bloatplan_record */ ixstat('G frag', 'g_frag_idx');
SELECT /* wiki_bloatplan_record */ xp('G seq full',  'SELECT id FROM g_seq  WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('G frag full', 'SELECT id FROM g_frag WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ predict('G seq full',  'g_seq_idx',  'g_seq');
SELECT /* wiki_bloatplan_record */ predict('G frag full', 'g_frag_idx', 'g_frag');
SQL
}

stage_fh() {
  say "fh: fixture H, 50,000 rows, a one-level tree against a two-level tree"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS h_l1, h_l2;
CREATE /* wiki_bloatplan_fixture */ TABLE h_l1 (id int NOT NULL, pad text NOT NULL);
CREATE /* wiki_bloatplan_fixture */ TABLE h_l2 (id int NOT NULL, pad text NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO h_l1 SELECT g, repeat('x', 24) FROM generate_series(1, 50000) g;
INSERT /* wiki_bloatplan_fixture */ INTO h_l2 SELECT g, repeat('x', 24) FROM generate_series(1, 50000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX h_l1_idx ON h_l1 (id);
CREATE /* wiki_bloatplan_fixture */ INDEX h_l2_idx ON h_l2 (id) WITH (fillfactor = 10);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) h_l1;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) h_l2;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('H l1', 'h_l1_idx');
SELECT /* wiki_bloatplan_record */ ixstat('H l2', 'h_l2_idx');
SELECT /* wiki_bloatplan_record */ xp('H l1 point', 'SELECT * FROM h_l1 WHERE id = 25000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('H l2 point', 'SELECT * FROM h_l2 WHERE id = 25000', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('H l1 point opcost1', 'SELECT * FROM h_l1 WHERE id = 25000', NULL, 'cpu_operator_cost=1', $IOS);
SELECT /* wiki_bloatplan_record */ xp('H l2 point opcost1', 'SELECT * FROM h_l2 WHERE id = 25000', NULL, 'cpu_operator_cost=1', $IOS);
SQL
}

stage_fi() {
  say "fi: fixture I, 2,000 rows, the v17 ScalarArrayOp descent clamp"
  # The two tables are analyzed but never vacuumed, so relallvisible is 0 and
  # the index-only scan is charged heap access exactly as a plain index scan
  # would be.  With the heap in key order that is one random heap page (4.0)
  # at every array length, plus cpu_tuple_cost for each row, the same for both
  # indexes.
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS i_small, i_big;
CREATE /* wiki_bloatplan_fixture */ TABLE i_small (id int NOT NULL);
CREATE /* wiki_bloatplan_fixture */ TABLE i_big   (id int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO i_small SELECT g FROM generate_series(1, 2000) g;
INSERT /* wiki_bloatplan_fixture */ INTO i_big   SELECT g FROM generate_series(1, 2000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX i_small_idx ON i_small (id);
CREATE /* wiki_bloatplan_fixture */ INDEX i_big_idx   ON i_big   (id) WITH (fillfactor = 10);
ANALYZE /* wiki_bloatplan_fixture */ i_small;
ANALYZE /* wiki_bloatplan_fixture */ i_big;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ixstat('I small', 'i_small_idx');
SELECT /* wiki_bloatplan_record */ ixstat('I big', 'i_big_idx');
SELECT /* wiki_bloatplan_heap */ 'heap ' || relname || ' pages=' || relpages || ' allvisible=' || relallvisible FROM pg_class WHERE relname IN ('i_small','i_big') ORDER BY 1;
SQL
  local n arr
  for n in 1 2 3 4 6 10; do
    arr=$(pgq -c "SELECT /* wiki_bloatplan_array */ '{' || string_agg((g * 150)::text, ',') || '}' FROM generate_series(1, $n) g") \
      || die "could not build the $n-element array"
    pgq <<SQL
SELECT /* wiki_bloatplan_record */ xp('I small saop $n', \$q\$SELECT * FROM i_small WHERE id = ANY ('$arr'::int[])\$q\$, NULL, 'cpu_operator_cost=1', $IOS);
SELECT /* wiki_bloatplan_record */ xp('I big saop $n',   \$q\$SELECT * FROM i_big   WHERE id = ANY ('$arr'::int[])\$q\$, NULL, 'cpu_operator_cost=1', $IOS);
SQL
  done
}

stage_fn() {
  say "fn: fixture N, 1,000,000 rows over 100 keys, deduplicate_items on against off"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS n_t;
CREATE /* wiki_bloatplan_fixture */ TABLE n_t (k int NOT NULL, pad text NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO n_t SELECT g % 100, repeat('x', 24) FROM generate_series(1, 1000000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX n_on  ON n_t (k);
CREATE /* wiki_bloatplan_fixture */ INDEX n_off ON n_t (k) WITH (deduplicate_items = off);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) n_t;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('N on', 'n_on');
SELECT /* wiki_bloatplan_record */ ixstat('N off', 'n_off');
SELECT /* wiki_bloatplan_record */ xp('N on k=5',  'SELECT * FROM n_t WHERE k = 5', 'n_off', $IOS);
SELECT /* wiki_bloatplan_record */ xp('N off k=5', 'SELECT * FROM n_t WHERE k = 5', 'n_on',  $IOS);
SELECT /* wiki_bloatplan_record */ xp('N on k>0',  'SELECT * FROM n_t WHERE k > 0', 'n_off', $IOS);
SELECT /* wiki_bloatplan_record */ xp('N off k>0', 'SELECT * FROM n_t WHERE k > 0', 'n_on',  $IOS);
SQL
}

stage_fstale() {
  say "fstale: a forged pg_class.relpages on a plain index, a forged reltuples on a partial one"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS b_stale, b_part;
CREATE /* wiki_bloatplan_fixture */ TABLE b_stale (id int NOT NULL);
CREATE /* wiki_bloatplan_fixture */ TABLE b_part  (id int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO b_stale SELECT g FROM generate_series(1, 200000) g;
INSERT /* wiki_bloatplan_fixture */ INTO b_part  SELECT g FROM generate_series(1, 200000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX b_stale_idx ON b_stale (id) WITH (fillfactor = 10);
CREATE /* wiki_bloatplan_fixture */ INDEX b_part_idx  ON b_part  (id) WITH (fillfactor = 10) WHERE id > 0;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) b_stale;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) b_part;
SQL
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('stale plain', 'b_stale_idx');
SELECT /* wiki_bloatplan_record */ ixstat('stale partial', 'b_part_idx');
SELECT /* wiki_bloatplan_record */ xp('stale plain honest',   'SELECT id FROM b_stale WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('stale partial honest', 'SELECT id FROM b_part  WHERE id > 0', NULL, $IOS);
SQL
  # Disposable-fixture catalog forgery.  Never run this against a real database.
  pg -q -c "UPDATE /* wiki_bloatplan_fixture_catalog_forgery */ pg_class SET relpages = 1 WHERE relname = 'b_stale_idx'" \
        -c "UPDATE /* wiki_bloatplan_fixture_catalog_forgery */ pg_class SET reltuples = 20 WHERE relname = 'b_part_idx'"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ fact('forged catalog ' || relname, 'relpages=' || relpages || ' reltuples=' || reltuples || ' live blocks=' || pg_relation_size(oid) / current_setting('block_size')::int) FROM pg_class WHERE relname IN ('b_stale_idx','b_part_idx') ORDER BY relname;
SELECT /* wiki_bloatplan_record */ xp('stale plain forged',   'SELECT id FROM b_stale WHERE id > 0', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('stale partial forged', 'SELECT id FROM b_part  WHERE id > 0', NULL, $IOS);
SQL
}

stage_fl3() {
  say "fl3: a bloated index dropped from a BitmapAnd, over two independent columns"
  # c is drawn from a seeded PRNG instead of being derived from g, so a and c
  # are independent of each other and a = 5 AND c = 7 is a combination that
  # exists.  setseed makes the draw reproducible, as it does for fixture G.
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS l3;
CREATE /* wiki_bloatplan_fixture */ TABLE l3 (a int NOT NULL, c int NOT NULL, pad text NOT NULL);
SELECT /* wiki_bloatplan_fixture */ setseed(0.42);
INSERT /* wiki_bloatplan_fixture */ INTO l3
  SELECT g % 2000, floor(random() * 20)::int, repeat('x', 24) FROM generate_series(1, 500000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX l3_a ON l3 (a);
CREATE /* wiki_bloatplan_fixture */ INDEX l3_c ON l3 (c);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) l3;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ixstat('l3 a', 'l3_a');
SELECT /* wiki_bloatplan_record */ ixstat('l3 c healthy', 'l3_c');
SELECT /* wiki_bloatplan_record */ xp('l3 healthy', 'SELECT pad FROM l3 WHERE a = 5 AND c = 7');
SELECT /* wiki_bloatplan_record */ xp('l3 a only',  'SELECT pad FROM l3 WHERE a = 5');
SELECT /* wiki_bloatplan_record */ xp('l3 c only',  'SELECT pad FROM l3 WHERE c = 7');
SQL
  pg -q -c "ALTER /* wiki_bloatplan_fixture */ INDEX l3_c SET (fillfactor = 10)" -c "REINDEX /* wiki_bloatplan_fixture */ INDEX l3_c"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ixstat('l3 c bloated', 'l3_c');
SELECT /* wiki_bloatplan_record */ xp('l3 bloated', 'SELECT pad FROM l3 WHERE a = 5 AND c = 7');
-- Exact statistics are not exact estimates.  Both columns are analyzed
-- exhaustively here, and the conjunction is still estimated by multiplying two
-- per-clause selectivities, so record each estimate beside the true count and
-- let the page report the gap rather than assert there is none.  c is a
-- random draw, so even with independent columns the true count of the
-- conjunction is itself random; only its expectation equals the product.
SELECT /* wiki_bloatplan_record */ fact('l3 actual rows', 'a=5 AND c=7 = ' || count(*) FILTER (WHERE a = 5 AND c = 7)
       || ', a=5 = ' || count(*) FILTER (WHERE a = 5)
       || ', c=7 = ' || count(*) FILTER (WHERE c = 7)
       || ', total = ' || count(*)) FROM l3;
SELECT /* wiki_bloatplan_record */ fact('l3 estimated rows', string_agg(label || ' = ' || plan_rows, ', ' ORDER BY seq))
  FROM r WHERE label IN ('l3 healthy', 'l3 a only', 'l3 c only', 'l3 bloated');
SQL
}

churn() {  # churn <table>: five whole-table non-HOT UPDATE rounds, one transaction each
  local t="$1" i
  for i in 1 2 3 4 5; do
    pg -q -c "UPDATE /* wiki_bloatplan_fixture_churn */ $t SET payload = payload + 1"
  done
}

hold_snapshot() {  # hold_snapshot <table>: REPEATABLE READ snapshot in a second session
  local t="$1" tries=0 n
  # The holder runs in the background with the same -X, ON_ERROR_STOP and
  # SESSION_OPTS as every other session.  Its 20-minute statement_timeout
  # outlasts the 900-second sleep, and release_snapshot() ends it long before.
  ( PGAPPNAME=bloatplan_holder PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -q -v ON_ERROR_STOP=1 \
      -c "BEGIN /* wiki_bloatplan_fixture_holder */ ISOLATION LEVEL REPEATABLE READ" \
      -c "SELECT /* wiki_bloatplan_fixture_holder */ count(*) FROM $t" \
      -c "SELECT /* wiki_bloatplan_fixture_holder */ pg_sleep(900)" > "$OUT/holder_$t.log" 2>&1 ) &
  HOLDER_PID=$!
  while :; do
    n=$(pgq -c "SELECT /* wiki_bloatplan_holder_probe */ count(*) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder' AND backend_xmin IS NOT NULL") \
      || die "the holder probe failed"
    [ "$n" = "1" ] && break
    tries=$((tries + 1)); [ "$tries" -gt 60 ] && die "the holder session never took its snapshot"
    sleep 1
  done
  # The xmin itself is a transaction id that moves with every run, so the
  # fact records only that the holder has one, which keeps summary.txt
  # identical between runs.
  pgq -c "SELECT /* wiki_bloatplan_record */ fact('holder $t', 'snapshot held, backend_xmin ' || CASE WHEN backend_xmin IS NULL THEN 'not set' ELSE 'set' END) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder'"
}

release_snapshot() {  # release_snapshot <table>: end the holder, which must still be alive
  local t="$1" ended n
  # A holder that died during the churn would have released its snapshot
  # early, so exactly one live session must be ended here.  The 10-second
  # timeout makes pg_terminate_backend() wait until the backend has exited.
  ended=$(pgq -c "SELECT /* wiki_bloatplan_holder_release */ count(*) FILTER (WHERE pg_terminate_backend(pid, 10000)) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder'") \
    || die "the holder release failed"
  [ "$ended" = "1" ] || die "expected to end exactly one live holder session after the churn, ended $ended"
  pgq -c "SELECT /* wiki_bloatplan_record */ fact('holder $t ended', 'sessions ended after the churn=$ended')"
  wait 2>/dev/null
  HOLDER_PID=""
  n=$(pgq -c "SELECT /* wiki_bloatplan_holder_probe */ count(*) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder'") \
    || die "the holder probe failed"
  [ "$n" = "0" ] || die "the holder session is still connected"
}

stage_fp() {
  say "fp: fixture P, version churn at 1,000 and at 100 tag values, with and without a held snapshot"
  local mod t
  for mod in 1000 100; do
    for t in p${mod}_free p${mod}_held; do
      pg -q <<SQL
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS $t;
CREATE /* wiki_bloatplan_fixture */ TABLE $t (id int NOT NULL, payload int NOT NULL, tag int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO $t SELECT g, g, g % $mod FROM generate_series(1, 200000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX ${t}_payload ON $t (payload);
CREATE /* wiki_bloatplan_fixture */ INDEX ${t}_tag     ON $t (tag);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) $t;
SQL
    done
    pgq -c "SELECT /* wiki_bloatplan_record */ ixstat('P$mod free start', 'p${mod}_free_tag')" \
        -c "SELECT /* wiki_bloatplan_record */ ixstat('P$mod held start', 'p${mod}_held_tag')"
    churn p${mod}_free
    pgq -c "SELECT /* wiki_bloatplan_record */ ixstat('P$mod free end', 'p${mod}_free_tag')"
    hold_snapshot p${mod}_held
    churn p${mod}_held
    pgq -c "SELECT /* wiki_bloatplan_record */ ixstat('P$mod held end', 'p${mod}_held_tag')"
    release_snapshot p${mod}_held
    pg -q -c "ANALYZE /* wiki_bloatplan_fixture */ p${mod}_free" -c "ANALYZE /* wiki_bloatplan_fixture */ p${mod}_held"
    pgq <<SQL
SELECT /* wiki_bloatplan_record */ xp('P$mod free bitmap', 'SELECT * FROM p${mod}_free WHERE tag = 7', NULL, 'enable_seqscan=off','enable_indexscan=off');
SELECT /* wiki_bloatplan_record */ xp('P$mod held bitmap', 'SELECT * FROM p${mod}_held WHERE tag = 7', NULL, 'enable_seqscan=off','enable_indexscan=off');
SQL
  done
}

q_state() {  # q_state <state>: catalog rows, live sizes and the charged index pages of fixture Q
  local s="$1"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ fact('Q $s ' || c.relname, 'relpages=' || c.relpages || ' reltuples=' || c.reltuples || ' live blocks=' || pg_relation_size(c.oid) / current_setting('block_size')::int) FROM pg_class c WHERE c.relname IN ('q_t','q_part','q_full') ORDER BY c.relname;
SELECT /* wiki_bloatplan_record */ xp('Q $s partial rpc4', 'SELECT v FROM q_t WHERE id <= 20000 AND v > 0', 'q_full', $BMP);
SELECT /* wiki_bloatplan_record */ xp('Q $s partial rpc1', 'SELECT v FROM q_t WHERE id <= 20000 AND v > 0', 'q_full', $BMP, 'random_page_cost=1');
SELECT /* wiki_bloatplan_record */ xp('Q $s plain rpc4',   'SELECT v FROM q_t WHERE v > 0', 'q_part', $BMP);
SELECT /* wiki_bloatplan_record */ xp('Q $s plain rpc1',   'SELECT v FROM q_t WHERE v > 0', 'q_part', $BMP, 'random_page_cost=1');
SELECT /* wiki_bloatplan_record */ fact('Q $s charged index pages',
       'partial=' || round((idxcost('Q $s partial rpc4') - idxcost('Q $s partial rpc1')) / 3, 2)
       || ' plain=' || round((idxcost('Q $s plain rpc4') - idxcost('Q $s plain rpc1')) / 3, 2));
SQL
}

stage_fq() {
  say "fq: fixture Q, a partial index that grows after its pg_class row was last written"
  # q_part covers the first 20,000 of 200,000 rows; q_full covers them all.
  # Ten UPDATE rounds over the partial index's rows, in one transaction so
  # that no pruning and no bottom-up deletion can reclaim a version while the
  # rounds run, add ten entries per row to both indexes and double the heap.
  # Nothing writes pg_class until the ANALYZE at the end.
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS q_t;
CREATE /* wiki_bloatplan_fixture */ TABLE q_t (id int NOT NULL, v int NOT NULL, pad text NOT NULL) WITH (fillfactor = 100);
INSERT /* wiki_bloatplan_fixture */ INTO q_t SELECT g, g, repeat('x', 20) FROM generate_series(1, 200000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX q_part ON q_t (v) WHERE id <= 20000;
CREATE /* wiki_bloatplan_fixture */ INDEX q_full ON q_t (v);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) q_t;
SQL
  q_state built
  pg -q <<'SQL'
DO /* wiki_bloatplan_fixture_churn */ $churn$
BEGIN
  FOR i IN 1..10 LOOP
    UPDATE /* wiki_bloatplan_fixture_churn */ q_t SET v = v + 1 WHERE id <= 20000;
  END LOOP;
END $churn$;
SQL
  pgq -c "SELECT /* wiki_bloatplan_record */ ixstat('Q part churned', 'q_part')" \
      -c "SELECT /* wiki_bloatplan_record */ ixstat('Q full churned', 'q_full')"
  q_state churned
  pg -q -c "ANALYZE /* wiki_bloatplan_fixture */ q_t"
  q_state analyzed
}

stage_fe() {
  say "fe: fixture E, dead entries at the end of a B-tree and the planner's endpoint probe"
  # The histogram is built while id runs to 200,000; the upper half is then
  # deleted and nothing vacuums or analyzes the table.  A predicate that lands
  # in the last histogram bucket makes the planner probe the index for the
  # true maximum.  Each probe reads dead entries until it has visited 100 heap
  # pages without a visible row, and the entries it read are marked dead in
  # the index, so each plan gets further than the one before.
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS e_t;
CREATE /* wiki_bloatplan_fixture */ TABLE e_t (id int NOT NULL, v int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO e_t SELECT g, g FROM generate_series(1, 200000) g;
ALTER /* wiki_bloatplan_fixture */ TABLE e_t ADD CONSTRAINT e_t_pkey PRIMARY KEY (id);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) e_t;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ fact('E histogram', 'bounds=' || array_length(histogram_bounds::text::int[], 1)
       || ' last bound=' || (histogram_bounds::text::int[])[array_length(histogram_bounds::text::int[], 1)]
       || ' heap blocks=' || pg_relation_size('e_t') / current_setting('block_size')::int)
  FROM pg_stats WHERE schemaname = 'public' AND tablename = 'e_t' AND attname = 'id';
SELECT /* wiki_bloatplan_record */ xp('E before delete', 'SELECT count(*) FROM e_t WHERE id > 199990');
SQL
  pg -q -c "DELETE /* wiki_bloatplan_fixture */ FROM e_t WHERE id > 100000"
  pgq -c "SELECT /* wiki_bloatplan_record */ fact('E after delete', 'index entries marked dead=' || sum(dead_items)) FROM bt_multi_page_stats('e_t_pkey', 1, -1)"
  local k
  for k in 1 2 3 4 5 6; do
    pgq <<SQL
SELECT /* wiki_bloatplan_record */ xp('E plan $k', 'SELECT count(*) FROM e_t WHERE id > 199990');
SELECT /* wiki_bloatplan_record */ fact('E plan $k', 'index entries marked dead=' || sum(dead_items)) FROM bt_multi_page_stats('e_t_pkey', 1, -1);
SQL
  done
  pg -q -c "VACUUM /* wiki_bloatplan_fixture */ e_t"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ fact('E after vacuum', 'index entries marked dead=' || sum(dead_items)) FROM bt_multi_page_stats('e_t_pkey', 1, -1);
SELECT /* wiki_bloatplan_record */ xp('E after vacuum', 'SELECT count(*) FROM e_t WHERE id > 199990');
SQL
}

# ---- GIN follow-up ----------------------------------------------------------

stage_gs() {
  say "gs: fixture S, one table, a btree_gin GIN index and a B-tree on the same column"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS t_both;
CREATE /* wiki_bloatplan_fixture */ TABLE t_both (id int NOT NULL, n int);
INSERT /* wiki_bloatplan_fixture */ INTO t_both SELECT i, i % 10000 FROM generate_series(1, 300000) i;
CREATE /* wiki_bloatplan_fixture */ INDEX t_both_n_gin ON t_both USING gin (n);
CREATE /* wiki_bloatplan_fixture */ INDEX t_both_n_bt  ON t_both (n);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) t_both;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('S gin', 't_both_n_gin');
SELECT /* wiki_bloatplan_record */ ixstat('S btree', 't_both_n_bt');
SELECT /* wiki_bloatplan_record */ xp('S gin n=42',      'SELECT * FROM t_both WHERE n = 42', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt n=42',       'SELECT * FROM t_both WHERE n = 42', 't_both_n_gin', 'enable_seqscan=off', 'enable_indexscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin between',   'SELECT * FROM t_both WHERE n BETWEEN 100 AND 200', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt between',    'SELECT * FROM t_both WHERE n BETWEEN 100 AND 200', 't_both_n_gin', 'enable_seqscan=off', 'enable_indexscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin n<20',      'SELECT * FROM t_both WHERE n < 20', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt n<20',       'SELECT * FROM t_both WHERE n < 20', 't_both_n_gin', 'enable_seqscan=off', 'enable_indexscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin in123',     'SELECT * FROM t_both WHERE n IN (1,2,3)', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt in123',      'SELECT * FROM t_both WHERE n IN (1,2,3)', 't_both_n_gin', 'enable_seqscan=off', 'enable_indexscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin orderby',   'SELECT * FROM t_both ORDER BY n LIMIT 10', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt orderby',    'SELECT * FROM t_both ORDER BY n LIMIT 10', 't_both_n_gin', 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin ios',       'SELECT n FROM t_both WHERE n = 42', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt ios',        'SELECT n FROM t_both WHERE n = 42', 't_both_n_gin', 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin isnull',    'SELECT * FROM t_both WHERE n IS NULL', 't_both_n_bt',  'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S bt isnull',     'SELECT * FROM t_both WHERE n IS NULL', 't_both_n_gin', 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('S gin n=42 rpc1', 'SELECT * FROM t_both WHERE n = 42', 't_both_n_bt',  'enable_seqscan=off', 'random_page_cost=1');
SELECT /* wiki_bloatplan_record */ xp('S default n=42',    'SELECT * FROM t_both WHERE n = 42');
SELECT /* wiki_bloatplan_record */ xp('S default between', 'SELECT * FROM t_both WHERE n BETWEEN 100 AND 200');
SELECT /* wiki_bloatplan_record */ xp('S default n<20',    'SELECT * FROM t_both WHERE n < 20');
SELECT /* wiki_bloatplan_record */ fact('S property ' || p, 'gin=' || pg_index_has_property('t_both_n_gin'::regclass, p) || ' btree=' || pg_index_has_property('t_both_n_bt'::regclass, p))
  FROM unnest(ARRAY['clusterable','index_scan','bitmap_scan','backward_scan']) p;
SELECT /* wiki_bloatplan_record */ fact('S column property ' || p, 'gin=' || pg_index_column_has_property('t_both_n_gin'::regclass, 1, p) || ' btree=' || pg_index_column_has_property('t_both_n_bt'::regclass, 1, p))
  FROM unnest(ARRAY['orderable','returnable','search_array','search_nulls']) p;
SQL
}

# docs_rows <from> <to>: the INSERT that fills fixture D.
#
# The rare lexeme is placed on every 4,999th row, not every 5,000th.  With a
# period of 5,000 the fixture was degenerate: 20 divides 5,000, so every row
# carrying the lexeme also had cat = 0, and the two-clause predicate
# "zebracorn AND cat = 7" could not match a single row however the planner
# priced it.  4,999 is prime, so it is coprime with the 20 category values and
# the two populations are independent: the lexeme rows walk all 20 residues of
# i % 20 in turn, so exactly one in twenty of them carries each category.
# cat stays i % 20, which keeps it exactly uniform and keeps the count of
# lexeme rows the same (80 in 400,000 either way).
docs_rows() {  # docs_rows <from> <to>
  printf "INSERT /* wiki_bloatplan_fixture */ INTO docs SELECT i, i %% 20, to_tsvector('simple', 'filler w' || (i %% 1000) || ' v' || (i %% 777) || CASE WHEN i %% 4999 = 0 THEN ' zebracorn' ELSE '' END) FROM generate_series(%s, %s) i" "$1" "$2"
}

docs_state() {  # docs_state <label>: what the planner does with the two-clause query in this state
  local l="$1"
  local q="SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7"
  local qg="SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn')"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ginstat('D $l', 'docs_tsv_gin');
SELECT /* wiki_bloatplan_record */ fact('D $l heap', 'relpages=' || relpages || ' reltuples=' || reltuples || ' live blocks=' || pg_relation_size(oid) / current_setting('block_size')::int) FROM pg_class WHERE relname = 'docs';
SELECT /* wiki_bloatplan_record */ xp('D $l chosen',     \$q\$$q\$q\$);
SELECT /* wiki_bloatplan_record */ xp('D $l btree only', \$q\$$q\$q\$, 'docs_tsv_gin');
SELECT /* wiki_bloatplan_record */ xp('D $l gin alone',      \$q\$$qg\$q\$, NULL, 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('D $l gin alone rpc1', \$q\$$qg\$q\$, NULL, 'enable_seqscan=off', 'random_page_cost=1');
SQL
}

stage_gd() {
  say "gd: fixture D, a tsvector GIN index with a fastupdate pending list, then a multicolumn GIN"
  pg -q <<SQL
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS docs;
CREATE /* wiki_bloatplan_fixture */ TABLE docs (id int NOT NULL, cat int NOT NULL, tsv tsvector NOT NULL);
$(docs_rows 1 200000);
CREATE /* wiki_bloatplan_fixture */ INDEX docs_tsv_gin ON docs USING gin (tsv) WITH (fastupdate = on);
CREATE /* wiki_bloatplan_fixture */ INDEX docs_cat_bt  ON docs (cat);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) docs;
SQL
  pgq -c "SELECT /* wiki_bloatplan_record */ ixstat('D cat btree', 'docs_cat_bt')"
  docs_state "vacuumed"
  # gin_pending_list_limit is PGC_USERSET; raised for these two sessions only so
  # that the inserts do not drain the pending list themselves.
  pgopt "-c gin_pending_list_limit=1GB" \
    -c "$(docs_rows 200001 300000)" -c "ANALYZE /* wiki_bloatplan_fixture */ docs"
  docs_state "pending one"
  pgopt "-c gin_pending_list_limit=1GB" \
    -c "$(docs_rows 300001 400000)" -c "ANALYZE /* wiki_bloatplan_fixture */ docs"
  docs_state "pending two"
  pgq -c "SELECT /* wiki_bloatplan_record */ fact('D gin_clean_pending_list returned', gin_clean_pending_list('docs_tsv_gin')::text)"
  docs_state "drained"
  pg -q -c "VACUUM /* wiki_bloatplan_fixture */ docs"
  docs_state "revacuumed"
  # The cat = 7 clause on its own, in the same state, so that all three
  # estimates the page sets beside true counts come from one statistics state.
  pgq -c "SELECT /* wiki_bloatplan_record */ xp('D revacuumed cat alone', 'SELECT * FROM docs WHERE cat = 7')"
  pg -q -c "CREATE /* wiki_bloatplan_fixture */ INDEX docs_multi_gin ON docs USING gin (tsv, cat)"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('D multi', 'docs_multi_gin');
SELECT /* wiki_bloatplan_record */ fact('D multi heap', 'relpages=' || relpages || ' reltuples=' || reltuples || ' live blocks=' || pg_relation_size(oid) / current_setting('block_size')::int) FROM pg_class WHERE relname = 'docs';
SELECT /* wiki_bloatplan_record */ xp('D multi chosen', $q$SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7$q$);
SELECT /* wiki_bloatplan_record */ xp('D multi hidden', $q$SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7$q$, 'docs_multi_gin');
-- The two clauses must be able to hold together, or every cost above prices a
-- plan for a predicate that matches nothing.  Record the true counts beside
-- the estimates, as fixture L3 does, so the page reports the gap instead of
-- assuming there is none.
SELECT /* wiki_bloatplan_record */ fact('D actual rows', 'zebracorn AND cat=7 = '
       || count(*) FILTER (WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7)
       || ', zebracorn = ' || count(*) FILTER (WHERE tsv @@ to_tsquery('simple','zebracorn'))
       || ', cat=7 = ' || count(*) FILTER (WHERE cat = 7)
       || ', total = ' || count(*)) FROM docs;
SELECT /* wiki_bloatplan_record */ fact('D estimated rows', string_agg(label || ' = ' || plan_rows, ', ' ORDER BY seq))
  FROM r WHERE label IN ('D revacuumed chosen', 'D revacuumed gin alone', 'D revacuumed cat alone', 'D multi chosen');
SQL
}

stage_gt() {
  say "gt: fixture T, a stale GIN metapage and the 4X fallback"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS t_stale;
CREATE /* wiki_bloatplan_fixture */ TABLE t_stale (id int NOT NULL, n int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO t_stale SELECT i, i % 100 FROM generate_series(1, 2000) i;
CREATE /* wiki_bloatplan_fixture */ INDEX t_stale_gin ON t_stale USING gin (n) WITH (fastupdate = off);
VACUUM /* wiki_bloatplan_fixture */ (ANALYZE) t_stale;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('T small', 't_stale_gin');
SELECT /* wiki_bloatplan_record */ xp('T small n=42', 'SELECT * FROM t_stale WHERE n = 42', NULL, 'enable_seqscan=off');
SQL
  pg -q -c "INSERT /* wiki_bloatplan_fixture */ INTO t_stale SELECT i, i % 200000 FROM generate_series(2001, 400000) i" \
        -c "ANALYZE /* wiki_bloatplan_fixture */ t_stale"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('T grown', 't_stale_gin');
SELECT /* wiki_bloatplan_record */ xp('T grown n=42',      'SELECT * FROM t_stale WHERE n = 42', NULL, 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('T grown n=42 rpc1', 'SELECT * FROM t_stale WHERE n = 42', NULL, 'enable_seqscan=off', 'random_page_cost=1');
SQL
  pg -q -c "VACUUM /* wiki_bloatplan_fixture */ t_stale"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('T vacuumed', 't_stale_gin');
SELECT /* wiki_bloatplan_record */ xp('T vacuumed n=42',      'SELECT * FROM t_stale WHERE n = 42', NULL, 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('T vacuumed n=42 rpc1', 'SELECT * FROM t_stale WHERE n = 42', NULL, 'enable_seqscan=off', 'random_page_cost=1');
SQL
}

stage_gp() {
  say "gp: fixture P-gin, the keyless path on a partial GIN index"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS t_part;
CREATE /* wiki_bloatplan_fixture */ TABLE t_part (id int NOT NULL, n int NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO t_part SELECT i, i % 1000 FROM generate_series(1, 100000) i;
CREATE /* wiki_bloatplan_fixture */ INDEX t_part_gin ON t_part USING gin (n) WHERE id <= 5000;
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) t_part;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('Pg partial', 't_part_gin');
SELECT /* wiki_bloatplan_record */ xp('Pg keyless',      'SELECT * FROM t_part WHERE id <= 5000', NULL, 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('Pg keyless rpc1', 'SELECT * FROM t_part WHERE id <= 5000', NULL, 'enable_seqscan=off', 'random_page_cost=1');
SELECT /* wiki_bloatplan_record */ xp('Pg keyed',        'SELECT * FROM t_part WHERE id <= 5000 AND n = 42', NULL, 'enable_seqscan=off');
SQL
}

stage_gb() {
  say "gb: fixture B-gin, a boolean column under a btree_gin bool opclass"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS t_bool;
CREATE /* wiki_bloatplan_fixture */ TABLE t_bool (id int NOT NULL, i boolean);
INSERT /* wiki_bloatplan_fixture */ INTO t_bool SELECT g, g % 100 = 0 FROM generate_series(1, 100000) g;
CREATE /* wiki_bloatplan_fixture */ INDEX t_bool_gin ON t_bool USING gin (i);
VACUUM /* wiki_bloatplan_fixture */ (FREEZE, ANALYZE) t_bool;
SQL
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ xp('Bg bare',   'SELECT * FROM t_bool WHERE i',         NULL, 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('Bg eqtrue', 'SELECT * FROM t_bool WHERE i = true',  NULL, 'enable_seqscan=off');
SELECT /* wiki_bloatplan_record */ xp('Bg istrue', 'SELECT * FROM t_bool WHERE i IS TRUE', NULL, 'enable_seqscan=off');
SQL
}

stage_grej() {
  say "grej: the four things no GIN index can be created for (four errors are the expected result)"
  # The four statements need fixture S.  Without it they would fail for the
  # wrong reason, "relation does not exist", so the index is checked first
  # and the errors are matched by their text, not merely counted.
  local have
  have=$(pgq -c "SELECT /* wiki_bloatplan_grej_precheck */ to_regclass('t_both_n_gin') IS NOT NULL") \
    || die "could not check for fixture S"
  [ "$have" = "t" ] || die "grej needs fixture S: run the gs stage first"
  pgerr <<'SQL' 2>&1 | grep -E 'ERROR' | tee "$OUT/grej.txt"
CREATE /* wiki_bloatplan_expected_error */ UNIQUE INDEX rej_u ON t_both USING gin (n);
CREATE /* wiki_bloatplan_expected_error */ INDEX rej_i ON t_both USING gin (n) INCLUDE (id);
ALTER /* wiki_bloatplan_expected_error */ TABLE t_both ADD CONSTRAINT rej_x EXCLUDE USING gin (n WITH =);
CLUSTER /* wiki_bloatplan_expected_error */ t_both USING t_both_n_gin;
SQL
  [ "$(grep -c ERROR "$OUT/grej.txt")" = "4" ] || die "expected four errors, see $OUT/grej.txt"
  [ "$(grep -c -E 'does not support unique indexes|does not support included columns|does not support exclusion constraints|access method does not support clustering' "$OUT/grej.txt")" = "4" ] \
    || die "the four errors are not the four GIN rejections, see $OUT/grej.txt"
}

stage_diag() {
  say "diag: run the page's two filed diagnostic blocks verbatim against my_table / my_col / my_gin_index"
  # The page's two sql blocks are read out of the page itself, so what runs is
  # what is filed.  The fence is assembled here because this script is
  # published inside a fenced block.
  local fence n=0 inblock=0 line
  fence=$(printf '\140\140\140')
  [ -f "$PAGE" ] || die "no page at $PAGE"
  while IFS= read -r line; do
    if [ "$inblock" = 0 ] && [ "$line" = "${fence}sql" ]; then
      n=$((n + 1)); inblock=1; : > "$OUT/diag$n.sql"; continue
    fi
    if [ "$inblock" = 1 ] && [ "$line" = "$fence" ]; then inblock=0; continue; fi
    [ "$inblock" = 1 ] && printf '%s\n' "$line" >> "$OUT/diag$n.sql"
  done < "$PAGE"
  [ "$n" = "2" ] || die "expected exactly two sql blocks on the page, found $n"
  pg -q <<'SQL'
DROP /* wiki_bloatplan_fixture */ TABLE IF EXISTS my_table;
CREATE /* wiki_bloatplan_fixture */ TABLE my_table (id int NOT NULL, my_col tsvector NOT NULL);
INSERT /* wiki_bloatplan_fixture */ INTO my_table SELECT i, to_tsvector('simple', 'filler w' || (i % 1000)) FROM generate_series(1, 100000) i;
CREATE /* wiki_bloatplan_fixture */ INDEX my_gin_index ON my_table USING gin (my_col) WITH (fastupdate = on);
VACUUM /* wiki_bloatplan_fixture */ (ANALYZE) my_table;
SQL
  pgopt "-c gin_pending_list_limit=1GB" \
    -c "INSERT /* wiki_bloatplan_fixture */ INTO my_table SELECT i, to_tsvector('simple', 'filler w' || (i % 1000)) FROM generate_series(100001, 150000) i"
  # The blocks SET their own timeouts; SESSION_OPTS is passed as well so that a
  # PGOPTIONS in the caller's environment cannot reach these two sessions.
  PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -v ON_ERROR_STOP=1 -f "$OUT/diag1.sql" > "$OUT/diag1.out" 2>&1 \
    || die "diagnostic block 1 failed, see $OUT/diag1.out"
  PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -v ON_ERROR_STOP=1 -f "$OUT/diag2.sql" > "$OUT/diag2.out" 2>&1 \
    || die "diagnostic block 2 failed, see $OUT/diag2.out"
  grep -E 'my_gin_index' "$OUT/diag1.out" "$OUT/diag2.out"
}

stage_predict() {
  say "predict: the closed form in (pages, tuples, fastlevel) against EXPLAIN, fixture A"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ predict('A dense full',  'a_dense_idx',  'a_dense');
SELECT /* wiki_bloatplan_record */ predict('A sparse full', 'a_sparse_idx', 'a_sparse');
SELECT /* wiki_bloatplan_record */ fact('closed form against EXPLAIN', 'matches on ' || count(*) FILTER (WHERE predicted = observed) || ' of ' || count(*)) FROM pr;
SQL
}

stage_summary() {
  say "summary: written to $OUT/summary.txt"
  # Only standard output goes to the file: a psql error or an abort message
  # lands on the terminal, where it is seen, not inside the results.
  pg <<'SQL' > "$OUT/summary.txt"
\pset pager off
SELECT /* wiki_bloatplan_summary */ label, idx, blocks, tree_level, fastlevel, leaf_pages AS leaf, internal_pages AS internal,
       deleted_pages AS deleted, density, frag FROM ix ORDER BY seq;
SELECT /* wiki_bloatplan_summary */ label, idx, blocks, pending_pages AS pending, n_total, n_entry, n_data, n_entries FROM gx ORDER BY seq;
SELECT /* wiki_bloatplan_summary */ label, pages, tuples, fastlevel, predicted, observed, predicted = observed AS match FROM pr ORDER BY seq;
SELECT /* wiki_bloatplan_summary */ c.relname AS fixture_table, c.relpages, c.relallvisible, c.reltuples
  FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE n.nspname = 'public' AND c.relkind = 'r' AND c.relname NOT IN ('r','ix','gx','pr','fx') ORDER BY 1;
SELECT /* wiki_bloatplan_summary */ label, value FROM fx ORDER BY seq;
\pset format unaligned
SELECT /* wiki_bloatplan_summary */ label, nodes(label) AS plan FROM r ORDER BY seq;
SQL
  cat "$OUT/platform.txt" >> "$OUT/summary.txt"
  [ -f "$OUT/grej.txt" ] && cat "$OUT/grej.txt" >> "$OUT/summary.txt"
  [ -f "$OUT/diag1.out" ] && cat "$OUT/diag1.out" "$OUT/diag2.out" >> "$OUT/summary.txt"
  date -u '+run finished %Y-%m-%dT%H:%M:%SZ' >> "$OUT/summary.txt"
  note "$(grep -c . "$OUT/summary.txt") lines"
}

# The teardown is asserted, not assumed: pg_ctl finds no server, there is no
# postmaster.pid, no process runs from this data directory, and no socket is
# left on the port.  $DATA is the resolved path pg_ctl was given, so the pgrep
# pattern is the text the postmaster's own command line carries.
assert_down() {
  # Without pgrep the process check below would fail with status 127 and
  # pass silently, so its absence is an error, not a pass.
  command -v pgrep >/dev/null 2>&1 || die "pgrep is needed to assert that no postgres process is left"
  "$BIN/pg_ctl" -D "$DATA" status >/dev/null 2>&1 && die "pg_ctl still reports a server running from $DATA"
  [ -f "$DATA/postmaster.pid" ] && die "postmaster.pid is still present in $DATA"
  pgrep -f -- "-D $DATA" >/dev/null 2>&1 && die "a postgres process is still running from $DATA"
  [ -e "$SOCK/.s.PGSQL.$PORT" ] && die "the socket $SOCK/.s.PGSQL.$PORT is still present"
  return 0
}

stage_stop() {
  say "stop: shut the measurement cluster down"
  own_sandbox || return 0
  stop_server
  assert_down
  note "stopped: no postmaster.pid, no process from $DATA, no socket on port $PORT; $SANDBOX is kept until the clean stage"
}

stage_clean() {
  say "clean: stop the server and delete the sandbox"
  own_sandbox || return 0
  stop_server
  assert_down
  rm -rf "$SANDBOX" || die "could not delete $SANDBOX"
  SANDBOX_OK=0
  note "removed $SANDBOX"
}

# The dispatcher checks every stage name before any stage runs, claims the
# sandbox for the stages that write into it, starts a server for the stages
# that need one, and turns a failed stage into a failed run.  Without the
# || die a stage that returns non-zero without calling die is skipped over and
# the script still exits 0.  CURRENT_STAGE only names the stage in the abort
# message; the abort itself comes from pg(), pgopt() or an explicit die inside
# the stage, because a stage function's own exit status is the status of its
# LAST command and cannot speak for the calls before it.
SQL_STAGES="fa fb ff fg fh fi fn fstale fl3 fp fq fe gs gd gt gp gb grej diag predict summary"
known_stage() {
  case " build check cluster stop clean $SQL_STAGES " in *" $1 "*) return 0 ;; esac
  return 1
}
run_stage() {
  local st="$1"
  CURRENT_STAGE="$st"
  case "$st" in
    stop|clean) ;;
    build|check|cluster) claim_sandbox ;;
    *) claim_sandbox; need_server ;;
  esac
  "stage_$st" || die "stage $st failed"
  CURRENT_STAGE=""
}

DEFAULT="build check cluster $SQL_STAGES stop"
[ $# -eq 0 ] && set -- $DEFAULT
for st in "$@"; do known_stage "$st" || die "unknown stage: $st; nothing was run"; done
for st in "$@"; do run_stage "$st"; done
```

## Context Reviewed

Planner and cost path: `src/backend/optimizer/util/plancat.c` (`get_relation_info`, `estimate_rel_size`), `src/backend/optimizer/path/costsize.c` (`cost_index`, `index_pages_fetched`, `cost_bitmap_heap_scan`, `compute_bitmap_pages`, `get_indexpath_pages`), `src/backend/optimizer/path/allpaths.c` (`total_table_pages`, `compute_parallel_worker`), `src/backend/optimizer/path/indxpath.c` (`choose_bitmap_and`, `bitmap_scan_cost_est`), `src/backend/optimizer/util/pathnode.c` (`compare_path_costs_fuzzily`), `src/backend/utils/adt/selfuncs.c` (`genericcostestimate`, `btcostestimate`, `hashcostestimate`, `gistcostestimate`, `spgcostestimate`, `gincostestimate`, `brincostestimate`, `add_predicate_to_index_quals`).

nbtree: `nbtpage.c` (`_bt_getrootheight`, fast-root update, page deletion), `nbtsplitloc.c` (`_bt_findsplitloc` fillfactor policy and single-value strategy), `nbtdedup.c` (`_bt_dedup_pass`, `_bt_bottomupdel_pass`), `nbtinsert.c` (pre-split deletion and deduplication), `nbtsort.c`, `src/include/access/nbtree.h`, and the nbtree `README` sections on page deletion, tree height, FSM placement, simple deletion, bottom-up deletion, split policy, and deduplication.

VACUUM: `src/backend/access/heap/vacuumlazy.c` (`BYPASS_THRESHOLD_PAGES`, `lazy_vacuum`, `lazy_check_wraparound_failsafe`, `LVRelState`).

Headers and catalogs: `src/include/nodes/pathnodes.h` (`IndexOptInfo`, `RelOptInfo`, `PlannerInfo`), `src/include/catalog/pg_class.h`, `src/backend/access/common/reloptions.c` (including the table `parallel_workers` option), `src/backend/utils/cache/spccache.c` (`get_tablespace_page_costs`), `src/backend/utils/misc/guc_tables.c`.

Contrib: `contrib/pgstattuple/pgstatindex.c` plus its `sql/` and `expected/` regression files, `contrib/pageinspect` `bt_metap()` definitions.

Documentation: `ref/reindex.sgml`, `maintenance.sgml` (routine reindexing), `glossary.sgml` (Bloat), `btree.sgml` (version churn, bottom-up deletion, deduplication), `ref/create_table.sgml` (`vacuum_index_cleanup`), `ref/create_index.sgml`, `indices.sgml`, `pgstattuple.sgml`.

History: `git log -L` on `genericcostestimate`, `btcostestimate`, `gincostestimate`, `index_pages_fetched` and `cost_index` over `615cebc94b..HEAD`, where `615cebc94b` is the `Stamp HEAD as 13devel.` commit that marks the v12 branch point; `git merge-base --is-ancestor` of each attributed commit against the five `Stamp HEAD as NNdevel.` commits and the twelve `Stamp 17.N.` commits; `git show -s` for every subject and date. The v17 checkout carries no tags, so `git tag --contains` and `git describe` cannot run in it, and the first filing's use of them was replaced by this method; every attribution it made survived. `REL_12_0` function text for `index_pages_fetched`, `cost_index`, `genericcostestimate`, `btcostestimate`, `gincostestimate`, the `get_relation_info()` index-size block, `estimate_rel_size()` and `reloptions.c` was read from the v12 checkout's `REL_12_0` tag. Also `contrib/pgstattuple` and `src/backend/access/nbtree` history in the same range, and the diff of the cited files between the two pins, `54eeefaedbee..786db8dcf168`. The 2026-09-25 pass added: `git log -L '/^get_actual_variable_endpoint(/,/^}/:src/backend/utils/adt/selfuncs.c' 615cebc94b..HEAD`, which lists `9c6ad5eaa9`, `dc7420c2c9` and `d3751adcf1` (the `:get_actual_variable_endpoint:` form matches the function's forward declaration and lists only `d3751adcf1`); `git show --stat` on `fccebe421`, `3ca930fc3` and `9c6ad5eaa9` for the probe's test absence; `git log -S 'stats->estimated_count = true'`, which attributes that `btvacuumcleanup()` line to `9f3665fbfc`; `3d351d916b2`'s `index.c` diff, which adds the keep-`-1` hack; the messages of `9c6ad5eaa9`, `19d8e2308b`, `9f3665fbfc` and `effdd3f3b6`; branch-point ancestry for `9c6ad5eaa9`, `19d8e2308b`, `5753d4ee32`, `e3fcca0d0d`, `4b754d6c16e`, `dc7420c2c9`, `e5d8a99903`, `3c569049b7b` and `71b66171d0`, and the `Stamp 17.0.` commit `d7ec59a63d` containing `71b66171d0`; in the v12 checkout, `git merge-base --is-ancestor` of `9c6ad5eaa9`'s `REL_12_STABLE` twin `ec10b6139c` against `REL_12_0`, `REL_12_14` and the wiki's v12 pin `45b88269` (only `REL_12_14` contains it), and of `d3751adcf1`'s twin `cee976c4e8` against `REL_12_0` (it is contained); and `git log 54eeefaedbee..786db8dcf168` over the cost path and every other cited file, which returns the 32 commits tabled in [Fixtures and method](#fixtures-and-method). `REL_12_0` text newly read from the v12 checkout: `heap_update()` in `heapam.c` and the update callback in `heapam_handler.c`; `get_actual_variable_endpoint()` and `estimate_array_length()` in `selfuncs.c`; the relfilenode reset in `relcache.c`; the empty-build path in `nbtsort.c`; `btvacuumcleanup()` in `nbtree.c`; `lazy_cleanup_index()` and the `useindex` rule in `vacuumlazy.c`; the `INDEX_CLEANUP` option in `vacuum.c`; and `btm_oldest_btpo_xact` in `nbtree.h`. `index_update_stats()`, the partitioned-index skip and `estimate_rel_size()` were read again for the `reltuples` and partitioned-index comparisons, and `gincostestimate()`'s full-scan test and `genericcostestimate()`'s array count for the v13 GIN change and the `= ANY` comparison.

Empirical: one isolated PostgreSQL 17.11 server built from the pin by the script under [Measurement Script](#measurement-script), whose B-tree stages cover density, real deletion bloat, a mostly-empty index whose fast root moved, the same draining taken to one surviving row so that the flat one-page estimate fires, height, seeded fragmentation, catalog forgery, plan flips, `BitmapAnd` pruning, parallel worker counts at three selectivities under both default and zeroed parallel costs, a repeated inner scan under two cache settings, the v17 SAOP clamp, version churn at two key cardinalities with and without a held snapshot, deduplication, a closed-form prediction checked against `EXPLAIN`, a partial and a plain index on one table priced before their growth reached `pg_class` and again after `ANALYZE` (fixture Q), and the planning-time endpoint probe of a primary key whose upper half was deleted, followed through six plans and a plain `VACUUM` with the dead-entry count read after each (fixture E).

Pin: `raw/postgres-17/` at commit `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11, seven commits past `Stamp 17.11.` `083ac03341`); repinned from `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on 2026-08-17. Every measured number on the page was taken on this pin by the script filed under [Measurement Script](#measurement-script), last on 2026-09-25, and every number filed since each fixture's last rebuild came back unchanged. The page no longer quotes numbers from the 17.10 filing or from superseded fixture versions; [Fixtures and method](#fixtures-and-method) says what changed.

Follow-up (GIN versus B-tree) additions. Path generation and gating: `src/backend/optimizer/path/indxpath.c` (`create_index_paths`, `match_restriction_clauses_to_index`, `match_clause_to_indexcol`, `match_opclause_to_indexcol`, `match_saopclause_to_indexcol`, `match_boolean_index_clause`, `IsBooleanOpfamily`, `get_index_clause_from_support`, `get_index_paths`, `build_index_paths`, `check_index_only`, `choose_bitmap_and`, `bitmap_and_cost_est`), `src/backend/optimizer/util/plancat.c` (the AM capability-flag copy and the `sortopfamily` branches), `src/backend/optimizer/util/pathnode.c` (`add_path`, `STD_FUZZ_FACTOR`), `src/backend/access/index/indexam.c` (`index_can_return`), `src/backend/optimizer/path/costsize.c` (`disable_cost`).

GIN internals: `src/backend/utils/adt/selfuncs.c` (`gincostestimate`, `gincost_pattern`, `gincost_opexpr`, `gincost_scalararrayopexpr`, `GinQualCounts`), `src/backend/access/gin/ginutil.c` (`ginhandler`, `ginGetStats`, `ginUpdateStats`), `src/backend/access/gin/ginfast.c` (`gin_clean_pending_list`), `src/include/access/gin.h` (`GinStatsData`, `GIN_SEARCH_MODE_*`), `src/include/access/ginblock.h` (`GinMetaPageData`).

Catalogs, errors and settings: `src/include/catalog/pg_amop.dat` (the four core GIN opfamilies plus the B-tree and hash `jsonb` families), `src/include/catalog/pg_opfamily.h` (`IsBuiltinBooleanOpfamily`), `src/backend/commands/indexcmds.c` (unique/`INCLUDE`/multicolumn/exclusion AM checks), `src/backend/commands/cluster.c` (`amclusterable`), `src/backend/access/common/reloptions.c` (`fastupdate`, per-index `gin_pending_list_limit`), `src/backend/utils/misc/guc_tables.c` (`gin_pending_list_limit`).

Contrib, tests and docs: `contrib/btree_gin` (`btree_gin--1.0.sql` operator classes, `sql/bool.sql`, `expected/bool.out`), `contrib/pgstattuple/pgstatindex.c` (`pgstatginindex`), `src/test/regress/expected/amutils.out`, `src/test/regress/sql/create_index.sql`, `src/test/regress/sql/tsearch.sql`, `doc/src/sgml/indices.sgml`, `doc/src/sgml/indexam.sgml`, `doc/src/sgml/gin.sgml`, `doc/src/sgml/btree-gin.sgml`, `doc/src/sgml/pgtrgm.sgml`.

Follow-up history: `git log -L` on `gincostestimate` over `615cebc94b..HEAD` returned exactly two commits, `cd9479af2af` and `4b754d6c16e`, whose first major versions by branch-point ancestry are 16 and 13. The follow-up describes only v17 behavior and makes no cross-version claim. The main answer's [v13: GIN's whole-index estimate narrowed](#v13-gins-whole-index-estimate-narrowed) uses the second commit, with `REL_12_0`'s `gincostestimate()` text read from the v12 checkout.

Follow-up empirical: the GIN stages of the same script on the same server, with `btree_gin`, `pgstattuple` and `pageinspect` installed, covering same-column GIN-versus-B-tree costing on identical statistics (the closed-form reconciliation of both cost models in [Gate 3](#gate-3-cost-and-why-gin-loses-on-the-same-column) was done by hand from the counters the script records; the script's own `predict()` checks only the seven B-tree whole-index scans), the three plan-shape cases of the fixture-S table, `fastupdate` pending-list bloat at two sizes with the B-tree-only alternative priced beside it, the drain and the following `VACUUM`, the 4X stale-metapage fallback, the keyless partial-index path, the boolean-column case, the live AM property matrix, the four `CREATE INDEX`/`CLUSTER` rejections, a multicolumn-GIN comparison, and verbatim execution of the two filed diagnostic blocks as read out of this page. The `clean` stage stopped the server and deleted the sandbox.

Review, 2026-09-25: the filed script, md5 `a7b99e2260848bdcbfa4dcc1f76bfc35`, was re-run unchanged from an empty sandbox first: it exited 0 after 1 minute 42 seconds and reproduced every number, and a second unchanged run under macOS `/bin/bash` 3.2.57 gave the same summary. A 23-agent review followed: eleven claim-by-claim checkers, eleven adversarial verifiers and one gap critic. It raised 157 findings; 19 were refuted and 138 survived, 52 medium, 86 low and none high, and none of them changed a recorded measurement. Before fixture Q was written, a probe on a throwaway database confirmed that a partial index's page charge lags its growth. The fix pass ran seven slice editors, each followed by an independent checker and a repairer. The script gained stages `fq` and `fe` for fixtures Q and E and an `idxcost()` helper. It now clears `PGSERVICE`, `PGSERVICEFILE` and `PGHOSTADDR`, checks that `grej`'s fixture exists and that each of its four error lines contains one of the four expected GIN rejection texts, rebuilds an install left without a build record, tags the `PERFORM` in `xp()`, requires `pgrep` before it asserts a teardown, and records each plan node's width and recheck condition. The edited script ran end to end four times from empty sandboxes, twice under each of bash 5.3.15 and `/bin/bash` 3.2.57, with identical results apart from timestamps and the bash version, and every earlier number came back unchanged; [Last run](#last-run) records the final pair, started together after the script text and the diagnostic blocks were final. The page dropped every measured number that the current script does not produce: those of the 17.10 first filing and those of superseded fixture versions. Seven glossary entries were added, checked on PostgreSQL 17 only (BitmapAnd, Mackert-Lohman formula, Nested loop join, Partial path, Planner support function, ScalarArrayOpExpr and Summarizing index), and the page gained glossary links in the same pass. Files or functions read for the first time on this page: for the endpoint probe, `selfuncs.c`'s `scalarineqsel()` wrappers and `scalargtsel()`, `ineq_histogram_selectivity()`, `mergejoinscansel()`, `get_variable_range()`, `get_actual_variable_range()` and `get_actual_variable_endpoint()`, `src/backend/utils/adt/like_support.c` (`prefix_selectivity`), `plancat.c`'s `restriction_selectivity()`, `pg_operator.dat` (`int4gt`), `src/include/utils/lsyscache.h` (`AttStatsSlot`), `costsize.c`'s merge-join selectivity calls and `clamp_row_est()`, `nbtree.c`'s `btendscan()`, `nbtsearch.c`'s killed-item and recovery tests, `src/backend/access/index/genam.c` (`RelationGetIndexScan`), `nbtxlog.c` (`btree_xlog_vacuum`) and `contrib/pageinspect/btreefuncs.c` (`dead_items`); for fixture Q and the `reltuples` history, `src/backend/access/table/tableam.c` (`table_block_relation_estimate_size`), `src/backend/access/heap/heapam_visibility.c` (`HeapTupleSatisfiesVacuumHorizon`), `analyze.c`'s `do_analyze_rel()` statistics writes and index cleanup, `compute_scalar_stats()`'s MCV rules and the partial-index population, `heapam.c`'s `heap_delete()` visibility-map clearing and the send and discard steps of its in-place update, `src/backend/utils/cache/inval.c`, `relcache.c`'s `RelationBuildLocalRelation()`, `index.c`'s `reindex_index()` and `index_build()`, `src/backend/commands/tablecmds.c` (`ExecuteTruncateGuts`), `heap.c`'s `RelationTruncateIndexes()`, `matview.c`, `heapam_handler.c`'s index-build scan and `nbtsort.c`'s `_bt_uppershutdown()`; for nbtree, `nbtsplitloc.c`'s `_bt_deltasortsplits()`, `_bt_defaultinterval()`, `_bt_bestsplitloc()`, `_bt_split_penalty()` and `_bt_strategy()`, `nbtinsert.c`'s `_bt_insertonpg()` fast-root update, `nbtpage.c`'s `_bt_allocbuf()` and `nbtutils.c` (`_bt_allequalimage`); for BRIN and GIN, `brin_revmap.c`, `brin.h` (`BrinStatsData`) and `ginfast.c`'s tail stop; for gates 0 and 3, `src/include/catalog/pg_index.h` (`indisvalid`, `indcheckxmin`), `plancat.c`'s `indisvalid` and `indcheckxmin` skips, `src/backend/utils/cache/plancache.c`, `src/backend/access/heap/README.HOT`, `indxpath.c`'s `predOK` handling, `costsize.c`'s `cost_bitmap_tree_node()` and `cost_seqscan()`, `createplan.c` (`create_bitmap_subplan`), `tidbitmap.c` (`tbm_shared_iterate`), `prepjointree.c` (`is_simple_subquery`) and `rel.h` (`RELATION_IS_OTHER_TEMP`); and `planner.c`'s `plan_cluster_use_sort()`, `src/include/optimizer/cost.h`, `src/include/storage/bufmgr.h`, `contrib/pgstattuple/pgstattuple.control` and, for the script's environment, `src/interfaces/libpq/fe-connect.c` (`conninfo_add_defaults`, the `hostaddr` option, `pqConnectOptions2()`'s host-type choice and `PQconnectPoll()`'s address resolution), `pqcomm.c`, `pqcomm.h` and `installation.sgml`. The page now links [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md) and [Mandatory GIN Bloat Tests (unverified)](../../common-concepts/mandatory-gin-bloat-tests.md) where it makes rebuild and pending-list claims, and says those claims are not scored against them. No common concept page was created or edited.

Review, 2026-09-23: the filed script was re-run unchanged from an empty sandbox first, and every number on the page reproduced. Six parallel claim-by-claim checks of the page's slices against the pin followed, and every material finding was re-verified before it was fixed. Files or functions read for the first time on this page: `src/backend/storage/buffer/bufmgr.c` (`RelationGetNumberOfBlocksInFork`), `src/backend/commands/vacuum.c` (`vac_update_relstats`, `vac_estimate_reltuples`), `src/backend/storage/lmgr/lmgr.c` (`LockRelationOid`), `heapam.c`'s in-place update invalidation, the nbtree `README` note on the cached metapage, `nbtinsert.c`'s `_bt_newlevel`, `src/include/access/nbtree.h`'s fillfactor comment, `nbtsplitloc.c`'s single-value condition, `src/backend/catalog/index.c` (`index_update_stats`), `src/backend/utils/cache/relcache.c` (`RelationSetNewRelfilenumber`), `src/backend/catalog/heap.c` (`AddNewRelationTuple`), `src/backend/catalog/Catalog.pm`, `src/backend/catalog/genbki.pl` and `src/include/catalog/pg_am.dat`, the `amcanparallel` lines of `hash.c`, `gist.c`, `spgutils.c`, `brin.c` and `contrib/bloom/blutils.c`, `brincostestimate()`'s total, `planmain.c`, `initsplan.c` and `relnode.c` for the call tree, `allpaths.c`'s `set_rel_pathlist_hook`, `pathnode.c`'s `create_index_path`, `clauses.c` (`simplify_boolean_equality`), `src/backend/executor/nodeBitmapHeapscan.c`, `doc/src/sgml/config.sgml` (`gin_pending_list_limit`), `contrib/pgstattuple/pgstattuple.c`, and `src/test/regress/sql/btree_index.sql`. The v12 checkout's `REL_12_0` `selfuncs.h`, `index.c` (`index_update_stats`), `plancat.c` (partitioned indexes skipped) and `estimate_rel_size()` were read for the history corrections, and `git log` over `54eeefaedbee..786db8dcf168` for the pin-range list. No common concept page was created, edited, or needed.

Review, 2026-09-20: ten reported defects were each checked against the pin before being fixed and each was confirmed; none was a false report. One was a P1 in the published script: `pg()` and `pgq()` returned their status to callers that did not look at it, so only the last command of a stage function decided the stage's exit status and a failed fixture build or `UPDATE` round in the middle of a stage was reported as a pass. Reproduced by injecting two such failures — both runs exited 0 before the fix and 1 after it. Files or functions read for the first time on this page: `src/backend/utils/cache/spccache.c` (`get_tablespace_page_costs`) and the `reloptions.c` table `parallel_workers` entry. Read again for the corrections: `nbtpage.c`'s `_bt_getroot` stale-cache discard and `_bt_gettrueroot` flush, `allpaths.c`'s `compute_parallel_worker` reloption and `RELOPT_BASEREL` branches, `heapam.c`'s `newbuf == buffer` gate and the `TU_All` fallthrough, `selfuncs.c`'s `genericcostestimate` guards and repeated-scan branch and its `gincost_scalararrayopexpr` `arrayScans` multiplication, `indxpath.c`'s `build_index_paths` step 4, and `gincostestimate`'s `ginGetStats` read. Fixture D was rebuilt on independent populations and every fixture-D number re-measured; fixture F-one was added to measure the `tuples <= 1` guard; the filed page-charge probe gained a tablespace-override check. No common concept page was created, edited, or needed.

Review, 2026-09-19, second pass: fourteen reported defects were each checked against the pin and each confirmed, then fixed. Files or functions read for the first time on this page: `src/backend/optimizer/path/clausesel.c` (`clauselist_selectivity_ext`), `src/backend/utils/cache/relcache.c` (`RelationInvalidateRelation`, the `rd_amcache` free sites), `src/backend/access/heap/heapam.c` (`heap_update`'s HOT and summarizing decision and its `update_indexes` result), `src/include/access/tableam.h` (`TU_UpdateIndexes`), `src/backend/executor/nodeModifyTable.c` (`ExecUpdateEpilogue`) and `src/backend/executor/execIndexing.c` (`ExecInsertIndexTuples`), `src/backend/access/nbtree/nbtree.c` (`btvacuumcleanup`), `src/backend/access/gin/ginvacuum.c` (`ginvacuumcleanup`), `src/backend/access/gin/ginfast.c` (`ginHeapTupleFastInsert`'s `needCleanup` test and `ginInsertCleanup`'s conditional lock), `src/backend/access/nbtree/nbtsort.c` (`_bt_allequalimage` at build), `src/include/catalog/genbki.h` (the `CATALOG()` macro), and `src/backend/utils/adt/selfuncs.c` (`brincostestimate`'s `brinGetStats` read and `gincostestimate`'s cache-effect block). `nbtsplitloc.c`'s `BTGetFillFactor` read, `costsize.c`'s `enable_bitmapscan` test, `allpaths.c`'s `create_partial_bitmap_paths` and its call site in `indxpath.c`, `pgstatindex.c`'s `fragments` counter, and `ref/create_index.sgml`'s `fastupdate` note were read for the first time as evidence. The script gained failure propagation, an exit trap and a `need_server` step, and fixture L3 was rebuilt on two independent columns; every number was re-measured from the edited script. No common concept page was created, edited, or needed.

Review, 2026-09-19, first pass: all 340 citations of the previous text (177 ranges in 46 files) were re-read against the pin; all were in bounds and all came from `raw/postgres-17/`. Six ranges were tightened or extended, one label pointed at a legacy entry point, and five behavioral claims had no citation. Files read for the first time in this pass: `contrib/bloom/blcost.c`, `src/include/optimizer/plancat.h` and the `get_relation_info_hook` call in `plancat.c`, `src/include/utils/selfuncs.h` (`GenericCosts`), `src/backend/commands/analyze.c` (`std_typanalyze`), `indxpath.c` (`check_index_predicates`, `match_restriction_clauses_to_index`), `costsize.c` (`cost_bitmap_heap_scan`), `gininsert.c` and `ginvacuum.c` (`ginUpdateStats()` callers), and `contrib/pgstattuple/pgstattuple--1.4--1.5.sql`. Common concepts: the three v17 concept pages are measurement protocols for claims about how much space an index wastes or whether it should be rebuilt, and the B-tree protocol excludes planner pricing, which is this page's subject. None was linked then, and none was edited. The 2026-09-25 review found that the page's rebuild-ranking advice and its description of a GIN pending list as bloat fall inside the B-tree and GIN protocols' scope; the page now links both where it makes those claims and says the claims are not scored against them.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pg_class` carries only three size statistics, `relpages`, `reltuples` and `relallvisible`, and `IndexOptInfo` only three index-size fields, `pages`, `tuples` and `tree_height`; neither has a bloat, density or fragmentation field | [pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69), [pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128) |
| `pages` comes from the live block count for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486); `RelationGetNumberOfBlocks()` counts the main fork only ([bufmgr.c#RelationGetNumberOfBlocksInFork](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4002-L4020), [bufmgr.h:280-281](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L280-L281)); measured: forged `relpages = 1` left cost at `24640.42`, and fixture Q's plain `q_full` was charged all 825 of its live blocks while `pg_class` still recorded 551 |
| `tuples` is the parent table's estimate for non-partial indexes, so removing index entries never lowers it, and `pages / tuples` also rises when the table's estimate falls while the index keeps its pages | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486), [plancat.c:476](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L476); measured on fixture F: the index kept 2,745 blocks while the table's estimate fell from 1,000,000 to 1,000 rows, and its one-row lookup rose from `4.44` to `12.29` |
| A partial index's `tuples` is its recorded `pg_class` density times its live blocks less the metapage, clamped to the table's rows; a forged `reltuples` caps the entries read but leaves every page charged | [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160), [plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146), [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486), [selfuncs.c#genericcostestimate-tuple-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6707-L6715), [selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091); measured: forged `reltuples = 20` moved the cost `24140.42` -> `23140.49` and the startup `0.42` -> `0.39`, with all 5,285 pages still charged |
| A partial index uses its recorded density only while `reltuples >= 0` and `relpages > 1`; otherwise it invents a density from the column widths | [plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146); not measured |
| A new index's row starts at `relpages = 0` and `reltuples = 0`, and a build writes its own counts, an empty B-tree build recording `relpages = 1`; a `REINDEX` or `TRUNCATE` first resets the row to `0` and `-1`, a rebuild with no entries leaves those, and a build during binary upgrade writes nothing | [index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024), [relcache.c#RelationBuildLocalRelation-rd_rel](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3656-L3659), [index.c#index_update_stats-write](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2923), [nbtsort.c#_bt_load-first-page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1287-L1290), [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1063-L1128), [relcache.c#RelationSetNewRelfilenumber-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3948-L3954), [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842); not measured |
| A partial index is charged about its estimated matching rows divided by its last recorded density, so pages it gains reach the charge only in proportion to the table's estimated growth, until `ANALYZE`, a `VACUUM` that counts its entries exactly, or a rebuild rewrites its `pg_class` row, or until the clamp binds | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160), [selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019), [selfuncs.c:6695](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6695), [selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732), [tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747), [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [vacuumlazy.c:512-513](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L512-L513), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842); measured on fixture Q: `q_part` grew from 57 to 331 blocks and was charged 57, then 113, then 331 after `ANALYZE`, while `q_full` grew from 551 to 825 blocks and was charged all 825 before `ANALYZE`. After the churn `pgstatindex` read `q_part` at 77.56% density and 49.85% fragmentation and `q_full` at 85.14% and 20% ([pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372)), and `bt_metap()` read their `fastlevel` as 1 and 2 ([btreefuncs.c:908](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L908)). The exact-count `VACUUM`, the rebuild and the clamp are not measured |
| Fixture Q's churned partial charge: the table estimate is `rint(200000 / 1471 * 2942) = 400000`, the partial `tuples` `rint(20000 / (57 - 1) * (331 - 1)) = 117857`, the entries read `0.1 * 400000 = 40000`, and the charge `ceil(40000 * 331 / 117857) = 113` pages, in a node total of `752.29`; after `ANALYZE` it is `ceil(20000 * 331 / 20000) = 331`. The plain index keeps its 825 pages after `ANALYZE`, and its total falls by `1500.00`: `200000 * (0.005 + 0.0025)` for the halved row estimate, plus one fewer descent comparison, `ceil(log2(400000)) = 19` against `ceil(log2(200000)) = 18`, at `0.0025`, which the rounding hides | [plancat.c#get_relation_info-table-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L196-L202), [tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160), [selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019), [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [analyze.c#do_analyze_rel-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645), [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810), [selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091); measured: `752.29` / `413.29` at `random_page_cost` 4 / 1, so 113 pages, and `1474.29` / `481.29` after `ANALYZE`, so 331, with the `pg_class` rows before and after; the plain index's `Bitmap Index Scan` total `6300.42` before `ANALYZE` and `4800.42` after |
| Fixture Q's ten `UPDATE` rounds are non-HOT, so each adds an entry to both indexes, the partial one included because every updated row still satisfies `id <= 20000`; and they share one transaction, so no replaced version is dead to anyone before the commit | [heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166), [heapam.c:4429](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4429), [nodeModifyTable.c#ExecUpdateEpilogue-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L2162-L2166), [execIndexing.c#ExecInsertIndexTuples-loop](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L343-L387), [heapam_visibility.c#HeapTupleSatisfiesVacuumHorizon-same-xact](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1253-L1266), [heapam_visibility.c#HeapTupleSatisfiesVacuumHorizon-xmax-in-progress](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1383-L1386); measured: the heap doubled exactly, 1,471 -> 2,942 blocks |
| The charged index pages are the `Bitmap Index Scan` node's cost difference between `random_page_cost` 4 and 1, divided by 3, and that node's cost is the access method's own total with no heap term | [selfuncs.c#genericcostestimate-single-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787), [costsize.c#cost_index-save-indextotalcost](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L623-L629), [createplan.c#create_bitmap_subplan-indexpath](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3480-L3485) |
| For a partial index the `ceil(log2(tuples))` comparison charge also grows with the live block count | [selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160); it enters fixture Q's `752.29` as `ceil(log2(117857)) * 0.0025` |
| Page count enters cost pro-rata | [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732); measured `28480.42` vs `123144.43` |
| Height charge prices descent CPU, and bloat is one of its two stated reasons | [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106) |
| Height charge is `50 * cpu_operator_cost` per level | [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145), `0.125` at the default `cpu_operator_cost` of `0.0025` ([cost.h:28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L28)); measured startup gap exactly `50.00` at `cpu_operator_cost = 1`. The height charge still sees bloat on a lone lookup: fixture B's falls from `4.44` to `4.31` when a rebuild removes a level, and fixture H's two lookups cost `8.31` and `8.43` |
| Planner height is the fast-root level, while `pgstatindex` reports the true root level `btm_level` as `tree_level` | [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381), [pgstatindex.c#metapage](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265), [pgstatindex.c:351](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L351), [pgstattuple--1.4--1.5.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L24), [pgstattuple.control:3](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L3); measured on fixture F: `tree_level` 2 against `fastlevel` 1 |
| Index pages enter cache modeling: `index_pages_fetched()` prorates `effective_cache_size` over the query's table pages plus the index, `total_table_pages` counts table pages only, and every B-tree-family call site passes `index->pages` (GIN passes its entry- or data-page count); `cost_index()` passes it on each of its three `index_pages_fetched()` calls, two for repeated scans and one for the normal case, and computes the normal case's perfectly-correlated estimate without the cache model | [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951), [allpaths.c#total_table_pages](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216), [pathnodes.h#total_table_pages](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L483-L484), [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-repeated-uncorrelated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L680-L683), [costsize.c#cost_index-repeated-correlated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L702-L707), [costsize.c#cost_index-normal-uncorrelated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L720-L723), [costsize.c#cost_index-normal-correlated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L733-L746), [selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779), [costsize.c#compute_bitmap_pages-repeated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476), [selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974); measured `0.66` against `2.50` per loop for a repeated inner index-only scan, and both higher at `effective_cache_size = 64MB` |
| The scan's touched-pages estimate, not the index size, chooses parallel workers | [costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772), [selfuncs.c#btcostestimate-outputs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7206-L7210), [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279); measured 6 / 5 / 5 workers at 100% / 50% / 20% of the 26,411-block index against 4 / 3 / 2 on its 2,745-block twin |
| The page-based worker calculation is conditional | a table `parallel_workers` reloption is used instead and skips it entirely, capped only by the caller's maximum, `max_parallel_workers_per_gather` for `cost_index()` ([allpaths.c#compute_parallel_worker-reloption](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4208-L4213), [allpaths.c:4276](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4276), [plancat.c:205](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L205), [reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382)); the `min_parallel_index_scan_size` rejection is guarded on `RELOPT_BASEREL`, exempting inheritance children ([allpaths.c#compute_parallel_worker-threshold](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227)). Neither case is measured here |
| `min_parallel_index_scan_size` is necessary but not sufficient: a plain index scan also needs a heap-page estimate of at least `min_parallel_table_scan_size` and takes the smaller of the two worker counts, while an index-only scan passes no heap estimate | [costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772), [allpaths.c#compute_parallel_worker-threshold](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227), [allpaths.c#compute_parallel_worker-index-ramp](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4253-L4272); fixture A's parallel scans are index-only, so only the index test is measured |
| A flat one-page estimate is a ceiling as well as a floor: `tuples <= 1` collapses an index of any size to one page | [selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732), [selfuncs.c#btcostestimate-log2-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7086-L7091); measured on fixture F-one: 551 blocks and one live leaf priced at `4.14`, the page term `1 * 4.0` |
| A repeated lookup is priced with the whole index as the page universe, even though a single one is not: few loops over a large index are charged about one page each whatever its size; once loops times touched pages reach the index size the charge approaches the whole index spread over the loops, it is capped at the whole index from twice the index size while the index fits its cache share, and it passes the cap only when the query's tables plus the index exceed `effective_cache_size` | [selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951); measured on fixture A over 50,000 loops: `0.66` against `2.50` per loop on indexes that both price a lone lookup at `4.44`. The page charges are `2745 * 4.0 / 50000 = 0.22`, at the cap, and `25687 * 4.0 / 50000 = 2.05`, just short of it, 9.36x the charge for 9.62x the pages; at `effective_cache_size = 64MB`, the case past the cap, `1.38` against `3.55`. The few-loops regime is not measured |
| v17 clamps SAOP descents to `ceil(pages/3)` | [selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042); measured plateau at 3 descents on the 8-block index; the 55-block index's cap of 19 is computed and not reached at 10 elements |
| The SAOP per-tuple CPU charge is divided by the descent count and multiplied back, so `rint()` rounding moves it, and a floor of one tuple per descent makes it `num_sa_scans * (cpu_index_tuple_cost + qual_op_cost)` when descents outnumber rows | [selfuncs.c:7064](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7064), [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810), [selfuncs.c#genericcostestimate-tuple-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6707-L6715); measured on fixture I's 8-block index, clamped to 3 descents: `rint(10 / 3) * 3 = 9` tuples at ten elements and `rint(4 / 3) * 3 = 3` at four, the same 3 tuples as at three elements, so `352.04` at three elements and `352.05` at four differ by one heap row's `cpu_tuple_cost`. Mechanism 4's five-element example is arithmetic |
| The endpoint probe is not a cost input: when `ineq_histogram_selectivity()`'s binary search reaches the first or last histogram bound, or the histogram has two bounds, it replaces that bound in its copy with the live minimum or maximum, and keeps `pg_statistic`'s bound when the probe fails | [selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136), [lsyscache.h#AttStatsSlot](../../../../raw/postgres-17/src/include/utils/lsyscache.h#L46-L62), [selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413) |
| The probe is reached from `scalarltsel()`, `scalarlesel()`, `scalargtsel()` and `scalargesel()` for a comparison against a constant, from `LIKE` fixed-prefix estimation, and from merge-join costing, which compares each side's column with the other side's `pg_statistic` extremes; `get_variable_range()`'s own call to the probe is compiled out | [selfuncs.c#scalarineqsel_wrapper](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1397-L1503), [selfuncs.c:690](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L690), [like_support.c#prefix_selectivity](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1213-L1270), [like_support.c:1245](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1245), [like_support.c:1266](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1266), [costsize.c:3577](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L3577), [costsize.c:4016](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L4016), [selfuncs.c#mergejoinscansel-ranges](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3141-L3156), [selfuncs.c#get_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5969-L6098), [selfuncs.c#get_variable_range-not-used](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5993-L6003), [selfuncs.c#mergejoinscansel-scalarineqsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3164-L3203); the chain from `clauselist_selectivity()`: [clausesel.c#clauselist_selectivity](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L100-L108), [clausesel.c:136](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L136), [clausesel.c:183](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L183), [clausesel.c:848](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L848), [plancat.c#restriction_selectivity](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1951-L1979), [pg_operator.dat#int4gt](../../../../raw/postgres-17/src/include/catalog/pg_operator.dat#L453-L456), [selfuncs.c#scalargtsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1490-L1494), [selfuncs.c:1461](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1461). Fixture E exercises only the `scalargtsel()` path |
| The probe reads only the first index that is a B-tree, not partial, not hypothetical, and whose first column matches the compared expression, collation and sort operator, and it gives up at once on a partitioned parent; no other access method and no partial index is probed | [selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6157-L6331), [selfuncs.c#get_actual_variable_range-skips](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6196-L6212), [selfuncs.c:6197](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6197) |
| The probe walks the index as an index-only scan under `SnapshotNonVacuumable`: an entry on an all-visible heap page is accepted without a heap visit, rows dead to every snapshot are skipped, and recently dead or uncommitted rows are accepted | [selfuncs.c#get_actual_variable_endpoint-snapshot](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6376-L6386), [selfuncs.c#get_actual_variable_endpoint-horizon](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6416), [selfuncs.c#get_actual_variable_endpoint-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6458), [selfuncs.c#get_actual_variable_endpoint-comment](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6371-L6414); fixture E covers only rows dead to every snapshot |
| Each probe counts a heap page when a rejected entry points to a different page than the last one counted, and gives up on the fetch that takes the count past `VISITED_PAGES_LIMIT` (100), a local `#define` with no GUC; each call starts at zero and the minimum and maximum are separate calls, so the limit binds each probe, not each plan, and one probe reads up to 101 heap pages | [selfuncs.c#get_actual_variable_endpoint-limit](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6436-L6457), [selfuncs.c#VISITED_PAGES_LIMIT](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447-L6455), [selfuncs.c:6447](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447), [selfuncs.c:6365](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6365), [selfuncs.c#get_actual_variable_range-endpoint-calls](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6282-L6314), [selfuncs.c#get_actual_variable_endpoint](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6349-L6501); measured on fixture E: 885 heap blocks of 226 rows, the last holding 216, and plan 1 marked `216 + 99 * 226 = 22590` entries, exactly 100 heap pages, and plans 2 to 4 marked `22600` each |
| Every entry the probe passes over because its whole HOT chain is dead is marked killed, so the next probe passes it without a heap visit; the entry whose fetch trips the 100-page limit is not, because no further `btgettuple()` call records it | [indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652), [nbtree.c#btgettuple-kill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L226-L245), [nbtsearch.c#_bt_steppage-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2049-L2051), [nbtree.c#btendscan-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L424-L426), [selfuncs.c#get_actual_variable_endpoint-killed](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6388-L6397); `bt_multi_page_stats()`'s `dead_items` counts the line pointers marked dead ([btreefuncs.c#GetBTPageStatistics-items](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L172-L187)); measured on fixture E: 0 entries marked dead after the delete, then 22,590, 45,190, 67,790, 90,390 and 100,000 after plans 1 to 5, and 0 after a plain `VACUUM` |
| In a transaction started during recovery the probe neither marks rejected entries killed nor skips marked ones, so on a hot standby a dead run over more than 100 heap pages keeps the stale bound until the standby replays the entries' removal | [genam.c#RelationGetIndexScan-recovery](../../../../raw/postgres-17/src/backend/access/index/genam.c#L107-L119), [nbtsearch.c:1721](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L1721), [nbtsearch.c:1850](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L1850), [indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652), [nbtxlog.c#btree_xlog_vacuum](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L598-L634); not measured: the script builds no standby |
| A probe that gives up leaves a stale row estimate; one that reaches a live entry moves the bound and corrects it | [selfuncs.c#ineq_histogram_selectivity-above-last-bound](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1162-L1168), [selfuncs.c#ineq_histogram_selectivity-flip-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1319-L1344), [costsize.c#clamp_row_est](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L202-L218); measured on fixture E: `id > 199990` stayed at `rows=10` for plans 1 to 4, `200000 * (1 - 9999.5 / 10000) = 10` from the stale bound 200000; plan 5 found 9,610 dead entries on 43 heap pages, reached `id` 100,000 and fell to `rows=1`, and the estimate stayed at 1 after `VACUUM` |
| Dead entries at the end of a B-tree add no page or level to the cost; they reach it only through the row estimate | [selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019), [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [selfuncs.c:6810](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6810), [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810), [costsize.c#cost_index-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800), [cost.h#default-costs](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25-L28); measured on fixture E: the Index Only Scan at `0.42..4.59` for `rows=10` and `0.42..4.44` for `rows=1`, which rebuild as `4.0 + 0.42 + 10 * (0.005 + 0.0025) + 10 * 0.01 = 4.595` and `4.4375` |
| Fixture E's Index Only Scan totals carry no heap term although the delete cleared the visibility-map bits of its pages, because the planner reads the all-visible fraction from `pg_class.relallvisible`, which `VACUUM` and `ANALYZE` refresh from the map, as does an index build on the table; the stage runs neither, and builds no index, between the delete and plan 6, and the server runs with `autovacuum=off` | [heapam.c#heap_delete-clear-vm](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3120-L3146), [plancat.c#get_relation_info-table-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L196-L202), [tableam.c#table_block_relation_estimate_size-allvisfrac](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L749-L760), [vacuumlazy.c#heap_vacuum_rel-relstats](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575), [analyze.c#do_analyze_rel-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645), [index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3131), [index.c#index_update_stats-relallvisible](../../../../raw/postgres-17/src/backend/catalog/index.c#L2851-L2928), [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-correlation-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L781-L800); measured `4.59` at `rows=10`, which leaves no room for the `4.00` one heap page would add |
| Four core AMs and contrib `bloom` use `genericcostestimate`; GIN and BRIN do not | [selfuncs.c:7073](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7073), [selfuncs.c:7221](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7221), [selfuncs.c:7265](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7265), [selfuncs.c:7320](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7320), [blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42); [selfuncs.c#gincostestimate-header](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L7671), [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8052-L8061) |
| A single contrib `bloom` scan is charged every index page; a repeated one over an index within its cache share is capped at the index's page count and spread over the loops | [blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42), [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951); not measured |
| GiST/SP-GiST estimate height from page count | [selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7256-L7308), [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7311-L7363) |
| Hash charges no descent cost | [selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7213-L7253) |
| `avg_leaf_density`/`leaf_fragmentation` ignore deleted pages | [pgstatindex.c#page-classification](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372); measured 89.18% on a 2,465-deleted-page index |
| Fragmentation contributes zero cost | no cost input carries it: `IndexOptInfo` has no fragmentation field ([pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)) and the page charge is a flat pro-rata share of `pages` ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)); measured gap `1616.00` = `404 * 4.0` with 49.87% vs 0% fragmentation |
| Deleted pages stay in the fork | [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659), [README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441); recycling goes through the FSM only ([nbtree.c:1168](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1168), [nbtpage.c:3050](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3050), [nbtpage.c#_bt_allocbuf-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L905)); measured 2,745 blocks after three VACUUMs; no `RelationTruncate()` or `smgrtruncate()` call exists under `src/backend/access/nbtree/` |
| VACUUM unlinks a deleted page from its siblings, so no later scan reaches it; a half-dead page stays in the chain and is read and skipped; the rightmost page of a level is never deleted | [nbtpage.c#unlink-siblings](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2612), [README#half-dead](../../../../raw/postgres-17/src/backend/access/nbtree/README#L247-L259), [nbtsearch.c#step-right](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2207-L2219), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381); measured: fixture M's whole-index scan, which by that source walks only its 276 live leaves, is priced at `12730.42` like fixture B's over 2,733 live leaves. What either scan reads is not measured |
| A split takes its new right half from `_bt_allocbuf()`, which asks the FSM before extending the file, and only VACUUM records recyclable pages there, so an index with no deletions behind it appends every split | [nbtinsert.c:1720](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1720), [nbtpage.c#_bt_allocbuf](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L868-L988), [nbtpage.c:903](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L903), [nbtpage.c:978](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L978), [nbtree.c:1168](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1168), [nbtpage.c:3050](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3050); nbtree asks the table AM which entries simple and bottom-up deletion may remove ([nbtpage.c:1526](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1526)); measured: fixture G's random-order index reads 49.87% `leaf_fragmentation` |
| Build and split policy set how full a page is left, not a density ceiling: a rightmost leaf split uses the leaf fillfactor, as does a leaf split that the "split after new item" optimization recognizes, unless it splits exactly after the new item; a rightmost internal split uses `BTREE_NONLEAF_FILLFACTOR` (70); every other split aims for an even balance and takes the lowest-penalty point near it; a page of many duplicates may split beside its group, and the last page of a single value splits at 96% | [nbtree.h#fillfactor-comment](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L197), [nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtsplitloc.c#_bt_deltasortsplits](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L561-L588), [nbtsplitloc.c#_bt_defaultinterval](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L852-L920), [nbtsplitloc.c#_bt_bestsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L770-L812), [nbtsplitloc.c#_bt_split_penalty](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1119-L1153), [nbtsplitloc.c#many-duplicates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L374-L405), [nbtsplitloc.c#_bt_strategy-many-duplicates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L985-L1009), [nbtsplitloc.c#single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L389-L416), [nbtsplitloc.c#single-value-condition](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1011-L1041), [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202); measured: fixture P's `tag` index read 91.47% after its build at the default fillfactor of 90 ([nbtree.h:200](../../../../raw/postgres-17/src/include/access/nbtree.h#L200)) and 98.01% after the unblocked churn |
| `_bt_strategy()` overrides a leaf split whose default interval cannot avoid a heap TID in the new high key: `SPLIT_MANY_DUPLICATES` for a page not entirely one value, and `SPLIT_SINGLE_VALUE` at 96% for the last page holding one value, whatever the fillfactor | [nbtsplitloc.c#split-strategies](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L364-L416), [nbtsplitloc.c#_bt_strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L985-L1041), [nbtree.h:202](../../../../raw/postgres-17/src/include/access/nbtree.h#L202) |
| `fillfactor` and `deduplicate_items` take `ShareUpdateExclusiveLock` because they apply only to later inserts; GIN's `fastupdate` and per-index `gin_pending_list_limit` take `AccessExclusiveLock`, and `gin_clean_pending_list()` takes `RowExclusiveLock` | [reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194), [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167), [reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L130), [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347), [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091) |
| Bloat can drop an index from a `BitmapAnd` | [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489); measured `c = 7` demoted to `Filter` |
| VACUUM can skip index vacuuming | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1949), considered only under `INDEX_CLEANUP` `AUTO` and before any round of index vacuuming ([vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402), [vacuumlazy.c#bypass-off-after-round](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L894-L896), [vacuumlazy.c:1900](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1900)), [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2347) |
| `index_pages_fetched` / `cost_index` unchanged since `REL_12_0` | function text byte-identical between the v12 checkout's `REL_12_0` tag and the pin; in `615cebc94b..HEAD`, `git log -L` finds no commit on `index_pages_fetched` and only a rename and its revert on `cost_index` |
| SAOP clamp is new in v17 | `5bf748b86bc`, first major version 17 by branch-point ancestry; absent from `REL_12_0` `btcostestimate` |
| Against v12, the clamp can only lower the descent count of a constant list in a boundary qual; an `= ANY` outside the boundary quals now adds no descents, where v12 added one per element; a non-constant array is sized from its element statistics and can exceed v12's flat 10 | [selfuncs.c#estimate_array_length](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2131-L2207), [selfuncs.c#estimate_array_length-statistics](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2162-L2206), [selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042), [selfuncs.c#btcostestimate-bound-saop](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6904-L6989), [selfuncs.c#btcostestimate-boundary-saop](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6950-L6961), [selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073), [selfuncs.c#genericcostestimate-num_sa_scans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6657-L6679), [selfuncs.h#GenericCosts](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L108-L138), [indxpath.c#match_saopclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2651-L2669); `REL_12_0`'s `genericcostestimate()` multiplies the length of every `ScalarArrayOpExpr` among the index quals, and its `estimate_array_length()` returns 10 for a non-constant array; `9391f71523b` is first in 17 by branch-point ancestry. Only the first case is measured (fixture I) |
| `50.0` -> macro is cosmetic and v16 | `eb5c4e953bb` diff, first major version 16 by branch-point ancestry; `REL_12_0` already had `* 50.0 *` |
| Deduplication is v13 | `0d861bbb702`, first major version 13 by branch-point ancestry; `deduplicate_items` absent from `REL_12_0` `reloptions.c`; measured 852 vs 2,749 blocks. An index `pg_upgrade` carried from v12 does not deduplicate until it is rebuilt, because only `CREATE INDEX` or `REINDEX` sets `btm_allequalimage` ([nbtree.h#allequalimage-upgrade](../../../../raw/postgres-17/src/include/access/nbtree.h#L135-L141), [btree.sgml#deduplication-safety](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L838), [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)), while bottom-up deletion runs before that test ([nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782)); not measured |
| Deduplication at build and at insert also needs `allequalimage`, decided at `CREATE INDEX` or `REINDEX` from each key's operator class and collation; the build also skips unique indexes; the types the manual lists and every `INCLUDE` index never deduplicate | [btree.sgml#deduplication-at-build](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L786-L797), [btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800), [README#notes-about-deduplication](../../../../raw/postgres-17/src/backend/access/nbtree/README#L904-L948), [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L33-L58), [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183), [nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564), [nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152), [nbtinsert.c#dedup-gate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2778-L2781), [btree.sgml#deduplication-restrictions](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L908); measured on fixture N: 852 against 2,749 blocks, both built after the load |
| The pre-split deduplication pass runs only after three exits have not fired, and the second exit returns without making room, so its page can split undeduplicated; the source relies on `allequalimage` being zero on an index `pg_upgrade` carried from 12 | [nbtinsert.c#early-returns](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2721-L2776), [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782), [nbtpage.c#_bt_metaversion-comment](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L720-L737) |
| Bottom-up deletion is v14 | `d168b666823`, first major version 14 by branch-point ancestry; [btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703), [README#bottom-up-added-in-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L981), [README#deduplication-and-bottom-up-note](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988); measured 543 vs 1,173 blocks on a 1,000-value column, and 1,020 vs 1,020 on a 100-value one |
| Newly deleted pages recyclable in the same VACUUM is v14 | `9dd963ae253`, first major version 14 by branch-point ancestry; [README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424) |
| `e5d8a99903` (first in 14) stores a 64-bit `safexid` in deleted pages and replaced `btm_oldest_btpo_xact` with `btm_last_cleanup_num_delpages`, which `_bt_vacuum_needs_cleanup()` compares with 5% of the index | [nbtree.h#BTDeletedPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L230-L236), [nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119), [nbtpage.c#_bt_vacuum_needs_cleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L172-L223); `REL_12_0`'s `nbtree.h` has `btm_oldest_btpo_xact`; `d10b19e224` is an ancestor of `e5d8a99903` and `596b5af1d3` is not. Not measured |
| `9f3665fbfc` (first in 14) marks a cleanup-only B-tree VACUUM's count as estimated, so VACUUM no longer rewrites that index's `relpages` and `reltuples`, which `REL_12_0` did; `effdd3f3b6` restored `vacuum_cleanup_index_scale_factor` only as a deprecated storage parameter that no code reads | [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [reloptions.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L462-L470), [nbtree.h:1134](../../../../raw/postgres-17/src/include/access/nbtree.h#L1134); `git log -S 'stats->estimated_count = true'` attributes that line to `9f3665fbfc`; `REL_12_0`'s `btvacuumcleanup()` leaves `estimated_count` false and its `lazy_cleanup_index()` writes the row from an exact count; `d10b19e224` is an ancestor of both commits and `596b5af1d3` of neither; both commit messages. Not measured |
| `REL_12_0` skipped index vacuuming only on request, and skipped index cleanup with it; v14 added the automatic 2% bypass and the wraparound failsafe, and `3499df0dee8` made the setting tri-valued with `auto` as the default | [reloptions.c#vacuum_index_cleanup](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L510-L520), [vacuum.c#index_cleanup-default](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179), [vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402), [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949), [vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326); `REL_12_0`'s `vacuum.c` `INDEX_CLEANUP` option and its `vacuumlazy.c` `useindex` rule; `5100010ee4d`, `1e55e7d1755` and `3499df0dee8` first in 14 by branch-point ancestry |
| `reltuples = -1` is v14, and an index holds it only after a rebuild through `reindex_index()` produced no entries; its catalog `relpages` is `0` then, so the new `reltuples >= 0` test never changes an index's branch | `3d351d916b2`, first major version 14 by branch-point ancestry, whose `index.c` diff adds the keep-`-1` hack; [plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146), [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66), [relcache.c#RelationSetNewRelfilenumber-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3948-L3954), [index.c#reindex_index-rebuild](../../../../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789), [index.c#index_build-index-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3133-L3135), [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842); not measured |
| Plain `REINDEX`, `TRUNCATE`, `VACUUM FULL`, `CLUSTER`, a table-rewriting `ALTER TABLE` and a non-concurrent `REFRESH MATERIALIZED VIEW` rebuild each index through `reindex_index()`; `TRUNCATE`'s in-place path for a table new in the subtransaction can keep a `-1` but never sets one; `REINDEX CONCURRENTLY` builds through `index_create()` and starts at `0` and `0` | [tablecmds.c#ExecuteTruncateGuts-rewrite](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2167-L2189), [vacuum.c:2260](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2260), [cluster.c:670](../../../../raw/postgres-17/src/backend/commands/cluster.c#L670), [tablecmds.c:5873](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L5873), [matview.c:890](../../../../raw/postgres-17/src/backend/commands/matview.c#L890), [cluster.c:1508](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1508), [index.c:4048](../../../../raw/postgres-17/src/backend/catalog/index.c#L4048), [tablecmds.c#ExecuteTruncateGuts-in-place](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2133-L2145), [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3083-L3087), [index.c:1459](../../../../raw/postgres-17/src/backend/catalog/index.c#L1459), [index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024); not measured |
| A B-tree build counts only the rows the heap scan hands it, and the scan skips rows dead to every transaction and rows that fail a partial index's predicate, so a rebuild can produce no entries on a table that is not empty | [nbtsort.c:599](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L599), [nbtsort.c:338](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L338), [heapam_handler.c#index-build-dead](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1419-L1423), [heapam_handler.c#index-build-predicate](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1636-L1644); not measured |
| `71b66171d0` (first in 17.0) skips index statistics during binary upgrade, so an index that `pg_upgrade` creates keeps `relpages = 0` and `reltuples = 0` until `ANALYZE`, or a `VACUUM` whose index pass reports an exact count, writes its row; a cleanup-only B-tree pass writes nothing | [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842), [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893); `5bcc7e6dc8` is an ancestor of `71b66171d0`, which the `Stamp 17.0.` commit `d7ec59a63d` contains; not measured |
| For a partial B-tree whose rebuild produced no entries and which inserts then filled, `REL_12_0` and v17 both take the width-based fallback, and v17, holding `relpages = 0`, skips the metapage discount and estimates one page's worth more | [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160), [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486); `REL_12_0` text in the v12 checkout: the relfilenode reset writes `relpages = 0` and `reltuples = 0` (`relcache.c`), `index_update_stats()` writes every build's counts (`index.c`), an empty B-tree build writes the metapage alone (`nbtsort.c`), and `estimate_rel_size()` discounts the metapage whenever `relpages > 0` (`plancat.c`); not measured |
| `3c569049b7b` (first in 16) lists partitioned indexes with `pages = 0`, `tuples = 0` and `tree_height = -1`, where `REL_12_0` skipped them, so neither version sizes them | [plancat.c:471](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471), [plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508); `REL_12_0`'s `get_relation_info()` comment "Ignore partitioned indexes, since they are not usable for queries"; `d31d30973a` is an ancestor of `3c569049b7b` and `5bcc7e6dc8` is not |
| The endpoint probe's 100-heap-page limit is `9c6ad5eaa9` (2022-11-22), first in 16 and back-patched; `REL_12_0`'s probe had no page limit and used `RecentGlobalXmin` as its horizon, which `dc7420c2c9` (first in 14) replaced with `GlobalVisTestFor()` | [selfuncs.c#VISITED_PAGES_LIMIT](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447-L6455), [selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413), [selfuncs.c#get_actual_variable_endpoint-horizon](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6416); `d31d30973a` is an ancestor of `9c6ad5eaa9` and `5bcc7e6dc8` is not; its message says "Back-patch to all supported branches"; in the v12 checkout its twin `ec10b6139c` is in `REL_12_14` and in neither `REL_12_0` nor the wiki's v12 pin `45b88269`; `git log -L` on `get_actual_variable_endpoint()` over `615cebc94b..HEAD` lists `9c6ad5eaa9`, `dc7420c2c9` and `d3751adcf1`, whose `REL_12_STABLE` twin `cee976c4e8` is in `REL_12_0`; the v17 side is measured by fixture E |
| `19d8e2308b` (2023-03-20, first in 16) lets an `UPDATE` of only summarizing-index columns stay HOT and skip every non-summarizing index; `REL_12_0` tested HOT against every indexed column, BRIN included, and told the executor to insert into all indexes or none | [relcache.c#summarizing-split](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5390-L5398), [brin.c:269](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L269), [heapam.c#heap_update-attr-bitmaps](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3434-L3437), [heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166), [heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429), [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127), [execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366); `d31d30973a` is an ancestor of `19d8e2308b` and `5bcc7e6dc8` is not; its message says it re-applies `5753d4ee32`, which `e3fcca0d0d` reverted, both descendants of the 15devel stamp `596b5af1d3` and not of the 16devel one; `REL_12_0`'s `heap_update()` uses `INDEX_ATTR_BITMAP_ALL` and its `heapam_handler.c` sets one boolean `update_indexes`; not measured |
| `4b754d6c16e` (2020-01-18, first in 13) made GIN's whole-index estimate fire only for a column with a full-scan key and no normal key; `REL_12_0` fired it for any `GIN_SEARCH_MODE_ALL` key | [selfuncs.c#gincost_pattern-searchmode](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7454-L7489), [selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877); `REL_12_0` text `if (counts.haveFullScan \|\| indexQuals == NIL)`; `615cebc94b` is an ancestor of `4b754d6c16e` and `d10b19e224` is not; not measured |
| No test asserts the bloat charge | no `src/test` match for `tree_height`, `btcostestimate`, `genericcostestimate`; `btree_index.sql` builds and splits a fast root as a correctness test only ([btree_index.sql#multilevel-page-deletion](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L252-L270)) |
| No in-tree test asserts the endpoint probe or its limit | no file under `src/test`, `contrib` or `doc` names `get_actual_variable_range`, `get_actual_variable_endpoint`, `VISITED_PAGES_LIMIT` or `SnapshotNonVacuumable`; `git show --stat` lists only `selfuncs.c` for `fccebe421` and `9c6ad5eaa9`, and only source and header files for `3ca930fc3`; the conditions for reaching the probe are in [selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136) and [selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239) |
| No test covers gate 0's `indcheckxmin` skip | no file under `src/test` mentions `indcheckxmin`, and its only `contrib` mention is `contrib/amcheck/verify_nbtree.c` |
| The `pgstatindex` test uses an empty index | [sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82), [expected/pgstattuple.out#partition-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L264-L268), [pgstatindex.c#density-guards](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372) |
| All four cost GUCs are `PGC_USERSET`, with defaults of 4.0, 0.0025, 524288 pages and 64 blocks | [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696), [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518), [guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540), [cost.h:25](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25), [cost.h:28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L28), [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34) |
| `random_page_cost` multiplies only the index pages the estimate counts as fetched; B-tree upper levels are charged CPU only, BRIN's range-map pages `seq_page_cost`, and a tablespace setting replaces the GUC | [selfuncs.c#genericcostestimate-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6786), [selfuncs.c#btcostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7106), [selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258), [spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196), [selfuncs.c#genericcostestimate-tablespace](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6734-L6737) |
| Every setting the script sets has the context its header names: `shared_buffers`, `port`, `listen_addresses` and `unix_socket_directories` are `PGC_POSTMASTER`, `autovacuum` is `PGC_SIGHUP`, and the rest are `PGC_USERSET` | [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270), [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2393-L2401), [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4436-L4445), [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4425-L4434), [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457), [guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2070-L2079), [guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792), [guc_tables.c#enable_indexscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L793-L802), [guc_tables.c#enable_indexonlyscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L803-L812), [guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822), [guc_tables.c#enable_mergejoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L893-L902), [guc_tables.c#enable_hashjoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L903-L912), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518), [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696), [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729), [guc_tables.c#max_parallel_workers_per_gather](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3419-L3428), [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439), [guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529), [guc_tables.c#parallel_tuple_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3730-L3740), [guc_tables.c#parallel_setup_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3741-L3751), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585), [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#application_name](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4644-L4653) |
| `ceil()` charges a lookup of `k` estimated rows one page while `k * pages <= tuples`, so a one-row lookup is charged `ceil(pages / tuples)` pages once pages outnumber rows | [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732), [selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732); measured on fixture F: `12.29` on 2,745 blocks over 1,000 rows, `4.29` after `REINDEX`; fixture P's 200-row bitmap scans are charged one page at 543 blocks and two at 1,173, `5.92` against `9.92` |
| Page deletion can lower the planner's height without a rebuild | [nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381); measured `tree_level` 2 against `fastlevel` 1, startup `0.42` -> `0.28` |
| The closed form has no standalone `qual_op_cost` term | [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810); `predict()` equals `EXPLAIN` on 7 of 7 whole-index scans |
| A plugin can rewrite `pages`, `tuples` and `tree_height` before costing | [plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576), [plancat.h#get_relation_info_hook_type](../../../../raw/postgres-17/src/include/optimizer/plancat.h#L20-L25); `gincostestimate()` skips the metapage read for a hypothetical index ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711)), and the endpoint probe skips a hypothetical index ([selfuncs.c#get_actual_variable_range-skips](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6196-L6212)); not exercised |
| The statistics hooks move the page charge through selectivity, and `set_rel_pathlist_hook` can delete or modify costed paths | [selfuncs.c:5203-5204](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5203-L5204), [selfuncs.c:5375-5376](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5375-L5376), [selfuncs.c#btcostestimate-stats-hooks](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7116-L7171), [selfuncs.c:8135-8136](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8135-L8136), [selfuncs.c:8166-8167](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8166-L8167), [allpaths.c#set_rel_pathlist_hook](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L532-L539); not exercised |
| `cost_index()` calls the index AM's `amcostestimate` through the pointer `get_relation_info()` copied into `IndexOptInfo`, and a custom AM gets `tree_height = -1` | [costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621), [plancat.c#amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L331-L332), [plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500); not exercised |
| A predicate-implied clause is dropped from a partial index's quals | [indxpath.c#check_index_predicates-indrestrictinfo](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3318-L3378), [indxpath.c#match_restriction_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1968-L1974); measured `500.00` = `200000 * cpu_operator_cost` between the plain and the partial twin |
| `ANALYZE` reads every fixture row at `default_statistics_target = 10000` | [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1876-L1894); every run of the filed script agrees on every recorded value; see [Last run](#last-run) |
| GIN's startup/total split never reaches a plan | [costsize.c#cost_bitmap_heap_scan-startup](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044-L1048), [ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79) |
| `pgstatindex` rejects every non-B-tree relation | [pgstatindex.c#pgstatindex_impl-btree-check](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228), [sql/pgstattuple.sql#wrong-index-type](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L63) |
| Nothing on the cost path reads the FSM | no file under `src/backend/optimizer/`, and neither `selfuncs.c` nor contrib `bloom`'s `blcost.c`, includes `freespace.h` or calls an FSM function: `/usr/bin/grep -rn -i -E 'freespace\|GetFreeIndexPage\|RecordFree\|GetRecordedFreeSpace\|fsm'` over those paths on the pin returns nothing; the planner's block count is the main fork's alone ([bufmgr.h:280-281](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L280-L281)) |
| A GIN *clause* is discarded at clause matching when the operator is not in its opfamily; the *index* is not necessarily discarded with it | [indxpath.c#match_opclause_to_indexcol-op_in_opfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2433-L2459); core GIN families carry no `<`/`<=`/`>=`/`>` ([pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244), [pg_amop.dat#gin-tsvector_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1290-L1296), [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611), [pg_amop.dat#gin-jsonb_path_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1613-L1622)); but `build_index_paths()` still generates a path on `useful_predicate` alone ([indxpath.c#build_index_paths-generate](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L954-L962)), and GIN's `amoptionalkey = true` removes the only hard stop ([indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897), [ginutil.c:49](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L49)); measured on the keyless partial GIN path |
| One GIN measurement here is priced with the array cache adjustment, not without it | `n IN (1,2,3)` sets `counts.arrayScans = 3` ([selfuncs.c#gincost_scalararrayopexpr-arrayScans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7648-L7658)), which satisfies the `outer_scans > 1 \|\| counts.arrayScans > 1` test ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974)); the recorded plan carries `Index Cond: (n = ANY ('{1,2,3}'::integer[]))` on the GIN index |
| The GIN array cache branch does not depend on `amsearcharray`, which only routes the array clause through the `ST_BITMAPSCAN` retry | [selfuncs.c#gincostestimate-saop-clause](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7823-L7833), [selfuncs.c#gincost_scalararrayopexpr-arrayScans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7648-L7658), [indxpath.c#build_index_paths-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L862-L885), [indxpath.c#get_index_paths-nonnative-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L753-L766) |
| The two-`random_page_cost` page-charge probe is invalid on an index whose tablespace overrides `random_page_cost` | [spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196) returns the reloption and ignores the GUC; [selfuncs.c#gincostestimate-tablespace-costs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7786-L7789) is what GIN calls; the filed block now checks for the override first and reported `pg_default` with no override on the measurement server |
| `btree_gin` adds strategies 1-5 but is documented not to outperform B-tree | [btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69), [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33) |
| A bare boolean `Var` still matches a GIN bool opclass in v17 | [indxpath.c#IsBooleanOpfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2271-L2286), [indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384), [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98); measured `38.26` for `i`, `i = true` and `i IS TRUE` |
| GIN yields no plain index scan, no pathkeys, no index-only scan, no `IS NULL`, no native array scan, and no partial index path of its own | [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89), with `NULL` `amcanreturn` and `amgettuple` ([ginutil.c:70](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L70), [ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79)) copied by [plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335), against B-tree's handler ([nbtree.c:143](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L143), [nbtree.c:108-109](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L108-L109), [nbtree.c:134](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L134), [nbtree.c:115](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L115), [nbtree.c:114](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L114), [nbtree.c:119](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L119)), [indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751), [indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944), [indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800), [indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129); measured: `ORDER BY` and `IS NULL` as `disable_cost` sequential scans, and the index-only case as a `Bitmap Heap Scan` at `119.83` |
| GIN charges every pending, entry and data page at `random_page_cost` plus `50 * cpu_operator_cost` | [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980), [selfuncs.c#gincostestimate-search-entry-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8008-L8015), [selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029); measured `12.97` GIN vs `4.52` B-tree on identical statistics, both reconciled by hand to the cent from the recorded counters |
| On a single scan every pending page is charged in full, `4.25` per page of total cost at default settings, `4.125` of it as startup cost | [selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029); measured 736 pending pages moving the GIN scan from `13.01` to `3141.12` inside a `BitmapAnd` the planner kept, charged `739.00` pages (736 pending, 2 entry and 1 data page), and 1,471 pages moving it to `6264.98` and out of the `BitmapAnd` |
| `gin_clean_pending_list()` drains the list but leaves `nTotalPages` stale | [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091), [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767); only [gininsert.c:406](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L406) and [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789) call `ginUpdateStats()`; measured 2,073 live blocks against a metapage still reading 302 |
| An `ANALYZE` run by an autovacuum worker flushes the GIN pending list in its final index-cleanup step, only as far as the tail page recorded when the flush begins, and leaves the metapage counters stale; a manual `ANALYZE` leaves the list alone | [ginvacuum.c#ginvacuumcleanup-analyze-only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717), [analyze.c#do_analyze_rel-index-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L694-L721), [analyze.c:527](../../../../raw/postgres-17/src/backend/commands/analyze.c#L527), [ginfast.c:847](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L847), [ginfast.c#ginInsertCleanup-stop-at-tail](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L881-L888), [gin.sgml#fast-update-flush](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515), [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789); not measured |
| `gincostestimate()` scales the last-`VACUUM` counters by `numPages / nTotalPages`, rounding up, and clamps them to the non-pending pages while the index has a page, the total, entry-page and entry counts are nonzero (the data-page count may be zero), and `nTotalPages` is at most the index and more than a quarter of it; otherwise it invents them from the live block count, at least 10 pages, 90% of them entry pages holding 100 entries each | [selfuncs.c#gincostestimate-stats-branch](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7729-L7747), [selfuncs.c#gincostestimate-invented](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7748-L7767), [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767), [selfuncs.c#gincostestimate-trust](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7727); measured on fixture T: 1,369 live blocks against a metapage reading 2, charged 4 pages; on fixture D at 736 pending pages: entry pages `ceil(273 * 1038 / 302) = 939` clamped to 302 and data pages `97` clamped to 0 |
| `gincostestimate()` uses the pending-page count only while it is below `index->pages`, and its data-page floor needs a nonzero selectivity and a nonzero `index->tuples` | [selfuncs.c#gincostestimate-pending-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7724-L7727), [selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7982-L8006), [selfuncs.c:7675](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7675); measured: one floor data page on every trusted-statistics row of the page-charge table |
| A keyless partial GIN path is priced as a whole-index scan | [selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877), [indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897); measured `1001.00` charged pages on a 10-block index |
| GIN rejects unique, `INCLUDE`, exclusion and `CLUSTER` | [indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879), [cluster.c#check_index_is_clusterable-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522); all four messages reproduced |
| A multicolumn GIN can beat a `BitmapAnd` of GIN + B-tree | [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33); measured `21.51` versus `240.13` |
| No test compares GIN and B-tree plan choice, and none asserts a `gincostestimate()` cost | no `src/test` match for `gincostestimate`; `btree_gin` tests set `enable_seqscan = off` ([bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9)) and use `EXPLAIN (COSTS OFF)` ([bool.sql#explain-costs-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L23-L26)) |
| The planner's tree height can be a stale per-backend cached copy of the metapage, until an invalidation or a failed cache check discards it | [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717) caches `BTMetaPageData` in `rel->rd_amcache`, never caches an index with no root, and its comment declines to refresh the cache; scans and inserts fill the same cache ([nbtpage.c#_bt_getroot-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L523-L528), [nbtpage.c#_bt_metaversion-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L773-L775)). Three events discard it: relcache invalidation ([relcache.c#RelationCacheInvalidateEntry](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2975-L2985), [relcache.c#RelationFlushRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2916-L2924), [relcache.c#RelationClearRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2600-L2603)), `_bt_getroot()` rejecting a cached fast root that is deleted or half-dead ([nbtree.h:225](../../../../raw/postgres-17/src/include/access/nbtree.h#L225)), at another level, or not alone on its level ([nbtpage.c#_bt_getroot-stale-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L380-L403)), and the unconditional flush in [nbtpage.c#_bt_gettrueroot-flush](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L592-L600). No fixture reads a stale height: every measurement is planned in a session opened after the last change to its index |
| The `pg_class` writes that VACUUM and `ANALYZE` make for an index reach a planning backend as a relcache invalidation; a root split, a split that moves the fast root up, an unchanged row, and a VACUUM that writes no index row send none | VACUUM writes an index's row only from exact counts with index cleanup on ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [vacuumlazy.c:512-513](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L512-L513)), and `ANALYZE` writes every index's row ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)). Both are in-place updates ([vacuum.c#vac_update_relstats-inplace](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1545-L1548)) that register an invalidation for the index's own entry ([heapam.c:6668](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6668), [inval.c#CacheInvalidateHeapTupleCommon-pg_class](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L1383-L1392), [inval.c:1447](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L1447)), send it after the write ([heapam.c#heap_inplace_update_and_unlock-send](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6888-L6892)) and discard it when the row is unchanged ([vacuum.c#vac_update_relstats-dirty](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1446-L1461), [heapam.c#heap_inplace_unlock](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6914-L6920)); the planner's index lock absorbs it ([plancat.c:253](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L253), [lmgr.c#LockRelationOid-accept](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L134-L138)), as [README#metapage-cache](../../../../raw/postgres-17/src/backend/access/nbtree/README#L776-L790) expects. A root split writes only index pages ([nbtinsert.c#_bt_newlevel-root-level](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2507-L2513), [nbtinsert.c#_bt_newlevel-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2515-L2519)), and splitting the only page of a level below the true root rewrites the fast root in the metapage alone ([nbtinsert.c#_bt_insertonpg-fastroot](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1250-L1295), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)). A B-tree VACUUM that never called `btbulkdelete()`, because it collected no dead items or bypassed index vacuuming, gets no statistics or estimated ones and writes no row ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893), [vacuumlazy.c:1051-1052](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1051-L1052), [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)). Read from source; not measured |
| `leaf_fragmentation` counts backward sibling links only, not physical adjacency | [pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323) |
| An `UPDATE` of only summarizing-index columns adds no entry to a B-tree, but only while the new version fits on the old page | the whole HOT decision is inside `if (newbuf == buffer)` and the `else` branch only hints the page full ([heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166)); a non-HOT update reports `TU_All` ([heapam.c:4429](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4429)); otherwise [heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429), [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127), [nodeModifyTable.c#ExecUpdateEpilogue-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L2162-L2166), [execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366) |
| A non-HOT update skips every partial index whose predicate the new row fails, and still inserts into every other index that accepts the row | [execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L387), [execIndexing.c#ExecInsertIndexTuples-loop](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L343-L387), [heapam.c:4429](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4429) |
| The 2% bypass skips index vacuuming but still runs index cleanup | [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949) clears only `do_index_vacuuming`; [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893) can still scan and recycle; [ginvacuum.c#ginvacuumcleanup-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L719-L729) flushes the pending list whenever `ginbulkdelete()` was not called, which also covers every VACUUM that collected no dead items ([vacuumlazy.c#lazy_vacuum-call](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1066)), and `ginbulkdelete()` flushes it on its first call ([ginvacuum.c#ginbulkdelete-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)) |
| The wraparound failsafe clears `do_index_cleanup` too, so it does stop cleanup | [vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326) |
| `fillfactor` reaches existing pages at their next rightmost or split-after-new-item leaf split, and `deduplicate_items` at their next pre-split deduplication pass, without a rebuild | [nbtsplitloc.c#leaffillfactor](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L170-L176), [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782), [nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564) |
| `IndexOptInfo` is not the whole size input for every AM: GIN and BRIN reopen the index for metapage values, and only GIN's track bloat; BRIN's are `pagesPerRange` and the revmap page count, which grows with the heap's ranges | [selfuncs.c:7674](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7674), [selfuncs.c:8063](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8063), [selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767), [selfuncs.c#brincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8086-L8101), [brin.c#brinGetStats](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1638-L1654), [brin.h#BrinStatsData](../../../../raw/postgres-17/src/include/access/brin.h#L32-L36), [brin_revmap.c#HEAPBLK_TO_REVMAP_BLK](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L35-L43), [brin_revmap.c#revmap_extend_and_get_blkno](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L494-L514) |
| `FormData_pg_class` is cpp output of the `CATALOG()` macro, not a `genbki.pl` product | [genbki.h:23](../../../../raw/postgres-17/src/include/catalog/genbki.h#L23), [pg_class.h#CATALOG](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L32), [pg_class.h#NOTES](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L11-L16) |
| The cost path and the endpoint probe compile against generated catalog headers: the `RELKIND_*` letters from `pg_class.h`'s client-code block, which `genbki.pl` copies into `pg_class_d.h`, and `BTREE_AM_OID`, an `oid_symbol` in `pg_am.dat` | [plancat.c:471](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471), [plancat.c:488](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488), [selfuncs.c:6197](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6197), [pg_class.h#relkinds](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L162-L173), [Catalog.pm#client_code](../../../../raw/postgres-17/src/backend/catalog/Catalog.pm#L61-L66), [Catalog.pm#client_code-push](../../../../raw/postgres-17/src/backend/catalog/Catalog.pm#L197-L205), [genbki.pl#client_code](../../../../raw/postgres-17/src/backend/catalog/genbki.pl#L563-L567), [pg_am.dat:18](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18), [genbki.pl#oid_symbol](../../../../raw/postgres-17/src/backend/catalog/genbki.pl#L675-L687) |
| `036decbba2a`, first contained in the `Stamp 17.7.` commit, added a `BTPageOpaqueData` size check to `pgstattuple`'s B-tree page handling; it changes `pgstattuple()` output, not `pgstatindex()` | [pgstattuple.c#pgstat_btree_page-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L419-L425) |
| A serial GIN bitmap index scan can feed a `Parallel Bitmap Heap Scan` | [indxpath.c:347](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L347), [allpaths.c#create_partial_bitmap_paths](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4162-L4185); the bitmap is built once and shared, and each participant takes the next heap page from one shared iterator ([nodeBitmapHeapscan.c#BitmapShouldInitializeSharedState](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L774-L806), [nodeBitmapHeapscan.c#shared-bitmap-build](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L130-L141), [nodeBitmapHeapscan.c#shared-iterator](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L143-L170), [nodeBitmapHeapscan.c:241](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L241), [tidbitmap.c#tbm_shared_iterate](../../../../raw/postgres-17/src/backend/nodes/tidbitmap.c#L1044-L1136)); not measured |
| On a repeated GIN scan the pending-page I/O charge is amortized and the per-page CPU charge is not | [selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974), [selfuncs.c:7980](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7980), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955) |
| `gin_pending_list_limit` triggers a non-forced cleanup after the insert, and is not a ceiling | [ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [ginfast.c#ginInsertCleanup-conditional-lock](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828), [gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529); the reference entry and GUC description call it a "maximum size" ([config.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9870-L9890), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)), filed under Open Questions |
| `fastupdate = off` does not flush the entries already in the pending list | [ref/create_index.sgml#fastupdate-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L524-L532) |
| `enable_bitmapscan = off` adds `disable_cost` and does not remove the bitmap path | [costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042) |
| Gate 3 is a comparison of computed costs, not a rule that GIN loses | [indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1399), [pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L622); measured: a multicolumn GIN at `21.51` beating a `BitmapAnd` at `240.13` |
| For two indexes on the identical clause set, `choose_bitmap_and()` keeps the lower `amcostestimate` total plus `0.1 * cpu_operator_cost` per row, and the row term is the same for both (`baserel->rows`), not the `rows` figure `EXPLAIN` prints; on a tie the path met first stays, and the B-tree's plain paths still compete in `add_path()` | [indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1399), [costsize.c:628](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L628), [costsize.c#cost_bitmap_tree_node-indexpath](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1114-L1128), [costsize.c#cost_index-rows](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L589-L604), [createplan.c#create_bitmap_subplan-indexpath](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3480-L3485), [indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751); measured on fixture S: the planner chose the B-tree for `n = 42`, `n BETWEEN 100 AND 200` and `n < 20` |
| For indexes on different clauses, an index joins an AND group only when the whole bitmap-heap-scan estimate drops, and `add_path()` then compares whole paths | [indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489), [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526-L1553), [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571), [pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L622); measured on fixture D at 736 pending pages: the GIN estimate `3141.12` against the B-tree's `168.80`, kept because the `BitmapAnd` plan, `3321.93`, is below the B-tree-only plan, `3486.80` |
| Gate 0: `get_relation_info()` builds no `IndexOptInfo` for an index that is not `indisvalid`, and skips an `indcheckxmin` index whose `pg_index` row does not precede `TransactionXmin`, marking the plan transient; a cached transient plan is planned again when `TransactionXmin` changes | [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L43), [plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281), [plancache.c#BuildCachedPlan-transient](../../../../raw/postgres-17/src/backend/utils/cache/plancache.c#L1022-L1033), [plancache.c#CheckCachedPlan-transient](../../../../raw/postgres-17/src/backend/utils/cache/plancache.c#L866-L873), [README.HOT#indcheckxmin](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L341-L353); a failed `CREATE INDEX CONCURRENTLY` leaves an invalid index ([ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667), [create_index.sgml#concurrently-invalid](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L651)); not measured |
| `index_build()` sets `indcheckxmin` only for a non-concurrent `CREATE INDEX` that met broken HOT chains; `reindex_index()` clears it after a rebuild that met none, and otherwise keeps it on a valid index and sets it on an invalid, not-ready or dead one | [index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3099-L3101), [index.c#reindex_index-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3800-L3850); not measured |
| A partial index whose predicate the query's restrictions do not imply is skipped before clause matching and left to `generate_bitmap_or_paths()`; with `build_index_paths()`'s missing-scan-type return, these are the hard stops that do not depend on clause matching | [indxpath.c#check_index_predicates-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3272-L3350), [indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266), [pathnodes.h:1168](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1168), [pathnodes.h:1181](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1181), [indxpath.c#build_index_paths-scantype](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L826-L842); not measured |
| Exhaustive sampling removes sampling noise; the independence assumption survives it, and so do changes made after `ANALYZE` | [clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263); a column whose every sampled value repeats and fits the target gets a complete MCV list ([analyze.c#compute_scalar_stats-stadistinct](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2550-L2560), [analyze.c#compute_scalar_stats-complete-mcv](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2614-L2636)); measured on fixture L3, 250, 24,971 and 12 estimated against 250, 24,971 and 12 actual, where the conjunction's match is the seed's (independence predicts 12.49 on average), and on fixture D, 80, 19,999 and 4 estimated against 80, 20,000 and 4 actual, where the conjunction is exact by construction. Fixtures Q and E change their tables after `ANALYZE` on purpose: Q estimates 40,000 rows for 20,000 live ones, and E 10 for none. No fixture here violates the independence assumption |
| An `INDEX_CLEANUP ON` VACUUM forces index vacuuming, not cleanup | [vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402), [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949) |
| BRIN charges every index page on every scan: the range-map pages at `seq_page_cost` as startup cost, the rest at `random_page_cost`, both multiplied by `loop_count` with no cache model, so BRIN bloat reaches the cost only through `index->pages` | [selfuncs.c:8063](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8063), [selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258), [brin.c#brinGetStats](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1638-L1654); not measured |
| Bloat flips a plan to a sequential scan | measured on fixture A: `Index Scan` at `9590.42` on the dense index against `Seq Scan` at `22353.00` on the bloated twin, for the same 25% range; `22353.00` is 7,353 heap pages at `seq_page_cost` plus 1,000,000 rows at `cpu_tuple_cost` and two operator evaluations each ([costsize.c#cost_seqscan](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L315-L322)) |
| An index-only scan of an all-visible table pays no heap I/O but still `cpu_tuple_cost` per expected row | [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800); measured: fixture A's whole-index totals each carry `10000.00`, so the 9.62x larger index costs 6.12x more on the index alone (`113144.43` against `18480.42`) and 4.32x in total |
| Fixture I pays one random heap page per scan at every array length because its heap is in key order | [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747), [costsize.c#cost_index-correlation-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L781-L800) |
| Blocks per row price plans but do not predict what `REINDEX` returns: the rebuild reapplies the index's `fillfactor` and `deduplicate_items`, and a partial index's population is the sampled rows that pass its predicate | [index.c#reindex_index-rebuild](../../../../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789), [nbtsort.c:665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L665), [nbtsort.c#_bt_pagestate-leaf-fill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665), [nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152), [nbtree.h#BTGetFillFactor-BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1150), [analyze.c#partial-index-population](../../../../raw/postgres-17/src/backend/commands/analyze.c#L902-L908), [analyze.c#tupleFract](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953); measured on fixture L3: `REINDEX` at `fillfactor = 10` turned 427 blocks into 3,801. That `n_off` and `a_sparse_idx` would rebuild at about their current size is a source reading; neither is rebuilt, and the reading is not scored against [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md) |
| This page's *bloat* is broader than the manual's: an index with deduplication off and a GIN pending list fit only the broad sense; the pending list is deferred insert work, yet a single GIN scan is charged for every pending page | [glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250), [btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800), [gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529), [gin.sgml#fast-update-flush](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515), [selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980); the pending list is not scored as waste under [Mandatory GIN Bloat Tests (unverified)](../../common-concepts/mandatory-gin-bloat-tests.md) |
| `CLUSTER` on a B-tree prices a whole-index scan, charged every index page unless the table's estimate is one row or less, against a sorted sequential scan, so bloat pushes it toward the sort without deciding it; `enable_indexscan = off` or an index missing from the planner's list forces the sort | [cluster.c#plan_cluster_use_sort-call](../../../../raw/postgres-17/src/backend/commands/cluster.c#L948-L951), [planner.c#plan_cluster_use_sort](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6761-L6874), [planner.c#plan_cluster_use_sort-compare](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6861-L6873), [planner.c#plan_cluster_use_sort-short-circuits](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6788-L6841), [selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019), [cluster.c#check_index_is_clusterable-partial](../../../../raw/postgres-17/src/backend/commands/cluster.c#L525-L534), [selfuncs.c#genericcostestimate-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6786), [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747); not measured |
| `RelOptInfo.tuples`, filled by `estimate_rel_size()`, is copied into a non-partial index and caps a partial one; `consider_bypass_optimization` gates the 2% bypass; `AttStatsSlot` carries the histogram bounds the probe overwrites | [pathnodes.h#RelOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L941-L944), [plancat.c:201](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L201), [plancat.c:476](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L476), [plancat.c:484](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L484), [vacuumlazy.c:1900](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1900), [vacuumlazy.c#LVRelState](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L148-L156), [lsyscache.h#AttStatsSlot](../../../../raw/postgres-17/src/include/utils/lsyscache.h#L46-L62) |
| The caller and callee path from `query_planner()` through `get_relation_info()`, `create_index_paths()`, `cost_index()` and the AM's `amcostestimate`, and from `clauselist_selectivity()` to the endpoint probe | [planmain.c:170](../../../../raw/postgres-17/src/backend/optimizer/plan/planmain.c#L170), [initsplan.c:165](../../../../raw/postgres-17/src/backend/optimizer/plan/initsplan.c#L165), [relnode.c:340](../../../../raw/postgres-17/src/backend/optimizer/util/relnode.c#L340), [planmain.c:280](../../../../raw/postgres-17/src/backend/optimizer/plan/planmain.c#L280), [allpaths.c:783](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L783), [indxpath.c:279](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L279), [pathnode.c:1024](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L1024), [costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621), [indxpath.c#choose_bitmap_and-call](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L340-L343), [clausesel.c#clauselist_selectivity](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L100-L108), [plancat.c#restriction_selectivity](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1951-L1979), [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6157-L6331), [selfuncs.c#get_actual_variable_endpoint](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6349-L6501) |
| Between the two pins, nine commits touch the cost path and twenty-three the other cited files; none changes a cost function or the planner's index-size inputs | `git log 54eeefaedbee..786db8dcf168` over the cost-path directories and files and over every other cited file; the two in `selfuncs.c` and `nbtsearch.c` are a `ctid` datatype check in `scalarineqsel()` ([selfuncs.c#scalarineqsel-ctid-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L599-L601)) and an empty-index recheck in `_bt_endpoint()` ([nbtsearch.c#_bt_endpoint-empty-index](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2590-L2606)) |
| The two filed diagnostic blocks run as printed | executed verbatim by the script's `diag` stage: `340` live blocks, `246` pending pages, `72.35` %; `2030.46` and `1121.46`, so `303.00` charged pages; `pg_default` with no `random_page_cost` override |
| `pgstatginindex()` raises an `ERROR` on a partitioned GIN index, an invalid one and another session's temporary one, so the first diagnostic block filters them out in an `OFFSET 0` subquery, which is never pulled up | [index.c:1015](../../../../raw/postgres-17/src/backend/catalog/index.c#L1015), [pg_class.h#relkinds](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L162-L173), [pgstatindex.c#IS_INDEX](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70-L72), [pgstatindex.c#pgstatginindex_internal-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L520-L543), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L667-L669), [pg_class.h:177](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L177), [create_index.sgml#concurrently-invalid](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L651), [prepjointree.c#is_simple_subquery](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1689-L1701); the filters ran against one valid, permanent, non-partitioned index only |
| `pgstatginindex()` needs `EXECUTE`, which the 1.5 upgrade grants to `pg_stat_scan_tables`, takes `AccessShareLock` and reads only the metapage | [pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57), [pgstatindex.c#pgstatginindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L497-L504), [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577) |
| Each `disable_cost` figure in fixture S's table is `1.0e10` plus the plan's normal cost | [costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130), [costsize.c#cost_seqscan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L304-L305); measured `10000004328.00` = `1.0e10 + 4328.00` and `10000010810.92` = `1.0e10 + 10810.92` |
| The `cat = 7` estimate depends on how `reltuples` was last written | [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1299-L1366); measured: 19,999 after a plain `VACUUM` left `reltuples` at 399,985, and 20,000 after `CREATE INDEX` rewrote it to 400,000, because the GIN build returns its heap scan's tuple count and `index_build()` writes it into the table's row ([gininsert.c#ginbuild-scan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L382-L384), [gininsert.c:424](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L424), [index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3131), [index.c#index_update_stats-write](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2923)) |
| The script clears `PGSERVICE` and `PGSERVICEFILE` because libpq fills options from a service entry before it reads the environment, and `PGHOSTADDR` because libpq gives a non-empty host address precedence over `host`, a socket directory included, and resolves it as a numeric network address, so the session goes over TCP rather than to the sandbox socket; the build needs GNU `make` 3.81 or newer, and the server refuses a socket path longer than its buffer | [fe-connect.c#conninfo_add_defaults](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L6201-L6245), [fe-connect.c#hostaddr-option](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L223-L225), [fe-connect.c#pqConnectOptions2-host-type](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L1189-L1204), [fe-connect.c#PQconnectPoll-host-address](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L2754-L2764), [installation.sgml:40](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L40), [pqcomm.h:60](../../../../raw/postgres-17/src/include/libpq/pqcomm.h#L60), [pqcomm.c:453](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L453) |

## Open Questions

- Fixture P's growth ratios (3.2x unblocked, 6.9x blocked) come from one deliberately harsh workload with autovacuum off, and fixture P-100 shows the result is shape-dependent: on a 100-value column the same workload grew the index to 1,020 blocks with or without a held snapshot. Why the free horizon bought nothing there was not traced to source. Both P-100 runs end with identical block counts, which points at the insertion pattern inside long runs of one key rather than at the horizon, but nothing on this page establishes that. The documentation's stronger claim, that some indexes "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)), was not reproduced here and would need a gentler, steady-state workload to test. Nor was the 98.01% density of fixture P's unblocked run traced to a mechanism: deduplication, bottom-up deletion and the single-value split strategy can each leave a leaf page fuller than the 91.47% `pgstatindex` measured after the build (the default leaf fillfactor is 90), and the script does not separate them.
- Mechanism 3 is measured on the index side only. [Index pages in the cache model](#index-pages-in-the-cache-model) prices a repeated inner scan at `0.66` against `2.50` per loop, but every table in that fixture is all-visible, so no heap page is fetched and the other half of the mechanism, a bloated index shrinking the heap's prorated share of `effective_cache_size`, was not isolated. It needs a nested loop whose inner scan fetches heap pages while the tables and the index together exceed the cache setting. The index side is measured in two regimes only: 50,000 loops at the default `effective_cache_size`, where `a_dense_idx` sits at the Mackert-Lohman cap and `a_sparse_idx` just short of it, and the same join at `effective_cache_size = 64MB`, past the cap. The few-loops regime, where the formula charges about one page per loop whatever the index's size, is read from source only ([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)).
- Whether GIN's per-page CPU charge, added in v16, ever decides a GIN-versus-B-tree choice by itself, rather than the `random_page_cost` term, was not isolated: no fixture varies `cpu_operator_cost` for a GIN plan. [Gate 3](#gate-3-cost-and-why-gin-loses-on-the-same-column) reconciles the charge to the cent on one fixture only.
- Two extension-boundary claims are read from source and were not exercised: that a custom index AM gets `tree_height = -1` unless `relam == BTREE_AM_OID` ([plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500)), and that a `get_relation_info_hook` plugin can rewrite `pages`, `tuples` or `tree_height` before costing ([plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576)). No custom AM and no hook plugin was built. The same goes for contrib `bloom`: a single scan is charged for every index page, as `blcostestimate()` says ([blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42)), and a repeated one over an index within its cache share is capped at the index's page count and spread over the loops ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)); no bloom fixture was measured.
- The `BitmapAnd` boundary in fixture D was located between two pending-list sizes, 736 pages (index kept) and 1,471 pages (index dropped), and not bisected. The comparison that decides it is identified, the two-index plan against the B-tree-only plan, but the crossover size was not measured.
- Fixture T's `VACUUM` refreshed the GIN metapage from `(2, 1, 0, 100)` to `(1369, 1368, 0, 200000)` and the `n = 42` cost moved by one cent, from `17.19` to `17.20`. The invented and trusted branches produce the same charged page count here, `4.00`, so this fixture does not separate them; a fixture where the two branches diverge visibly was not built.
- The GIN entry-page estimate `ceil(searchEntries * rint(pow(numEntryPages, 0.15)))` was reconciled arithmetically in every measured case, but `rint()`'s banker's rounding at exact `.5` boundaries was not exercised. Mechanism 4's five-element `= ANY` example, `rint(5 / 3) * 3 = 6` tuples, is likewise arithmetic from the pinned formula; fixture I measures the ten- and four-element cases.
- Whether `contrib/btree_gin`'s partial-match path (`gincost_pattern()` charging `partialEntries += 100` per key, [selfuncs.c:7467](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7467)) systematically over- or under-charges a range predicate was not investigated; the measurements report only the resulting costs.
- The parallel-worker counts were measured on `count()` queries, whose partial aggregate makes each worker return one row, and there the planner chose the same parallel plans at the default parallel costs as with both costs set to `0`. A query that ships its rows through `Gather` pays `parallel_tuple_cost` for each of them, so whether bloat changes the worker count of such a scan depends on the parallel path winning on cost first, which was not measured.
- Every row estimate on this page comes from exhaustive statistics, because the script runs with `default_statistics_target = 10000`. That is not the same as the model being exact, and this page does not bound the difference. Two fixtures record two-clause estimates beside true counts, L3 and D, and both were built with *independent* columns, the one case where the multiplication in `clauselist_selectivity_ext()` is right, and even then only on average ([clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)): L3's conjunction matches its true count by luck of the seed, and D's is exact by construction. D's single-clause `cat = 7` estimate, 19,999 against 20,000 true, is off by a row for a different reason, VACUUM's extrapolated `reltuples`. Fixture A's range predicates on its unique key are estimated by histogram interpolation, because a column whose every value appears once gets no MCV list, and they land on their true counts by construction; the script records no true count beside them. Nothing here exercises a correlated column pair or an extended-statistics object. Earlier versions of fixtures L3 and D did pair two dependent columns by accident, so that their two-clause predicates could match no row while the planner estimated a few rows; both were rebuilt on independent columns on 2026-09-19 and 2026-09-20, and rebuilding D left every one of its recorded costs unchanged, because a cost is computed from the estimate and the estimate assumes independence in both versions alike. The page no longer quotes the old versions' numbers, so it carries no measured case in which the assumption fails. A server at the default target of 100 samples only 30,000 rows ([guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2070-L2079), [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1876-L1894)), and no sampled run is filed on this page.
- Seven claims added on the second 2026-09-19 pass are read from source and have no fixture on this page: that a backend keeps costing from a stale cached metapage until an invalidation reaches it, so that two backends can price one index differently; that an `UPDATE` touching only summarizing-index columns leaves an unrelated B-tree untouched; that a partial index whose predicate the new row fails receives no entry; that VACUUM's 2% bypass still runs `amvacuumcleanup`, so it can flush a GIN pending list or recycle B-tree pages while skipping index vacuuming; that a changed `fillfactor` reaches already-existing pages at their next rightmost or split-after-new-item split, and a changed `deduplicate_items` at their next pre-split pass; that a serial GIN bitmap index scan can feed a `Parallel Bitmap Heap Scan`; and that a repeated GIN scan amortizes the pending-page I/O charge through `index_pages_fetched()` while leaving the per-page CPU charge unamortized. Each is cited to the pinned source, which `MANDATORY Evidence` treats as primary, but none was reproduced on the measurement server, and the last one in particular would change how [A GIN index with a large pending list loses to a B-tree](#a-gin-index-with-a-large-pending-list-loses-to-a-b-tree) reads for a nested-loop inner scan. The array half of that last claim is now half-exercised and half not: fixture S's `n IN (1,2,3)` does enter the `outer_scans > 1 || counts.arrayScans > 1` branch at `arrayScans = 3`, so the cache adjustment fires on a measured row, but that index has no pending pages, so nothing here shows the pending-list charge being amortized. A fixture with both a pending list and a repeated scan was not built.
- Four claims added on the 2026-09-20 pass are likewise read from source with no fixture. Three are about paths the measurement server never takes: that a table `parallel_workers` reloption makes `compute_parallel_worker()` skip the page-based calculation altogether, capped only by `max_parallel_workers_per_gather`, and that an inheritance child is exempt from the minimum-size rejection — every fixture here is a plain unpartitioned table with no such reloption; and that an update whose new version does not fit on its own page reports `TU_All` and so writes into every index that accepts the row. Fixture P cannot show that last one either way: its `UPDATE` changes an indexed column, so it is non-HOT whether or not the new version fits. Isolating the fit-decided route needs an update of only summarizing or unindexed columns on full pages. The fourth is that an index on a tablespace carrying a `random_page_cost` reloption makes the two-`random_page_cost` page-charge probe report zero charged pages: the source path is unambiguous, but no second tablespace was created, and the filed diagnostic block was run only against `pg_default`, where it correctly reported no override. Measuring any of these needs a fixture shape the script does not build.
- The documentation and the source disagree about `gin_pending_list_limit`. The setting's reference entry and its GUC description call it the "maximum size" of the pending list, and the reference entry says that a list growing past it "is cleaned up" ([config.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9870-L9890), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)). The source allows the overshoot the entry describes but does not guarantee the cleanup: `ginHeapTupleFastInsert()` tests the size only after adding entries and then asks for a cleanup that is not forced and is skipped when another process holds the metapage lock ([ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [ginfast.c#ginInsertCleanup-conditional-lock](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828)). The page follows the source. Fixture D grows its pending list by raising the limit rather than by exceeding it, so no measurement here shows the overshoot.
- Several claims added on the 2026-09-23 pass are read from source with no fixture: that VACUUM's statistics update reaches a planning backend as a relcache invalidation, and that a root split, or a VACUUM that writes no statistics, leaves a stale height in place; that BRIN charges every index page on every scan; that the statistics hooks move the page charge through selectivity and that `set_rel_pathlist_hook` can edit costed paths; that the cost path compiles against generated catalog headers; what `e5d8a99903` and `9f3665fbfc` changed in page recycling; and that a query needing no column passes `check_index_only()` trivially.
- Fixture Q measures the partial-index lag through one path only, `ANALYZE` rewriting the index's `pg_class` row. Four neighboring claims are read from source: that a `VACUUM` whose counts for the index are exact, or a rebuild, ends the lag the same way ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842)); that the lag stops once the scaled `tuples` reaches the table's row estimate and `get_relation_info()`'s clamp binds ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)); that a partial index whose row records `relpages <= 1` or `reltuples = -1` takes the width-based density instead ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)); and which commands leave an index's row at `-1`, or at `0` and `0`, as [v14: reltuples turns negative for never-analyzed relations](#v14-reltuples-turns-negative-for-never-analyzed-relations) sets out. No fixture builds an index whose rebuild produced no entries, whether from an empty table, from rows dead to every transaction, or from a partial predicate that no row satisfies.
- Fixture E measures the endpoint probe on one plain primary key, for one `>` clause, with no other session, on a primary server. The rest is read from source: that a snapshot old enough to see the deleted rows makes `SnapshotNonVacuumable` accept them as recently dead, so the probe returns the deleted maximum at once and marks nothing ([selfuncs.c#get_actual_variable_endpoint-comment](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6371-L6414)); that on a hot standby the probe neither marks nor honors killed entries, so every plan walks the same dead entries until the standby replays their removal ([genam.c#RelationGetIndexScan-recovery](../../../../raw/postgres-17/src/backend/access/index/genam.c#L107-L119), [nbtxlog.c#btree_xlog_vacuum](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L598-L634)); that `LIKE` prefix estimation and merge-join costing reach the probe too ([like_support.c#prefix_selectivity](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1213-L1270), [selfuncs.c#mergejoinscansel-scalarineqsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3164-L3203)); and that no partial index and no non-B-tree index is ever probed ([selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)). The page claims one probe per histogram end that a clause's search reaches, and several probes for several such clauses; whether one clause can be probed more than once in a plan, through a selectivity path that does not reuse the cached restriction selectivity, such as join-clause or index-predicate selectivity, was not traced. Nothing in fixture E is timed, so the planning time a probe costs, up to 101 heap pages, is not measured.
- Most of the history added to [What Changed Since PostgreSQL 12](#what-changed-since-postgresql-12) on 2026-09-25 is read from the pinned v17 source and from the v12 checkout's `REL_12_0` text, with no fixture and no v12 server: the v16 summarizing-only HOT change, the v13 narrowing of GIN's whole-index estimate, the `pg_upgrade` caveat for deduplication, the two `= ANY` cases fixture I does not build (an array outside the boundary quals, and a non-constant array sized from statistics), the `reltuples` states of new, rebuilt and upgraded indexes, `9f3665fbfc` stopping a cleanup-only B-tree VACUUM from writing `pg_class`, and the `REL_12_0` comparison of the width-based fallback. Only the endpoint probe's 100-page limit is measured, and only on the v17 side (fixture E). For a partial index of an access method whose empty build leaves more than one block, rebuilt with no entries, `REL_12_0` would have recorded a density of zero where v17 falls back to column widths ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146), [relcache.c#RelationSetNewRelfilenumber-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3948-L3954), [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)). A fresh `CREATE INDEX` of such an index with no matching rows records a zero density in v17 too, because the new row starts at `reltuples = 0`, not `-1`, so the keep-`-1` rule does not fire ([relcache.c#RelationBuildLocalRelation-rd_rel](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3656-L3659), [index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024)). Which core access methods build that way was not worked out.
- The gate-0 checks are read from source only: no fixture builds an invalid index, an `indcheckxmin` index inside its horizon, or a partial GIN index whose predicate the query does not imply ([plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281), [indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266)). No fixture exercises the pending-list flush that an autovacuum `ANALYZE` performs, or shows that it leaves the metapage counters stale ([ginvacuum.c#ginvacuumcleanup-analyze-only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717)); the script runs with `autovacuum = off`. The first diagnostic block's filters were exercised only against one valid, permanent, non-partitioned GIN index, so the `ERROR`s they avoid on a partitioned, an invalid and another session's temporary GIN index are read from `pgstatginindex()`'s checks ([pgstatindex.c#pgstatginindex_internal-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L520-L543)) and were not reproduced. The documentation and the source also disagree on one point here. The `CREATE INDEX` reference says a concurrently built index "may not be immediately usable for queries" while transactions that predate the build exist ([ref/create_index.sgml#not-immediately-usable](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L640-L642)), but `index_build()` never sets `indcheckxmin` for a concurrent build ([index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3099-L3101)), and `get_relation_info()` holds back a valid index only through that flag ([plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281)). Which mechanism the sentence refers to was not traced; the page follows the source.
- Several other claims added on 2026-09-25 rest on source reading alone: that fixture M's whole-index scan walks only its 276 live leaves because VACUUM unlinked each deleted page from its siblings, since no fixture measures what a scan reads, with `EXPLAIN (ANALYZE, BUFFERS)` or otherwise ([nbtpage.c#unlink-siblings](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2612)); that splitting the only page of a level below the true root moves the fast root without an invalidation and so leaves a cached height stale ([nbtinsert.c#_bt_insertonpg-fastroot](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1250-L1295)); that an `ANALYZE`'s write of an index's row clears a cached height ([analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)); that a B-tree VACUUM that never called `btbulkdelete()` writes no `pg_class` row for the index ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)); that `CLUSTER` prices a whole-index scan against a sort ([planner.c#plan_cluster_use_sort](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6761-L6874)); and that a `REINDEX` of fixture N's `deduplicate_items = off` index or of fixture A's `fillfactor = 10` twin would come back at about its current size ([nbtsort.c#_bt_pagestate-leaf-fill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665), [nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)), which the script does not run. The page's size-against-rows reading and its first Practical Interpretation bullet have not been scored against [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md), which governs rebuild decisions.

## Source References

- [plancat.c#get_relation_info-index-block](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)
- [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)
- [selfuncs.c#genericcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6626-L6828)
- [selfuncs.c#btcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6870-L7211)
- [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)
- [costsize.c#cost_index](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L821)
- [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)
- [costsize.c#compute_bitmap_pages](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6420-L6518)
- [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)
- [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489)
- [pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)
- [pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)
- [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)
- [nbtpage.c#_bt_getroot-stale-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L380-L403)
- [nbtpage.c#_bt_gettrueroot-flush](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L592-L600)
- [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659)
- [spccache.c#get_tablespace_page_costs](../../../../raw/postgres-17/src/backend/utils/cache/spccache.c#L181-L196)
- [reloptions.c#parallel_workers](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L374-L382)
- [nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334)
- [nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L280-L320)
- [nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)
- [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)
- [README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)
- [README#bottom-up-deletion](../../../../raw/postgres-17/src/backend/access/nbtree/README#L557-L619)
- [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1949)
- [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)
- [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372)
- [ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)
- [glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)
- [btree.sgml#bottom-up-deletion](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L678)
- [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)
- [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L8050)
- [selfuncs.c#gincost_pattern](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7380-L7492)
- [selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7370-L7378)
- [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89)
- [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)
- [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091)
- [gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L40-L50)
- [ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101)
- [indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L413)
- [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767)
- [indxpath.c#build_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L804-L1057)
- [indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800)
- [indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269)
- [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2386-L2503)
- [indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384)
- [indexam.c#index_can_return](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L780-L797)
- [pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L622)
- [costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130)
- [pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244)
- [pg_opfamily.h#IsBuiltinBooleanOpfamily](../../../../raw/postgres-17/src/include/catalog/pg_opfamily.h#L59-L65)
- [indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879)
- [cluster.c#check_index_is_clusterable-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522)
- [reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L130)
- [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)
- [btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69)
- [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)
- [gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)
- [indexam.sgml#amgetbitmap](../../../../raw/postgres-17/doc/src/sgml/indexam.sgml#L991-L1010)
- [pgtrgm.sgml#index-support](../../../../raw/postgres-17/doc/src/sgml/pgtrgm.sgml#L413-L425)
- [pgstatindex.c#pgstatginindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L497-L504)
- [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)
- [pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57)
- [pgstatindex.c#pgstatindex_impl-btree-check](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228)
- [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129)
- [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)
- [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810)
- [selfuncs.c#btcostestimate-outputs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7206-L7210)
- [selfuncs.h#GenericCosts](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L108-L138)
- [blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42)
- [plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576)
- [plancat.h#get_relation_info_hook_type](../../../../raw/postgres-17/src/include/optimizer/plancat.h#L20-L25)
- [costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)
- [costsize.c#cost_bitmap_heap_scan-startup](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044-L1048)
- [indxpath.c#check_index_predicates-indrestrictinfo](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3318-L3378)
- [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1876-L1894)
- [gininsert.c:406](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L406)
- [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789)
- [guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822)
- [clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)
- [relcache.c#RelationCacheInvalidateEntry](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2975-L2985)
- [relcache.c#RelationFlushRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2916-L2924)
- [relcache.c#RelationClearRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2600-L2603)
- [heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4166)
- [heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429)
- [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127)
- [nodeModifyTable.c#ExecUpdateEpilogue-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L2162-L2166)
- [execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366)
- [execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L387)
- [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)
- [vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)
- [ginvacuum.c#ginvacuumcleanup-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L719-L729)
- [nbtsplitloc.c#leaffillfactor](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L170-L176)
- [nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564)
- [genbki.h:23](../../../../raw/postgres-17/src/include/catalog/genbki.h#L23)
- [allpaths.c#create_partial_bitmap_paths](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4162-L4185)
- [ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471)
- [ginfast.c#ginInsertCleanup-conditional-lock](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828)
- [ref/create_index.sgml#fastupdate-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L524-L532)
- [costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042)
- [selfuncs.c#brincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8086-L8101)
- [selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974)
- [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955)
- [pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323)
- [indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1399)
- [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4002-L4020)
- [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)
- [vacuum.c#vac_update_relstats-inplace](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1545-L1548)
- [vacuum.c#vac_update_relstats-dirty](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1446-L1461)
- [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1299-L1366)
- [vacuumlazy.c#index_cleanup-options](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L402)
- [vacuumlazy.c#bypass-off-after-round](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L894-L896)
- [lmgr.c#LockRelationOid-accept](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L134-L138)
- [heapam.c:6668](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6668)
- [README#metapage-cache](../../../../raw/postgres-17/src/backend/access/nbtree/README#L776-L790)
- [nbtinsert.c#_bt_newlevel-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2515-L2519)
- [nbtpage.c#_bt_getroot-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L523-L528)
- [nbtpage.c#_bt_metaversion-cache](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L773-L775)
- [nbtpage.c#side-links](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2600)
- [nbtree.h#fillfactor-comment](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L197)
- [nbtsplitloc.c#single-value-condition](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1011-L1041)
- [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2782)
- [index.c#index_update_stats-empty](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)
- [relcache.c#RelationSetNewRelfilenumber-reset](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3948-L3954)
- [heap.c#AddNewRelationTuple-empty](../../../../raw/postgres-17/src/backend/catalog/heap.c#L1006-L1009)
- [pg_class.h#relkinds](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L162-L173)
- [genbki.pl#oid_symbol](../../../../raw/postgres-17/src/backend/catalog/genbki.pl#L675-L687)
- [pg_am.dat:18](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18)
- [hash.c:75](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L75)
- [gist.c:77](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L77)
- [spgutils.c:62](../../../../raw/postgres-17/src/backend/access/spgist/spgutils.c#L62)
- [blutils.c:124](../../../../raw/postgres-17/contrib/bloom/blutils.c#L124)
- [brin.c:265](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L265)
- [selfuncs.c#genericcostestimate-tablespace](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6734-L6737)
- [selfuncs.c#brincostestimate-page-charge](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8244-L8258)
- [selfuncs.c#gincostestimate-search-entry-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8008-L8015)
- [selfuncs.c#gincostestimate-total](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8027-L8029)
- [allpaths.c#set_rel_pathlist_hook](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L532-L539)
- [clauses.c#simplify_boolean_equality](../../../../raw/postgres-17/src/backend/optimizer/util/clauses.c#L3990-L4045)
- [nodeBitmapHeapscan.c#BitmapShouldInitializeSharedState](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L774-L806)
- [config.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9870-L9890)
- [pgstattuple.c#pgstat_btree_page-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L419-L425)
- [btree_index.sql#multilevel-page-deletion](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L252-L270)
- [btree.sgml#deduplication-at-build](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L786-L797)
- [maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1048-L1054)
- [nbtsearch.c#_bt_endpoint-empty-index](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2590-L2606)
- [indices.sgml#other-operators](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L505-L508)
- [ref/create_table.sgml#vacuum_index_cleanup](../../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1558-L1575)
- [bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9)
- [sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37)
- [create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268)
- [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)
- [bool.sql#explain-costs-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L23-L26)
- [btreefuncs.c#GetBTPageStatistics-items](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L172-L187)
- [btreefuncs.c:908](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L908)
- [expected/pgstattuple.out#partition-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L264-L268)
- [pgstatindex.c#IS_INDEX](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70-L72)
- [pgstatindex.c#metapage](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265)
- [pgstatindex.c#page-classification](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331)
- [pgstatindex.c:351](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L351)
- [pgstatindex.c#density-guards](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)
- [pgstatindex.c#pgstatginindex_internal-checks](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L520-L543)
- [pgstattuple--1.4--1.5.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L24)
- [pgstattuple.control:3](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L3)
- [sql/pgstattuple.sql#wrong-index-type](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L63)
- [btree.sgml#version-churn](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L638-L655)
- [btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703)
- [btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)
- [btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800)
- [btree.sgml#deduplication-restrictions](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L908)
- [btree.sgml#deduplication-safety](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L838)
- [gin.sgml#fast-update-flush](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515)
- [gin.sgml#fast-update-searches](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L521-L529)
- [indices.sgml#ordering](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L530-L538)
- [indices.sgml#bitmap-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L643-L656)
- [indices.sgml#index-only-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L1125-L1136)
- [installation.sgml:40](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L40)
- [maintenance.sgml#non-btree-bloat](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)
- [ref/create_index.sgml#not-immediately-usable](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L640-L642)
- [ref/create_index.sgml#invalid-index](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L645-L667)
- [create_index.sgml#concurrently-invalid](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L651)
- [ref/reindex.sgml#storage-parameter](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L66-L71)
- [brin.c:269](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L269)
- [brin.c#brinGetStats](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1638-L1654)
- [brin_revmap.c#HEAPBLK_TO_REVMAP_BLK](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L35-L43)
- [brin_revmap.c#revmap_extend_and_get_blkno](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L494-L514)
- [reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)
- [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)
- [reloptions.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L462-L470)
- [reloptions.c#vacuum_index_cleanup](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L510-L520)
- [ginfast.c:847](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L847)
- [ginfast.c#ginInsertCleanup-stop-at-tail](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L881-L888)
- [gininsert.c#ginbuild-scan](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L382-L384)
- [gininsert.c:424](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L424)
- [ginutil.c:44-45](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L44-L45)
- [ginutil.c:49](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L49)
- [ginutil.c:50](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L50)
- [ginutil.c:51](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L51)
- [ginutil.c:55](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L55)
- [ginutil.c:70](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L70)
- [ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79)
- [ginvacuum.c#ginbulkdelete-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L591-L602)
- [ginvacuum.c#ginvacuumcleanup-analyze-only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717)
- [README.HOT#indcheckxmin](../../../../raw/postgres-17/src/backend/access/heap/README.HOT#L341-L353)
- [heapam.c#heap_delete-clear-vm](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3120-L3146)
- [heapam.c#heap_update-attr-bitmaps](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3434-L3437)
- [heapam.c:4429](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4429)
- [heapam.c#heap_inplace_update_and_unlock-send](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6888-L6892)
- [heapam.c#heap_inplace_unlock](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6914-L6920)
- [heapam_handler.c#index-build-dead](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1419-L1423)
- [heapam_handler.c#index-build-predicate](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1636-L1644)
- [heapam_visibility.c#HeapTupleSatisfiesVacuumHorizon-same-xact](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1253-L1266)
- [heapam_visibility.c#HeapTupleSatisfiesVacuumHorizon-xmax-in-progress](../../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1383-L1386)
- [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89)
- [vacuumlazy.c#LVRelState](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L148-L156)
- [vacuumlazy.c:512-513](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L512-L513)
- [vacuumlazy.c#heap_vacuum_rel-relstats](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575)
- [vacuumlazy.c#lazy_vacuum-call](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1066)
- [vacuumlazy.c:1051-1052](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1051-L1052)
- [vacuumlazy.c:1900](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1900)
- [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2347)
- [genam.c#RelationGetIndexScan-recovery](../../../../raw/postgres-17/src/backend/access/index/genam.c#L107-L119)
- [indexam.c#index_fetch_heap-kill](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L644-L652)
- [README#half-dead](../../../../raw/postgres-17/src/backend/access/nbtree/README#L247-L259)
- [README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424)
- [README#notes-about-deduplication](../../../../raw/postgres-17/src/backend/access/nbtree/README#L904-L948)
- [README#bottom-up-added-in-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L981)
- [README#deduplication-and-bottom-up-note](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)
- [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L33-L58)
- [nbtinsert.c#_bt_insertonpg-fastroot](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1250-L1295)
- [nbtinsert.c:1720](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1720)
- [nbtinsert.c#_bt_newlevel-root-level](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2507-L2513)
- [nbtinsert.c#early-returns](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2721-L2776)
- [nbtinsert.c#dedup-gate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2778-L2781)
- [nbtpage.c#_bt_vacuum_needs_cleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L172-L223)
- [nbtpage.c#_bt_metaversion-comment](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L720-L737)
- [nbtpage.c#_bt_allocbuf](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L868-L988)
- [nbtpage.c#_bt_allocbuf-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L905)
- [nbtpage.c:903](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L903)
- [nbtpage.c:978](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L978)
- [nbtpage.c:1526](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1526)
- [nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)
- [nbtpage.c#unlink-siblings](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2597-L2612)
- [nbtpage.c:3050](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3050)
- [nbtree.c:108-109](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L108-L109)
- [nbtree.c:114](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L114)
- [nbtree.c:115](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L115)
- [nbtree.c:119](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L119)
- [nbtree.c:134](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L134)
- [nbtree.c:143](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L143)
- [nbtree.c#btgettuple-kill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L226-L245)
- [nbtree.c#btendscan-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L424-L426)
- [nbtree.c:1168](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1168)
- [nbtsearch.c:1721](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L1721)
- [nbtsearch.c:1850](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L1850)
- [nbtsearch.c#_bt_steppage-killitems](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2049-L2051)
- [nbtsearch.c#step-right](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2207-L2219)
- [nbtsort.c:338](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L338)
- [nbtsort.c:599](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L599)
- [nbtsort.c#_bt_pagestate-leaf-fill](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L665)
- [nbtsort.c:665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L665)
- [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1063-L1128)
- [nbtsort.c#_bt_load-deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)
- [nbtsort.c#_bt_load-first-page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1287-L1290)
- [nbtsplitloc.c#split-strategies](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L364-L416)
- [nbtsplitloc.c#many-duplicates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L374-L405)
- [nbtsplitloc.c#single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L389-L416)
- [nbtsplitloc.c#_bt_deltasortsplits](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L561-L588)
- [nbtsplitloc.c#_bt_bestsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L770-L812)
- [nbtsplitloc.c#_bt_defaultinterval](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L852-L920)
- [nbtsplitloc.c#_bt_strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L985-L1041)
- [nbtsplitloc.c#_bt_strategy-many-duplicates](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L985-L1009)
- [nbtsplitloc.c#_bt_split_penalty](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1119-L1153)
- [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)
- [nbtxlog.c#btree_xlog_vacuum](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L598-L634)
- [tableam.c#table_block_relation_estimate_size-density](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747)
- [tableam.c#table_block_relation_estimate_size-allvisfrac](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L749-L760)
- [Catalog.pm#client_code](../../../../raw/postgres-17/src/backend/catalog/Catalog.pm#L61-L66)
- [Catalog.pm#client_code-push](../../../../raw/postgres-17/src/backend/catalog/Catalog.pm#L197-L205)
- [genbki.pl#client_code](../../../../raw/postgres-17/src/backend/catalog/genbki.pl#L563-L567)
- [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3083-L3087)
- [index.c#index_create-pg_class](../../../../raw/postgres-17/src/backend/catalog/index.c#L1008-L1024)
- [index.c:1015](../../../../raw/postgres-17/src/backend/catalog/index.c#L1015)
- [index.c:1459](../../../../raw/postgres-17/src/backend/catalog/index.c#L1459)
- [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842)
- [index.c#index_update_stats-write](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2923)
- [index.c#index_update_stats-relallvisible](../../../../raw/postgres-17/src/backend/catalog/index.c#L2851-L2928)
- [index.c#index_build-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3099-L3101)
- [index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3131)
- [index.c#index_build-index-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3133-L3135)
- [index.c#reindex_index-rebuild](../../../../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789)
- [index.c#reindex_index-indcheckxmin](../../../../raw/postgres-17/src/backend/catalog/index.c#L3800-L3850)
- [index.c:4048](../../../../raw/postgres-17/src/backend/catalog/index.c#L4048)
- [analyze.c:527](../../../../raw/postgres-17/src/backend/commands/analyze.c#L527)
- [analyze.c#do_analyze_rel-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645)
- [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)
- [analyze.c#do_analyze_rel-index-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L694-L721)
- [analyze.c#partial-index-population](../../../../raw/postgres-17/src/backend/commands/analyze.c#L902-L908)
- [analyze.c#tupleFract](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)
- [analyze.c#compute_scalar_stats-stadistinct](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2550-L2560)
- [analyze.c#compute_scalar_stats-complete-mcv](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2614-L2636)
- [cluster.c#check_index_is_clusterable-partial](../../../../raw/postgres-17/src/backend/commands/cluster.c#L525-L534)
- [cluster.c:670](../../../../raw/postgres-17/src/backend/commands/cluster.c#L670)
- [cluster.c#plan_cluster_use_sort-call](../../../../raw/postgres-17/src/backend/commands/cluster.c#L948-L951)
- [cluster.c:1508](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1508)
- [matview.c:890](../../../../raw/postgres-17/src/backend/commands/matview.c#L890)
- [tablecmds.c#ExecuteTruncateGuts-in-place](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2133-L2145)
- [tablecmds.c#ExecuteTruncateGuts-rewrite](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2167-L2189)
- [tablecmds.c:5873](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L5873)
- [vacuum.c#index_cleanup-default](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179)
- [vacuum.c:2260](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2260)
- [execIndexing.c#ExecInsertIndexTuples-loop](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L343-L387)
- [nodeBitmapHeapscan.c#shared-bitmap-build](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L130-L141)
- [nodeBitmapHeapscan.c#shared-iterator](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L143-L170)
- [nodeBitmapHeapscan.c:241](../../../../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L241)
- [pqcomm.c:453](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L453)
- [tidbitmap.c#tbm_shared_iterate](../../../../raw/postgres-17/src/backend/nodes/tidbitmap.c#L1044-L1136)
- [allpaths.c#total_table_pages](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216)
- [allpaths.c:221](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L221)
- [allpaths.c:322](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L322)
- [allpaths.c:351](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L351)
- [allpaths.c:411](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L411)
- [allpaths.c:499](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L499)
- [allpaths.c:581](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L581)
- [allpaths.c:783](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L783)
- [allpaths.c#compute_parallel_worker-reloption](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4208-L4213)
- [allpaths.c#compute_parallel_worker-threshold](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4216-L4227)
- [allpaths.c#compute_parallel_worker-index-ramp](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4253-L4272)
- [allpaths.c:4276](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4276)
- [clausesel.c#clauselist_selectivity](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L100-L108)
- [clausesel.c:136](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L136)
- [clausesel.c:183](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L183)
- [clausesel.c:848](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L848)
- [costsize.c#clamp_row_est](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L202-L218)
- [costsize.c#cost_seqscan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L304-L305)
- [costsize.c#cost_seqscan](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L315-L322)
- [costsize.c#cost_index-rows](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L589-L604)
- [costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)
- [costsize.c#cost_index-save-indextotalcost](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L623-L629)
- [costsize.c:628](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L628)
- [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)
- [costsize.c#cost_index-repeated-uncorrelated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L680-L683)
- [costsize.c#cost_index-repeated-correlated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L702-L707)
- [costsize.c#cost_index-normal-uncorrelated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L720-L723)
- [costsize.c#cost_index-normal-correlated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L733-L746)
- [costsize.c:765](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L765)
- [costsize.c#cost_index-correlation-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L781-L800)
- [costsize.c#cost_index-cpu](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L795-L800)
- [costsize.c:1044](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044)
- [costsize.c#cost_bitmap_tree_node](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1114-L1144)
- [costsize.c#cost_bitmap_tree_node-indexpath](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1114-L1128)
- [costsize.c:3577](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L3577)
- [costsize.c:4016](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L4016)
- [costsize.c:5264](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L5264)
- [costsize.c#compute_bitmap_pages-repeated](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476)
- [indxpath.c#create_index_paths-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L260-L266)
- [indxpath.c:279](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L279)
- [indxpath.c#choose_bitmap_and-call](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L340-L343)
- [indxpath.c:341](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L341)
- [indxpath.c:343](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L343)
- [indxpath.c:347](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L347)
- [indxpath.c:722](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L722)
- [indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751)
- [indxpath.c#get_index_paths-nonnative-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L753-L766)
- [indxpath.c#build_index_paths-scantype](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L826-L842)
- [indxpath.c#build_index_paths-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L862-L885)
- [indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897)
- [indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944)
- [indxpath.c#build_index_paths-generate](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L954-L962)
- [indxpath.c:963](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L963)
- [indxpath.c#build_index_paths-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L975-L1002)
- [indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489)
- [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526-L1553)
- [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571)
- [indxpath.c#match_restriction_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1968-L1974)
- [indxpath.c#match_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2051-L2064)
- [indxpath.c#match_clause_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2084-L2136)
- [indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266)
- [indxpath.c#IsBooleanOpfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2271-L2286)
- [indxpath.c#match_opclause_to_indexcol-op_in_opfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2433-L2459)
- [indxpath.c#get_index_clause_from_support](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2557-L2615)
- [indxpath.c#match_saopclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2651-L2669)
- [indxpath.c#check_index_predicates-predOK](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3272-L3350)
- [createplan.c#create_bitmap_subplan-indexpath](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3480-L3485)
- [initsplan.c:165](../../../../raw/postgres-17/src/backend/optimizer/plan/initsplan.c#L165)
- [planmain.c:170](../../../../raw/postgres-17/src/backend/optimizer/plan/planmain.c#L170)
- [planmain.c:280](../../../../raw/postgres-17/src/backend/optimizer/plan/planmain.c#L280)
- [planner.c#plan_cluster_use_sort](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6761-L6874)
- [planner.c#plan_cluster_use_sort-short-circuits](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6788-L6841)
- [planner.c#plan_cluster_use_sort-compare](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L6861-L6873)
- [prepjointree.c#is_simple_subquery](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1689-L1701)
- [pathnode.c#STD_FUZZ_FACTOR](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L42-L47)
- [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L163-L212)
- [pathnode.c:452](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L452)
- [pathnode.c:1024](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L1024)
- [pathnode.c#create_bitmap_heap_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L1042-L1068)
- [plancat.c#get_relation_info-table-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L196-L202)
- [plancat.c:201](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L201)
- [plancat.c:205](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L205)
- [plancat.c:253](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L253)
- [plancat.c#get_relation_info-index-skips](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L281)
- [plancat.c#get_relation_info-canreturn](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L296-L301)
- [plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335)
- [plancat.c#amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L331-L332)
- [plancat.c#get_relation_info-sortopfamily](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L340-L422)
- [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)
- [plancat.c:471](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471)
- [plancat.c:476](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L476)
- [plancat.c:484](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L484)
- [plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500)
- [plancat.c:488](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488)
- [plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)
- [plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)
- [plancat.c#restriction_selectivity](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1951-L1979)
- [relnode.c:340](../../../../raw/postgres-17/src/backend/optimizer/util/relnode.c#L340)
- [like_support.c#prefix_selectivity](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1213-L1270)
- [like_support.c:1245](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1245)
- [like_support.c:1266](../../../../raw/postgres-17/src/backend/utils/adt/like_support.c#L1266)
- [selfuncs.c#scalarineqsel-ctid-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L599-L601)
- [selfuncs.c:690](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L690)
- [selfuncs.c#ineq_histogram_selectivity-endpoints](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1076-L1136)
- [selfuncs.c#ineq_histogram_selectivity-above-last-bound](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1162-L1168)
- [selfuncs.c#ineq_histogram_selectivity-flip-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1319-L1344)
- [selfuncs.c#scalarineqsel_wrapper](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1397-L1503)
- [selfuncs.c:1461](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1461)
- [selfuncs.c#scalargtsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1490-L1494)
- [selfuncs.c#estimate_array_length](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2131-L2207)
- [selfuncs.c#estimate_array_length-statistics](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L2162-L2206)
- [selfuncs.c#mergejoinscansel-ranges](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3141-L3156)
- [selfuncs.c#mergejoinscansel-scalarineqsel](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L3164-L3203)
- [selfuncs.c:5203-5204](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5203-L5204)
- [selfuncs.c:5375-5376](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5375-L5376)
- [selfuncs.c#get_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5969-L6098)
- [selfuncs.c#get_variable_range-not-used](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5993-L6003)
- [selfuncs.c#get_actual_variable_range](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6157-L6331)
- [selfuncs.c#get_actual_variable_range-index-choice](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6179-L6239)
- [selfuncs.c#get_actual_variable_range-skips](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6196-L6212)
- [selfuncs.c:6197](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6197)
- [selfuncs.c#get_actual_variable_range-endpoint-calls](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6282-L6314)
- [selfuncs.c#get_actual_variable_endpoint](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6349-L6501)
- [selfuncs.c:6365](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6365)
- [selfuncs.c#get_actual_variable_endpoint-comment](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6371-L6414)
- [selfuncs.c#get_actual_variable_endpoint-snapshot](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6376-L6386)
- [selfuncs.c#get_actual_variable_endpoint-killed](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6388-L6397)
- [selfuncs.c#get_actual_variable_endpoint-give-up](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6409-L6413)
- [selfuncs.c#get_actual_variable_endpoint-horizon](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6416)
- [selfuncs.c#get_actual_variable_endpoint-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6415-L6458)
- [selfuncs.c#get_actual_variable_endpoint-limit](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6436-L6457)
- [selfuncs.c#VISITED_PAGES_LIMIT](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447-L6455)
- [selfuncs.c:6447](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6447)
- [selfuncs.c#genericcostestimate-num_sa_scans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6657-L6679)
- [selfuncs.c:6682](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6682)
- [selfuncs.c:6695](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6695)
- [selfuncs.c#genericcostestimate-tuple-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6707-L6715)
- [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)
- [selfuncs.c#genericcostestimate-numIndexPages-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6732)
- [selfuncs.c#genericcostestimate-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6729-L6786)
- [selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)
- [selfuncs.c:6767](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6767)
- [selfuncs.c#genericcostestimate-single-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787)
- [selfuncs.c:6810](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6810)
- [selfuncs.c#btcostestimate-bound-saop](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6904-L6989)
- [selfuncs.c#btcostestimate-boundary-saop](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6950-L6961)
- [selfuncs.c#btcostestimate-numIndexTuples](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7007-L7019)
- [selfuncs.c:7015](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7015)
- [selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042)
- [selfuncs.c:7064](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7064)
- [selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073)
- [selfuncs.c:7073](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7073)
- [selfuncs.c#btcostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7106)
- [selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091)
- [selfuncs.c#btcostestimate-log2-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7086-L7091)
- [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)
- [selfuncs.c#btcostestimate-stats-hooks](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7116-L7171)
- [selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7213-L7253)
- [selfuncs.c:7221](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7221)
- [selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7256-L7308)
- [selfuncs.c:7265](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7265)
- [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7311-L7363)
- [selfuncs.c:7320](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7320)
- [selfuncs.c#gincost_pattern-searchmode](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7454-L7489)
- [selfuncs.c:7467](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7467)
- [selfuncs.c#gincost_scalararrayopexpr](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7550-L7660)
- [selfuncs.c#gincost_scalararrayopexpr-arrayScans](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7648-L7658)
- [selfuncs.c#gincostestimate-header](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L7671)
- [selfuncs.c:7674](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7674)
- [selfuncs.c:7675](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7675)
- [selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711)
- [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767)
- [selfuncs.c#gincostestimate-trust](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7727)
- [selfuncs.c#gincostestimate-pending-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7724-L7727)
- [selfuncs.c#gincostestimate-stats-branch](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7729-L7747)
- [selfuncs.c#gincostestimate-invented](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7748-L7767)
- [selfuncs.c#gincostestimate-tablespace-costs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7786-L7789)
- [selfuncs.c#gincostestimate-saop-clause](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7823-L7833)
- [selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877)
- [selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886)
- [selfuncs.c:7886](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7886)
- [selfuncs.c#gincostestimate-entrypages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7888-L7914)
- [selfuncs.c:7895](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7895)
- [selfuncs.c#gincostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7919-L7935)
- [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)
- [selfuncs.c:7980](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7980)
- [selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7982-L8006)
- [selfuncs.c#gincostestimate-datapages-cache](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8017-L8025)
- [selfuncs.c#gincostestimate-qualcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8031-L8048)
- [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8052-L8061)
- [selfuncs.c:8063](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8063)
- [selfuncs.c:8135-8136](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8135-L8136)
- [selfuncs.c:8166-8167](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8166-L8167)
- [inval.c#CacheInvalidateHeapTupleCommon-pg_class](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L1383-L1392)
- [inval.c:1447](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L1447)
- [plancache.c#CheckCachedPlan-transient](../../../../raw/postgres-17/src/backend/utils/cache/plancache.c#L866-L873)
- [plancache.c#BuildCachedPlan-transient](../../../../raw/postgres-17/src/backend/utils/cache/plancache.c#L1022-L1033)
- [relcache.c#RelationBuildLocalRelation-rd_rel](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3656-L3659)
- [relcache.c#summarizing-split](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5390-L5398)
- [guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792)
- [guc_tables.c#enable_indexscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L793-L802)
- [guc_tables.c#enable_indexonlyscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L803-L812)
- [guc_tables.c#enable_mergejoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L893-L902)
- [guc_tables.c#enable_hashjoin](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L903-L912)
- [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)
- [guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2070-L2079)
- [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270)
- [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2393-L2401)
- [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)
- [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)
- [guc_tables.c#max_parallel_workers_per_gather](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3419-L3428)
- [guc_tables.c#max_parallel_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3430-L3439)
- [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)
- [guc_tables.c#min_parallel_table_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3520-L3529)
- [guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540)
- [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)
- [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729)
- [guc_tables.c#parallel_tuple_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3730-L3740)
- [guc_tables.c#parallel_setup_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3741-L3751)
- [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4425-L4434)
- [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4436-L4445)
- [guc_tables.c#application_name](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4644-L4653)
- [brin.h#BrinStatsData](../../../../raw/postgres-17/src/include/access/brin.h#L32-L36)
- [gin.h#GIN_SEARCH_MODE](../../../../raw/postgres-17/src/include/access/gin.h#L34-L37)
- [nbtree.h#allequalimage-upgrade](../../../../raw/postgres-17/src/include/access/nbtree.h#L135-L141)
- [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202)
- [nbtree.h:200](../../../../raw/postgres-17/src/include/access/nbtree.h#L200)
- [nbtree.h:202](../../../../raw/postgres-17/src/include/access/nbtree.h#L202)
- [nbtree.h:225](../../../../raw/postgres-17/src/include/access/nbtree.h#L225)
- [nbtree.h#BTDeletedPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L230-L236)
- [nbtree.h:1134](../../../../raw/postgres-17/src/include/access/nbtree.h#L1134)
- [nbtree.h#BTGetFillFactor-BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1150)
- [pg_amop.dat#gin-tsvector_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1290-L1296)
- [pg_amop.dat#btree-hash-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1571-L1591)
- [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611)
- [pg_amop.dat#gin-jsonb_path_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1613-L1622)
- [pg_class.h#NOTES](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L11-L16)
- [pg_class.h#CATALOG](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L32)
- [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)
- [pg_class.h:177](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L177)
- [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L43)
- [pg_operator.dat#int4gt](../../../../raw/postgres-17/src/include/catalog/pg_operator.dat#L453-L456)
- [pqcomm.h:60](../../../../raw/postgres-17/src/include/libpq/pqcomm.h#L60)
- [pathnodes.h#total_table_pages](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L483-L484)
- [pathnodes.h#RelOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L941-L944)
- [pathnodes.h:1168](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1168)
- [pathnodes.h:1181](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1181)
- [cost.h#default-costs](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25-L28)
- [cost.h:25](../../../../raw/postgres-17/src/include/optimizer/cost.h#L25)
- [cost.h:28](../../../../raw/postgres-17/src/include/optimizer/cost.h#L28)
- [cost.h:34](../../../../raw/postgres-17/src/include/optimizer/cost.h#L34)
- [bufmgr.h:280-281](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L280-L281)
- [lsyscache.h#AttStatsSlot](../../../../raw/postgres-17/src/include/utils/lsyscache.h#L46-L62)
- [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L667-L669)
- [fe-connect.c#conninfo_add_defaults](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L6201-L6245)
- [fe-connect.c#hostaddr-option](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L223-L225)
- [fe-connect.c#pqConnectOptions2-host-type](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L1189-L1204)
- [fe-connect.c#PQconnectPoll-host-address](../../../../raw/postgres-17/src/interfaces/libpq/fe-connect.c#L2754-L2764)
- [amutils.out#column-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L96-L108)
- [amutils.out#am-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L152-L157)
- [nbtsplitloc.c:317](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L317)
- [nbtsplitloc.c:364](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L364)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Mandatory B-Tree Bloat Tests (unverified)](../../common-concepts/mandatory-btree-bloat-tests.md) - the protocol for claims about wasted B-tree space and rebuild decisions; this page's rebuild advice is not scored against it.
- [Mandatory GIN Bloat Tests (unverified)](../../common-concepts/mandatory-gin-bloat-tests.md) - the protocol for claims about wasted GIN space, under which a pending list is deferred work rather than waste.
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [log](../../../log.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](../indexing/reindex-index-concurrently.md) - the online rebuild that resets every input on this page.
- [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](../indexing/partial-indexes-pros-cons.md) - more on the partial-index costing path that behaves differently here.
- [How Bottom-Up Index Deletion and B-Tree Deduplication Work in PostgreSQL 17 (unverified)](../indexing/bottom-up-deletion-and-btree-deduplication.md) - the two mechanisms fixtures N, P and P-100 exercise, including the heap-block budget that decides what a bottom-up pass can free.
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](../../../v12/questions/query-planning/bloated-indexes-query-planner.md) - the same question, and the same GIN-versus-B-tree follow-up, answered against the v12 pin.
