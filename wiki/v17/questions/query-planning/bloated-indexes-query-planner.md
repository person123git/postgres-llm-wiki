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
- [Answer](#answer)
- [Planner Mechanisms](#planner-mechanisms)
  - [How the planner obtains its index-size inputs](#how-the-planner-obtains-its-index-size-inputs)
  - [1. Physical page count enters index cost](#1-physical-page-count-enters-index-cost)
  - [2. B-tree height carries an explicit anti-bloat charge](#2-b-tree-height-carries-an-explicit-anti-bloat-charge)
  - [3. Index pages enter cache modeling and parallel worker counts](#3-index-pages-enter-cache-modeling-and-parallel-worker-counts)
  - [4. v17 only: index pages cap the ScalarArrayOp descent estimate](#4-v17-only-index-pages-cap-the-scalararrayop-descent-estimate)
  - [Which access methods share these mechanisms](#which-access-methods-share-these-mechanisms)
- [What The Planner Does Not See](#what-the-planner-does-not-see)
- [Types Of Bloated Indexes](#types-of-bloated-indexes)
  - [Low-density live leaf pages](#low-density-live-leaf-pages)
  - [Deleted and half-dead pages](#deleted-and-half-dead-pages)
  - [Extra tree levels](#extra-tree-levels)
  - [Physically fragmented leaf chains](#physically-fragmented-leaf-chains)
  - [Version-churn duplicates from non-HOT UPDATEs](#version-churn-duplicates-from-non-hot-updates)
  - [Duplicate-heavy indexes with deduplication disabled](#duplicate-heavy-indexes-with-deduplication-disabled)
  - [Bloat that VACUUM deliberately skipped](#bloat-that-vacuum-deliberately-skipped)
  - [Non-B-tree bloat](#non-b-tree-bloat)
- [How Density And Fragmentation Affect Different Queries](#how-density-and-fragmentation-affect-different-queries)
- [Exact-Pin Measurements](#exact-pin-measurements)
  - [Fixtures and method](#fixtures-and-method)
  - [Index cost is a closed form in pages, tuples and tree height](#index-cost-is-a-closed-form-in-pages-tuples-and-tree-height)
  - [The point lookup is nearly blind to bloat](#the-point-lookup-is-nearly-blind-to-bloat)
  - [The tree-height charge, isolated to the cent](#the-tree-height-charge-isolated-to-the-cent)
  - [A mostly-empty index: the fast root drops and pages outnumber rows](#a-mostly-empty-index-the-fast-root-drops-and-pages-outnumber-rows)
  - [Leaf fragmentation contributes exactly zero](#leaf-fragmentation-contributes-exactly-zero)
  - [Two indexes with the same cost and opposite avg_leaf_density](#two-indexes-with-the-same-cost-and-opposite-avg_leaf_density)
  - [The planner reads the live block count, not pg_class.relpages](#the-planner-reads-the-live-block-count-not-pg_classrelpages)
  - [Bloat changes plans](#bloat-changes-plans)
  - [Bloat changes parallel worker counts](#bloat-changes-parallel-worker-counts)
  - [Index pages in the cache model](#index-pages-in-the-cache-model)
  - [The v17 ScalarArrayOp descent clamp](#the-v17-scalararrayop-descent-clamp)
  - [Version churn with and without a held snapshot](#version-churn-with-and-without-a-held-snapshot)
  - [Deduplication](#deduplication)
- [What Changed Since PostgreSQL 12](#what-changed-since-postgresql-12)
  - [The cost code that did not change at all](#the-cost-code-that-did-not-change-at-all)
  - [v16: the 50x page charge became a macro](#v16-the-50x-page-charge-became-a-macro)
  - [v17: index pages now cap ScalarArrayOp descents](#v17-index-pages-now-cap-scalararrayop-descents)
  - [v16: partitioned indexes are zeroed out](#v16-partitioned-indexes-are-zeroed-out)
  - [v14: reltuples turns negative for never-analyzed relations](#v14-reltuples-turns-negative-for-never-analyzed-relations)
  - [v13: deduplication](#v13-deduplication)
  - [v14: bottom-up index deletion](#v14-bottom-up-index-deletion)
  - [v14: faster recycling of deleted pages](#v14-faster-recycling-of-deleted-pages)
  - [v14: VACUUM can skip index vacuuming entirely](#v14-vacuum-can-skip-index-vacuuming-entirely)
  - [Since-v12 summary table](#since-v12-summary-table)
- [Settings That Move The Boundary](#settings-that-move-the-boundary)
- [Practical Interpretation](#practical-interpretation)
- [Key Data Structures](#key-data-structures)
- [Caller And Callee Boundary](#caller-and-callee-boundary)
- [Build, Generated-Header, And Extension Boundary](#build-generated-header-and-extension-boundary)
- [Tests And Explicit Test Absence](#tests-and-explicit-test-absence)
- [Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead](#follow-up-when-a-gin-index-is-discarded-and-a-b-tree-is-used-instead)
  - [Short answer](#short-answer)
  - [Gate 1: the clause never matches the GIN index](#gate-1-the-clause-never-matches-the-gin-index)
  - [Gate 2: the required plan shape rules GIN out](#gate-2-the-required-plan-shape-rules-gin-out)
  - [Gate 3: cost, and why GIN loses on the same column](#gate-3-cost-and-why-gin-loses-on-the-same-column)
  - [A bloated GIN index loses to a B-tree](#a-bloated-gin-index-loses-to-a-b-tree)
  - [Stale GIN metapage statistics](#stale-gin-metapage-statistics)
  - [The keyless full-index path on a partial GIN index](#the-keyless-full-index-path-on-a-partial-gin-index)
  - [Jobs no GIN index can be created for](#jobs-no-gin-index-can-be-created-for)
  - [Where GIN still wins](#where-gin-still-wins)
  - [GIN exact-pin measurements](#gin-exact-pin-measurements)
  - [GIN settings that move the boundary](#gin-settings-that-move-the-boundary)
  - [GIN key data structures](#gin-key-data-structures)
  - [GIN caller and callee boundary](#gin-caller-and-callee-boundary)
  - [GIN tests and explicit test absence](#gin-tests-and-explicit-test-absence)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [The script](#the-script)
- [Open Questions](#open-questions)
- [Related Pages](#related-pages)
- [Evidence Map](#evidence-map)
- [Context Reviewed](#context-reviewed)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, are there mechanisms to penalize bloated indexes in the query planner? If there are, give a comprehensive explanation with examples of types of bloated indexes and how leaf fragmentation or density affects them, and what changed since PostgreSQL 12.

Follow-up:

When might a GIN index be discarded by the query planner and a B-tree used instead?

Review prompt, 2026-09-19, corrected form: `Follow AGENTS.md. In PostgreSQL 17, review: Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified).` The request as written read `follow agents.md, in postgresql 17 , review : # Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)`; the defects were `agents.md` for AGENTS.md, lowercase `postgresql`, a space before the comma and before the colon, the lowercase sentence opening `follow`, a stray Markdown heading marker `#` carried in with the pasted title, and no terminal period. The asker chose **correct and restate** and **report only**, read the findings, and then asked for them to be fixed (`fix issues`). That pass corrected nine source readings, replaced the history method, re-measured every number on the 17.11 pin from the script now filed under [Measurement Script](#measurement-script), and is recorded in the first 2026-09-19 entry of [log](../../../log.md).

Second review prompt, 2026-09-19, corrected form: `Follow AGENTS.md. In PostgreSQL 17, for the question "Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)", fix these issues:` followed by a list of twelve findings, fourteen counting the three folded into the last one. The request as written read `follow agents.md, in postgresql 17 , for question : # Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified) fix these issues :`; the defects were `agents.md` for AGENTS.md, lowercase `postgresql`, the lowercase sentence openings `follow` and `fix`, a space before the comma in `17 ,` and before each of the two colons, a stray Markdown heading marker `#` carried in with the pasted title, and no terminal punctuation. The pasted finding list carried two defects of its own: the numbering stopped at 11 and the twelfth item began mid-sentence at `planner.md:148) omits potentially stale cached metadata`, its opening clause missing, and item 2's `fa predict` was unquoted so it did not read as a command. The asker chose **correct and restate**, and chose to **replace** fixture `l3` rather than keep its degenerate form. All fourteen sub-findings were confirmed against the pin and fixed; that pass is the second 2026-09-19 entry of [log](../../../log.md).

## Answer

Yes, but every mechanism is indirect. PostgreSQL 17 stores no planner field called `bloat`, `avg_leaf_density`, or `leaf_fragmentation`, and `pg_class` carries only `relpages`, `reltuples`, and `relallvisible` ([pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)). Bloat is penalized only when it shows up as one of exactly four planner inputs:

| Planner input | Filled by | Penalizes bloat? |
|---|---|---|
| `IndexOptInfo.pages` | live block count from the storage manager (non-partial) or `estimate_rel_size()` (partial) | Yes, this is the main channel |
| `IndexOptInfo.tree_height` | `_bt_getrootheight()`, B-tree only | Yes, one explicit charge per extra level |
| `index->pages` in the cache model, and the touched-pages estimate `numIndexPages` in parallel-worker selection | the same `pages` value: whole for the cache model, prorated by selectivity for worker counts | Yes, second-order |
| `ceil(index->pages * 0.3333333)` descent clamp | the same `pages` value | Yes, and this channel is new in v17 |

`get_relation_info()` fills all of them while it has the index open ([plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)), and the access method's `amcostestimate` turns them into cost ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)).

The consequences are sharply uneven, and measurements on an isolated server built from this exact pin make that concrete:

- A **broad scan** is penalized in direct proportion to page count. A 1,000,000-row index at 90.06% leaf density (2,745 blocks) costs `28480.42` for a full index-only scan; its byte-for-byte logical twin at 9.62% density (26,411 blocks) costs `123144.43`.
- A **point lookup** is almost blind to bloat for as long as the index has no more pages than the table has rows. The same two indexes both cost exactly `4.44`. The only bloat signal available to a single-leaf-page search is tree height, worth `(tree_height + 1) * 50 * cpu_operator_cost`, which is `0.125` per extra level at default settings. Once pages outnumber rows the blindness ends: a 2,745-block index over 1,000 surviving rows prices the same kind of lookup at `12.29`, and at `4.29` after a rebuild.
- **Leaf fragmentation is worth exactly zero.** A 1,148-block index at 49.87% `leaf_fragmentation` and a 744-block index at 0% differ in cost by `1616.00`, which is precisely `(1148 - 744) * random_page_cost`. There is no residual for fragmentation.
- **`avg_leaf_density` and planner cost can point in opposite directions.** Two 2,745-block indexes over the same 100,000 rows cost an identical `12730.42`, while `pgstatindex` reports 9.27% density for one and a healthy-looking 89.18% for the other (the second hides 2,465 deleted pages).

The v17 manual describes B-tree bloat operationally as an index that "contains many empty or nearly-empty pages" and recommends `REINDEX` ([ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)), notes that a page keeps its space when "all but a few index keys on a page have been deleted" ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)), and separately says a freshly built B-tree is slightly faster because logically adjacent pages are usually physically adjacent ([maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1048-L1054)). `leaf_fragmentation` is a one-sided proxy for that last effect rather than a measure of it: it counts only leaf pages whose right-sibling link points at a *lower* block number ([pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323)). A leaf chain that always moves forward but in thousand-block jumps scores 0% while being no more physically adjacent than one that moves backwards. Whatever part of the effect the metric does capture, the planner ignores all of it.

Since PostgreSQL 12, the cost formulas themselves barely moved: `index_pages_fetched()` and `cost_index()` are byte-identical to their `REL_12_0` text, and the bloat-relevant core of `genericcostestimate()` is unchanged. What changed is (a) one new v17 penalty channel through `index->pages`, and (b) a great deal of nbtree work in v13 and v14 that reduces how much bloat exists to be penalized in the first place.

## Planner Mechanisms

### How the planner obtains its index-size inputs

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

1. **Bloat is visible immediately, without `ANALYZE`.** `pages` is a live `smgrnblocks()` answer, not `pg_class.relpages`. Measured below: forging `pg_class.relpages` to `1` leaves the cost unchanged.
2. **Removing index entries does not lower `tuples`.** For a non-partial index, `tuples` is the *table's* row estimate, so the `pages / tuples` ratio rises purely as pages accumulate.
3. **Partial indexes take a different path.** `estimate_rel_size()` still reports live blocks as `*pages`, but derives `*tuples` from `pg_class` tuple density after discounting the metapage ([plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)). A partial index therefore inherits stale-statistics behavior that an ordinary index does not.

For B-trees only, the height comes from the metapage while the index is open; every other AM is left at `-1` ([plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500)). That read is not necessarily a fresh one — it can be answered from a per-backend cached copy of the metapage, which is the third caveat in [2. B-tree height carries an explicit anti-bloat charge](#2-b-tree-height-carries-an-explicit-anti-bloat-charge). Partitioned indexes have no storage, so all three fields are zeroed ([plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)).

### 1. Physical page count enters index cost

`genericcostestimate()` estimates touched index pages as a pro-rata share of the whole index, and its comment states it counts only leaf pages, ignoring the metapage and upper levels:

```c
if (index->pages > 1 && index->tuples > 1)
    numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
else
    numIndexPages = 1.0;
```

([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732))

For a single scan it charges `spc_random_page_cost` per touched page ([selfuncs.c#genericcostestimate-single-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6780-L6787)); for repeated scans it first runs the page count through the Mackert-Lohman cache model ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)).

This is the dominant penalty, and it scales linearly with bloat for anything that reads a meaningful fraction of the index. Two guards blunt it: the `index->pages > 1 && index->tuples > 1` test floors a tiny index at one page, and `ceil()` collapses a selective lookup to one page for as long as the index has no more pages than the table has rows. Past that point the second guard fails. A one-row lookup is charged `ceil(pages / tuples)` pages, which is the shape a drained queue table leaves behind and is measured in [A mostly-empty index: the fast root drops and pages outnumber rows](#a-mostly-empty-index-the-fast-root-drops-and-pages-outnumber-rows).

### 2. B-tree height carries an explicit anti-bloat charge

`btcostestimate()` adds two CPU charges after delegating to `genericcostestimate()`. The first is roughly `log2(N)` comparisons ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7075-L7091)). The second exists specifically to stop bloated indexes from looking free:

```c
descentCost = (index->tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost;
```

([selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106))

The in-tree comment is explicit: "if we had no such charge at all, bloated indexes would appear to have the same search cost as unbloated ones, at least in cases where only a single leaf page is expected to be visited." `DEFAULT_PAGE_CPU_MULTIPLIER` is `50.0` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), so each extra level costs `50 * cpu_operator_cost` = `0.125` at the default `cpu_operator_cost` of `0.0025`.

Three caveats matter.

First, this is a *level* charge, not a density charge: it changes only when the B-tree gains or loses a level.

Second, the planner's height is the **fast-root** level, not the true root level. `_bt_getrootheight()` returns `btm_fastlevel` ([nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)), and page deletion can lower `btm_fastlevel` in place ([nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)); the nbtree README explains the fast-root idea and states that tree height can never *decrease* by page deletion alone ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)). `pgstatindex` reports `btm_level` as `tree_level` ([pgstatindex.c#metapage](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265)), so `pgstatindex.tree_level` and the planner's `tree_height` are not guaranteed to be the same number. Fixture F below measures them apart: `tree_level` 2 against `fastlevel` 1.

Third, the number the planner gets can be stale. `_bt_getrootheight()` reads the metapage only on the first call in a backend, copies the whole `BTMetaPageData` into `rel->rd_amcache`, and answers every later call from that copy without rechecking it. The function's comment says the staleness is deliberate: "Since it's only an estimate, slightly-stale data is fine, hence we don't worry about updating previously cached data" ([nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)). The cache is per-backend and is freed only when the index's relcache entry is invalidated ([relcache.c#RelationInvalidateRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2534-L2557)), so a long-lived backend can keep charging for a level that VACUUM has already removed, and two backends can price the same index differently at the same moment. Every measurement below is taken from a fresh `psql` session, so none of them is reading a stale height.

### 3. Index pages enter cache modeling and parallel worker counts

`index_pages_fetched()` prorates `effective_cache_size` across "all the tables in the query and the index currently under consideration":

```c
total_pages = root->total_table_pages + index_pages;
b = (double) effective_cache_size * T / total_pages;
```

([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951))

`root->total_table_pages` counts only non-dummy *table* pages ([allpaths.c#total_table_pages](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216)), so the index's own page count is added separately at every call site. `cost_index()` passes `index->pages` on each of its three `index_pages_fetched()` calls, two for repeated scans and one for the normal case; the normal case's perfectly-correlated estimate is computed without the cache model ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)). `genericcostestimate()` passes it for repeated index scans ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)), and `compute_bitmap_pages()` passes `get_indexpath_pages()` for repeated bitmap scans ([costsize.c#compute_bitmap_pages](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476)). A bloated index therefore claims a larger notional share of cache for itself while shrinking the share available to the heap.

Parallel worker selection reads a different number with a similar name. The `index_pages` variable that `cost_index()` hands to `compute_parallel_worker()` is the access method's `*indexPages` output, which `btcostestimate()` sets to `numIndexPages`: the pages this scan is expected to touch, not the size of the index ([costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772), [selfuncs.c#btcostestimate-outputs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7206-L7210)). The source comment says workers are computed "based on number of index pages fetched". An index-only scan passes only that estimate; a plain index scan also passes its heap-page estimate, and the smaller of the two worker counts wins. `compute_parallel_worker()` rejects a parallel path outright when the estimate is below `min_parallel_index_scan_size`, and otherwise adds one worker each time the estimate triples ([allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)). Because `numIndexPages` is a pro-rata share of `index->pages`, bloat still raises the worker count, but in proportion to the scan's selectivity rather than to the whole index. Bloat can therefore change the *shape* of a plan, not only its price.

### 4. v17 only: index pages cap the ScalarArrayOp descent estimate

`btcostestimate()` clamps the number of estimated array descents to one third of the index's physical pages:

```c
num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333));
num_sa_scans = Max(num_sa_scans, 1);
```

([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042))

`num_sa_scans` then multiplies both descent charges, and is handed to `genericcostestimate()` through `GenericCosts` ([selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7067-L7073)), where it multiplies the page charge: `numIndexPages * num_sa_scans` pages go through the Mackert-Lohman formula ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6739-L6779)). The per-tuple CPU charge is not inflated, because `btcostestimate()` first divides the tuple count by the same number ([selfuncs.c:7064](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7064)). The clamp was introduced with v17's native `ScalarArrayOpExpr` execution and does not exist in `REL_12_0`; see [v17: index pages now cap ScalarArrayOp descents](#v17-index-pages-now-cap-scalararrayop-descents). Its effect on bloat is counter-intuitive but real: a bloated index has a *higher* cap, so it is charged for more descents than its dense twin on the identical query.

### Which access methods share these mechanisms

Four core AMs route through `genericcostestimate()`: B-tree, hash, GiST and SP-GiST ([selfuncs.c:7073](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7073), [selfuncs.c:7221](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7221), [selfuncs.c:7265](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7265), [selfuncs.c:7320](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7320)). So does contrib `bloom`, the only other caller in the tree, and it is the most exposed of the five: `blcostestimate()` sets `numIndexTuples = index->tuples` because "We have to visit all index tuples anyway", so every bloom scan is charged for every page of the index ([blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42)). GIN and BRIN have their own models and never call it; both source comments say their search behavior is "completely different from other index types" ([selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L7671), [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8052-L8061)).

The height charge is B-tree-only in the sense that only B-trees supply a *measured* height. GiST and SP-GiST fill the `-1` themselves by assuming a fanout of 100 and taking `log100(index->pages)`, then apply the same formula "calculated the same as for btrees" ([selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7256-L7308), [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7311-L7363)); their descent charge therefore tracks physical page count rather than real height. `hashcostestimate()` adds no descent charge at all ([selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7213-L7253)).

Every `pgstatindex` metric discussed on this page is B-tree-only: the function reads B-tree page structures and rejects any other relation with `is not a btree index` ([pgstatindex.c#pgstatindex_impl-btree-check](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228)), which its regression test asserts for a GIN and a hash index ([sql/pgstattuple.sql#wrong-index-type](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L63)).

## What The Planner Does Not See

| Bloat signal | Planner-visible in v17? | Effect |
|---|---|---|
| More physical index blocks for the same useful keys | Yes | Raises `numIndexPages` through `index->pages / index->tuples` |
| An extra B-tree level | Yes | One `50 * cpu_operator_cost` charge per level, fast-root based |
| Deleted, half-dead, or empty pages | Yes, but as ordinary pages | They are inside `RelationGetNumberOfBlocks()`, so the planner charges for pages no scan will ever read |
| Low `avg_leaf_density` from `pgstatindex` | Not directly | Only matters when it produces extra pages or an extra level |
| High `leaf_fragmentation` from `pgstatindex` | No | Measured contribution below is exactly zero |
| Free space recorded in the index FSM | No | Nothing on the cost path reads the FSM |
| Index entries removed by VACUUM | No | For a non-partial index `tuples` stays pinned to the table estimate |
| Where inside the index the useful entries sit | No | `numIndexPages` is a flat pro-rata share |

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

So `avg_leaf_density` answers "how full are the leaf pages that still hold data", and `leaf_fragmentation` answers "how often does the right-sibling link point backwards". Neither answers "how many blocks is the planner charged for", which is the only question the cost model asks. The measurement section shows a case where `avg_leaf_density` reads 89.18% on an index that is ten times larger than it needs to be.

## Types Of Bloated Indexes

### Low-density live leaf pages

The classic case the manual describes: scattered deletions leave every leaf page allocated but nearly empty ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)). Measured: deleting 90% of 1,000,000 rows with `id % 10 <> 0` and vacuuming twice left the index at **2,745 blocks with 2,733 live leaf pages, zero deleted pages, and 9.27% `avg_leaf_density`**. This is the one bloat shape that both `avg_leaf_density` and the planner agree on, because low density here means high `pages / tuples`.

Split policy sets the ceiling on density. A rightmost leaf split uses the index fillfactor; a leaf split that the "split after new item" optimization recognizes as a localized ascending insertion either uses the fillfactor too or splits exactly after the new item; a rightmost internal split uses `BTREE_NONLEAF_FILLFACTOR`; every other internal or leaf split is `0.50`; and an all-duplicates page uses `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtsplitloc.c#single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L399-L413)). Those constants are `90`, `70`, and `96` ([nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202)), and the per-index `fillfactor` reloption takes `ShareUpdateExclusiveLock` because it only affects later inserts ([reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)).

### Deleted and half-dead pages

When a leaf page becomes completely empty, VACUUM can delete it, but the page stays in the relation as a tombstone with its sibling links intact, labelled with a `safexid` ([nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2648)). It becomes recyclable through the FSM only once no scan can still hold a reference ([README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)). nbtree never shortens its own file: nothing under `src/backend/access/nbtree/` calls `RelationTruncate()` or `smgrtruncate()`, and recycling goes through the FSM only. Those blocks therefore remain inside `RelationGetNumberOfBlocks()` and remain charged.

Measured: deleting a *contiguous* 90% (`id > 100000`) let VACUUM empty whole pages, giving **2,745 blocks made of 276 live leaf pages plus 2,465 deleted pages**, and `avg_leaf_density` of **89.18%**. A second and a third VACUUM did not shrink the fork. `REINDEX` cut it to 276 blocks. This is the shape where `avg_leaf_density` is actively misleading and the planner is right.

### Extra tree levels

Because a taller tree means more descent pages, the level charge fires. Measured: 50,000 rows produced a 139-block index at `fastlevel = 1`, while the same 50,000 rows with `fillfactor = 10` produced a 1,323-block index at `fastlevel = 2`. Point-lookup cost differed by exactly `0.125`. The README's rule that the height of the tree cannot decrease is about the true root, `btm_level`. The planner charges the fast-root level, and page deletion does lower that one: in fixture F, deleting all but the top 1,000 of 1,000,000 keys and vacuuming twice, with no rebuild, left `tree_level` at 2 and moved `fastlevel` from 2 to 1, which removed one level charge from every scan of that index.

### Physically fragmented leaf chains

`leaf_fragmentation` counts leaf pages whose right sibling lives at a lower block number. It rises when pages split in the middle of the key space and the new right half is appended at the end of the file, which is what random-order insertion produces. Only *backward* links count ([pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323)), so the metric under-reports: a right-sibling link that jumps a thousand blocks forward defeats read-ahead just as thoroughly and scores nothing. The manual ties physical adjacency to real runtime cost ([maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1048-L1054)). The planner never reads the metric at all. Measured contribution to cost: exactly `0.00`.

### Version-churn duplicates from non-HOT UPDATEs

An `UPDATE` that modifies a column used by a *hot-blocking* index cannot be HOT, so it writes a new index entry in every index that accepts the new row, including indexes whose own columns did not change ([btree.sgml#version-churn](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L638-L655)). Those entries are logically unchanged duplicates.

Two cases fall outside that, and neither produces a duplicate in an unrelated B-tree:

- **Only summarizing-index columns changed.** `heap_update()` tests the modified columns against the hot-blocking set and the summarizing set separately. An update that misses the hot-blocking set still takes the HOT path, and sets `summarized_update` if it touched the summarizing set ([heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4161)), which becomes `TU_Summarizing` — "Only summarized columns were updated, TID is unchanged" ([heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429), [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127)). `ExecUpdateEpilogue()` passes that through as `onlySummarizing` ([nodeModifyTable.c#ExecUpdateEpilogue-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L2162-L2166)), and `ExecInsertIndexTuples()` then skips every non-summarizing index ([execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366)). A BRIN-indexed column can therefore be churned without adding one B-tree entry.
- **A partial index whose predicate the new row fails.** `ExecInsertIndexTuples()` evaluates each index's `ii_Predicate` against the new tuple and skips the insert when it is not satisfied ([execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L387)), so a non-HOT update whose new row leaves a partial index's predicate adds nothing to that index.

Since v14, nbtree attacks the duplicates it does get with bottom-up index deletion passes triggered when a version-churn page split is anticipated ([btree.sgml#bottom-up-deletion](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L678), [nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L307-L320), [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785)).

Measured on the pin: five whole-table non-HOT `UPDATE` rounds over 200,000 rows grew an unrelated index on a 1,000-value column from 169 to **543 blocks**. Repeating the identical workload with a long-lived `REPEATABLE READ` snapshot open in another session, which is the condition the README names as blocking deletion, grew it to **1,173 blocks** instead. The benefit depends on the shape of the index: the same workload over a 100-value column grew its index from 180 to **1,020 blocks in both runs**, so there the unheld horizon bought nothing.

### Duplicate-heavy indexes with deduplication disabled

Deduplication merges duplicate leaf tuples into posting lists, lazily, at the point a page would otherwise split ([btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800), [README#notes-about-deduplication](../../../../raw/postgres-17/src/backend/access/nbtree/README#L904-L948), [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L45-L70)). It is on by default and can be turned off per index with the `deduplicate_items` reloption ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)).

Measured: 1,000,000 rows over 100 distinct keys built **852 blocks** with `deduplicate_items = on` and **2,749 blocks** with it off, a 3.2x difference in the planner's `pages` input for identical logical content.

### Bloat that VACUUM deliberately skipped

Two v14-era escape hatches let index bloat accumulate without any VACUUM touching it:

- The 2% bypass. `lazy_vacuum()` skips index *vacuuming* when fewer than `BYPASS_THRESHOLD_PAGES` (2% of `rel_pages`) hold `LP_DEAD` items and the TID store is under 32MB ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1949)).
- The wraparound failsafe, which makes the ongoing VACUUM bypass all further index vacuuming ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2347)).

The 2% bypass is not a skip of all index maintenance, and reading it that way overstates how much bloat it can hide. It clears only `do_index_vacuuming`, and the branch says so in as many words — "bypass index vacuuming, but do index cleanup" ([vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949)) — so `amvacuumcleanup` still runs. What that buys differs by access method:

| AM | Under the 2% bypass |
|---|---|
| B-tree | No entry is deleted, but `btvacuumcleanup()` is still called with `stats == NULL` and asks `_bt_vacuum_needs_cleanup()`; when that says yes it runs a full `btvacuumscan()`, which can place previously deleted pages in the FSM ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893)) |
| GIN | `ginvacuumcleanup()` flushes the pending list whenever `ginbulkdelete` was not called, which is exactly the bypass case ([ginvacuum.c#ginvacuumcleanup-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L719-L729)) |

The wraparound failsafe is the stronger escape hatch, because it clears `do_index_cleanup` as well as `do_index_vacuuming` ([vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326)), so it does stop `amvacuumcleanup` and with it the GIN pending-list flush.

`vacuum_index_cleanup = off` has the same effect by request, and the manual warns it "may also lead to severely bloated indexes if table modifications are frequent" ([ref/create_table.sgml#vacuum_index_cleanup](../../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1558-L1575)).

### Non-B-tree bloat

The manual states plainly that "the potential for bloat in non-B-tree indexes has not been well researched" and recommends monitoring physical size ([maintenance.sgml#non-btree-bloat](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)). Hash, GiST and SP-GiST still get the page-count penalty through `genericcostestimate()`. GIN and BRIN do not: their cost models are separate, and for GIN the pending list is a bloat source with its own cost consequences, covered in [Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead](#follow-up-when-a-gin-index-is-discarded-and-a-b-tree-is-used-instead) below.

## How Density And Fragmentation Affect Different Queries

| Query shape | Sensitivity to extra pages | Sensitivity to extra levels | Sensitivity to fragmentation |
|---|---|---|---|
| Equality point lookup on a unique or highly selective key | None while `pages <= tuples`: `ceil()` keeps `numIndexPages` at 1. Past that, `ceil(pages / tuples)` pages per row | Full: the only signal below that threshold | None |
| Range scan / broad index-only scan | Linear in `pages / tuples` | One charge per level | None |
| Bitmap index scan feeding a `BitmapAnd` | Linear, and can get the index dropped by `choose_bitmap_and()` ([indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287)) | One charge per level | None |
| Nested-loop inner index scan | Sub-linear: Mackert-Lohman damps repeated fetches | Charged once per loop | None |
| `= ANY (array)` on a B-tree | Linear, plus the v17 descent clamp raises the descent count | Charged once per estimated descent | None |
| Parallel index or index-only scan | Threshold and worker count both move with the pages the scan is expected to touch | One charge per level | None |
| GIN / BRIN | Separate cost models | Not applicable | None |

The practical shape of this: bloat mostly hurts plans that were already reading a lot of the index, and it barely registers on the selective lookups that dominate OLTP traffic. That is why a badly bloated index can keep serving primary-key lookups with an almost unchanged plan and price while quietly wrecking reporting queries over the same table. The exception is an index with more pages than its table has rows, which is what a drained queue table leaves behind: there a one-row lookup is charged several pages, measured at `12.29` against `4.44`.

## Exact-Pin Measurements

### Fixtures and method

All numbers below come from one isolated server built from the pinned checkout by the script filed under [Measurement Script](#measurement-script), last run on 2026-09-19 (`PostgreSQL 17.11 on aarch64-apple-darwin27.0.0`, pin `786db8dcf168bd9df8f55047337525ac19118b1c`, `block_size` 8192, maximum data alignment 8), with `autovacuum = off`, `shared_buffers = 256MB`, and default planner cost settings (`random_page_cost = 4`, `seq_page_cost = 1`, `cpu_tuple_cost = 0.01`, `cpu_index_tuple_cost = 0.005`, `cpu_operator_cost = 0.0025`, `effective_cache_size = 4GB`). `pgstattuple` supplied `pgstatindex`, `pageinspect` supplied `bt_metap()` so the planner's fast-root height could be read directly, and `pg_class.relallvisible` against `relpages` confirmed visibility-map state.

Measurement provenance: the first filing of this page measured these fixtures on the previous pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on Linux x86_64, from scripts that were never published and no longer exist. The 2026-09-19 run rebuilt every fixture from that filing's prose on the current pin, on a different operating system and architecture. Every number that depends only on an index and on exact statistics came back identical to the cent: all block counts and densities of fixtures A, B, H, I, M and N, `28480.42`, `123144.43`, `4.44` and `4.31`, `12730.42`, `2854.29`, `8226.42`, the four fixture-H costs, all twelve fixture-I costs, the `22353.00` sequential scan, 4 against 6 parallel workers, and in the follow-up `12.97`, `4.52`, `3.97`, `30.10`, `13.57`, `4.82`, `8.31`, `8.64`, `8.55` and the `1001.00` charged pages. That agreement is consistent with the range between the two pins, which changes no cost function: in the files this page cites it touches `nbtsearch.c` (`8434c938598`, an empty-index recheck under `SERIALIZABLE`, [nbtsearch.c#_bt_endpoint-empty-index](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2590-L2606)), `selfuncs.c` (`0ebf896f44d`, a `tid` type guard in `scalarineqsel()`, [selfuncs.c#scalarineqsel-ctid-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L599-L601)), `allpaths.c` (`c3f1db2b88`, inside `check_output_expressions()`), `ginvacuum.c` (`ba5e463295`, a restored `vacuum_delay_point()`), and `guc_tables.c` (a new GUC, which moved line numbers only). The numbers that did move are the ones that depended on `ANALYZE`'s random sample or on fixture details the first filing did not record; each is marked where it appears.

Method notes that matter for reading the numbers:

- Each comparison uses **two separate tables with identical contents**, so every query has exactly one candidate index and no index-choice tie-breaking is involved. Where one table carries two indexes, each is priced with the other dropped inside a rolled-back subtransaction, so the statistics are literally the same.
- Comparisons use `Index Only Scan` on tables vacuum-frozen to 100% all-visible, so the heap component of `cost_index()` is zero and the reported cost is the pure index cost. Three fixtures are exceptions on purpose, and each pays the same heap fetches on both sides of its comparison: H and N select a non-indexed column, and I is analyzed but never vacuumed. The plan-choice tests select a non-indexed column too, because a heap fetch is what makes the sequential scan competitive.
- `enable_seqscan` and `enable_bitmapscan` were disabled where a specific scan type had to be priced; they were left at their defaults for the plan-choice tests.
- **Statistics are exact, so the numbers are reproducible.** Every session runs with `default_statistics_target = 10000` (`PGC_USERSET`, session scope). `ANALYZE` samples 300 rows per unit of that target ([analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1876-L1894)), which at 3,000,000 is more rows than any fixture holds, so it reads every row and the statistics carry no sampling noise. The setting does not enter the cost model. Two further complete passes of every fixture stage reproduced all 33 recorded index rows, all 11 GIN rows, all 7 closed-form predictions and all 118 recorded plans byte for byte, one of them from an empty sandbox; and so did three runs of the previous pass, two of them on freshly initialized clusters, for the 112 plans they shared.
- **Exact statistics are not exact row estimates.** Exhaustive sampling makes the statistics, and therefore every cost on this page, the same on every run. It does not make a row estimate equal the true row count, because the selectivity model applies its own assumptions on top of exact statistics: `clauselist_selectivity_ext()` multiplies the per-clause selectivities of clauses it cannot pair into a range query, which assumes they are independent ([clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)). Fixture L3 is the only measurement here with two clauses on two columns, and [Bloat changes plans](#bloat-changes-plans) records its estimates beside its true row counts: on that fixture, whose columns are independent by construction, all three agree exactly. No fixture on this page violates the assumption, so none of them bounds the error it can cause.

| Fixture | Contents | Dense index | Bloated twin |
|---|---|---|---|
| A | 1,000,000 rows, `int` key | `a_dense_idx`, 2,745 blocks, 90.06% density, `fastlevel` 2 | `a_sparse_idx`, `fillfactor = 10`, 26,411 blocks, 9.62% density, `fastlevel` 2 |
| B | 1,000,000 rows, then `DELETE` of 90% by `id % 10 <> 0` | after `REINDEX`: 276 blocks, 89.83%, `fastlevel` 1 | before: 2,745 blocks, 9.27%, `fastlevel` 2 |
| F | 1,000,000 rows, then `DELETE` of all but the top 1,000 keys, two VACUUMs | after `REINDEX`: 5 blocks, `fastlevel` 1 | 2,745 blocks = 4 live leaves + 2,738 deleted pages, `tree_level` 2, **`fastlevel` 1** |
| G | 300,000 rows | `g_seq_idx`, `fillfactor = 100`, 744 blocks, 99.89% density, 0% fragmentation | `g_frag_idx` filled by `setseed(0.42)` random-order inserts, 1,148 blocks, 64.69% density, **49.87% fragmentation** |
| H | 50,000 rows | `h_l1_idx`, 139 blocks, `fastlevel` 1 | `h_l2_idx`, `fillfactor = 10`, 1,323 blocks, `fastlevel` 2 |
| I | 2,000 rows, analyzed but not vacuumed | `i_small_idx`, 8 blocks | `i_big_idx`, `fillfactor = 10`, 55 blocks |
| M | 1,000,000 rows, then `DELETE` of a contiguous 90% | after `REINDEX`: 276 blocks | 2,745 blocks = 276 live leaves + **2,465 deleted pages**, 89.18% density |
| N | 1,000,000 rows over 100 distinct keys | `deduplicate_items = on`, 852 blocks | `deduplicate_items = off`, 2,749 blocks |
| P | 200,000 rows, `tag = id % 1000`, five whole-table non-HOT `UPDATE` rounds | no blocking snapshot: 169 -> 543 blocks | `REPEATABLE READ` snapshot held: 169 -> 1,173 blocks |
| P-100 | the same with `tag = id % 100` | no blocking snapshot: 180 -> 1,020 blocks | snapshot held: 180 -> 1,020 blocks |
| L3 | 500,000 rows, `a = g % 2000` plus an independent seeded `c` over 0..19; `l3_a` is 449 blocks at 85.47% throughout | `l3_c` at the default fillfactor: 427 blocks, 89.81% density | `l3_c` rebuilt at `fillfactor = 10`: 3,801 blocks, 10.37% density |

Fixture G's random order is seeded, so its block count is reproducible; the first filing's unseeded build measured 1,101 blocks at 49.73% fragmentation. Fixture P's first filing did not record the tag column's cardinality and measured 583 and 1,174 blocks. Fixture L3 was rebuilt on the second 2026-09-19 pass: its `c` column used to be `g % 20`, which made `a = 5 AND c = 7` an impossible combination, because 20 divides 2,000 and so `a = 5` forced `c = 5` and the predicate matched no row at all. `c` is now an independent seeded draw over the same twenty values, and the figures that moved with it are 428 blocks at 89.59% density becoming 427 at 89.81%, and a `BitmapAnd` total of `332.49` with a `275.92` `c` bitmap becoming `328.51` and `275.70`.

### Index cost is a closed form in pages, tuples and tree height

Fixture A, whole-index scan (`WHERE id > 0`, selectivity exactly 1 so both row estimates are 1,000,000):

| Index | Blocks | Density | `fastlevel` | Observed total cost | Predicted from `(pages, tuples, fastlevel)` |
|---|---:|---:|---:|---:|---:|
| `a_dense_idx` | 2,745 | 90.06% | 2 | `28480.42` | `28480.42` |
| `a_sparse_idx` | 26,411 | 9.62% | 2 | `123144.43` | `123144.43` |

The prediction is `pages * random_page_cost + tuples * (cpu_index_tuple_cost + qual_op_cost) + ceil(log2(tuples)) * cpu_operator_cost + (fastlevel + 1) * 50 * cpu_operator_cost + rows * cpu_tuple_cost`, where `qual_op_cost` is one `cpu_operator_cost` for the single index qual. `genericcostestimate()` adds one more term, `qual_arg_cost`, which is zero when the comparison value is a constant ([selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810)). The script's `predict()` evaluates that expression in `float8`, in the planner's order of operations, from three inputs only: the live block count, the table's `reltuples`, and `bt_metap()`'s `fastlevel`. It equals the `EXPLAIN` total on 7 of 7 whole-index scans: both fixture-A indexes, fixtures B and M before the rebuild, fixture B after it, and both fixture-G indexes. Both fixture-A totals are exactly `x.425` in decimal; the planner's doubles print one as `.42` and the other as `.43`, and the `float8` recomputation lands on the same side both times. The first filing's formula carried a standalone `+ qual_op_cost` that the source does not have, and that term, not rounding, produced the one-cent gaps it reported on three rows. A 9.62x larger index costs 4.32x more.

Fixture A, 10% range scan (`id BETWEEN 1 AND 100000`, 100,000 rows estimated by both): `3100.43` dense against `12568.42` bloated. The first filing's `3136.70` and `12502.38` came from two independently sampled row estimates.

### The point lookup is nearly blind to bloat

Fixture A, `WHERE id = 42`, both tables 100% all-visible:

```text
Index Only Scan using a_dense_idx  on a_dense   (cost=0.42..4.44 rows=1 width=4)
Index Only Scan using a_sparse_idx on a_sparse  (cost=0.42..4.44 rows=1 width=4)
```

Identical to the cent, across a 9.62x size difference and an 80-point density difference, because `ceil(1 * pages / tuples)` is `1` for both and both trees have `fastlevel = 2`.

Fixture B shows the same thing where the bloat is real rather than synthetic. Before `REINDEX` the index is 2,745 blocks at 9.27% density; after, 276 blocks at 89.83%. The whole-index scan drops from `12730.42` to `2854.29` (4.46x), but the point lookup drops only from `4.44` to `4.31`, and that entire `0.125` difference is the level the rebuild removed.

### The tree-height charge, isolated to the cent

Fixture H holds row count constant at 50,000 and changes only the level:

| Index | Blocks | `fastlevel` | `WHERE id = 25000` | Same with `cpu_operator_cost = 1` |
|---|---:|---:|---|---|
| `h_l1_idx` | 139 | 1 | `cost=0.29..8.31` | `cost=116.00..125.02` |
| `h_l2_idx` | 1,323 | 2 | `cost=0.41..8.43` | `cost=166.00..175.01` |

With `cpu_operator_cost = 1` the startup costs are `116.00` and `166.00`. Both decompose exactly: `ceil(log2(50000)) = 16` comparison charges plus `(fastlevel + 1) * 50`. The gap is `50.00`, exactly `DEFAULT_PAGE_CPU_MULTIPLIER`. At default settings the same gap is `0.125`, so a 9.5x larger index raises a point-lookup cost by about 1.5%. Fixture H selects a non-indexed column, so all four costs include one heap page at `4.0`.

### A mostly-empty index: the fast root drops and pages outnumber rows

Fixture F builds the same 2,745-block index as fixture A, deletes every row except the top 1,000 keys (`id > 999000`), and vacuums twice. Nothing is rebuilt until the last row.

| State | Blocks | Live leaf pages | Deleted pages | `tree_level` | `fastlevel` | `WHERE id = 999950` | Whole-index scan |
|---|---:|---:|---:|---:|---:|---|---:|
| built, 1,000,000 rows | 2,745 | 2,733 | 0 | 2 | 2 | `cost=0.42..4.44` | |
| after the delete and two VACUUMs, 1,000 rows | 2,745 | 4 | 2,738 | 2 | **1** | `cost=0.28..12.29` | `10997.77` |
| after `REINDEX` | 5 | 3 | 0 | 1 | 1 | `cost=0.28..4.29` | |

Two things happen here that none of the other fixtures reach.

**The planner's height fell without a rebuild.** Page deletion left the true root where it was, at block 290 and level 2, and moved the fast root to block 2570 at level 1, which is the adjustment the README describes for deleting the next-to-last page on a level ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381), [nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)). `pgstatindex` still reports `tree_level` 2, while the startup cost shows the planner charging for one level fewer: `0.28` is `ceil(log2(1000)) * 0.0025 + (1 + 1) * 0.125 = 0.275`.

**The point lookup stopped being cheap.** With 1,000 rows and 2,745 pages, `numIndexPages = ceil(1 * 2745 / 1000) = 3`, so the one-row lookup is charged three random pages: `3 * 4.0 + 0.0075 + 0.025 + 0.25 + 0.01 = 12.2925`, printed `12.29`. The rebuilt index prices it at `4.29`. A whole-index scan of the 1,000 rows costs `10997.77`, of which `10980.00` is the 2,745 pages. This is the shape a drained queue table leaves behind, and it is the one case on this page where bloat reaches the selective lookups of an OLTP workload.

### Leaf fragmentation contributes exactly zero

Fixture G, both tables frozen to 100% all-visible so the index-only scan pays no heap cost:

| Index | Blocks | Density | Fragmentation | `fastlevel` | Observed | Predicted |
|---|---:|---:|---:|---:|---:|---:|
| `g_seq_idx` | 744 | 99.89% | **0.00%** | 2 | `8226.42` | `8226.42` |
| `g_frag_idx` | 1,148 | 64.69% | **49.87%** | 2 | `9842.42` | `9842.42` |

The observed gap is `9842.42 - 8226.42 = 1616.00`. The page-count gap is `1148 - 744 = 404`, and `404 * random_page_cost = 404 * 4.0 = 1616.00`. The residual attributable to a 49.87-point fragmentation difference is `0.00`, and the closed-form prediction, which has no fragmentation term, reproduces both costs. The first filing's unseeded build of `g_frag_idx` came out at 1,101 blocks and showed the same identity, `1428.00 = 357 * 4.0`.

### Two indexes with the same cost and opposite avg_leaf_density

Fixtures B and M both end at 2,745 blocks over 100,000 surviving rows, and both price a whole-index scan at exactly `12730.42`, dropping to `2854.29` after `REINDEX`. Their `pgstatindex` output could hardly be more different:

| Fixture | Deletion pattern | Blocks | Live leaf pages | Deleted pages | `avg_leaf_density` | Whole-index cost |
|---|---|---:|---:|---:|---:|---:|
| B | scattered (`id % 10 <> 0`) | 2,745 | 2,733 | 0 | **9.27%** | `12730.42` |
| M | contiguous (`id > 100000`) | 2,745 | 276 | **2,465** | **89.18%** | `12730.42` |

A monitoring rule that flags low `avg_leaf_density` catches fixture B and completely misses fixture M, even though the planner is charged the same 2,745 pages in both and a rebuild recovers 10x in both. Physical size against row count is the reliable signal; `avg_leaf_density` alone is not.

### The planner reads the live block count, not pg_class.relpages

A 200,000-row table with a `fillfactor = 10` index occupying 5,285 blocks priced its whole-index scan at `24640.42`. Forging the catalog on the scratch server, which is a disposable-fixture step and must never be run against a database anyone cares about:

```text
UPDATE pg_class SET relpages = 1 WHERE relname = 'b_stale_idx';
-- pg_class.relpages = 1, real size = 5285 blocks
Index Only Scan using b_stale_idx on b_stale  (cost=0.42..24640.42 rows=200000 width=4)
```

The cost did not move, confirming that `get_relation_info()` used `RelationGetNumberOfBlocks()`. A **partial** index behaves differently, because `estimate_rel_size()` derives its tuple count from `pg_class` density: forging `reltuples` from 200,000 to 20 on an otherwise identical partial index (`WHERE id > 0`) moved the cost from `24140.42` to `23140.49` and the startup cost from `0.42` to `0.39`. With 20 claimed tuples the scan is charged `20 * cpu_index_tuple_cost` instead of `200000 *`, and `ceil(log2(20)) = 5` comparisons instead of 18, while the page charge stays at all 5,285 pages.

The partial index starts `500.00` below the plain one for a reason that has nothing to do with size. Its predicate implies the query's only clause, so `check_index_predicates()` drops that clause from the list the index is matched against, the scan carries no index qual, and its 200,000 tuples pay no `qual_op_cost` ([indxpath.c#check_index_predicates-indrestrictinfo](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3318-L3378), [indxpath.c#match_restriction_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1968-L1974)). The first filing reported `25528.42` for the plain index. This review reproduced that figure to the cent only by leaving the table un-vacuumed, which adds `888.00` of heap fetches to the index-only scan, against a method note that said every comparison paid none; this run vacuums the table like every other fixture.

### Bloat changes plans

Fixture A with all `enable_*` settings at their defaults, selecting a non-indexed column so a heap fetch is required:

| Selectivity | Dense index | Bloated twin |
|---|---|---|
| 25% (`id BETWEEN 1 AND 250000`) | `Index Scan using a_dense_idx` at `9590.42` | `Seq Scan` at `22353.00`; the index path is rejected |
| 12% (`id BETWEEN 1 AND 120000`) | `Index Scan` at `4606.43` | `Index Scan` at `15966.42` |

The sequential scan is the same `22353.00` on both tables: 7,353 heap pages plus 1,000,000 rows at `cpu_tuple_cost` and two operator evaluations each. The first filing's three index-scan costs (`9731.06`, `4706.74`, `16026.44`) carried sampled row estimates; with exact statistics the estimates are 250,000 and 120,000 rows.

Bloat also removes an index from a `BitmapAnd`. Fixture L3 is a 500,000-row table with `a = g % 2000` and an independent `c` drawn from a seeded PRNG over 0..19, so the two columns are independent of each other and `a = 5 AND c = 7` selects real rows. With both indexes healthy — `l3_a` at 449 blocks and 85.47% density, `l3_c` at 427 and 89.81% — the two-clause predicate produced a `BitmapAnd` over both, total `328.51`, of which the `c` bitmap alone cost `275.70` against the `a` bitmap's `6.30`. Bloating only the `c` index to 3,801 blocks at 10.37% density, by rebuilding it at `fillfactor = 10`, made the planner drop it and demote `c = 7` to a `Filter`:

```text
Bitmap Heap Scan on l3  (cost=6.30..806.01 rows=12 width=25)
  Recheck Cond: (a = 5)
  Filter: (c = 7)
  ->  Bitmap Index Scan on l3_a  (cost=0.00..6.30 rows=250 width=0)
        Index Cond: (a = 5)
```

This is also the one fixture on the page that can check the method note's claim that exhaustive statistics make every cost reproducible without thereby making every estimate true. Both columns are analyzed exhaustively, and the conjunction is still estimated by multiplying two per-clause selectivities, which assumes independence ([clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)). Recording each estimate beside the true count:

| Predicate | Estimated rows | Actual rows |
|---|---:|---:|
| `a = 5` | 250 | 250 |
| `c = 7` | 24,971 | 24,971 |
| `a = 5 AND c = 7` | 12 | 12 |

Every one of them is exact. The two single-column estimates are exact because `default_statistics_target = 10000` buys each column an MCV list long enough to hold all of its values at their true frequencies, and the conjunction is exact because the columns really are independent, which is how the fixture is built. That is agreement, not proof: the multiplication is an assumption, and no fixture on this page violates it, so nothing here bounds the error it can cause. That limit is filed under [Open Questions](#open-questions).

### Bloat changes parallel worker counts

With `max_parallel_workers_per_gather = 8`, `max_parallel_workers = 8`, `min_parallel_table_scan_size = 0` and `enable_seqscan = off`, fixture A's parallel index-only scans differed only in the index:

| Index | Blocks | Workers, whole index | Workers, 50% of the keys | Workers, 20% of the keys |
|---|---:|---:|---:|---:|
| `a_dense_idx` | 2,745 | 4 | 3 | 2 |
| `a_sparse_idx` | 26,411 | 6 | 5 | 5 |

Every count follows `compute_parallel_worker()`'s powers-of-three ramp from `min_parallel_index_scan_size` (default 512kB, that is 64 blocks: one worker from 64 pages, two from 192, three from 576, four from 1,728, five from 5,184, six from 15,552), applied to the pages the scan is expected to touch. The whole-index scans touch every page, so they read as if the index size set the count. The partial scans show that it does not: the bloated index plans 5 workers for a 20% scan because 20% of 26,411 pages is still 5,283 pages, where the dense index's 549 pages earn 2. If `index->pages` chose the count, every cell in the bloated row would read 6. The two partial-scan columns were planned with `parallel_setup_cost = 0` and `parallel_tuple_cost = 0` (both `PGC_USERSET`) so that the parallel path is the one chosen; the whole-index column reads 4 and 6 with or without them. Bloat bought two or three extra workers for no extra useful data.

### Index pages in the cache model

Fixture A again, this time as the inner side of a 50,000-iteration nested loop: `a_outer` holds 50,000 keys spread evenly over the 1,000,000, and hash and merge joins are disabled. The index-only scan is now repeated, so `genericcostestimate()` prices its pages through `index_pages_fetched()`:

| Inner index | Blocks | Inner scan, per loop | Join total | Inner scan at `effective_cache_size = 64MB` | Join total at 64MB |
|---|---:|---|---:|---|---:|
| `a_dense_idx` | 2,745 | `cost=0.42..0.66` | `34452.01` | `cost=0.42..1.38` | `70448.01` |
| `a_sparse_idx` | 26,411 | `cost=0.42..2.50` | `126220.01` | `cost=0.42..3.55` | `178748.01` |

Each loop touches one index page, so with no cache model every loop would pay `4.0`. At the default 4GB the Mackert-Lohman formula caps the dense index's 50,000 fetches at its 2,745 pages, `2745 * 4.0 / 50000 = 0.22` per loop, while the bloated twin's 50,000 fetches are spread over 26,411 pages and come to about 25,700 distinct ones, `2.05` per loop. The remaining `0.44` is the same CPU and descent charge on both. Every table is all-visible, so the heap contributes nothing and the whole difference is the index. Shrinking `effective_cache_size` to 64MB (`PGC_USERSET`, session scope) makes the prorated cache share smaller than either index, and both costs rise. A bloated index is therefore charged more for *repeated* lookups even though each one touches a single page, which is the part of the penalty a lone point lookup never shows.

### The v17 ScalarArrayOp descent clamp

Fixture I, with `cpu_operator_cost = 1` so that each additional estimated descent adds about 116 cost units and the clamp is directly readable: 111 of descent CPU (`ceil(log2(2000)) = 11` comparisons plus `(1 + 1) * 50` for two pages), one more index page at `4.0`, and about `1.0` of per-tuple CPU. The tables are analyzed but never vacuumed, so every row also pays a heap fetch, identically for both indexes. The clamp is `ceil(pages * 0.3333333)`: `3` for the 8-block index, `19` for the 55-block index.

| Array length | `i_small_idx` (8 blocks, clamp 3) | `i_big_idx` (55 blocks, clamp 19) |
|---:|---:|---:|
| 1 | `120.02` | `120.02` |
| 2 | `236.03` | `236.03` |
| 3 | `352.04` | `352.04` |
| 4 | `352.05` | `468.06` |
| 6 | `355.09` | `700.09` |
| 10 | `358.14` | `1164.15` |

The dense index plateaus at exactly three descents and stops charging for longer arrays; the bloated index keeps paying. At ten elements the bloated index is charged `1164.15` against `358.14` for the identical query and identical row estimates, purely because its page count raised the cap. Residual growth on the plateaued rows is the per-tuple term, not descents.

### Version churn with and without a held snapshot

Fixture P: a 200,000-row table `(id, payload, tag)` with indexes on `payload` and `tag`, where `UPDATE ... SET payload = payload + 1` is non-HOT and therefore writes a logically unchanged duplicate into the `tag` index on every round. Each of the five rounds is its own transaction, and no VACUUM ran. Fixture P sets `tag = id % 1000`; fixture P-100 sets `tag = id % 100` and changes nothing else.

| Fixture | Run | `tag` index blocks after 5 rounds | Density | Fragmentation | `fastlevel` | Bitmap index scan cost, `tag = 7` |
|---|---|---:|---:|---:|---:|---:|
| P, 1,000 tag values | no blocking snapshot | 543 (from 169) | 98.01% | 64.31% | 1 -> 2 | `5.92` |
| P, 1,000 tag values | `REPEATABLE READ` snapshot held elsewhere | 1,173 (from 169) | 82.52% | 42.88% | 1 -> 2 | `9.92` |
| P-100, 100 tag values | no blocking snapshot | 1,020 (from 180) | 91.16% | 18.48% | 1 -> 2 | `59.42` |
| P-100, 100 tag values | `REPEATABLE READ` snapshot held elsewhere | 1,020 (from 180) | 91.11% | 18.48% | 1 -> 2 | `59.42` |

Six row versions exist per logical row after five rounds. In fixture P the blocked run grew 6.9x, close to storing every version, and the unblocked run grew 3.2x. The difference is the work bottom-up index deletion was able to do, and the README names an old snapshot holding up cleanup as exactly the condition that defeats it ([README#deduplication-in-unique-indexes](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). The `4.00` gap between the two bitmap index scan costs is one page: `ceil(200 * 1173 / 200000) = 2` against `ceil(200 * 543 / 200000) = 1`. Note also that the unblocked run's density reads 98.01%, above its 90 fillfactor, because the single-value split strategy and deduplication pack duplicate-heavy pages tighter.

Fixture P-100 is the control that keeps this result from being over-read. With 2,000 rows per tag value instead of 200, the same five rounds grew the index 5.7x to the same 1,020 blocks whether or not a snapshot was held, so there the free horizon bought nothing. Why was not traced; it is filed under [Open Questions](#open-questions).

### Deduplication

Fixture N, 1,000,000 rows over 100 distinct keys, two indexes on the same column:

| Index | Blocks | Density | `WHERE k = 5` | `WHERE k > 0` |
|---|---:|---:|---:|---:|
| `deduplicate_items = on` | 852 | 89.70% | `24021.01` | `50111.17` |
| `deduplicate_items = off` | 2,749 | 90.16% | `24097.01` | `57623.17` |

Each index is priced with the other one dropped inside a rolled-back subtransaction. These are plain index scans on `SELECT *`, so every cost also contains the same heap fetches in both rows; only the gaps belong to the index, and both gaps are pure page arithmetic. For `k > 0` (990,000 rows estimated), touched pages are `ceil(990000 * 852 / 1000000) = 844` against `ceil(990000 * 2749 / 1000000) = 2722`, and `(2722 - 844) * 4.0 = 7512.00`, exactly the observed `57623.17 - 50111.17`. For `k = 5` (10,000 rows), `(28 - 9) * 4.0 = 76.00`, exactly the observed gap. The first filing's `7516.00` and `68.00` were the same arithmetic on sampled estimates of 990,367 and about 9,400 rows. Note that `avg_leaf_density` is ~90% in both cases: it cannot see that one index stores 3.2x more pages for the same information.

## What Changed Since PostgreSQL 12

Every attribution below is anchored to the pinned v17 checkout's own commit history, and the method needs no tags, because this checkout carries none: it is a single-branch clone of `REL_17_STABLE`. A commit on the main line is assigned to the first major version whose branch point follows it, and the branch points are the `Stamp HEAD as NNdevel.` commits that open each development cycle: `615cebc94b` (13devel, 2019-07-01), `d10b19e224` (14devel, 2020-06-07), `596b5af1d3` (15devel, 2021-06-28), `d31d30973a` (16devel, 2022-06-30) and `5bcc7e6dc8` (17devel, 2023-06-29). A commit that has `d10b19e224` as an ancestor but not `596b5af1d3` first shipped in PostgreSQL 14, which is what "first in `REL_14_0`" means below. Within 17.x, a commit is assigned to the first `Stamp 17.N.` commit that contains it. "Since `REL_12_0`" in a `git log -L` claim means the range `615cebc94b..HEAD`. The `REL_12_0` release commit lives on the `REL_12_STABLE` branch and is not an ancestor of `REL_17_STABLE`, so without tags this checkout cannot show its text. The byte-for-byte comparisons below therefore read it from the v12 checkout's own `REL_12_0` tag (`git -C raw/postgres-12 show REL_12_0:<path>`), which is the matching-version evidence a cross-version claim needs.

### The cost code that did not change at all

Comparing the `REL_12_0` file text with the pin:

- `index_pages_fetched()` is byte-identical.
- `cost_index()` is byte-identical.
- The bloat-relevant core of `genericcostestimate()` is unchanged: the `numIndexPages` prorating formula, the `index->pages > 1 && index->tuples > 1` guard, the Mackert-Lohman call, and the single-scan `numIndexPages * spc_random_page_cost` charge. The function's whole diff against `REL_12_0` is one hunk from two v17 commits: `5bf748b86bc` made `num_sa_scans` a caller-supplied input, and `9391f71523b` changed the `estimate_array_length()` call to pass `root` and to return a `double`.
- Exactly five commits touched `btcostestimate()` since `REL_12_0`, and only two are functional: `5bf748b86bc` and `9391f71523b`. The remaining three are a typo fix (`950d4a2cb1d`), a `MemSet`-to-struct-initializer change (`9fd45870c14`), and the macro extraction (`eb5c4e953bb`).

So a reader who knows the v12 model already knows most of the v17 model.

### v16: the 50x page charge became a macro

`eb5c4e953bb` "Extract the multiplier for CPU process cost of index page into a macro" (2023-01-08, first in `REL_16_0`) replaced three literal `50.0` occurrences with `DEFAULT_PAGE_CPU_MULTIPLIER`. `REL_12_0` already charged `(index->tree_height + 1) * 50.0 * cpu_operator_cost` in `btcostestimate`, `gistcostestimate` and `spgcostestimate`. The value and the behavior are unchanged; only the spelling moved ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).

The companion commit `cd9479af2af` "Improve GIN cost estimation" (also first in `REL_16_0`) newly applied the same per-page CPU multiplier inside `gincostestimate()` ([selfuncs.c#gincostestimate-page-charges](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955)), which changes how GIN bloat is priced but does not touch the B-tree path.

### v17: index pages now cap ScalarArrayOp descents

`5bf748b86bc` "Enhance nbtree ScalarArrayOp execution." (2024-04-06, first in `REL_17_0`) is the only commit since v12 that added a *new* way for a bloated B-tree to be charged more. It introduced the clamp `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))`, moved `num_sa_scans` into `GenericCosts` so `btcostestimate()` can hand its own estimate to `genericcostestimate()`, and reworded the descent comments from "per SA scan" to "per estimated SA index descent" ([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042)).

In `REL_12_0` there was no clamp: `num_sa_scans` was simply the product of array lengths, computed inside `genericcostestimate()`. The measured consequence in v17 is that `index->pages` now sets the ceiling on descent charges, so the dense and bloated twins in fixture I diverge by 3.25x on a ten-element `= ANY` where v12's formula had no page-count input at that point at all.

The related commit `9391f71523b` "Teach estimate_array_length() to use statistics where available." (also `REL_17_0`) changed how the *unclamped* array length is estimated, which feeds the same variable.

### v16: partitioned indexes are zeroed out

`3c569049b7b` "Allow left join removals and unique joins on partitioned tables" (2023-01-09, first in `REL_16_0`) added the `RELKIND_PARTITIONED_INDEX` guard that sets `pages = 0`, `tuples = 0.0`, `tree_height = -1` for partitioned indexes ([plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)). This is the only structural change to `get_relation_info()`'s index-size block since `REL_12_0`.

### v14: reltuples turns negative for never-analyzed relations

`3d351d916b2` "Redefine pg_class.reltuples to be -1 before the first VACUUM or ANALYZE." (2020-08-30, first in `REL_14_0`) changed `estimate_rel_size()`'s index branch from `if (relpages > 0)` to `if (reltuples >= 0 && relpages > 0)` ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)), and the catalog header now documents `-1` as "unknown" ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). A never-analyzed partial index therefore falls through to the attribute-width density fallback in v17 instead of computing a density from a zero tuple count. Non-partial indexes are unaffected, because they never consult `reltuples`.

### v13: deduplication

`0d861bbb702` "Add deduplication to nbtree." (2020-02-26, first in `REL_13_0`) introduced posting-list tuples, the lazy pre-split deduplication pass, and the `deduplicate_items` reloption, none of which exist in `REL_12_0`'s `reloptions.c`. It does not change any cost formula; it changes how many pages a duplicate-heavy index needs, and therefore what the unchanged formula is fed. Fixture N measures 852 blocks against 2,749 for the same data with the feature disabled.

### v14: bottom-up index deletion

`d168b666823` "Enhance nbtree index tuple deletion." (2021-01-13, first in `REL_14_0`) added `_bt_bottomupdel_pass()`. The v17 documentation states the boundary directly: "Prior to PostgreSQL 14, the only category of B-Tree deletion was simple deletion" ([btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703)), and the README says bottom-up index deletion "was added to PostgreSQL 14" ([README#added-to-postgresql-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). The docs claim it is possible for such an index's on-disk size to "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)); fixture P measures a 3.2x growth under a deliberately harsh five-round whole-table churn with no VACUUM, against 6.9x when an old snapshot blocks deletion, and fixture P-100 measures no difference at all on a column with ten times fewer distinct values.

### v14: faster recycling of deleted pages

`9dd963ae253` "Recycle nbtree pages deleted during same VACUUM." (2021-03-21, first in `REL_14_0`). The README describes the change in its own words: before v14 VACUUM placed only *previously* deleted pages in the FSM, and "PostgreSQL 14 added the ability for VACUUM to consider if it's possible to recycle newly deleted pages at the end of the full index scan where the page deletion took place" ([README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424)). This shortens the window in which deleted pages are dead weight, but it never shrinks the fork, so the planner keeps paying for them until a rebuild. Fixture M measures 2,465 deleted pages still inside a 2,745-block index after three VACUUMs.

### v14: VACUUM can skip index vacuuming entirely

Two `REL_14_0` commits created bloat paths that do not exist in v12: `5100010ee4d` "Teach VACUUM to bypass unnecessary index vacuuming." (the 2% `BYPASS_THRESHOLD_PAGES` rule) and `1e55e7d1755` "Add wraparound failsafe to VACUUM.". A third, `3499df0dee8` "Support disabling index bypassing by VACUUM." (2021-06-18, `REL_14_0`), added the tri-valued `auto` handling so `INDEX_CLEANUP`/`vacuum_index_cleanup` can force cleanup back on. The `vacuum_index_cleanup` reloption itself already existed in `REL_12_0` as a plain boolean.

### Since-v12 summary table

| Change | First release | Effect on planner-visible bloat |
|---|---|---|
| `0d861bbb702` deduplication | 13 | Fewer pages for duplicate-heavy indexes; measured 3.2x |
| `3d351d916b2` `reltuples = -1` | 14 | Different fallback for never-analyzed partial indexes |
| `d168b666823` bottom-up index deletion | 14 | Fewer pages under non-HOT `UPDATE` churn; measured 2.2x on a 1,000-value column and nothing on a 100-value one |
| `9dd963ae253` recycle newly deleted pages | 14 | Deleted pages reusable sooner; fork still not shortened |
| `5100010ee4d` 2% index-vacuum bypass | 14 | New way for index bloat to accumulate untouched |
| `1e55e7d1755` wraparound failsafe | 14 | Same, under wraparound pressure |
| `3499df0dee8` `INDEX_CLEANUP` auto | 14 | Lets an operator force cleanup back on |
| `eb5c4e953bb` `DEFAULT_PAGE_CPU_MULTIPLIER` | 16 | Cosmetic; value stays 50.0 |
| `cd9479af2af` GIN page CPU charges | 16 | Changes GIN bloat pricing, not B-tree |
| `3c569049b7b` partitioned-index zeroing | 16 | Partitioned parents contribute no size |
| `5bf748b86bc` SAOP descent clamp | 17 | **New** `index->pages` penalty channel for `= ANY` |
| `9391f71523b` `estimate_array_length()` statistics | 17 | Feeds the clamped descent count |
| `index_pages_fetched()`, `cost_index()` | unchanged | Byte-identical to `REL_12_0` |
| `numIndexPages` prorating, height charge | unchanged | Same formulas and same `50.0` constant as v12 |

## Settings That Move The Boundary

Every setting below is `PGC_USERSET`, so each takes effect at session or transaction scope with no reload or restart. None of them is a bloat control; they change how heavily the existing page count is weighted.

| Setting | v17 default | Role in bloat pricing | Apply scope |
|---|---|---|---|
| `random_page_cost` | `DEFAULT_RANDOM_PAGE_COST` (4.0) | Multiplies every touched index page ([guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)) | session/transaction |
| `cpu_operator_cost` | `DEFAULT_CPU_OPERATOR_COST` (0.0025) | Scales the height charge and the log2 descent charge ([guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729)) | session/transaction |
| `effective_cache_size` | `DEFAULT_EFFECTIVE_CACHE_SIZE` | Splits notional cache between heap and index pages ([guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)) | session/transaction |
| `min_parallel_index_scan_size` | 512kB ([guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540)) | Threshold and ramp for index-driven worker counts | session/transaction |

Two per-index storage parameters change the physical layout rather than its price, and both take `ShareUpdateExclusiveLock` with the in-tree reason "since it applies only to later inserts": `fillfactor` ([reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)) and `deduplicate_items` ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)).

Neither is inert on the pages that already exist, so "no effect until a rebuild" is too strong. Both are read from the relation at the moment a leaf page is about to split:

- `_bt_findsplitloc()` takes `leaffillfactor = BTGetFillFactor(rel)` on every split ([nbtsplitloc.c#leaffillfactor](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L170-L176)), so a changed `fillfactor` governs the next split of a page that was built under the old value.
- `_bt_delete_or_dedup_one_page()` runs `_bt_dedup_pass()` over the existing leaf page before splitting it ([nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785)), gated on `BTGetDeduplicateItems(rel)` and on `allequalimage`, which is an opclass property fixed into the metapage at build time and is not affected by the reloption ([nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564)). Turning `deduplicate_items` back on therefore compacts existing leaves in place as inserts reach them.

What a rebuild adds is reach and immediacy: `REINDEX` applies the new layout to every page at once instead of page by page as traffic happens to touch them, which is one of the scenarios it documents ([ref/reindex.sgml#storage-parameter](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L66-L71)). Fixture N and the `l3_c` bloating step below both use `REINDEX` for exactly that reason.

## Practical Interpretation

- **Rank rebuild candidates by physical size against useful rows, not by `avg_leaf_density`.** The planner is charged for blocks, and fixture M shows a 10x-oversized index reporting 89.18% density.
- **Do not expect a bloated index to be abandoned by OLTP queries.** Point lookups are charged only the height difference: `0.125` per level at default settings, about 1.5% on a typical lookup.
- **The exception is an index with more pages than rows.** A drained queue table is the usual way to get one. There a one-row lookup is charged `ceil(pages / tuples)` random pages, measured at `12.29` against `4.29` after a rebuild, so this is the one shape where a rebuild changes what OLTP lookups cost.
- **Do not read the planner's height from `pgstatindex`.** `tree_level` is the true root level; the planner charges the fast-root level, which VACUUM's page deletion can lower without a rebuild. `bt_metap()` shows both. Even `bt_metap()` only shows what is on disk now: a long-lived backend can still be costing from its own cached copy of an older metapage, so two sessions can price the same index differently.
- **Expect plan changes on the analytical side.** Fixture A flipped to a sequential scan at 25% selectivity, and a bloated index was dropped from a `BitmapAnd` entirely.
- **Treat `leaf_fragmentation` as a runtime concern only.** It has real I/O consequences the manual acknowledges, and exactly zero cost-model consequences.
- **Watch for `= ANY (...)` regressions on v17 specifically.** The descent clamp means bloat is now charged through a channel that did not exist before v17.
- **A bloated partial index is priced through a different path.** Its tuple estimate comes from `pg_class`, so its cost can be wrong in either direction until it is analyzed.

## Key Data Structures

| Structure | Field | Role |
|---|---|---|
| `IndexOptInfo` | `pages`, `tuples`, `tree_height` | The size inputs `get_relation_info()` fills for every AM, and the only ones `genericcostestimate()` reads ([pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)). Not the complete set for every AM: `gincostestimate()` and `brincostestimate()` each reopen the index and read their own metapage counters on top ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [selfuncs.c#brincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8086-L8101)) |
| `RelOptInfo` | `pages`, `tuples`, `allvisfrac` | Parent-table estimates; `tuples` is copied into a non-partial index's `tuples` ([pathnodes.h#RelOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L941-L944)) |
| `PlannerInfo` | `total_table_pages` | Table-only page total used to prorate `effective_cache_size` ([pathnodes.h#total_table_pages](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L483-L484)) |
| `GenericCosts` | `numIndexPages`, `numIndexTuples`, `num_sa_scans` | Shared cost scratchpad; `numIndexPages` is what every caller returns as `*indexPages`, and `num_sa_scans` became an input in v17 ([selfuncs.h#GenericCosts](../../../../raw/postgres-17/src/include/utils/selfuncs.h#L108-L138)) |
| `BTMetaPageData` | `btm_level`, `btm_fastlevel` | True root level versus the fast-root level the planner uses ([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)) |
| `LVRelState` | `consider_bypass_optimization`, `do_index_vacuuming`, `do_index_cleanup` | Two separate switches, not one: the 2% bypass clears only `do_index_vacuuming`, so `amvacuumcleanup` still runs; the wraparound failsafe clears both ([vacuumlazy.c#LVRelState](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L148-L156)) |
| `pg_class` | `relpages`, `reltuples`, `relallvisible` | The only physical-size catalog columns; no density or fragmentation column exists ([pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)) |

## Caller And Callee Boundary

```text
query_planner
  └─ get_relation_info                       plancat.c
       ├─ RelationGetNumberOfBlocks          -> smgrnblocks (non-partial index)
       ├─ estimate_rel_size                  -> pg_class density (partial index)
       ├─ _bt_getrootheight                  -> btm_fastlevel (B-tree only)
       └─ get_relation_info_hook             a plugin may rewrite pages, tuples, tree_height
  └─ make_one_rel -> set_base_rel_sizes -> root->total_table_pages
  └─ create_index_paths                      indxpath.c
       ├─ cost_index                         costsize.c
       │    ├─ amcostestimate  ==  btcostestimate / hash / gist / spgist / gin / brin
       │    │    └─ genericcostestimate      (bt, hash, gist, spgist, contrib bloom)
       │    │         └─ index_pages_fetched (repeated scans, index->pages)
       │    ├─ index_pages_fetched           (heap fetches, 3 call sites, index->pages)
       │    └─ compute_parallel_worker       allpaths.c (numIndexPages, not index->pages)
       ├─ choose_bitmap_and                  indxpath.c
       │    └─ bitmap_scan_cost_est -> cost_bitmap_heap_scan -> compute_bitmap_pages
       └─ add_path -> compare_path_costs_fuzzily   pathnode.c
```

Symbols: [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508), [costsize.c#cost_index](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L621), [selfuncs.c#genericcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6626-L6828), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951), [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279), [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287), [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526), [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L164).

On the write side, the code that decides how many pages exist runs entirely inside nbtree: [nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334) chooses split points, [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785) tries bottom-up deletion and then deduplication before splitting, and [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659) marks pages deleted and can lower the fast-root level.

## Build, Generated-Header, And Extension Boundary

- Nothing on this path depends on a generated parser or catalog artifact at cost time. `pg_class.relpages` / `reltuples` / `relallvisible` come from the hand-written catalog header `src/include/catalog/pg_class.h`, whose `BKI_DEFAULT(-1)` on `reltuples` encodes the v14 redefinition ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). The `FormData_pg_class` struct that `estimate_rel_size()` reads is not generated by `genbki.pl`: it is the `CATALOG()` macro in that same hand-written header expanding through cpp, `#define CATALOG(name,oid,oidmacro) typedef struct CppConcat(FormData_,name)` ([genbki.h:23](../../../../raw/postgres-17/src/include/catalog/genbki.h#L23)), which the header's own comment states — "cpp turns this into typedef struct FormData_pg_class" ([pg_class.h#CATALOG](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L32)). What `genbki.pl` does with the header, by way of `Catalog.pm` reading it ([pg_class.h#NOTES](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L11-L16)), is emit the derived `pg_class_d.h` and the bootstrap `.bki` data that `initdb` consumes. So changing one of these columns' declarations is an initdb-visible catalog change and not only a recompile, but the struct the planner reads is plain cpp output.
- `DEFAULT_PAGE_CPU_MULTIPLIER` is a private `#define` inside `selfuncs.c` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), not an exported symbol and not a GUC, so the multiplier itself is fixed at compile time. What it multiplies is not: an extension can change `cpu_operator_cost`, and it can rewrite `tree_height` through `get_relation_info_hook`, below.
- `pathnodes.h` deliberately types `IndexOptInfo.amcostestimate` weakly to avoid including `amapi.h`, so `cost_index()` casts it before calling ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). A custom index AM therefore participates in bloat pricing entirely through its own `amcostestimate`, and gets `tree_height = -1` unless it is a B-tree.
- Three hooks sit on this path. `get_relation_info_hook` runs at the end of `get_relation_info()`, after every `IndexOptInfo` is filled, so that a plugin can "editorialize on the info we obtained from the catalogs. Actions might include altering the assumed relation size, removing an index, or adding a hypothetical index to the indexlist" ([plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576), [plancat.h#get_relation_info_hook_type](../../../../raw/postgres-17/src/include/optimizer/plancat.h#L20-L25)). A plugin there can substitute `pages`, `tuples`, `tree_height` or the `amcostestimate` pointer itself before any path is costed, and the cost code expects it: `gincostestimate()` skips the metapage read for an index marked `hypothetical` ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711)). The other two, `get_relation_stats_hook` and `get_index_stats_hook`, are consulted by `btcostestimate()` for correlation statistics only ([selfuncs.c#btcostestimate-stats-hooks](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7116-L7171)) and cannot substitute a page count or a tree height.
- Every density and fragmentation metric on this page comes from `contrib`, not core: `pgstatindex` in `pgstattuple` and `bt_metap()` in `pageinspect`. Core SQL exposes physical size (`pg_relation_size`) and the catalog columns, and nothing else about index page structure.
- Within the 17.x series, `036decbba2a` "pgstattuple: Improve reports generated for indexes (hash, gist, btree)" (first contained in the `Stamp 17.7.` commit; its message says `Backpatch-through: 13`) added a `BTPageOpaqueData` size check to `pgstattuple`'s B-tree handling and made new, all-zero pages count as free space. The pin contains it. It touches only `pgstattuple.c`, so it changes `pgstattuple()` output, not `pgstatindex()` density or fragmentation.

## Tests And Explicit Test Absence

- **No test exercises the bloat charge.** `src/test` contains no reference to `tree_height`, `btcostestimate`, or `genericcostestimate` anywhere. There is no regression test that asserts a bloated index is costed higher than an unbloated one.
- **Every successful in-tree `pgstatindex` call runs against an empty index**, so none asserts anything about density or fragmentation. The test creates `test (a int primary key, b int[])` with no rows and expects `avg_leaf_density` and `leaf_fragmentation` to be `NaN` through four spellings of the call ([sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)), and a fifth call on the empty index of an empty partition expects `(4,0,8192,0,0,0,0,0,NaN,NaN)` ([expected/pgstattuple.out#partition-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L264-L268)). The remaining calls are error paths: GIN, hash, a partitioned table, a view, a foreign table, a partition and a sequence are each rejected as `not a btree index`. The `NaN` comes from the `max_avail > 0` and `leaf_pages > 0` guards ([pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
- The nbtree features that limit bloat do have tests, but they test correctness, not size: deduplication and deletion are covered through `src/test/regress/sql/btree_index.sql` and `contrib/amcheck`, neither of which asserts page counts.
- All measurements on this page were therefore produced by this page's own script on an isolated exact-pin server, not by any in-tree test.

## Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead

### Short answer

PostgreSQL 17 discards a GIN index at three separate gates, and only the third one is about cost. Gates 1 and 2 are absolute: they are catalog and access-method properties, and no setting moves them. Gate 3 is a comparison of computed costs, not a rule. `choose_bitmap_and()` keeps only the cheapest path in each group of paths that use an identical clause set ([indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1357)), and `add_path()` prunes on cost ([pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L453)), so a GIN index that clears the first two gates loses exactly when its estimate comes out higher, and not otherwise.

For ordinary comparison predicates on the same column it does come out higher, and the reason is a model asymmetry rather than a size difference: `gincostestimate()` charges `random_page_cost` for every pending, entry and data page it expects to touch and adds a `50 * cpu_operator_cost` charge per page on top ([selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L8050)), while `genericcostestimate()` charges the B-tree only a pro-rata share of `index->pages` plus cheap CPU descent ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732)). That is the outcome on every comparison predicate measured below, and it is what `contrib/btree_gin`'s own documentation says in general terms: "In general, these operator classes will not outperform the equivalent standard B-tree index methods" ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)). Nothing in the source makes it a guarantee, and [Where GIN still wins](#where-gin-still-wins) measures a case where the GIN path is the cheaper one.

| Gate | Where | What makes GIN lose | Recovery |
|---|---|---|---|
| 1. Clause matching | `match_clause_to_indexcol()` | The query operator is not in the GIN index's operator family and no planner support function rewrites it, so no `IndexClause` and therefore no GIN path is ever built ([indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269), [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2386-L2500)) | Use a matching operator, or add the operators with `contrib/btree_gin` |
| 2. Plan shape | `build_index_paths()` / `get_index_paths()` | GIN has no `amgettuple`, no ordering, no `amcanreturn`, no null search, no native array search and no `amcanparallel`, so it cannot produce a plain `Index Scan`, satisfy `ORDER BY` pathkeys, feed an `Index Only Scan`, serve `IS NULL`, or contribute a *partial* index path of its own ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89), [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767)). It can still sit under a parallel plan; see [Gate 2](#gate-2-the-required-plan-shape-rules-gin-out) | None. These are AM properties, not costs |
| 3. Cost | `gincostestimate()` versus `btcostestimate()`, then `add_path()` / `choose_bitmap_and()` | GIN's page charges are all at `random_page_cost` and include the whole pending list, as startup cost ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)) | Drain the pending list with `gin_clean_pending_list()` or `VACUUM`. Turning `fastupdate` off is not enough on its own: it stops future entries from joining the pending list but "does not in itself flush previous entries" ([ref/create_index.sgml#fastupdate-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L524-L532)), so it still needs one of the other two afterward |

Measured at this pin on one table carrying both indexes over the same 300,000 rows, so the two `EXPLAIN` runs saw literally identical statistics: the same `n = 42` predicate cost **`12.97` through a `btree_gin` GIN index and `4.52` through a B-tree**, and the planner chose the B-tree. The GIN index was 279 blocks and the B-tree 280, so GIN lost while being the physically smaller index.

### Gate 1: the clause never matches the GIN index

`create_index_paths()` walks `rel->indexlist` and calls `match_restriction_clauses_to_index()` for each index before any cost model runs ([indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L310)). For an `OpExpr`, `match_opclause_to_indexcol()` accepts the clause only when the index column's collation matches and `op_in_opfamily(expr_op, opfamily)` is true; otherwise it falls through to `get_index_clause_from_support()`, the planner-support-function escape hatch ([indxpath.c#match_opclause_to_indexcol-op_in_opfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2433-L2459), [indxpath.c#get_index_clause_from_support](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2557-L2615)).

The core GIN operator families are bootstrap catalog data, and none of them lists `<`, `<=`, `>=`, or `>`:

| GIN opfamily | Operators declared | Evidence |
|---|---|---|
| `gin/array_ops` | `&&`, `@>`, `<@`, `=` (whole-array equality, GIN strategy 4) | [pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244) |
| `gin/tsvector_ops` | `@@`, `@@@` | [pg_amop.dat#gin-tsvector_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1290-L1296) |
| `gin/jsonb_ops` | `@>`, `?`, `?|`, `?&`, `@?`, `@@` | [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611) |
| `gin/jsonb_path_ops` | `@>`, `@?`, `@@` | [pg_amop.dat#gin-jsonb_path_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1613-L1621) |

So `WHERE jb = '{"k": 42}'::jsonb` cannot use a `jsonb_ops` GIN index at all: `jsonb`'s `=` lives in the B-tree and hash families, not the GIN one ([pg_amop.dat#btree-hash-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1571-L1591)). The manual states the rule directly: "each column must be used with operators appropriate to the index type; clauses that involve other operators will not be considered" ([indices.sgml#other-operators](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L505-L508)). `contrib/pg_trgm` says the same about its own classes: "Inequality operators are not supported. Note that those indexes may not be as efficient as regular B-tree indexes for equality operator." ([pgtrgm.sgml#index-support](../../../../raw/postgres-17/doc/src/sgml/pgtrgm.sgml#L413-L425)).

`contrib/btree_gin` closes gate 1 deliberately. Each of its operator classes declares exactly strategies 1 through 5 — `<`, `<=`, `=`, `>=`, `>` — with the type's B-tree comparison proc as GIN support function 1 ([btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69)). Its own documentation states the conclusion this follow-up asks about: "In general, these operator classes will not outperform the equivalent standard B-tree index methods, and they lack one major feature of the standard B-tree code: the ability to enforce uniqueness." ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)).

One case that looks like a gate-1 rejection but is not: a boolean column. `WHERE i = true` is simplified to a bare boolean `Var`, so no `OpExpr` survives, but v17 still matches it. `IsBooleanOpfamily()` accepts any opfamily containing `BooleanEqualOperator`, falling back to a catcache lookup for non-built-in opfamilies ([indxpath.c#IsBooleanOpfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2271-L2286), [pg_opfamily.h#IsBuiltinBooleanOpfamily](../../../../raw/postgres-17/src/include/catalog/pg_opfamily.h#L59-L65)), and `match_boolean_index_clause()` rewrites the bare `Var` back into `indexkey = true` ([indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384)). The upstream expected output shows the resulting `Index Cond: (i = true)` on a `btree_gin` bool index ([bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)), and the measurement below reproduces it for `i`, `i = true` and `i IS TRUE` alike.

### Gate 2: the required plan shape rules GIN out

`get_relation_info()` copies a fixed set of AM capability flags into each `IndexOptInfo`, deriving `amhasgettuple` and `amhasgetbitmap` from whether the AM supplies those callbacks ([plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335)). GIN and B-tree differ on almost every one:

| `IndexAmRoutine` field | GIN | B-tree | Planner consequence |
|---|---|---|---|
| `amgettuple` | `NULL` ([ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79)) | `btgettuple` | `get_index_paths()` submits a path to `add_path()` only when `index->amhasgettuple`; a GIN path can only be collected into `*bitindexpaths` ([indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751)), and `build_index_paths()` returns `NIL` outright for `ST_INDEXSCAN` ([indxpath.c#build_index_paths-scantype](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L826-L842)) |
| `amcanorder` / `amcanorderbyop` | `false` / `false` ([ginutil.c:44](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L44)) | `true` / `false` | `get_relation_info()` fills `sortopfamily` only for `BTREE_AM_OID` or another `amcanorder` AM ([plancat.c#get_relation_info-sortopfamily](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L340-L422)), so `index_is_ordered` is false for GIN and `useful_pathkeys` stays `NIL` ([indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944)) |
| `amcanreturn` | `NULL` ([ginutil.c:70](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L70)) | `btcanreturn` | `index_can_return()` returns false when `amcanreturn` is `NULL` ([indexam.c#index_can_return](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L780-L797)), so every `canreturn[i]` is false and `check_index_only()` fails ([indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800)) |
| `amsearchnulls` | `false` ([ginutil.c:51](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L51)) | `true` | `match_clause_to_indexcol()` accepts a `NullTest` only when `index->amsearchnulls`, so `IS NULL` never reaches GIN ([indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266)) |
| `amsearcharray` | `false` ([ginutil.c:50](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L50)) | `true` | A `ScalarArrayOpExpr` is omitted from plain paths and re-offered only as a bitmap path ([indxpath.c#build_index_paths-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L862-L885), [indxpath.c#get_index_paths-nonnative-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L753-L766)), and `counts.arrayScans` multiplies the GIN estimate ([selfuncs.c#gincost_scalararrayopexpr](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7550-L7660)) |
| `amcanparallel` | `false` ([ginutil.c:55](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L55)) | `true` | No partial GIN *index* path: `build_index_paths()` gates parallel paths on `index->amcanparallel` ([indxpath.c#build_index_paths-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L975-L1002)). It does not keep GIN out of a parallel plan, because the bitmap heap scan above it can still be partial |

**GIN is not shut out of parallel plans, though, and "no parallelism" overstates the flag.** What `amcanparallel = false` removes is a partial *index* path. The bitmap heap scan built on top of a GIN bitmap can still be parallel: `create_index_paths()` hands whatever `choose_bitmap_and()` produced straight to `create_partial_bitmap_paths()` ([indxpath.c:347](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L347)), which sizes workers from the heap pages alone — it passes `index_pages = -1` to `compute_parallel_worker()` — and adds a `Parallel Bitmap Heap Scan` over the unchanged bitmapqual ([allpaths.c#create_partial_bitmap_paths](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4162-L4185)). The GIN scan itself is built once in the leader; the heap fetches that follow it are divided among workers. So a GIN index cannot *drive* parallelism, and it does not prevent it either.

Three consequences follow, and none can be reversed by tuning:

- **No plain index scan.** The AM developer documentation explains why: `amgetbitmap` returns tuples in a bitmap that "doesn't have any specific ordering", "Ordering operators will never be supplied for such a scan", and "there is no provision for index-only scans with `amgetbitmap`, since there is no way to return the contents of index tuples" ([indexam.sgml#amgetbitmap](../../../../raw/postgres-17/doc/src/sgml/indexam.sgml#L991-L1010)). Two upstream test comments say the same operationally: "GIN currently supports only bitmap scans, not plain indexscans" and "GIN only supports bitmapscan, so no need to test plain indexscan" ([create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268), [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)).
- **No sorted output.** "Of the index types currently supported by PostgreSQL, only B-tree can produce sorted output — the other index types return matching rows in an unspecified, implementation-dependent order." ([indices.sgml#ordering](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L530-L538)). Even a bitmap plan built from a B-tree loses order, because the bitmap is laid out in physical order ([indices.sgml#bitmap-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L643-L656)).
- **No index-only scan.** "As a counterexample, GIN indexes cannot support index-only scans because each index entry typically holds only part of the original data value" ([indices.sgml#index-only-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L1125-L1136)), matching the `amcanreturn` contract.

The upstream `amutils` regression test asserts exactly this property matrix for `gin` versus `btree`: `orderable`, `returnable`, `search_array` and `search_nulls` are all `f` for GIN while `bitmap_scan` is `t` and `index_scan` is `f`, and `can_order`, `can_unique`, `can_exclude`, `can_include` are all `f` ([amutils.out#column-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L96-L108), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129), [amutils.out#am-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L152-L157)).

### Gate 3: cost, and why GIN loses on the same column

`cost_index()` calls the AM's `amcostestimate` through the `IndexOptInfo` function pointer, so GIN and B-tree paths for the same clause are priced by different code ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). `gincostestimate()` builds its estimate like this:

1. Read the metapage counters with `ginGetStats()`. Only `nPendingPages` is current; the rest are as of the last `VACUUM` ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)).
2. Seed the startup page count with the **entire pending list**: `entryPagesFetched = numPendingPages` ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886)).
3. Add `ceil(counts.searchEntries * rint(pow(numEntryPages, 0.15)))` entry pages, plus a proportional share of entry and data pages for partial-match keys ([selfuncs.c#gincostestimate-entrypages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7888-L7914)).
4. Charge about `log2(numEntries)` comparisons per search entry for the entry-tree descent ([selfuncs.c#gincostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7919-L7935)).
5. Charge `DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost`, that is `50 * cpu_operator_cost`, for every entry and data page ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).
6. Charge all pending and entry pages at `random_page_cost` as **startup** cost, "because logically-close pages could be far apart on disk" ([selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)).
7. Add scan-time data pages, taking the larger of the per-entry estimate and a selectivity-derived floor of `ceil(indexSelectivity * numTuples / (BLCKSZ / 3))`, again at `random_page_cost`, then per-qual CPU with no descent-height charge and no ordering support ([selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7982-L8029), [selfuncs.c#gincostestimate-qualcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8031-L8048)).

`btcostestimate()` instead delegates to `genericcostestimate()`, which prorates `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)`, then adds a `log2(index->tuples)` comparison charge and the `(tree_height + 1) * 50 * cpu_operator_cost` descent charge described in [2. B-tree height carries an explicit anti-bloat charge](#2-b-tree-height-carries-an-explicit-anti-bloat-charge).

That asymmetry is the whole story for a selective equality lookup, and it survives GIN being the physically smaller index. Reproducing both closed forms in SQL from the catalog and the GIN metapage matched `EXPLAIN` to the cent for `n = 42` (30 matching rows out of 300,000, `numEntryPages` 278, `numEntries` 10,000, `nDataPages` 0):

- GIN: 2 entry pages plus 1 data page at `random_page_cost`, predicted total `12.9725`, printed `12.97`.
- B-tree: `ceil(30 * 280 / 300000) = 1` page at `4.0`, plus `ceil(log2(300000)) = 19` comparisons, plus `(1 + 1) * 50 * 0.0025 = 0.25`, plus `30 * (0.005 + 0.0025)`, predicted total `4.5225`, printed `4.52`.

The losing path is then dropped by ordinary path pruning. `add_path()` compares candidates with `compare_path_costs_fuzzily()` at `STD_FUZZ_FACTOR = 1.01` and refuses or removes a dominated path ([pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L453), [pathnode.c#STD_FUZZ_FACTOR](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L42-L47)). Because a GIN path is only ever a bitmap input, the decisive filter is usually `choose_bitmap_and()`: it first keeps only the cheapest path in each group of paths using identical clause sets, sorts the survivors by index access cost, and then adds a further index to the AND group only when `bitmap_and_cost_est()` reports a lower total ([indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489), [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571)). A GIN index whose own scan cost exceeds the saving it produces is therefore dropped from the bitmap tree entirely, and its predicate reappears as a `Filter` above the surviving B-tree bitmap scan.

### A bloated GIN index loses to a B-tree

This is the case that connects the follow-up back to this page's subject. GIN bloat in the `fastupdate` pending list is charged to the planner in full and immediately, because `nPendingPages` is the one metapage counter `gincostestimate()` treats as current ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711)). The manual states the runtime consequence: "searches must scan the list of pending entries in addition to searching the regular index, and so a large list of pending entries will slow searches significantly" ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L521-L529)).

Measured at the pin on fixture D: a `docs` table with a `tsvector` GIN index (`fastupdate = on`) and a B-tree index on an `int` category column, querying `tsv @@ to_tsquery('simple','zebracorn') AND cat = 7`. The table starts at 200,000 vacuumed rows and takes two further 100,000-row inserts, with `gin_pending_list_limit` raised for the inserting sessions so that nothing drains the list.

| State | Pending pages (`pgstatginindex`) | GIN blocks | GIN scan cost for `tsv @@ …` alone | Two-clause plan using the GIN index | Same query with the GIN index hidden | Plan chosen |
|---|---:|---:|---:|---:|---:|---|
| vacuumed, 200,000 rows | 0 | 302 | `13.01` | `132.40` | `2323.30` | `BitmapAnd` of GIN and B-tree |
| first insert, 300,000 rows | 736 | 1,038 | `3141.12` | `3321.93` | `3486.80` | `BitmapAnd` still, at 25 times the vacuumed price |
| second insert, 400,000 rows | 1,471 | 1,773 | `6264.98` | not chosen | `4646.42` | B-tree bitmap scan only; `tsv @@ …` demoted to `Filter` |
| after `gin_clean_pending_list()` | 0 | 2,073 | `17.49` | `255.85` | `4646.42` | `BitmapAnd` of GIN and B-tree |
| after `VACUUM` | 0 | 2,073 | `17.46` | `255.81` | `4646.40` | `BitmapAnd` of GIN and B-tree |

The GIN index leaves the `BitmapAnd` at the point where its own scan costs more than it saves. `choose_bitmap_and()` keeps it for as long as the two-index plan is cheaper than the B-tree-only plan, and on this fixture that comparison flips between 736 and 1,471 pending pages: `3321.93` against `3486.80` keeps it, and a GIN scan of `6264.98` against a whole alternative plan of `4646.42` drops it. At 736 pages the planner goes on using the GIN index at 25 times the vacuumed plan's cost. The boundary therefore depends on what the alternative costs and not only on the pending list; the B-tree-only plan is expensive here because `cat = 7` alone selects one row in twenty. The GIN-scan column is a separate `EXPLAIN` of the GIN qual on its own, and the hidden-index column drops the GIN index inside a rolled-back subtransaction.

`gin_clean_pending_list()` returned exactly `1471`, matching `pgstatginindex` ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091), [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)).

Note the direction of the sign: unlike the B-tree page-count penalty in [1. Physical page count enters index cost](#1-physical-page-count-enters-index-cost), this is not a mild pro-rata increase. On a single scan every pending page is charged in full, at `random_page_cost` plus the page CPU charge ([selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980)). `gincostestimate()` books that charge as startup cost, but the distinction never reaches a plan: a GIN path is only ever a bitmap input, and `cost_bitmap_heap_scan()` takes the index's total cost as its own startup cost whatever the split was ([costsize.c#cost_bitmap_heap_scan-startup](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044-L1048)).

"In full on every scan" needs one qualification, because the two halves of the charge behave differently once the scan repeats. The pending pages are seeded into `entryPagesFetched` ([selfuncs.c:7886](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7886)), and when there is more than one iteration — a nested-loop inner scan with `loop_count > 1`, or an array qual with `counts.arrayScans > 1` — `gincostestimate()` runs `entryPagesFetched` and `dataPagesFetched` through `index_pages_fetched()` and divides by `outer_scans`, exactly as `genericcostestimate()` does for the B-tree ([selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974)):

| Half of the charge | Repeated scans |
|---|---|
| The `random_page_cost` I/O charge | **Amortized.** It is applied after the cache adjustment, so the Mackert-Lohman cap covers the pending pages too ([selfuncs.c:7980](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7980)) |
| The `50 * cpu_operator_cost` per-page charge | **Not amortized.** It is computed from the pre-adjustment page count, and the source says why in as many words: "This is not amortized over a loop" ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955)) |

Every GIN measurement on this page prices a single scan, where `outer_scans` and `arrayScans` are both 1 and the adjustment does not fire, so the numbers below are the unamortized case throughout.

### Stale GIN metapage statistics

`gincostestimate()` trusts the last-`VACUUM` counters only when the index has not grown too much. It requires `nTotalPages <= numPages`, `nTotalPages > numPages / 4`, `nEntryPages > 0` and `nEntries > 0`; otherwise it invents statistics from the live block count — 90% entry pages, the rest data pages, and 100 entries per entry page — after clamping the page count to at least 10 ([selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767)). The source comment names the 4X cutoff and calls the 100-entries figure "rather bogus".

Two details matter operationally. First, `numPages` comes from `index->pages`, which `get_relation_info()` reads live with `RelationGetNumberOfBlocks()`, so index growth reaches the cost model before any `ANALYZE` ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)). Second, `numPendingPages` is discarded when it is not smaller than `numPages`, which is a sanity guard, not a cost reduction ([selfuncs.c#gincostestimate-pending-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7724-L7727)).

Measured on fixture T: a 2,000-row table's GIN index (`fastupdate = off`, 100 distinct keys) sat at 2 blocks with metapage counters `(nTotalPages, nEntryPages, nDataPages, nEntries) = (2, 1, 0, 100)` and priced `n = 42` at `8.64`. Adding 398,000 rows over 200,000 distinct keys, with an `ANALYZE` but no `VACUUM`, grew it to 1,369 live blocks while the metapage stayed at 2, so `1369 > 2 * 4` put the estimate on the invented branch (1,232 entry pages, 137 data pages, 123,200 entries) and the cost rose to `17.19`. A later `VACUUM` rewrote the metapage to `(1369, 1368, 0, 200000)` and the cost moved by one cent, to `17.20`.

The same staleness is visible after a manual pending-list drain, which is a trap worth naming: `gin_clean_pending_list()` moves entries into the tree and grows the fork but does **not** refresh `nTotalPages`, because only an index build and `VACUUM`'s cleanup call `ginUpdateStats()` ([gininsert.c:406](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L406), [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789)). In fixture D, the freshly vacuumed index read `(302, 273, 28, 1779)` at 302 live blocks, and after the drain it was 2,073 live blocks with the metapage still reading `302`, so `2073 > 302 * 4` kept the cost model on invented statistics until the next `VACUUM` wrote `(2073, 547, 54, 1779)`.

### The keyless full-index path on a partial GIN index

GIN sets `amoptionalkey = true` ([ginutil.c:49](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L49)), so `build_index_paths()` does not bail out when no clause matches the first index column ([indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897)), and a partial index whose predicate is proven still yields a path through `useful_predicate` ([indxpath.c#build_index_paths-generate](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L954-L1003)). `gincostestimate()` then prices that clauseless path as a whole-index scan: when `fullIndexScan` is set or `indexQuals == NIL`, it sets `searchEntries = numEntries`, "as if every key in the index had been listed in the query" ([selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877)). The same branch fires when an attribute has a full scan but no normal scan, which is how `GIN_SEARCH_MODE_ALL` reaches the estimate ([gin.h#GIN_SEARCH_MODE](../../../../raw/postgres-17/src/include/access/gin.h#L34-L37), [selfuncs.c#gincost_pattern-searchmode](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7454-L7489)).

Measured: a **10-block** partial GIN index with 1,000 entries priced `WHERE id <= 5000` (its own predicate, no GIN-indexable clause) at `4430.38`, and the two-`random_page_cost` probe recovered exactly `1001.00` charged pages — 100 times the index's physical size, because `ceil(1000 * rint(pow(9, 0.15))) = 1000`. Adding `AND n = 42` dropped the same index's cost to `8.55`.

### Jobs no GIN index can be created for

Before any planner gate, four things simply cannot be built on GIN in v17, all rejected from the AM flags ([indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879), [cluster.c#cluster_rel-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522)). All four messages were reproduced verbatim at the pin:

| Attempt | v17 error |
|---|---|
| `CREATE UNIQUE INDEX … USING gin` | `access method "gin" does not support unique indexes` |
| `CREATE INDEX … USING gin (…) INCLUDE (…)` | `access method "gin" does not support included columns` |
| `EXCLUDE USING gin (… WITH =)` | `access method "gin" does not support exclusion constraints` |
| `CLUSTER … USING <gin index>` | `cannot cluster on index "…" because access method does not support clustering` |

### Where GIN still wins

- **Operators only GIN has.** Gate 1 runs in both directions: `@@`, `@>`, `?`, `&&` and the `jsonpath` operators are in GIN families and not in B-tree ones ([pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611)), so for those predicates there is no B-tree candidate to lose to.
- **One multicolumn GIN instead of a `BitmapAnd`.** The `btree_gin` documentation says that "for queries that test both a GIN-indexable column and a B-tree-indexable column, it might be more efficient to create a multicolumn GIN index that uses one of these operator classes than to create two separate indexes that would have to be combined via bitmap ANDing" ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)). Measured on fixture D after its `VACUUM`: a `gin (tsv, cat)` index priced the two-column predicate at `21.51` and won the plan at `37.20`, against `240.13` for the `BitmapAnd` of the separate GIN and B-tree indexes at a plan cost of `255.82`.
- **Write amortization.** The pending list exists to make GIN insertion cheap, at the documented cost of slower searches ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)).

### GIN exact-pin measurements

All numbers in this follow-up come from the same run of the same script as the rest of the page: `PostgreSQL 17.11` built from the pin, `autovacuum = off`, `shared_buffers = 256MB`, default planner cost settings (`random_page_cost = 4`, `cpu_operator_cost = 0.0025`, `cpu_index_tuple_cost = 0.005`), and exact statistics; see [Fixtures and method](#fixtures-and-method). `btree_gin` supplied the int4 and bool GIN opclasses, `pgstattuple` supplied `pgstatginindex`, and `pageinspect` supplied `gin_metapage_info()` and `bt_metap()`.

| Fixture | Contents | GIN index | B-tree index |
|---|---|---|---|
| S | `t_both`, 300,000 rows, `n = i % 10000` | `t_both_n_gin`, `btree_gin` int4 opclass, **279 blocks**, `nEntryPages` 278, `nEntries` 10,000, `nDataPages` 0 | `t_both_n_bt`, **280 blocks**, `fastlevel` 1, 90.31% `avg_leaf_density` |
| D | `docs`, 200,000 then 300,000 then 400,000 rows; each `tsvector` is `filler`, `w` plus `i % 1000`, `v` plus `i % 777`, and `zebracorn` on every 5,000th row | `docs_tsv_gin` on `tsvector`, `fastupdate = on`, 302 blocks when vacuumed | `docs_cat_bt` on `cat = i % 20`, 171 blocks |
| T | `t_stale`, 2,000 rows with `n = i % 100`, then 398,000 more with `n = i % 200000` | `t_stale_gin`, `fastupdate = off`, vacuumed at 2,000 rows and not again until the last step | none |
| P-gin | `t_part`, 100,000 rows, `n = i % 1000` | `t_part_gin`, partial `WHERE id <= 5000`, 10 blocks, 1,000 entries | none |
| B-gin | `t_bool`, 100,000 boolean rows, 1% true | `t_bool_gin`, `btree_gin` bool opclass | none |

#### Same table, same statistics, both indexes

Each row below is two `EXPLAIN` runs on fixture S, with the *other* index dropped inside a rolled-back transaction, so `pg_statistic`, `reltuples` and every selectivity estimate are identical. `enable_seqscan` was off so that a missing index path is visible as a `disable_cost`-priced sequential scan.

| Predicate | GIN | B-tree | Row estimate (both) |
|---|---:|---:|---:|
| `n = 42` | `12.97` | `4.52` | 30 |
| `n BETWEEN 100 AND 200` | `66.30` | `42.60` | 3,030 |
| `n < 20` | `28.57` | `8.80` | 600 |
| `n IN (1,2,3)` | `30.10` | `13.57` | 90 |
| `ORDER BY n LIMIT 10` | no index path: `Sort` over `Seq Scan`, `Limit` at `10000010810.92` | `Limit` at `0.66` | 10 |
| `SELECT n WHERE n = 42` | no index-only scan: `Bitmap Heap Scan` at `119.83` | `Index Only Scan` at `4.82` | 30 |
| `n IS NULL` | no index path: `Seq Scan` at `10000004328.00` | `Index Scan` at `8.31` | 1 |

The last three rows are gate-2 outcomes rather than cost losses, and they are not the same outcome. Two of them, `ORDER BY n LIMIT 10` and `n IS NULL`, have **no GIN path at all**, so their costs are `disable_cost = 1.0e10` ([costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130)) — what a forced-off sequential scan costs when it is the only path available. The other, `SELECT n WHERE n = 42`, still **uses the GIN index**: it loses only the index-*only* scan and falls back to a `Bitmap Heap Scan` at `119.83`, against the B-tree's `Index Only Scan` at `4.82`. That is `amcanreturn` being `NULL`, not a missing path, and it is why only two of the seven rows carry a `disable_cost` figure. With every `enable_*` setting at its default, the planner chose the B-tree for `n = 42`, `n BETWEEN 100 AND 200` and `n < 20` alike.

#### The page charge, isolated

Running the same query at `random_page_cost = 4` and `random_page_cost = 1` isolates the page count, because every other GIN charge is a CPU charge that does not scale with it:

| Case | Cost at `rpc = 4` | Cost at `rpc = 1` | Difference / 3 | Reconciliation |
|---|---:|---:|---:|---|
| Fixture S, `n = 42` | `12.97` | `3.97` | `3.00` | 2 entry pages + 1 data page |
| Fixture D, vacuumed, `nTotalPages` 302 = 302 live blocks | `13.01` | `4.01` | `3.00` | trusted stats: `rint(273^0.15) = 2` entry pages + 1 data page |
| Fixture D, 736 pending pages, 1,038 live blocks | `3141.12` | `924.12` | `739.00` | scaled stats (`1038 <= 302 * 4`): 736 pending + 2 entry + 1 data page |
| Fixture D, 1,471 pending pages, 1,773 live blocks | `6264.98` | `1842.98` | `1474.00` | invented stats (`1773 > 302 * 4`): 1,471 pending + 2 entry + 1 data page |
| Fixture D, drained, 2,073 live blocks, metapage still 302 | `17.49` | `5.49` | `4.00` | invented stats: 3 entry pages + 1 data page |
| Fixture T, stale metapage, 1,369 live blocks | `17.19` | `5.19` | `4.00` | invented stats: 3 entry pages + 1 data page |
| Fixture T after `VACUUM`, metapage `(1369, 1368, 0, 200000)` | `17.20` | `5.20` | `4.00` | trusted stats: `rint(1368^0.15) = 3` entry pages + 1 data page |
| Fixture P-gin, keyless partial index, 10 live blocks | `4430.38` | `1427.38` | `1001.00` | `searchEntries = numEntries = 1000` entry pages + 1 data page |

The two pending-list rows are the sharpest result: 736 and 1,471 pages that hold no tree structure at all are charged page for page, `739.00` and `1474.00`, on a scan that touches 3 pages once the list is gone. They also land on different branches. At 1,038 live blocks the last-`VACUUM` counters are still within the 4X window (`302 > 1038 / 4`), so they are scaled; at 1,773 blocks they are not, and the estimate is invented from the block count. The plan consequences are in [A bloated GIN index loses to a B-tree](#a-bloated-gin-index-loses-to-a-b-tree).

#### Boolean column

Fixture B priced `WHERE i`, `WHERE i = true` and `WHERE i IS TRUE` identically at `38.26` for the GIN index scan, each with `Index Cond: (i = true)`. So a bare boolean `Var` is not a gate-1 rejection in v17. `i IS TRUE` additionally left `Filter: (i IS TRUE)` on the heap node while still using the index.

#### Live property matrix

Queried on fixture S, `pg_index_has_property` and `pg_index_column_has_property` returned exactly the values the `amutils` expected output asserts: `index_scan`, `clusterable` and `backward_scan` false for GIN and true for B-tree; `bitmap_scan` true for both; `orderable`, `returnable`, `search_array` and `search_nulls` false for GIN and true for B-tree.

#### A diagnostic pair for a live server

Both blocks below were executed verbatim at the pin against objects literally named `my_table`, `my_col` and `my_gin_index`. The first reports how much of each GIN index is pending list, which is the bloat the planner charges first:

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';

SELECT /* wiki_gin_pending_list_share */
       c.relname                                        AS gin_index,
       pg_relation_size(c.oid) / current_setting('block_size')::int AS live_blocks,
       g.pending_pages,
       g.pending_tuples,
       round(100.0 * g.pending_pages
             / greatest(pg_relation_size(c.oid)
                        / current_setting('block_size')::int, 1), 2)
                                                        AS pending_pct_of_index
  FROM pg_class c
  JOIN pg_index i ON i.indexrelid = c.oid
  JOIN pg_am    a ON a.oid = c.relam
  CROSS JOIN LATERAL pgstatginindex(c.oid) AS g
 WHERE a.amname = 'gin'
   AND c.relname = 'my_gin_index'
 ORDER BY g.pending_pages DESC;

RESET statement_timeout;
RESET lock_timeout;
```

It reported `live_blocks = 340`, `pending_pages = 246`, `pending_tuples = 50000`, `pending_pct_of_index = 72.35`, against a `my_table` of 100,000 vacuumed rows with 50,000 more waiting in the pending list. `pgstatginindex` comes from `pgstattuple`. Since extension version 1.5 the SQL function binds to a C entry point with no superuser check of its own and relies on `EXECUTE`, which the upgrade script revokes from `PUBLIC` and grants to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57), [pgstatindex.c#pgstatginindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L497-L504)). It reads only the metapage ([pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)).

The second recovers the number of pages the planner is charging, without any contrib module:

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
SET enable_seqscan = off;

SET random_page_cost = 4;
EXPLAIN (COSTS ON, TIMING OFF) /* wiki_gin_page_charge_probe_high */
SELECT count(*) FROM my_table
 WHERE my_col @@ to_tsquery('simple', 'filler');

SET random_page_cost = 1;
EXPLAIN (COSTS ON, TIMING OFF) /* wiki_gin_page_charge_probe_low */
SELECT count(*) FROM my_table
 WHERE my_col @@ to_tsquery('simple', 'filler');

RESET random_page_cost;
RESET enable_seqscan;
RESET statement_timeout;
RESET lock_timeout;
```

The `Bitmap Index Scan` costs were `2030.46` and `1121.46`, so `(2030.46 - 1121.46) / 3 = 303.00` pages, of which 246 were pending list. Divide by three because the two runs differ by exactly `3.0` per charged page. The block changes `enable_seqscan` and `random_page_cost` for its own session only and resets them; both are `PGC_USERSET`, so neither needs a reload or a restart ([guc_tables.c#enable_seqscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L783-L792), [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)).

### GIN settings that move the boundary

| Setting | v17 default | Role in the GIN-versus-B-tree decision | Apply scope |
|---|---|---|---|
| `random_page_cost` | 4.0 | Multiplies every pending, entry and data page GIN expects to touch ([guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696)) | session/transaction (`PGC_USERSET`) |
| `cpu_operator_cost` | 0.0025 | Scales GIN's entry-tree descent and its `50 *` per-page CPU charge, and the B-tree's height charge ([guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729)) | session/transaction (`PGC_USERSET`) |
| `gin_pending_list_limit` | 4MB | The size at which an insert *asks* for a cleanup. Not a ceiling: see below ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)) | session/transaction (`PGC_USERSET`) |
| `enable_bitmapscan` | on | Turning it off does not remove GIN's plan shape; it adds `disable_cost` to it. See below ([guc_tables.c#enable_bitmapscan](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L813-L822)) | session/transaction (`PGC_USERSET`) |

Two of those four need the fine print spelled out, because the obvious reading of each is wrong.

**`gin_pending_list_limit` triggers a cleanup; it does not cap the list.** `ginHeapTupleFastInsert()` writes the new entries first and only then compares the resulting size against the limit, setting `needCleanup` after the fact, so the list is already over the limit when the test fires ([ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471)). The cleanup that follows is deliberately not forced: `ginInsertCleanup()` called from a regular insert takes the metapage lock only conditionally and, if another process holds it, returns at once "in hope that concurrent process will clean up pending list" ([ginfast.c#ginInsertCleanup-conditional-lock](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828)). The manual frames it the same way, as a condition under which entries are moved, and notes that "the overhead work can be done by a background process" ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)). So the setting bounds the *typical* startup penalty, not its maximum — fixture D reaches 1,471 pending pages precisely by raising the limit so that no insert ever asks.

**`enable_bitmapscan = off` does not remove the path.** It is a cost penalty, not a veto: `cost_bitmap_heap_scan()` adds `disable_cost` to the startup cost and then prices the path normally ([costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042)). A GIN plan therefore still wins when every alternative is also disabled or more expensive — which is exactly how the `disable_cost`-priced sequential scans in the table above were produced, with `enable_seqscan = off`. What turning it off reliably does is make any other viable plan win.

Two per-index storage parameters change the physical shape rather than its price, and both take `AccessExclusiveLock`: `fastupdate` (default on) and a per-index `gin_pending_list_limit` override ([reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L130), [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)). `gin_clean_pending_list()` drains the list on demand and takes `RowExclusiveLock` on the index ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091)).

### GIN key data structures

| Structure | Field | Role |
|---|---|---|
| `GinQualCounts` | `partialEntries`, `exactEntries`, `searchEntries`, `arrayScans`, `attHasFullScan`, `attHasNormalScan` | The whole per-qual working set `gincostestimate()` derives from the index quals ([selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7370-L7378)) |
| `GinStatsData` | `nPendingPages`, `nTotalPages`, `nEntryPages`, `nDataPages`, `nEntries` | The metapage counters; only `nPendingPages` and `ginVersion` are current ([gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L40-L50)) |
| `GinMetaPageData` | same counters, on disk | Where those numbers live, and why `VACUUM` is what refreshes them ([ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101)) |
| `IndexOptInfo` | `amhasgettuple`, `amhasgetbitmap`, `amcanparallel`, `amsearcharray`, `amsearchnulls`, `amoptionalkey`, `sortopfamily`, `canreturn[]` | The gate-2 flags, all copied once in `get_relation_info()` ([plancat.c#get_relation_info-am-flags](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L317-L335)) |

### GIN caller and callee boundary

```text
create_index_paths                              indxpath.c
  ├─ match_restriction_clauses_to_index
  │    └─ match_clause_to_indexcol
  │         ├─ match_boolean_index_clause       (IsBooleanOpfamily opfamilies)
  │         ├─ match_opclause_to_indexcol       -> op_in_opfamily   GATE 1
  │         │    └─ get_index_clause_from_support
  │         ├─ match_saopclause_to_indexcol
  │         └─ NullTest branch                  needs amsearchnulls GATE 2
  ├─ get_index_paths
  │    ├─ build_index_paths(ST_ANYSCAN)         amhasgettuple / amoptionalkey / pathkeys
  │    │    ├─ check_index_only                 -> index_can_return  GATE 2
  │    │    └─ create_index_path -> cost_index
  │    │         └─ amcostestimate == gincostestimate     GATE 3
  │    │              ├─ ginGetStats            -> GIN metapage
  │    │              ├─ gincost_opexpr / gincost_scalararrayopexpr
  │    │              │    └─ gincost_pattern   -> extractQuery support proc
  │    │              └─ index_pages_fetched    (nestloop / array scans)
  │    ├─ add_path                              amhasgettuple only
  │    └─ build_index_paths(ST_BITMAPSCAN)       non-native SAOP retry
  └─ choose_bitmap_and                          GATE 3, drops the loser
       └─ bitmap_and_cost_est -> bitmap_scan_cost_est -> cost_bitmap_heap_scan
```

Symbols: [indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L310), [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767), [indxpath.c#build_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L804-L1003), [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L8050), [selfuncs.c#gincost_pattern](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7380-L7492), [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89).

### GIN tests and explicit test absence

- **No in-tree test compares a GIN plan against a B-tree plan on the same column**, and none exercises `gincostestimate()`. `src/test` contains no reference to `gincostestimate`.
- The `btree_gin` regression suite sets `enable_seqscan = off` and uses `EXPLAIN (COSTS OFF)` throughout, so it asserts plan *shape* only, never cost ([bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9), [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)).
- What *is* covered is the gate-2 property matrix, by `amutils` ([amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129)), and the bitmap-only restriction, by comments and expected plans in `create_index` and `tsearch` ([create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268), [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)).
- Every measurement in this follow-up was therefore produced by this page's own script on an isolated exact-pin server.

## Measurement Script

### Usage

| Item | Detail |
|---|---|
| Purpose | produces every measured number on this page, in the main answer and in the GIN follow-up: block counts, `pgstatindex` density and fragmentation, `bt_metap()` fast-root levels, GIN metapage counters and pending pages, every `EXPLAIN` cost and worker count, the closed-form predictions, the four GIN rejection messages, and the verbatim run of the two filed diagnostic blocks |
| Invocation | `bash .wiki-runtime/tmp/bloatplan.sh` from the repository root, with the script saved at that path. Any path works: it resolves everything from `WIKI_ROOT`, which defaults to `$PWD` |
| Stages | `build check cluster fa fb ff fg fh fi fn fstale fl3 fp gs gd gt gp gb grej diag predict summary stop` in that default order, plus `clean` on request. Select stages as arguments: `bash .wiki-runtime/tmp/bloatplan.sh fa predict`. `build` configures out of tree, installs core plus `pgstattuple`, `pageinspect` and `btree_gin`, and skips when the binary exists. `check` runs `make check` and the three contrib suites. `cluster` runs `initdb` once, starts the server, and installs the recording helpers `xp()`, `nodes()`, `ixstat()`, `ginstat()` and `predict()`. `fa` `fb` `ff` `fg` `fh` `fi` `fn` `fstale` `fl3` `fp` build the B-tree fixtures A, B with M, F, G, H, I, N, the forged catalog rows, the `BitmapAnd` pair L3, and P with P-100. `gs` `gd` `gt` `gp` `gb` build the GIN fixtures S, D, T, P-gin and B-gin. `grej` sends the four statements that must fail. `diag` reads the two `sql` blocks out of this page and runs them verbatim. `predict` compares the closed form with `EXPLAIN`. `summary` writes the result file. `stop` stops the server and asserts the teardown. Every fixture stage drops and rebuilds its own tables and replaces its own result rows, so any stage can be re-run alone; `grej` needs `gs`, and `predict` needs `fa`. Every stage except `build` and `check` starts the server itself if it is not up, so a selected re-run works after a default run has already stopped the cluster |
| Failure handling | Every stage's exit status is checked and the first failure aborts the run with a non-zero status, so a failed `configure`, `make`, `make check` or `psql` cannot be reported as a pass; an unknown stage name is rejected before anything runs. An `EXIT` trap stops the server and the snapshot-holding session on every path out, including a `die` in the middle of a fixture and an interrupt, and leaves the sandbox for inspection |
| Environment | `WIKI_ROOT` (`$PWD`), `SRC` (`$WIKI_ROOT/raw/postgres-17`), `SANDBOX` (`$WIKI_ROOT/.wiki-runtime/tmp/bloatplan`), `PAGE` (this page under `$WIKI_ROOT`), `PORT` (`55437`), `JOBS` (`8`), `STATS_TARGET` (`10000`) |
| Prerequisites | a C toolchain, `make`, `flex`, `bison`, `perl` for the build, and zlib headers. The tree is configured `--without-icu --without-readline`, so neither library is needed. `initdb` runs with `--locale=C --encoding=UTF8`. About 3 GiB of disk for the build, install and data directories |
| Output | `$SANDBOX/out/summary.txt`: the B-tree index table, the GIN table, the prediction table, each fixture table's visibility state, one line per recorded plan with every node's index, costs, rows, workers, index condition and filter, then the platform facts, the four error messages and the two diagnostic outputs. Read it first. `server.log`, the build and test logs, and the extracted `diag1.sql` and `diag2.sql` sit beside it |
| Runtime | about 2 minutes for a full run on 10 cores, of which roughly 1 minute is the build and the four test suites; a re-run of every fixture stage from a built tree takes about 35 seconds |
| Cleanup | `bash .wiki-runtime/tmp/bloatplan.sh clean` stops the server and deletes `$SANDBOX`. The `stop` stage, which runs by default, stops the server and asserts that no `postmaster.pid`, no process from the data directory and no socket on `$PORT` is left. Neither is the only safety net: the `EXIT` trap stops the server on any exit path, so an aborted run leaves no postmaster behind either |

Isolation: the pinned checkout is read only, the build is out of tree, the cluster has its own data directory, socket directory and port `55437`, and every fixture table is disposable. Stage `fstale` forges two `pg_class` rows on purpose, tagged `wiki_bloatplan_fixture_catalog_forgery`; they belong to a throwaway fixture and must never be pointed at a database anyone cares about. GUC apply scopes are named in the script's own header. Every `psql` call runs `-X -v ON_ERROR_STOP=1` with session-scoped `statement_timeout` and `lock_timeout`, except the `grej` stage, whose four statements are meant to fail and which asserts that exactly four errors came back. Every statement carries a `/* wiki_bloatplan_... */` tag after its leading verb.

### Last run

| Item | Value |
|---|---|
| Date | 2026-09-19, three passes with the script text filed below: a full default run, then a second pass over every fixture stage with the cluster stopped at the start, then a third full run from an empty sandbox |
| Server | `PostgreSQL 17.11 on aarch64-apple-darwin27.0.0, compiled by Apple clang version 21.0.0 (clang-2100.3.34.2), 64-bit`, built from pin `786db8dcf168bd9df8f55047337525ac19118b1c`, configured `--without-icu --without-readline` |
| Platform | Darwin arm64, `block_size` 8192, maximum data alignment 8, `initdb --locale=C --encoding=UTF8` |
| Test suites | `make check` All 225 tests passed; `pgstattuple` All 1, `pageinspect` All 8, `btree_gin` All 30 — twice, once in each full run |
| Agreement | all three passes wrote a byte-identical `summary.txt` apart from its two run timestamps: 33 B-tree index rows, 11 GIN rows, 7 of 7 closed-form predictions equal to `EXPLAIN`, 118 plans, 26 fixture-table rows, four error messages, both diagnostic outputs |
| Runtime | 1 minute 38 seconds for the full run from an empty sandbox, of which roughly 77 seconds is the build and the four test suites; 27 seconds for a second pass over every fixture stage on a built tree |
| Teardown | the `stop` stage asserted no `postmaster.pid`, no process from the data directory and no socket on port 55437 after each run. A deliberately aborted run (`cluster bogus`, which starts the server and then hits an unknown stage) was checked to leave that same state through the `EXIT` trap. The `clean` stage then stopped the server and deleted the sandbox |

The published text was extracted from this page and `md5`-compared with the script text that ran, before each full run started.

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
# The pinned checkout under raw/postgres-17 is read only.  Everything this
# script writes lives under $SANDBOX, and the clean stage deletes it.  The
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
#   build check cluster fa fb ff fg fh fi fn fstale fl3 fp
#   gs gd gt gp gb grej diag predict summary stop
# and, on request only: clean
#
# Failure and teardown.  Every stage's exit status is checked and any failure
# aborts the run, so a failed configure, make, make check or psql can never be
# reported as a pass.  An EXIT trap stops the server and the snapshot-holding
# session on every path out of the script, including a failure in the middle of
# a fixture and an interrupt, so no postmaster is left behind; the sandbox is
# kept for inspection unless the clean stage ran.  Any stage that needs a
# server starts one first, so a selected re-run works after a default run has
# stopped the cluster; only build and check do not need one.
#
# Environment: WIKI_ROOT SRC SANDBOX PAGE PORT JOBS STATS_TARGET
#
# GUC apply scopes, from the pinned v17 guc_tables.c:
#   shared_buffers, port, listen_addresses, unix_socket_directories
#       -> PGC_POSTMASTER, restart.  Set once on the postmaster command line.
#   autovacuum -> PGC_SIGHUP, reload.  Set once on the postmaster command line.
#   default_statistics_target, enable_seqscan, enable_bitmapscan,
#   enable_indexscan, enable_hashjoin, enable_mergejoin, effective_cache_size,
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

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; DATA="$SANDBOX/data"
SOCK="$SANDBOX/sock"; OUT="$SANDBOX/out"; BIN="$INST/bin"; DB=bloatplan
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE="$DB" PGUSER=postgres

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# -X ignores ~/.psqlrc so a stray file cannot change a result; ON_ERROR_STOP
# means no failed statement passes silently.  The three settings are
# PGC_USERSET, so passing them through libpq applies them at session scope.
SESSION_OPTS="-c statement_timeout=20min -c lock_timeout=60s -c default_statistics_target=$STATS_TARGET"
pg()    { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -v ON_ERROR_STOP=1 "$@"; }
pgq()   { pg -At "$@"; }
# grej sends four statements that are meant to fail, so it runs without
# ON_ERROR_STOP and keeps the error text.
pgerr() { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X "$@"; }

running() { [ -x "$BIN/pg_ctl" ] && "$BIN/pg_ctl" -D "$DATA" status >/dev/null 2>&1; }
stop_server() { running && "$BIN/pg_ctl" -D "$DATA" -m fast -w stop >/dev/null 2>&1; return 0; }

# Teardown on every path out of the script, not only on the stop stage: a die
# in the middle of a fixture, a failed psql, or an interrupt all land here, so
# no measurement postmaster and no snapshot holder survives the run.  The
# sandbox is left alone; only the clean stage deletes it.
HOLDER_PID=""
on_exit() {
  local st=$?
  trap - EXIT INT TERM
  if [ "$st" -ne 0 ] && running; then
    printf '!! exit status %s with the server still up; stopping it\n' "$st" >&2
  fi
  # Stopping the server first drops the holder session's connection, so its
  # pg_sleep ends and the background subshell exits on its own; the kill is only
  # there for the case where the shutdown could not reach it.
  stop_server
  [ -n "$HOLDER_PID" ] && kill "$HOLDER_PID" 2>/dev/null
  wait 2>/dev/null
  [ "$st" -ne 0 ] && printf '!! failed with status %s; %s was kept\n' "$st" "$SANDBOX" >&2
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
  mkdir -p "$BUILD" "$OUT" "$SOCK"
  if [ -x "$BIN/postgres" ]; then note "already built: $("$BIN/postgres" --version)"; return 0; fi
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --without-icu --without-readline \
      > "$OUT/configure.log" 2>&1 ) || die "configure failed, see $OUT/configure.log"
  ( cd "$BUILD" && make -s -j "$JOBS" > "$OUT/make.log" 2>&1 \
      && make -s install > "$OUT/install.log" 2>&1 \
      && for m in pgstattuple pageinspect btree_gin; do
           make -s -C contrib/$m install >> "$OUT/install.log" 2>&1 || exit 1
         done ) || die "make failed, see $OUT/make.log"
  note "$("$BIN/postgres" --version), pin $(cd "$SRC" && git rev-parse HEAD)"
}

stage_check() {
  say "check: core regression suite, then the three contrib suites, on the built tree"
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
  mkdir -p "$OUT" "$SOCK"
  [ -f "$DATA/PG_VERSION" ] || "$BIN/initdb" -D "$DATA" -U postgres --locale=C --encoding=UTF8 \
    > "$OUT/initdb.log" 2>&1 || die "initdb failed, see $OUT/initdb.log"
  running || "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w \
    -o "-p $PORT -k $SOCK -c listen_addresses='' -c autovacuum=off -c shared_buffers=256MB" start \
    >/dev/null 2>&1 || die "server did not start, see $OUT/server.log"
  PGDATABASE=postgres pgq -c "SELECT /* wiki_bloatplan_dbcheck */ 1 FROM pg_database WHERE datname = '$DB'" | grep -q 1 \
    || PGDATABASE=postgres pg -q -c "CREATE /* wiki_bloatplan_fixture */ DATABASE $DB"
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
    PERFORM set_config(split_part(s, '=', 1), split_part(s, '=', 2), true);
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
  DELETE FROM r WHERE label = p_label;
  INSERT INTO r(label, top_node, startup, total, plan_rows, plan)
  VALUES (p_label, j->0->'Plan'->>'Node Type', (j->0->'Plan'->>'Startup Cost')::numeric,
          (j->0->'Plan'->>'Total Cost')::numeric, (j->0->'Plan'->>'Plan Rows')::numeric, j::jsonb);
  RETURN format('%s | %s', p_label, nodes(p_label));
END $fn$;

-- nodes(): one line per recorded plan, every node with its index and costs.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION nodes(p_label text)
RETURNS text LANGUAGE sql AS $fn$
  SELECT string_agg(
           (n->>'Node Type')
           || coalesce(' ' || (n->>'Index Name'), '')
           || ' ' || (n->>'Startup Cost') || '..' || (n->>'Total Cost')
           || ' rows=' || (n->>'Plan Rows')
           || coalesce(' workers=' || (n->>'Workers Planned'), '')
           || coalesce(' cond=' || (n->>'Index Cond'), '')
           || coalesce(' filter=' || (n->>'Filter'), ''), ' ; ')
    FROM r, LATERAL jsonb_path_query(plan, 'strict $.** ? (exists(@."Node Type"))') AS n
   WHERE label = p_label
$fn$;

-- ixstat(): block count, pgstatindex figures and the fast-root level of one B-tree.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION ixstat(p_label text, p_idx regclass)
RETURNS text LANGUAGE plpgsql AS $fn$
DECLARE s record; m record; b bigint;
BEGIN
  SELECT * INTO s FROM pgstatindex(p_idx);
  SELECT * INTO m FROM bt_metap(p_idx::text);
  b := pg_relation_size(p_idx) / current_setting('block_size')::int;
  DELETE FROM ix WHERE label = p_label;
  INSERT INTO ix(label, idx, blocks, tree_level, fastlevel, leaf_pages, internal_pages, empty_pages,
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
  SELECT * INTO m FROM gin_metapage_info(get_raw_page(p_idx::text, 0));
  SELECT * INTO g FROM pgstatginindex(p_idx);
  b := pg_relation_size(p_idx) / current_setting('block_size')::int;
  DELETE FROM gx WHERE label = p_label;
  INSERT INTO gx(label, idx, blocks, pending_pages, n_total, n_entry, n_data, n_entries)
  VALUES (p_label, p_idx::text, b, g.pending_pages, m.n_total_pages, m.n_entry_pages, m.n_data_pages, m.n_entries);
  RETURN format('%s | %s blocks=%s pending=%s meta(total,entry,data,entries)=(%s,%s,%s,%s)',
                p_label, p_idx, b, g.pending_pages, m.n_total_pages, m.n_entry_pages, m.n_data_pages, m.n_entries);
END $fn$;

-- predict(): the whole-index-scan cost of one B-tree, recomputed in float8 in
-- the planner's own order of operations from three inputs only: the live
-- block count, the table's reltuples and the fast-root level.  It is compared
-- with the EXPLAIN total recorded under p_label.  The index-only scan pays no
-- heap cost because every fixture it is used on is 100% all-visible.
CREATE OR REPLACE /* wiki_bloatplan_fixture */ FUNCTION predict(p_label text, p_idx regclass, p_tbl regclass)
RETURNS text LANGUAGE plpgsql AS $fn$
DECLARE
  pages float8 := pg_relation_size(p_idx) / current_setting('block_size')::int;
  tuples float8 := (SELECT reltuples FROM pg_class WHERE oid = p_tbl);
  fl float8 := (SELECT fastlevel FROM bt_metap(p_idx::text));
  rpc float8 := current_setting('random_page_cost')::float8;
  citc float8 := current_setting('cpu_index_tuple_cost')::float8;
  coc float8 := current_setting('cpu_operator_cost')::float8;
  ctc float8 := current_setting('cpu_tuple_cost')::float8;
  generic float8; d1 float8; d2 float8; startup float8; idx_total float8; total float8; obs text;
BEGIN
  generic := (pages * rpc) + tuples * (citc + coc * 1);   -- genericcostestimate, one index qual
  d1 := ceil(ln(tuples) / ln(2.0::float8)) * coc;         -- btcostestimate, log2(N) comparisons
  d2 := (fl + 1) * 50.0 * coc;                            -- btcostestimate, per-page descent charge
  startup := d1 + d2;
  idx_total := generic + d1 + d2;
  total := startup + ((idx_total - startup) + ctc * tuples);   -- cost_index
  SELECT to_char(r.total, 'FM999999990.00') INTO obs FROM r WHERE label = p_label;
  DELETE FROM pr WHERE label = p_label;
  INSERT INTO pr(label, pages, tuples, fastlevel, predicted, observed)
  VALUES (p_label, pages, tuples, fl, to_char(total, 'FM999999990.00'), obs);
  RETURN format('%s | pages=%s tuples=%s fastlevel=%s predicted=%s observed=%s',
                p_label, pages, tuples, fl, to_char(total, 'FM999999990.00'), obs);
END $fn$;
SQL
  {
    date -u '+run started %Y-%m-%dT%H:%M:%SZ'
    uname -sm
    pgq -c "SELECT /* wiki_bloatplan_version */ version()"
    echo "pin $(cd "$SRC" && git rev-parse HEAD)"
    "$BIN/pg_controldata" "$DATA" | grep -E 'Database block size|Maximum data alignment'
    pgq -c "SELECT /* wiki_bloatplan_settings */ name || ' = ' || setting FROM pg_settings WHERE name IN ('autovacuum','shared_buffers','random_page_cost','seq_page_cost','cpu_tuple_cost','cpu_index_tuple_cost','cpu_operator_cost','effective_cache_size','min_parallel_index_scan_size','default_statistics_target','gin_pending_list_limit') ORDER BY 1"
  } | tee "$OUT/platform.txt"
}

# Settings that force an index or index-only scan so that one named index is
# priced, not chosen.  Both are PGC_USERSET and last for one statement.
IOS="'enable_seqscan=off','enable_bitmapscan=off'"
PAR="'max_parallel_workers_per_gather=8','max_parallel_workers=8','min_parallel_table_scan_size=0','enable_seqscan=off','enable_bitmapscan=off'"
PAR0="$PAR,'parallel_setup_cost=0','parallel_tuple_cost=0'"
# A nested loop with the fixture index on the inner side, so that the scan is
# repeated and index_pages_fetched() prices the index pages through the cache model.
NL="'enable_hashjoin=off','enable_mergejoin=off','max_parallel_workers_per_gather=0'"

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
SELECT /* wiki_bloatplan_metap */ 'F deleted metapage root=' || root || ' level=' || level || ' fastroot=' || fastroot || ' fastlevel=' || fastlevel FROM bt_metap('f_root_idx');
SELECT /* wiki_bloatplan_record */ xp('F deleted point', 'SELECT id FROM f_root WHERE id = 999950', NULL, $IOS);
SELECT /* wiki_bloatplan_record */ xp('F deleted full',  'SELECT id FROM f_root WHERE id > 0', NULL, $IOS);
SQL
  pg -q -c "REINDEX /* wiki_bloatplan_fixture */ INDEX f_root_idx"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ixstat('F rebuilt', 'f_root_idx');
SELECT /* wiki_bloatplan_record */ xp('F rebuilt point', 'SELECT id FROM f_root WHERE id = 999950', NULL, $IOS);
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
  # every row of the scan is charged a heap fetch, as in a plain index scan.
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
    arr=$(pgq -c "SELECT /* wiki_bloatplan_array */ '{' || string_agg((g * 150)::text, ',') || '}' FROM generate_series(1, $n) g")
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
SELECT /* wiki_bloatplan_catalog */ 'catalog ' || relname || ' relpages=' || relpages || ' reltuples=' || reltuples || ' live blocks=' || pg_relation_size(oid) / 8192 FROM pg_class WHERE relname IN ('b_stale_idx','b_part_idx') ORDER BY 1;
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
-- let the page report the gap rather than assert there is none.
SELECT /* wiki_bloatplan_estimate */ 'l3 actual rows: a=5 AND c=7 = ' || count(*) FILTER (WHERE a = 5 AND c = 7)
       || ', a=5 = ' || count(*) FILTER (WHERE a = 5)
       || ', c=7 = ' || count(*) FILTER (WHERE c = 7)
       || ', total = ' || count(*) FROM l3;
SELECT /* wiki_bloatplan_estimate */ 'l3 estimated rows: ' || string_agg(label || ' = ' || plan_rows, ', ' ORDER BY seq)
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
  local t="$1" tries=0
  ( PGAPPNAME=bloatplan_holder PGOPTIONS="-c statement_timeout=0" "$BIN/psql" -X -q \
      -c "BEGIN /* wiki_bloatplan_fixture_holder */ ISOLATION LEVEL REPEATABLE READ" \
      -c "SELECT /* wiki_bloatplan_fixture_holder */ count(*) FROM $t" \
      -c "SELECT /* wiki_bloatplan_fixture_holder */ pg_sleep(900)" > "$OUT/holder_$t.log" 2>&1 ) &
  HOLDER_PID=$!
  until [ "$(pgq -c "SELECT /* wiki_bloatplan_holder_probe */ count(*) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder' AND backend_xmin IS NOT NULL")" = "1" ]; do
    tries=$((tries + 1)); [ "$tries" -gt 60 ] && die "the holder session never took its snapshot"
    sleep 1
  done
  pgq -c "SELECT /* wiki_bloatplan_holder_probe */ 'holder backend_xmin=' || backend_xmin FROM pg_stat_activity WHERE application_name = 'bloatplan_holder'"
}

release_snapshot() {
  pgq -c "SELECT /* wiki_bloatplan_holder_release */ 'holder terminated=' || pg_terminate_backend(pid) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder'"
  wait 2>/dev/null
  HOLDER_PID=""
  [ "$(pgq -c "SELECT /* wiki_bloatplan_holder_probe */ count(*) FROM pg_stat_activity WHERE application_name = 'bloatplan_holder'")" = "0" ] \
    || die "the holder session is still connected"
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
    release_snapshot
    pg -q -c "ANALYZE /* wiki_bloatplan_fixture */ p${mod}_free" -c "ANALYZE /* wiki_bloatplan_fixture */ p${mod}_held"
    pgq <<SQL
SELECT /* wiki_bloatplan_record */ xp('P$mod free bitmap', 'SELECT * FROM p${mod}_free WHERE tag = 7', NULL, 'enable_seqscan=off','enable_indexscan=off');
SELECT /* wiki_bloatplan_record */ xp('P$mod held bitmap', 'SELECT * FROM p${mod}_held WHERE tag = 7', NULL, 'enable_seqscan=off','enable_indexscan=off');
SQL
  done
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
SELECT /* wiki_bloatplan_props */ 'S property ' || p || ' gin=' || pg_index_has_property('t_both_n_gin'::regclass, p) || ' btree=' || pg_index_has_property('t_both_n_bt'::regclass, p)
  FROM unnest(ARRAY['clusterable','index_scan','bitmap_scan','backward_scan']) p;
SELECT /* wiki_bloatplan_props */ 'S column property ' || p || ' gin=' || pg_index_column_has_property('t_both_n_gin'::regclass, 1, p) || ' btree=' || pg_index_column_has_property('t_both_n_bt'::regclass, 1, p)
  FROM unnest(ARRAY['orderable','returnable','search_array','search_nulls']) p;
SQL
}

docs_rows() {  # docs_rows <from> <to>: the INSERT that fills fixture D
  printf "INSERT /* wiki_bloatplan_fixture */ INTO docs SELECT i, i %% 20, to_tsvector('simple', 'filler w' || (i %% 1000) || ' v' || (i %% 777) || CASE WHEN i %% 5000 = 0 THEN ' zebracorn' ELSE '' END) FROM generate_series(%s, %s) i" "$1" "$2"
}

docs_state() {  # docs_state <label>: what the planner does with the two-clause query in this state
  local l="$1"
  local q="SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7"
  local qg="SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn')"
  pgq <<SQL
SELECT /* wiki_bloatplan_record */ ginstat('D $l', 'docs_tsv_gin');
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
  PGOPTIONS="$SESSION_OPTS -c gin_pending_list_limit=1GB" "$BIN/psql" -X -q -v ON_ERROR_STOP=1 \
    -c "$(docs_rows 200001 300000)" -c "ANALYZE /* wiki_bloatplan_fixture */ docs"
  docs_state "pending one"
  PGOPTIONS="$SESSION_OPTS -c gin_pending_list_limit=1GB" "$BIN/psql" -X -q -v ON_ERROR_STOP=1 \
    -c "$(docs_rows 300001 400000)" -c "ANALYZE /* wiki_bloatplan_fixture */ docs"
  docs_state "pending two"
  pgq -c "SELECT /* wiki_bloatplan_drain */ 'gin_clean_pending_list returned ' || gin_clean_pending_list('docs_tsv_gin')"
  docs_state "drained"
  pg -q -c "VACUUM /* wiki_bloatplan_fixture */ docs"
  docs_state "revacuumed"
  pg -q -c "CREATE /* wiki_bloatplan_fixture */ INDEX docs_multi_gin ON docs USING gin (tsv, cat)"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ ginstat('D multi', 'docs_multi_gin');
SELECT /* wiki_bloatplan_record */ xp('D multi chosen', $q$SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7$q$);
SELECT /* wiki_bloatplan_record */ xp('D multi hidden', $q$SELECT * FROM docs WHERE tsv @@ to_tsquery('simple','zebracorn') AND cat = 7$q$, 'docs_multi_gin');
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
  pgerr <<'SQL' 2>&1 | grep -E 'ERROR' | tee "$OUT/grej.txt"
CREATE /* wiki_bloatplan_expected_error */ UNIQUE INDEX rej_u ON t_both USING gin (n);
CREATE /* wiki_bloatplan_expected_error */ INDEX rej_i ON t_both USING gin (n) INCLUDE (id);
ALTER /* wiki_bloatplan_expected_error */ TABLE t_both ADD CONSTRAINT rej_x EXCLUDE USING gin (n WITH =);
CLUSTER /* wiki_bloatplan_expected_error */ t_both USING t_both_n_gin;
SQL
  [ "$(grep -c ERROR "$OUT/grej.txt")" = "4" ] || die "expected four errors, see $OUT/grej.txt"
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
  PGOPTIONS="$SESSION_OPTS -c gin_pending_list_limit=1GB" "$BIN/psql" -X -q -v ON_ERROR_STOP=1 \
    -c "INSERT /* wiki_bloatplan_fixture */ INTO my_table SELECT i, to_tsvector('simple', 'filler w' || (i % 1000)) FROM generate_series(100001, 150000) i"
  "$BIN/psql" -X -v ON_ERROR_STOP=1 -f "$OUT/diag1.sql" > "$OUT/diag1.out" 2>&1 || die "diagnostic block 1 failed, see $OUT/diag1.out"
  "$BIN/psql" -X -v ON_ERROR_STOP=1 -f "$OUT/diag2.sql" > "$OUT/diag2.out" 2>&1 || die "diagnostic block 2 failed, see $OUT/diag2.out"
  grep -E 'my_gin_index' "$OUT/diag1.out" "$OUT/diag2.out"
}

stage_predict() {
  say "predict: the closed form in (pages, tuples, fastlevel) against EXPLAIN, fixture A"
  pgq <<'SQL'
SELECT /* wiki_bloatplan_record */ predict('A dense full',  'a_dense_idx',  'a_dense');
SELECT /* wiki_bloatplan_record */ predict('A sparse full', 'a_sparse_idx', 'a_sparse');
SELECT /* wiki_bloatplan_predict */ 'closed form matches EXPLAIN on ' || count(*) FILTER (WHERE predicted = observed) || ' of ' || count(*) FROM pr;
SQL
}

stage_summary() {
  say "summary: written to $OUT/summary.txt"
  pg <<'SQL' > "$OUT/summary.txt" 2>&1
\pset pager off
SELECT /* wiki_bloatplan_summary */ label, idx, blocks, tree_level, fastlevel, leaf_pages AS leaf, internal_pages AS internal,
       deleted_pages AS deleted, density, frag FROM ix ORDER BY seq;
SELECT /* wiki_bloatplan_summary */ label, idx, blocks, pending_pages AS pending, n_total, n_entry, n_data, n_entries FROM gx ORDER BY seq;
SELECT /* wiki_bloatplan_summary */ label, pages, tuples, fastlevel, predicted, observed, predicted = observed AS match FROM pr ORDER BY seq;
SELECT /* wiki_bloatplan_summary */ c.relname AS fixture_table, c.relpages, c.relallvisible, c.reltuples
  FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE n.nspname = 'public' AND c.relkind = 'r' AND c.relname NOT IN ('r','ix','gx','pr') ORDER BY 1;
\pset format unaligned
SELECT /* wiki_bloatplan_summary */ label, nodes(label) AS plan FROM r ORDER BY seq;
SQL
  cat "$OUT/platform.txt" >> "$OUT/summary.txt"
  [ -f "$OUT/grej.txt" ] && cat "$OUT/grej.txt" >> "$OUT/summary.txt"
  [ -f "$OUT/diag1.out" ] && cat "$OUT/diag1.out" "$OUT/diag2.out" >> "$OUT/summary.txt"
  date -u '+run finished %Y-%m-%dT%H:%M:%SZ' >> "$OUT/summary.txt"
  note "$(grep -c . "$OUT/summary.txt") lines"
}

# The teardown is asserted, not assumed: no postmaster.pid, no process still
# running from this data directory, and no socket left on the port.
assert_down() {
  [ -f "$DATA/postmaster.pid" ] && die "postmaster.pid is still present in $DATA"
  pgrep -f -- "-D $DATA" >/dev/null 2>&1 && die "a postgres process is still running from $DATA"
  [ -e "$SOCK/.s.PGSQL.$PORT" ] && die "the socket $SOCK/.s.PGSQL.$PORT is still present"
  return 0
}

stage_stop() {
  say "stop: shut the measurement cluster down"
  stop_server
  assert_down
  note "stopped: no postmaster.pid, no process from $DATA, no socket on port $PORT; $SANDBOX is kept until the clean stage"
}

stage_clean() {
  say "clean: stop the server and delete the sandbox"
  stop_server
  assert_down
  rm -rf "$SANDBOX"
  note "removed $SANDBOX"
}

# The dispatcher rejects an unknown stage, starts a server for the stages that
# need one, and turns a failed stage into a failed run.  Without the || die a
# stage that returns non-zero without calling die is skipped over and the script
# still exits 0.
run_stage() {
  local st="$1"
  case "$st" in
    build|check|cluster|stop|clean) ;;
    fa|fb|ff|fg|fh|fi|fn|fstale|fl3|fp|gs|gd|gt|gp|gb|grej|diag|predict|summary) need_server ;;
    *) die "unknown stage: $st" ;;
  esac
  "stage_$st" || die "stage $st failed"
}

DEFAULT="build check cluster fa fb ff fg fh fi fn fstale fl3 fp gs gd gt gp gb grej diag predict summary stop"
[ $# -eq 0 ] && set -- $DEFAULT
for st in "$@"; do run_stage "$st"; done
```

## Open Questions

- Fixture P's growth ratios (3.2x unblocked, 6.9x blocked) come from one deliberately harsh workload with autovacuum off, and fixture P-100 shows the result is shape-dependent: on a 100-value column the same workload grew the index to 1,020 blocks with or without a held snapshot. Why the free horizon bought nothing there was not traced to source. Both P-100 runs end with identical block counts, which points at the insertion pattern inside long runs of one key rather than at the horizon, but nothing on this page establishes that. The documentation's stronger claim, that some indexes "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)), was not reproduced here and would need a gentler, steady-state workload to test.
- Mechanism 3 is measured on the index side only. [Index pages in the cache model](#index-pages-in-the-cache-model) prices a repeated inner scan at `0.66` against `2.50` per loop, but every fixture table is all-visible, so no heap page is fetched and the other half of the mechanism, a bloated index shrinking the heap's prorated share of `effective_cache_size`, was not isolated. It needs a nested loop whose inner scan fetches heap pages while the tables and the index together exceed the cache setting.
- Whether GIN's v16 `DEFAULT_PAGE_CPU_MULTIPLIER` charges make GIN bloat pricing behave qualitatively like the B-tree page charge was not investigated here; the GIN cost model is separate and is covered only to the extent needed to scope this page.
- Two extension-boundary claims are read from source and were not exercised: that a custom index AM gets `tree_height = -1` unless `relam == BTREE_AM_OID`, and that a `get_relation_info_hook` plugin can rewrite `pages`, `tuples` or `tree_height` before costing. No custom AM and no hook plugin was built. The same goes for contrib `bloom` being charged for every index page on every scan: it is what `blcostestimate()` says, and no bloom fixture was measured.
- The `BitmapAnd` boundary in fixture D was located between two pending-list sizes, 736 pages (index kept) and 1,471 pages (index dropped), and not bisected. The first filing of this page saw both outcomes at 982 pending pages on two different fixtures and could not explain the difference; the comparison that decides it is now identified, the two-index plan against the B-tree-only plan, but the crossover size on any one fixture was not measured.
- Fixture T's `VACUUM` refreshed the GIN metapage from `(2, 1, 0, 100)` to `(1369, 1368, 0, 200000)` and the `n = 42` cost moved by one cent, from `17.19` to `17.20`. The invented and trusted branches produce the same charged page count here, `4.00`, so this fixture does not separate them; a fixture where the two branches diverge visibly was not built.
- The GIN entry-page estimate `ceil(searchEntries * rint(pow(numEntryPages, 0.15)))` was reconciled arithmetically in every measured case, but `rint()`'s banker's rounding at exact `.5` boundaries was not exercised.
- Whether `contrib/btree_gin`'s partial-match path (`gincost_pattern()` charging `partialEntries += 100` per key) systematically over- or under-charges a range predicate was not investigated; the measurements report only the resulting costs.
- The partial-scan columns of the parallel-worker table were planned with `parallel_setup_cost = 0` and `parallel_tuple_cost = 0`. At the default parallel costs the planner chose a serial plan for the 50% and 20% scans on both indexes, so whether bloat changes the worker count of a real partial scan depends on the parallel path winning on cost first, which was not explored.
- Every row estimate on this page comes from exhaustive statistics, because the script runs with `default_statistics_target = 10000`, and every estimate the script recorded beside a true count matched it. That is not the same as the model being exact, and this page does not bound the difference. Fixture L3 is the only measurement here with two clauses on two columns, and it was deliberately built with *independent* columns, which is the one case where the multiplication in `clauselist_selectivity_ext()` is right by construction ([clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263)). Nothing here exercises a correlated column pair, histogram interpolation rather than a complete MCV list, or an extended-statistics object. The fixture L3 that the first 2026-09-19 filing used did exercise it, by accident: with `c = g % 20` against `a = g % 2000`, `a = 5 AND c = 7` could not match a row and was estimated at 13, an error of 13 rows on a true count of 0. That fixture was replaced on the second pass, so the page no longer carries a measured case in which the assumption fails. A server at the default target additionally samples only 30,000 rows, and the first filing's numbers show what that does: its range-scan and plan-flip costs differed from the ones filed here by between 0.4% and 2.2%. The plan shapes did not change, but no sampled run is filed on this page.
- Seven claims added on the second 2026-09-19 pass are read from source and have no fixture on this page: that a long-lived backend can keep costing from a stale cached metapage, and that two backends can therefore price one index differently; that an `UPDATE` touching only summarizing-index columns leaves an unrelated B-tree untouched; that a partial index whose predicate the new row fails receives no entry; that VACUUM's 2% bypass still runs `amvacuumcleanup`, so it can flush a GIN pending list or recycle B-tree pages while skipping index vacuuming; that a changed `fillfactor` or `deduplicate_items` reaches already-existing pages at their next split; that a serial GIN bitmap index scan can feed a `Parallel Bitmap Heap Scan`; and that a repeated GIN scan amortizes the pending-page I/O charge through `index_pages_fetched()` while leaving the per-page CPU charge unamortized. Each is cited to the pinned source, which `MANDATORY Evidence` treats as primary, but none was reproduced on the measurement server, and the last one in particular would change how [A bloated GIN index loses to a B-tree](#a-bloated-gin-index-loses-to-a-b-tree) reads for a nested-loop inner scan.

## Related Pages

- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](../indexing/reindex-index-concurrently.md) - the online rebuild that resets every input on this page.
- [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](../indexing/partial-indexes-pros-cons.md) - more on the partial-index costing path that behaves differently here.
- [How Bottom-Up Index Deletion and B-Tree Deduplication Work in PostgreSQL 17 (unverified)](../indexing/bottom-up-deletion-and-btree-deduplication.md) - the two mechanisms fixtures N, P and P-100 exercise, including the heap-block budget that decides what a bottom-up pass can free.
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](../../../v12/questions/query-planning/bloated-indexes-query-planner.md) - the same question, and the same GIN-versus-B-tree follow-up, answered against the v12 pin.

## Evidence Map

| Claim | Evidence |
|---|---|
| Planner has no bloat/density/fragmentation field | [pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69), [pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128) |
| `pages` comes from the live block count for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486); measured: forged `relpages = 1` left cost at `24640.42` |
| `tuples` is the parent table's estimate for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486) |
| Partial indexes use `pg_class` density | [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160); measured: forged `reltuples = 20` moved cost `24140.42` -> `23140.49` |
| Page count enters cost pro-rata | [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732); measured `28480.42` vs `123144.43` |
| Height charge exists to stop bloated indexes looking free | [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106) |
| Height charge is `50 * cpu_operator_cost` per level | [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145); measured startup gap exactly `50.00` at `cpu_operator_cost = 1` |
| Planner height is the fast-root level | [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381) |
| Index pages enter cache modeling | [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951), [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747); measured `0.66` against `2.50` per loop for a repeated inner index-only scan |
| The scan's touched-pages estimate, not the index size, chooses parallel workers | [costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772), [selfuncs.c#btcostestimate-outputs](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7206-L7210), [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279); measured 6 / 5 / 5 workers at 100% / 50% / 20% of the 26,411-block index against 4 / 3 / 2 on its 2,745-block twin |
| v17 clamps SAOP descents to `ceil(pages/3)` | [selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7021-L7042); measured plateau at 3 vs 19 |
| Four core AMs and contrib `bloom` use `genericcostestimate`; GIN and BRIN do not | [selfuncs.c:7073](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7073), [selfuncs.c:7221](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7221), [selfuncs.c:7265](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7265), [selfuncs.c:7320](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7320), [blcost.c#blcostestimate](../../../../raw/postgres-17/contrib/bloom/blcost.c#L22-L42); [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L7671), [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8052-L8061) |
| GiST/SP-GiST estimate height from page count | [selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7256-L7308), [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7311-L7363) |
| Hash charges no descent cost | [selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7213-L7253) |
| `avg_leaf_density`/`leaf_fragmentation` ignore deleted pages | [pgstatindex.c#page-classification](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372); measured 89.18% on a 2,465-deleted-page index |
| Fragmentation contributes zero cost | measured gap `1616.00` = `404 * 4.0` with 49.87% vs 0% fragmentation |
| Deleted pages stay in the fork | [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2648), [README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441); measured 2,745 blocks after three VACUUMs; no `RelationTruncate()` or `smgrtruncate()` call exists under `src/backend/access/nbtree/` |
| Split policy sets density ceilings | [nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202) |
| Bloat can drop an index from a `BitmapAnd` | [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287); measured `c = 7` demoted to `Filter` |
| VACUUM can skip index vacuuming | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1949), [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2347) |
| `index_pages_fetched` / `cost_index` unchanged since `REL_12_0` | function text byte-identical between the v12 checkout's `REL_12_0` tag and the pin; in `615cebc94b..HEAD`, `git log -L` finds no commit on `index_pages_fetched` and only a rename and its revert on `cost_index` |
| SAOP clamp is new in v17 | `5bf748b86bc`, first major version 17 by branch-point ancestry; absent from `REL_12_0` `btcostestimate` |
| `50.0` -> macro is cosmetic and v16 | `eb5c4e953bb` diff, first major version 16 by branch-point ancestry; `REL_12_0` already had `* 50.0 *` |
| Deduplication is v13 | `0d861bbb702`, first major version 13 by branch-point ancestry; `deduplicate_items` absent from `REL_12_0` `reloptions.c`; measured 852 vs 2,749 blocks |
| Bottom-up deletion is v14 | `d168b666823`, first major version 14 by branch-point ancestry; [btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703), [README#added-to-postgresql-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988); measured 543 vs 1,173 blocks on a 1,000-value column, and 1,020 vs 1,020 on a 100-value one |
| Newly deleted pages recyclable in the same VACUUM is v14 | `9dd963ae253`, first major version 14 by branch-point ancestry; [README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424) |
| `reltuples = -1` is v14 | `3d351d916b2`, first major version 14 by branch-point ancestry; [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66) |
| No test covers the bloat charge | no `src/test` match for `tree_height`, `btcostestimate`, `genericcostestimate` |
| The `pgstatindex` test uses an empty index | [sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52) |
| All four cost GUCs are session-scoped | [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3686-L3696), [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3719-L3729), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518), [guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3531-L3540) |
| A one-row lookup is charged `ceil(pages / tuples)` pages once pages outnumber rows | [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6717-L6732); measured `12.29` on 2,745 blocks over 1,000 rows, `4.29` after `REINDEX` |
| Page deletion can lower the planner's height without a rebuild | [nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381); measured `tree_level` 2 against `fastlevel` 1, startup `0.42` -> `0.28` |
| The closed form has no standalone `qual_op_cost` term | [selfuncs.c#genericcostestimate-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6789-L6810); `predict()` equals `EXPLAIN` on 7 of 7 whole-index scans |
| A plugin can rewrite `pages`, `tuples` and `tree_height` before costing | [plancat.c#get_relation_info_hook](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L570-L576), [plancat.h#get_relation_info_hook_type](../../../../raw/postgres-17/src/include/optimizer/plancat.h#L20-L25) |
| A predicate-implied clause is dropped from a partial index's quals | [indxpath.c#check_index_predicates-indrestrictinfo](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3318-L3378), [indxpath.c#match_restriction_clauses_to_index](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1968-L1974); measured `500.00` = `200000 * cpu_operator_cost` between the plain and the partial twin |
| `ANALYZE` reads every fixture row at `default_statistics_target = 10000` | [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1876-L1894); two complete runs agree on every recorded value |
| GIN's startup/total split never reaches a plan | [costsize.c#cost_bitmap_heap_scan-startup](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1044-L1048), [ginutil.c:79](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L79) |
| `pgstatindex` rejects every non-B-tree relation | [pgstatindex.c#pgstatindex_impl-btree-check](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228), [sql/pgstattuple.sql#wrong-index-type](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L63) |
| Nothing on the cost path reads the FSM | no file under `src/backend/optimizer/`, and not `selfuncs.c`, includes `freespace.h` or calls `GetRecordedFreeSpace()` |
| GIN is discarded at clause matching when the operator is not in its opfamily | [indxpath.c#match_opclause_to_indexcol-op_in_opfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2433-L2459); core GIN families carry no `<`/`<=`/`>=`/`>` ([pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244), [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611)) |
| `btree_gin` adds strategies 1-5 but is documented not to outperform B-tree | [btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69), [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33) |
| A bare boolean `Var` still matches a GIN bool opclass in v17 | [indxpath.c#IsBooleanOpfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2271-L2286), [indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384), [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98); measured `38.26` for `i`, `i = true` and `i IS TRUE` |
| GIN yields no plain index scan, no pathkeys, no index-only scan, no `IS NULL`, no native array scan, and no partial index path of its own | [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89), [indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751), [indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944), [indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800), [indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129); measured as `disable_cost` sequential scans |
| GIN charges every pending, entry and data page at `random_page_cost` plus `50 * cpu_operator_cost` | [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7976-L7980); measured `12.97` GIN vs `4.52` B-tree on identical statistics, both predicted to the cent |
| The pending list is charged to startup cost in full | [selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7882-L7886); measured 736 pending pages moving the GIN scan from `13.01` to `3141.12` inside a `BitmapAnd` the planner kept, and 1,471 pages moving it to `6264.98` and out of the `BitmapAnd` |
| `gin_clean_pending_list()` drains the list but leaves `nTotalPages` stale | [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091), [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767); only [gininsert.c:406](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L406) and [ginvacuum.c:789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L789) call `ginUpdateStats()`; measured 2,073 live blocks against a metapage still reading 302 |
| Stats older than 4X growth are replaced by invented ones | [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7713-L7767); measured 1,369 live blocks against a metapage reading 2, charged 4 pages |
| A keyless partial GIN path is priced as a whole-index scan | [selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7852-L7877), [indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897); measured `1001.00` charged pages on a 10-block index |
| GIN rejects unique, `INCLUDE`, exclusion and `CLUSTER` | [indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879), [cluster.c#cluster_rel-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522); all four messages reproduced |
| A multicolumn GIN can beat a `BitmapAnd` of GIN + B-tree | [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33); measured `21.51` versus `240.13` |
| No test compares GIN and B-tree plan choice, and none covers `gincostestimate` | no `src/test` match for `gincostestimate`; `btree_gin` tests use `EXPLAIN (COSTS OFF)` ([bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9)) |
| The planner's tree height can be a stale per-backend cached copy of the metapage | [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717) caches `BTMetaPageData` in `rel->rd_amcache` and its comment declines to refresh it; freed only on relcache invalidation ([relcache.c#RelationInvalidateRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2534-L2557)) |
| `leaf_fragmentation` counts backward sibling links only, not physical adjacency | [pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323) |
| An `UPDATE` of only summarizing-index columns adds no entry to a B-tree | [heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4161), [heapam.c#update_indexes](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4415-L4429), [tableam.h#TU_UpdateIndexes](../../../../raw/postgres-17/src/include/access/tableam.h#L113-L127), [nodeModifyTable.c#ExecUpdateEpilogue-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L2162-L2166), [execIndexing.c#ExecInsertIndexTuples-onlySummarizing](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L361-L366) |
| A partial index whose predicate the new row fails gets no entry | [execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L387) |
| The 2% bypass skips index vacuuming but still runs index cleanup | [vacuumlazy.c#lazy_vacuum-bypass-branch](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1949) clears only `do_index_vacuuming`; [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L893) can still scan and recycle; [ginvacuum.c#ginvacuumcleanup-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L719-L729) flushes the pending list |
| The wraparound failsafe clears `do_index_cleanup` too, so it does stop cleanup | [vacuumlazy.c#lazy_check_wraparound_failsafe-clears](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2326) |
| `fillfactor` and `deduplicate_items` reach existing pages at the next split, without a rebuild | [nbtsplitloc.c#leaffillfactor](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L170-L176), [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785), [nbtsort.c#allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L560-L564) |
| `IndexOptInfo` is not the whole size input for every AM: GIN and BRIN read metapage counters as well | [selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7697-L7711), [selfuncs.c#brincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8086-L8101) |
| `FormData_pg_class` is cpp output of the `CATALOG()` macro, not a `genbki.pl` product | [genbki.h:23](../../../../raw/postgres-17/src/include/catalog/genbki.h#L23), [pg_class.h#CATALOG](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L32), [pg_class.h#NOTES](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L11-L16) |
| A serial GIN bitmap index scan can feed a `Parallel Bitmap Heap Scan` | [indxpath.c:347](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L347), [allpaths.c#create_partial_bitmap_paths](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4162-L4185) |
| On a repeated GIN scan the pending-page I/O charge is amortized and the per-page CPU charge is not | [selfuncs.c#gincostestimate-cache-effects](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7957-L7974), [selfuncs.c:7980](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7980), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7937-L7955) |
| `gin_pending_list_limit` triggers a non-forced cleanup after the insert, and is not a ceiling | [ginfast.c#needCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), [ginfast.c#ginInsertCleanup-conditional-lock](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828), [gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529) |
| `fastupdate = off` does not flush the entries already in the pending list | [ref/create_index.sgml#fastupdate-note](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L524-L532) |
| `enable_bitmapscan = off` adds `disable_cost` and does not remove the bitmap path | [costsize.c#cost_bitmap_heap_scan-disable](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L1041-L1042) |
| Gate 3 is a comparison of computed costs, not a rule that GIN loses | [indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1357), [pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L453); measured: a multicolumn GIN at `21.51` beating a `BitmapAnd` at `240.13` |
| Exhaustive sampling removes sampling noise; the independence assumption survives it | [clausesel.c#clauselist_selectivity_ext-multiply](../../../../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L253-L263); measured on fixture L3, whose independent columns make the product exact: 250, 24,971 and 12 estimated against 250, 24,971 and 12 actual. No fixture here violates the assumption |

## Context Reviewed

Planner and cost path: `src/backend/optimizer/util/plancat.c` (`get_relation_info`, `estimate_rel_size`), `src/backend/optimizer/path/costsize.c` (`cost_index`, `index_pages_fetched`, `cost_bitmap_heap_scan`, `compute_bitmap_pages`, `get_indexpath_pages`), `src/backend/optimizer/path/allpaths.c` (`total_table_pages`, `compute_parallel_worker`), `src/backend/optimizer/path/indxpath.c` (`choose_bitmap_and`, `bitmap_scan_cost_est`), `src/backend/optimizer/util/pathnode.c` (`compare_path_costs_fuzzily`), `src/backend/utils/adt/selfuncs.c` (`genericcostestimate`, `btcostestimate`, `hashcostestimate`, `gistcostestimate`, `spgcostestimate`, `gincostestimate`, `brincostestimate`, `add_predicate_to_index_quals`).

nbtree: `nbtpage.c` (`_bt_getrootheight`, fast-root update, page deletion), `nbtsplitloc.c` (`_bt_findsplitloc` fillfactor policy and single-value strategy), `nbtdedup.c` (`_bt_dedup_pass`, `_bt_bottomupdel_pass`), `nbtinsert.c` (pre-split deletion and deduplication), `nbtsort.c`, `src/include/access/nbtree.h`, and the nbtree `README` sections on page deletion, tree height, FSM placement, simple deletion, bottom-up deletion, split policy, and deduplication.

VACUUM: `src/backend/access/heap/vacuumlazy.c` (`BYPASS_THRESHOLD_PAGES`, `lazy_vacuum`, `lazy_check_wraparound_failsafe`, `LVRelState`).

Headers and catalogs: `src/include/nodes/pathnodes.h` (`IndexOptInfo`, `RelOptInfo`, `PlannerInfo`), `src/include/catalog/pg_class.h`, `src/backend/access/common/reloptions.c`, `src/backend/utils/misc/guc_tables.c`.

Contrib: `contrib/pgstattuple/pgstatindex.c` plus its `sql/` and `expected/` regression files, `contrib/pageinspect` `bt_metap()` definitions.

Documentation: `ref/reindex.sgml`, `maintenance.sgml` (routine reindexing), `glossary.sgml` (Bloat), `btree.sgml` (version churn, bottom-up deletion, deduplication), `ref/create_table.sgml` (`vacuum_index_cleanup`), `ref/create_index.sgml`, `indices.sgml`, `pgstattuple.sgml`.

History: `git log -L` on `genericcostestimate`, `btcostestimate`, `gincostestimate`, `index_pages_fetched` and `cost_index` over `615cebc94b..HEAD`, where `615cebc94b` is the `Stamp HEAD as 13devel.` commit that marks the v12 branch point; `git merge-base --is-ancestor` of each attributed commit against the five `Stamp HEAD as NNdevel.` commits and the twelve `Stamp 17.N.` commits; `git show -s` for every subject and date. The v17 checkout carries no tags, so `git tag --contains` and `git describe` cannot run in it, and the first filing's use of them was replaced by this method; every attribution it made survived. `REL_12_0` function text for `index_pages_fetched`, `cost_index`, `genericcostestimate`, `btcostestimate`, `gincostestimate`, the `get_relation_info()` index-size block, `estimate_rel_size()` and `reloptions.c` was read from the v12 checkout's `REL_12_0` tag. Also `contrib/pgstattuple` and `src/backend/access/nbtree` history in the same range, and the diff of the cited files between the two pins, `54eeefaedbee..786db8dcf168`.

Empirical: one isolated PostgreSQL 17.11 server built from the pin by the script under [Measurement Script](#measurement-script), whose B-tree stages cover density, real deletion bloat, a mostly-empty index whose fast root moved, height, seeded fragmentation, catalog forgery, plan flips, `BitmapAnd` pruning, parallel worker counts at three selectivities, a repeated inner scan under two cache settings, the v17 SAOP clamp, version churn at two key cardinalities with and without a held snapshot, deduplication, and a closed-form prediction checked against `EXPLAIN`.

Pin: `raw/postgres-17/` at commit `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11, seven commits past `Stamp 17.11.` `083ac03341`); repinned from `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on 2026-08-17. Every measured number on the page was taken on this pin on 2026-09-19; what the 17.10 filing reported, and which of its numbers reproduced, is in [Fixtures and method](#fixtures-and-method).

Follow-up (GIN versus B-tree) additions. Path generation and gating: `src/backend/optimizer/path/indxpath.c` (`create_index_paths`, `match_restriction_clauses_to_index`, `match_clause_to_indexcol`, `match_opclause_to_indexcol`, `match_saopclause_to_indexcol`, `match_boolean_index_clause`, `IsBooleanOpfamily`, `get_index_clause_from_support`, `get_index_paths`, `build_index_paths`, `check_index_only`, `choose_bitmap_and`, `bitmap_and_cost_est`), `src/backend/optimizer/util/plancat.c` (the AM capability-flag copy and the `sortopfamily` branches), `src/backend/optimizer/util/pathnode.c` (`add_path`, `STD_FUZZ_FACTOR`), `src/backend/access/index/indexam.c` (`index_can_return`), `src/backend/optimizer/path/costsize.c` (`disable_cost`).

GIN internals: `src/backend/utils/adt/selfuncs.c` (`gincostestimate`, `gincost_pattern`, `gincost_opexpr`, `gincost_scalararrayopexpr`, `GinQualCounts`), `src/backend/access/gin/ginutil.c` (`ginhandler`, `ginGetStats`, `ginUpdateStats`), `src/backend/access/gin/ginfast.c` (`gin_clean_pending_list`), `src/include/access/gin.h` (`GinStatsData`, `GIN_SEARCH_MODE_*`), `src/include/access/ginblock.h` (`GinMetaPageData`).

Catalogs, errors and settings: `src/include/catalog/pg_amop.dat` (the four core GIN opfamilies plus the B-tree and hash `jsonb` families), `src/include/catalog/pg_opfamily.h` (`IsBuiltinBooleanOpfamily`), `src/backend/commands/indexcmds.c` (unique/`INCLUDE`/multicolumn/exclusion AM checks), `src/backend/commands/cluster.c` (`amclusterable`), `src/backend/access/common/reloptions.c` (`fastupdate`, per-index `gin_pending_list_limit`), `src/backend/utils/misc/guc_tables.c` (`gin_pending_list_limit`).

Contrib, tests and docs: `contrib/btree_gin` (`btree_gin--1.0.sql` operator classes, `sql/bool.sql`, `expected/bool.out`), `contrib/pgstattuple/pgstatindex.c` (`pgstatginindex`), `src/test/regress/expected/amutils.out`, `src/test/regress/sql/create_index.sql`, `src/test/regress/sql/tsearch.sql`, `doc/src/sgml/indices.sgml`, `doc/src/sgml/indexam.sgml`, `doc/src/sgml/gin.sgml`, `doc/src/sgml/btree-gin.sgml`, `doc/src/sgml/pgtrgm.sgml`.

Follow-up history: `git log -L` on `gincostestimate` over `615cebc94b..HEAD` returned exactly two commits, `cd9479af2af` and `4b754d6c16e`, whose first major versions by branch-point ancestry are 16 and 13. The follow-up describes only v17 behavior and makes no cross-version claim.

Follow-up empirical: the GIN stages of the same script on the same server, with `btree_gin`, `pgstattuple` and `pageinspect` installed, covering same-column GIN-versus-B-tree costing on identical statistics, closed-form reconciliation of both cost models, all six plan-shape cases, `fastupdate` pending-list bloat at two sizes with the B-tree-only alternative priced beside it, the drain and the following `VACUUM`, the 4X stale-metapage fallback, the keyless partial-index path, the boolean-column case, the live AM property matrix, the four `CREATE INDEX`/`CLUSTER` rejections, a multicolumn-GIN comparison, and verbatim execution of the two filed diagnostic blocks as read out of this page. The `clean` stage stopped the server and deleted the sandbox.

Review, 2026-09-19, second pass: fourteen reported defects were each checked against the pin and each confirmed, then fixed. Files read for the first time on this page: `src/backend/optimizer/path/clausesel.c` (`clauselist_selectivity_ext`), `src/backend/utils/cache/relcache.c` (`RelationInvalidateRelation`, the `rd_amcache` free sites), `src/backend/access/heap/heapam.c` (`heap_update`'s HOT and summarizing decision and its `update_indexes` result), `src/include/access/tableam.h` (`TU_UpdateIndexes`), `src/backend/executor/nodeModifyTable.c` (`ExecUpdateEpilogue`) and `src/backend/executor/execIndexing.c` (`ExecInsertIndexTuples`), `src/backend/access/nbtree/nbtree.c` (`btvacuumcleanup`), `src/backend/access/gin/ginvacuum.c` (`ginvacuumcleanup`), `src/backend/access/gin/ginfast.c` (`ginHeapTupleFastInsert`'s `needCleanup` test and `ginInsertCleanup`'s conditional lock), `src/backend/access/nbtree/nbtsort.c` (`_bt_allequalimage` at build), `src/include/catalog/genbki.h` (the `CATALOG()` macro), and `src/backend/utils/adt/selfuncs.c` (`brincostestimate`'s `brinGetStats` read and `gincostestimate`'s cache-effect block). `nbtsplitloc.c`'s `BTGetFillFactor` read, `costsize.c`'s `enable_bitmapscan` test, `allpaths.c`'s `create_partial_bitmap_paths` and its call site in `indxpath.c`, `pgstatindex.c`'s `fragments` counter, and `ref/create_index.sgml`'s `fastupdate` note were read for the first time as evidence. The script gained failure propagation, an exit trap and a `need_server` step, and fixture L3 was rebuilt on two independent columns; every number was re-measured from the edited script. No common concept page was created, edited, or needed.

Review, 2026-09-19, first pass: all 340 citations of the previous text (177 ranges in 46 files) were re-read against the pin; all were in bounds and all came from `raw/postgres-17/`. Six ranges were tightened or extended, one label pointed at a legacy entry point, and five behavioral claims had no citation. Files read for the first time in this pass: `contrib/bloom/blcost.c`, `src/include/optimizer/plancat.h` and the `get_relation_info_hook` call in `plancat.c`, `src/include/utils/selfuncs.h` (`GenericCosts`), `src/backend/commands/analyze.c` (`std_typanalyze`), `indxpath.c` (`check_index_predicates`, `match_restriction_clauses_to_index`), `costsize.c` (`cost_bitmap_heap_scan`), `gininsert.c` and `ginvacuum.c` (`ginUpdateStats()` callers), and `contrib/pgstattuple/pgstattuple--1.4--1.5.sql`. Common concepts: the three v17 concept pages are bloat-test protocols for estimator pages; none defines a concept this page explains, so none is linked, and none was edited.

## Source References

- [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)
- [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)
- [selfuncs.c#genericcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6626-L6828)
- [selfuncs.c#btcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6870-L7211)
- [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)
- [costsize.c#cost_index](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L621)
- [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L859-L951)
- [costsize.c#compute_bitmap_pages](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6420-L6518)
- [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)
- [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287)
- [pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)
- [pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)
- [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)
- [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659)
- [nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334)
- [nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L307-L320)
- [nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)
- [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)
- [README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)
- [README#bottom-up-deletion](../../../../raw/postgres-17/src/backend/access/nbtree/README#L557-L619)
- [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1949)
- [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)
- [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372)
- [ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1054)
- [glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)
- [btree.sgml#bottom-up-deletion](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L733)
- [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)
- [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7662-L8050)
- [selfuncs.c#gincost_pattern](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7380-L7492)
- [selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7370-L7378)
- [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89)
- [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)
- [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1091)
- [gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L40-L50)
- [ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101)
- [indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L310)
- [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767)
- [indxpath.c#build_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L804-L1003)
- [indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800)
- [indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269)
- [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2386-L2500)
- [indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384)
- [indexam.c#index_can_return](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L780-L797)
- [pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L453)
- [costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130)
- [pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244)
- [pg_opfamily.h#IsBuiltinBooleanOpfamily](../../../../raw/postgres-17/src/include/catalog/pg_opfamily.h#L59-L65)
- [indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879)
- [cluster.c#cluster_rel-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522)
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
- [relcache.c#RelationInvalidateRelation](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L2534-L2557)
- [heapam.c#heap_update-hot-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4140-L4161)
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
- [indxpath.c#choose_bitmap_and-cheapest-of-group](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1353-L1357)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [log](../../../log.md)
