---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, what has more impact on index scan I/O: B-tree leaf density or fragmentation? Provide a comprehensive comparison, analyze different levels of density and fragmentation, and estimate the impact on index scan I/O.

## Answer Up Front

Leaf density usually has the more predictable impact on PostgreSQL-visible index scan I/O, because it changes the number of leaf pages a scan must visit and the planner's estimated index page count. Fragmentation can have the larger wall-clock impact on cold, seek-sensitive storage, but PostgreSQL 12 does not cost `leaf_fragmentation` directly and `EXPLAIN (ANALYZE, BUFFERS)` does not report physical-order disorder as a separate counter ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).

For a fixed live tuple count and fixed average index tuple size, the leaf-page multiplier from density is approximately:

```text
leaf_page_multiplier = baseline_density / observed_density
```

So, compared with a 90% dense B-tree, a 60% dense B-tree needs about `90 / 60 = 1.5x` as many leaf pages for the same live index tuples. That extra page count feeds planner costing through `IndexOptInfo.pages` and `genericcostestimate`, and it feeds executor work because `_bt_next` steps through leaf pages with `_bt_steppage` and `_bt_readnextpage` after `_bt_first` positions the scan ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800)).

`leaf_fragmentation` is different. In `pgstatindex`, it is the percentage of live leaf pages whose logical right link points to a lower physical block number. It measures physical-order reversals in the leaf chain, not free space, tuple count, jump distance, run length, or read latency ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)). A fragmented index can still visit the same number of leaf pages as an unfragmented index with the same density; the difference is that logical neighbors are less likely to be physical neighbors. The PostgreSQL 12 manual explicitly says updated B-tree indexes can be slower than freshly built ones because logically adjacent pages are usually physically adjacent in a newly built index ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889)).

Practical ranking:

| Workload shape | More important factor | Why |
|---|---|---|
| Point lookup on a unique or highly selective key | Usually neither, unless density bloat raises tree height | The scan positions to one leaf page; `btcostestimate` adds a small height-based descent charge so bloated indexes do not look free for one-leaf probes. |
| Warm-cache range scan | Usually density | Same logical range touches more leaf buffers when density is lower; physical order has little storage I/O penalty when pages are already cached. |
| Cold large range scan on SSD or storage with low random-read penalty | Usually density, sometimes tied | Density changes page count; fragmentation changes order, but low random-read penalty reduces the order cost. |
| Cold large range scan on mechanical or seek-sensitive storage | Fragmentation can dominate latency | The same leaf count may become many nonsequential fetches; v12 docs say random access on mechanical storage is normally much more expensive than sequential access. |
| Full index scan or index-only `COUNT(*)` style scan | Density determines page count; fragmentation determines locality | `_bt_endpoint` starts from an end when there is no usable boundary key, and `_bt_next` walks the leaf chain page by page. |

The ranking above depends on the same executor path and planner constants: v12 documents `seq_page_cost` as the cost of a sequential page fetch and `random_page_cost` as the cost of a nonsequential page fetch, with defaults 1.0 and 4.0; it also says setting them equal is sensible for fully cached data and lowering `random_page_cost` can model SSDs ([config.sgml#planner-cost-constants](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4670-L4765), [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L20-L35)).

## What PostgreSQL 12 Measures

`pgstatindex` computes `avg_leaf_density` by summing `PageGetFreeSpace(page)` and available leaf capacity over live leaf pages, then returning `100 - free_space / max_avail * 100`. `PageGetFreeSpace` subtracts one line-pointer slot when possible, so the value is not the raw `pd_upper - pd_lower` gap ([pgstatindex.c#leaf-density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L356), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L581-L597)).

`pgstatindex` computes `leaf_fragmentation` as `fragments / leaf_pages * 100`, where `fragments` increments only when a live leaf page's `btpo_next` is not `P_NONE` and points to an earlier physical block. This is a one-bit-per-leaf-page physical-order test, not a cost model ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)).

The default B-tree leaf fillfactor is 90%, while non-leaf pages use a fixed 70% fillfactor. The v12 header says the leaf fillfactor is applied during index build and rightmost-page splits; non-rightmost splits generally divide data equally, and single-value duplicate splits use an effective 96% fillfactor ([nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171), [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330)).

## Where Index Scan I/O Comes From

For a plain B-tree scan, `_bt_first` preprocesses scan keys, descends with `_bt_search`, positions on the first leaf page with `_bt_binsrch`, and loads matching items from that page with `_bt_readpage`. Later calls to `_bt_next` return saved items from the current page until the page is exhausted, then call `_bt_steppage` ([nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381)).

For forward scans, `_bt_steppage` uses the saved `nextPage` right link, and `_bt_readnextpage` reads each candidate leaf page with `_bt_getbuf`. `_bt_readpage` then checks tuples against the scan keys and stores matching TIDs in the backend-local scan position array ([nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [nbtsearch.c#_bt_readpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1383-L1616)).

For backward scans, `_bt_readnextpage` uses `_bt_walk_left`, which is more complex because it must handle concurrent splits and deletion cases while walking the left-link direction ([nbtsearch.c#backward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1813-L1905), [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2007)).

`EXPLAIN (ANALYZE, BUFFERS)` prints node-level shared, local, and temp hit/read/dirtied/written counters from `Instrumentation.bufusage`. It does not split a node's buffers into index pages versus heap pages, and it does not label sequential versus nonsequential reads ([explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978), [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L24-L45)).

## Density Impact Estimates

These estimates assume the same live index tuple count, the same average tuple width, the same visibility state, and a scan wide enough that leaf-page walking dominates one-time descent. They estimate leaf pages visited, not heap fetches.

| Average leaf density | Leaf pages versus 90% baseline | Extra leaf page visits | PostgreSQL-visible I/O effect |
|---:|---:|---:|---|
| 95% | 0.95x | 5% fewer | Possible after compact data or duplicate-heavy effective fill behavior; fewer leaf buffers than 90% for the same live tuple volume. |
| 90% | 1.00x | Baseline | Default fresh-build leaf target. |
| 80% | 1.13x | 12.5% more | Small but visible for wide scans. |
| 70% | 1.29x | 28.6% more | Often enough to change large range-scan buffer counts. |
| 60% | 1.50x | 50.0% more | Strong direct impact on range scans and full index scans. |
| 50% | 1.80x | 80.0% more | Bloat-sized effect; more planner cost and more executor leaf-page steps. |
| 40% | 2.25x | 125.0% more | More than doubles leaf-page work for the same logical key volume. |
| 30% | 3.00x | 200.0% more | Severe density loss; index-side page count can dominate. |

The planner path matches this page-count model. For ordinary non-partial indexes, `get_relation_info` sets `IndexOptInfo.pages` from `RelationGetNumberOfBlocks(indexRelation)` and sets `IndexOptInfo.tuples` from the parent relation estimate; for partial indexes it calls `estimate_rel_size`, which still reports current blocks as pages. `genericcostestimate` then computes `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)` when both page and tuple counts are greater than one ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

For a one-leaf point probe, density often does not change the number of leaf pages visited. PostgreSQL 12 still adds an explicit B-tree descent CPU charge of `(tree_height + 1) * 50.0 * cpu_operator_cost`, and the comment says this prevents bloated indexes from appearing to have the same search cost as unbloated ones when only one leaf page is expected ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417)).

## Fragmentation Impact Estimates

These estimates assume the same density and the same logical key range. Under those assumptions, fragmentation does not add leaf pages by itself. It changes the physical order of the page numbers read while following the logical leaf chain.

Let:

```text
L = leaf pages visited by the logical scan
F = leaf_fragmentation / 100
S = cost of a sequential page fetch
R = cost of a nonsequential page fetch
```

`pgstatindex`'s `F` implies about `F * L` right-link steps where the next logical leaf page has a lower physical block number than the current page. It does not tell how far those jumps go, whether later pages form short or long runs, or whether the operating system or storage device can prefetch them ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).

A simple storage-order model is:

```text
order_cost_multiplier = 1 + F * (R / S - 1)
```

This is not PostgreSQL 12 planner output. It is a sensitivity model based on v12's documented distinction between sequential and nonsequential page-fetch costs. The docs define `seq_page_cost` as the cost of a sequential disk page fetch and `random_page_cost` as the cost of a nonsequential disk page fetch, with defaults of 1.0 and 4.0 ([config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765), [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L20-L35)).

Using the default `R / S = 4` as a cold, nonsequential-sensitive model:

| `leaf_fragmentation` | Approx. backward right-link steps in an `L`-page forward scan | Order-cost multiplier at `R / S = 4` | PostgreSQL buffer counter expectation |
|---:|---:|---:|---|
| 0% | `0` | 1.00x | Same leaf-page count as density predicts; best physical locality. |
| 10% | `0.10 * L` | 1.30x | Same distinct leaf pages, but some order breaks. |
| 25% | `0.25 * L` | 1.75x | Same distinct leaf pages; cold storage latency can become visible. |
| 50% | `0.50 * L` | 2.50x | Same distinct leaf pages; order cost can exceed a 60%-vs-90% density penalty in cold-storage latency. |
| 75% | `0.75 * L` | 3.25x | Same distinct leaf pages; physical-order cost can dominate elapsed time. |
| 100% | `1.00 * L` | 4.00x | Worst-case by this model, though the metric still lacks jump-distance and run-length information. |

When pages are already cached, the docs say setting `random_page_cost` equal to `seq_page_cost` makes sense because there is no penalty for touching pages out of sequence. Under `R / S = 1`, the order-cost multiplier is 1.00x at every fragmentation level, so density dominates the PostgreSQL-visible buffer work ([config.sgml#cached-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4748-L4765)).

For SSD-like storage, the docs say a lower `random_page_cost` can model low random-read cost relative to sequential reads. For example, if `R / S = 1.2`, the same model gives `1 + F * 0.2`; 50% fragmentation is only a 1.10x order penalty, which is smaller than the 1.50x density penalty from 60% versus 90% leaf density ([config.sgml#ssd-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4748-L4755)).

## Combined Estimate Matrix

The combined model for a large cold forward range scan is:

```text
combined_multiplier = (0.90 / density) * (1 + fragmentation * (R / S - 1))
```

Here `density` and `fragmentation` are fractions, such as `0.60` and `0.50`. The first term estimates extra leaf pages. The second term estimates physical-order penalty. PostgreSQL 12 directly models the first term through physical index pages, but not the second term through `leaf_fragmentation` ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307)).

At `R / S = 4`:

| Density | Fragmentation 0% | Fragmentation 25% | Fragmentation 50% | Fragmentation 75% |
|---:|---:|---:|---:|---:|
| 90% | 1.00x | 1.75x | 2.50x | 3.25x |
| 80% | 1.13x | 1.97x | 2.81x | 3.66x |
| 70% | 1.29x | 2.25x | 3.21x | 4.18x |
| 60% | 1.50x | 2.63x | 3.75x | 4.88x |
| 50% | 1.80x | 3.15x | 4.50x | 5.85x |
| 40% | 2.25x | 3.94x | 5.63x | 7.31x |

At `R / S = 1.2`, which represents much lower random-read penalty:

| Density | Fragmentation 0% | Fragmentation 25% | Fragmentation 50% | Fragmentation 75% |
|---:|---:|---:|---:|---:|
| 90% | 1.00x | 1.05x | 1.10x | 1.15x |
| 80% | 1.13x | 1.18x | 1.24x | 1.29x |
| 70% | 1.29x | 1.35x | 1.41x | 1.48x |
| 60% | 1.50x | 1.58x | 1.65x | 1.72x |
| 50% | 1.80x | 1.89x | 1.98x | 2.07x |
| 40% | 2.25x | 2.36x | 2.48x | 2.59x |

These tables estimate index-side leaf-page I/O pressure only. They do not estimate heap reads, visibility-map effects in index-only scans, CPU spent evaluating scan keys, kernel read-ahead behavior, controller caches, concurrent buffer churn, or tuple visibility checks outside the B-tree access method.

## How Fragmentation Arises

During retail insertion, `_bt_insertonpg` splits a page when `PageGetFreeSpace(page) < itemsz`. `_bt_split` obtains a new right page with `_bt_getbuf(rel, P_NEW, BT_WRITE)`, then updates the left page's `btpo_next` to the new page and sets the new page's `btpo_prev` and `btpo_next` fields ([nbtinsert.c#split-needed](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L945-L1005), [nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1408-L1437)).

When `_bt_getbuf` is asked for `P_NEW`, it first asks `GetFreeIndexPage` for a reusable page. If it finds a recyclable page, it reinitializes and returns that block; if not, it extends the relation by one page with `ReadBuffer(rel, P_NEW)`. Reuse can place a new logical neighbor at an older physical block, while extension places it at the end of the relation ([nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L744-L850)).

VACUUM can record recyclable index pages with `RecordFreeIndexPage` when the B-tree page is recyclable. That supplies the free-page source that later `P_NEW` allocations may reuse ([nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1120-L1180), [nbtpage.c#_bt_getbuf-free-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L790-L825)).

The manual's operational summary matches those mechanics: B-tree pages that become completely empty are reclaimed for reuse, partly empty pages can remain allocated and waste space, and freshly constructed B-tree indexes are slightly faster because logical neighbors are usually physical neighbors ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)).

## Which One Matters More?

Density matters more when the question is "how many index leaf buffers will PostgreSQL touch?" For wide scans, the executor walks more leaf pages when density is lower, and the planner estimates more index pages when the physical index is larger ([nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

Fragmentation matters more when the question is "how expensive are those page reads on this storage path?" The executor still follows the logical leaf chain, but `leaf_fragmentation` says some right-link steps go backward in physical block order. PostgreSQL 12 documents nonsequential page fetches as more expensive than sequential ones, especially for mechanical storage, while also documenting that fully cached data has no out-of-sequence page penalty ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [config.sgml#random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4765)).

Point lookups usually hide both effects. A unique equality lookup usually pays one descent and one leaf-page visit; density affects it mainly if bloat increases tree height, and fragmentation has little opportunity to matter because the scan is not walking many leaf links ([nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).

Full index scans expose both effects. When `_bt_first` has no usable boundary keys, `_bt_endpoint` starts from the first or last leaf page, and later calls to `_bt_next` continue through the leaf chain. Density controls how many pages are in that chain for the same live tuple volume; fragmentation controls how physically ordered that chain is ([nbtsearch.c#endpoint-start](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L990), [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2137-L2228), [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307)).

Bitmap index scans use the B-tree access method's `btgetbitmap`, which loops with `_bt_first` and `_bt_next` while adding TIDs to the bitmap. Density and fragmentation affect the index-side scan path similarly, although the heap page access pattern after bitmap creation is a separate executor concern ([nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L340), [nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381)).

Index-only scans still walk the same index pages through `index_getnext_tid`; they avoid heap fetches only when the visibility map says the heap page is all-visible. That means density and fragmentation affect index-side I/O even when heap I/O is avoided ([indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L502-L545), [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)).

## Operational Reading

For index scan I/O visible in PostgreSQL counters, compare `leaf_pages`, `index_size`, and `avg_leaf_density` first. A fall from 90% to 60% density implies roughly 50% more leaf-page visits for broad scans, and that can appear as more shared buffer hits or reads in the scan node depending on cache state ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).

For elapsed I/O latency on cold scans, inspect `leaf_fragmentation` next. A 50% fragmented index does not necessarily read 50% more buffers, but it can turn many logical next-page steps into nonsequential block accesses. The severity depends on cache residency, storage random-read cost, read-ahead behavior, and the fragmented run structure that `leaf_fragmentation` does not expose ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [config.sgml#random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4765)).

For maintenance decisions, PostgreSQL 12 source and docs do not define a universal threshold such as "reindex at 60% density" or "reindex at 40% fragmentation". The docs recommend `REINDEX` for B-tree indexes with many empty or nearly-empty pages and note that rebuilding can improve access speed by restoring physical adjacency ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)).

## Tests and Coverage

The v12 `pgstattuple` regression test checks empty B-tree `pgstatindex` output and error paths for unsupported relation kinds and access methods, but it does not create populated B-trees with non-`NaN` density or nonzero fragmentation assertions ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236)).

The core `btree_index` regression test creates a deliberately tall B-tree with `fillfactor = 10` and later covers page deletion and FSM page recycling, but it does not compare planner or executor I/O at different `avg_leaf_density` and `leaf_fragmentation` levels ([btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L119-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L161)).

## Open Questions

- `leaf_fragmentation` does not encode jump distance, fragmented-run length, or operating-system read-ahead behavior, so the order-cost tables are sensitivity estimates rather than source-measured PostgreSQL behavior ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307)).
- `EXPLAIN (ANALYZE, BUFFERS)` cannot separate index-relation buffers from heap-relation buffers inside a node, and it cannot distinguish sequential from nonsequential index page reads ([explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).
- The breakeven point where fragmentation dominates density depends on cache residency, storage hardware, kernel and storage read-ahead, tablespace cost settings, and concurrent buffer churn; PostgreSQL 12 does not derive that point from `pgstatindex` output ([config.sgml#planner-cost-constants](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4670-L4765), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

## Related Pages

- [v12/index](../index.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](explain-analyze-buffers-output.md)
- [versions](../../versions.md)

## Evidence Map

| Claim | Evidence |
|---|---|
| `avg_leaf_density` is computed from leaf free space and available capacity | [pgstatindex.c#leaf-density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L356), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L581-L597) |
| `leaf_fragmentation` counts live leaf pages whose right link points backward in physical block order | [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68) |
| Default leaf fillfactor is 90%, and split behavior differs for rightmost, non-rightmost, and duplicate-heavy pages | [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171), [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330) |
| Planner B-tree I/O costing uses physical index pages and tuple estimates, not `leaf_fragmentation` | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L754-L877) |
| B-tree scans position once and then walk leaf pages through `_bt_next`, `_bt_steppage`, `_bt_readnextpage`, and `_bt_getbuf` | [nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L744-L850) |
| Fresh B-tree indexes are usually physically ordered better than indexes updated many times | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889) |
| v12 cost constants distinguish sequential and nonsequential page fetches | [config.sgml#planner-cost-constants](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4670-L4765), [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L20-L35), [indexam.sgml#index-costs](../../../raw/postgres-12/doc/src/sgml/indexam.sgml#L1276-L1290) |
| Page splits allocate a new right page through `_bt_getbuf(P_NEW)`, which may reuse FSM pages or extend the relation | [nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1408-L1437), [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L744-L850), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1120-L1180) |
| `EXPLAIN BUFFERS` reports node-level buffer counters, not relation-kind or physical-order counters | [explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978), [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L24-L45) |
| Existing tests do not compare density and fragmentation I/O levels | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L119-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L161) |

## Context Reviewed

- Required wiki navigation: [versions](../../versions.md), [wiki index](../../index.md), [v12/index](../index.md), and recent [log](../../log.md).
- PostgreSQL 12 diagnostic sources: `contrib/pgstattuple/pgstatindex.c`, `doc/src/sgml/pgstattuple.sgml`, `src/backend/storage/page/bufpage.c`, and `src/include/access/nbtree.h`.
- PostgreSQL 12 B-tree scan and maintenance sources: `nbtsearch.c`, `nbtinsert.c`, `nbtpage.c`, `nbtree.c`, `nbtsplitloc.c`, and `nbtsort.c`.
- PostgreSQL 12 planner and costing sources: `plancat.c`, `selfuncs.c`, `costsize.c`, `cost.h`, `config.sgml`, and `indexam.sgml`.
- PostgreSQL 12 measurement sources: `explain.c`, `instrument.h`, and `bufmgr.c`.
- Same-checkout docs and tests: `maintenance.sgml`, `ref/reindex.sgml`, `contrib/pgstattuple/sql/pgstattuple.sql`, `contrib/pgstattuple/expected/pgstattuple.out`, and `src/test/regress/sql/btree_index.sql`.

## Source References

- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)
- [pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L189-L268)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L581-L597)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)
- [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)
- [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330)
- [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L746-L1328)
- [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381)
- [nbtsearch.c#_bt_readpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1383-L1616)
- [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724)
- [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1726-L1905)
- [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2007)
- [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2137-L2228)
- [nbtinsert.c#_bt_insertonpg](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L945-L1005)
- [nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1408-L1437)
- [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L744-L850)
- [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1120-L1180)
- [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L340)
- [indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L502-L545)
- [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)
- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417)
- [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)
- [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)
- [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L754-L877)
- [config.sgml#planner-cost-constants](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4670-L4765)
- [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L20-L35)
- [indexam.sgml#index-costs](../../../raw/postgres-12/doc/src/sgml/indexam.sgml#L1276-L1290)
- [explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866)
- [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)
- [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L24-L45)
- [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889)
- [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)
- [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236)
- [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L119-L123)
- [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L161)