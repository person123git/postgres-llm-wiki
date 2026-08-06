---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: claude-opus-5-max 2026-07-29T19:54:40Z
---

# B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [What PostgreSQL 12 measures](#what-postgresql-12-measures)
  - [Where index scan reads come from](#where-index-scan-reads-come-from)
  - [Density impact estimates](#density-impact-estimates)
  - [Fragmentation impact estimates](#fragmentation-impact-estimates)
  - [Combined estimate matrix](#combined-estimate-matrix)
  - [Exact-pin measurements](#exact-pin-measurements)
  - [How fragmentation arises](#how-fragmentation-arises)
  - [What both metrics hide](#what-both-metrics-hide)
  - [Which one matters more](#which-one-matters-more)
  - [Settings and apply scope](#settings-and-apply-scope)
  - [Operational reading](#operational-reading)
  - [Tests and coverage](#tests-and-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, what has more impact on index scan I/O: B-tree leaf density or fragmentation? Provide a comprehensive comparison, analyze different levels of density and fragmentation, and estimate the impact on index scan I/O.

## Answer

Leaf density has the larger effect on the index scan I/O that PostgreSQL 12 can see, count, and price. Density changes how many leaf pages a scan visits and how many blocks the planner charges for ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800)). Fragmentation reorders those same reads without changing their count: v12 never reads `leaf_fragmentation` in costing, and `EXPLAIN (ANALYZE, BUFFERS)` has no counter for physical-order disorder ([selfuncs.c#genericcostestimate-page-costs](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5782-L5835), [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2985)). Fragmentation's cost therefore lands entirely on the storage path, where it can still dominate elapsed time on seek-sensitive devices ([config.sgml#random_page_cost-mechanical](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4737-L4744)).

Measured on an isolated server built from this page's pinned commit, over the same 1,000,000 `bigint` keys (details in [Exact-pin measurements](#exact-pin-measurements)):

| Index state | `leaf_pages` | `avg_leaf_density` | `leaf_fragmentation` | Warm index-only scan buffers |
|---|---:|---:|---:|---:|
| Built with `fillfactor = 90` | 2733 | 90.06 | 0 | 2738 |
| Built with `fillfactor = 60` | 4116 | 59.90 | 0 | 4121 |
| Random-order retail inserts | 3694 | 66.71 | 49.95 | 3697 |
| Same index rebuilt at `fillfactor = 67` | 3677 | 67.02 | 0 | 3680 |

Dropping density from 90 to 60 raised buffer accesses by 50.5 %. Removing 49.95 % fragmentation at unchanged density moved them by 0.5 %, entirely explained by the 17-page size difference.

For a fixed live tuple count and fixed average index tuple size, the leaf-page multiplier from density is approximately:

```text
leaf_page_multiplier = baseline_density / observed_density
```

So, compared with a 90 % dense B-tree, a 60 % dense B-tree needs about `90 / 60 = 1.5x` as many leaf pages for the same live index tuples. That extra page count feeds planner costing through `IndexOptInfo.pages` and `genericcostestimate`, and it feeds executor work because `_bt_next` steps through leaf pages with `_bt_steppage` and `_bt_readnextpage` after `_bt_first` positions the scan ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780), [nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1244-L1321), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800)).

`leaf_fragmentation` is different. In `pgstatindex`, it is the percentage of live leaf pages whose logical right link points to a lower physical block number. It measures physical-order reversals in the leaf chain, not free space, tuple count, jump distance, run length, or read latency ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#leaf_fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)). A fragmented index can visit exactly the same number of leaf pages as an unfragmented index with the same density; the difference is that logical neighbors are less likely to be physical neighbors. The PostgreSQL 12 manual makes the same point: updated B-tree indexes can be slower than freshly built ones because logically adjacent pages are usually physically adjacent in a newly built index ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888)).

Practical ranking:

| Workload shape | More important factor | Why |
|---|---|---|
| Point lookup on a unique or highly selective key | Usually neither, unless density bloat raises tree height | The scan positions to one leaf page; `btcostestimate` adds a height-based descent charge so bloated indexes do not look free for one-leaf probes ([nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1244-L1321), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)). |
| Warm-cache range scan | Usually density | The same logical range touches more leaf buffers when density is lower, and the docs say fully cached data has no penalty for touching pages out of sequence ([nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [config.sgml#random_page_cost-cached](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4757-L4768)). |
| Cold large range scan on SSD or storage with low random-read penalty | Usually density, sometimes tied | Density changes page count; fragmentation changes order, and the docs model low-random-cost storage with a reduced `random_page_cost` ([config.sgml#random_page_cost-ssd](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4746-L4755)). |
| Cold large range scan on mechanical or seek-sensitive storage | Fragmentation can dominate latency | The same leaf count becomes many nonsequential fetches, and v12 documents random access to mechanical storage as normally much more expensive than four times sequential access ([config.sgml#random_page_cost-mechanical](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4737-L4744)). |
| Full index scan or index-only `COUNT(*)` style scan | Density sets the page count; fragmentation sets the locality | `_bt_endpoint` starts from an end when there is no usable boundary key, and `_bt_next` then walks the leaf chain page by page ([nbtsearch.c#endpoint-start](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L998), [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2136-L2229), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381)). |

The ranking above rests on the same executor path and the same two planner constants. v12 defines `seq_page_cost` as the cost of a disk page fetch that is part of a series of sequential fetches, default 1.0, and `random_page_cost` as the cost of a non-sequentially-fetched disk page, default 4.0 ([config.sgml#seq_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4711), [config.sgml#random_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4770), [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L21-L30)).

### What PostgreSQL 12 measures

`pgstatindex` computes `avg_leaf_density` by summing `PageGetFreeSpace(page)` and the per-page available capacity over live leaf pages, then returning `100 - free_space / max_avail * 100`. `PageGetFreeSpace` subtracts one line-pointer slot when possible, so the value is not the raw `pd_upper - pd_lower` gap ([pgstatindex.c#leaf-page-accounting](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L300), [pgstatindex.c#avg_leaf_density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)).

`pgstatindex` computes `leaf_fragmentation` as `fragments / leaf_pages * 100`, where `fragments` increments only when a live leaf page's `btpo_next` is not `P_NONE` and points to an earlier physical block. This is a one-bit-per-leaf-page physical-order test, not a cost model, and the v12 documentation defines the column only as "Leaf page fragmentation" ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#leaf_fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [pgstattuple.sgml#leaf_fragmentation-column](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L257-L261)).

Both numbers come from a full physical read of the index: `pgstatindex` walks every block from 1 to `RelationGetNumberOfBlocks`, share-locking each page, using a `BAS_BULKREAD` strategy ([pgstatindex.c:222](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L222), [pgstatindex.c#full-block-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).

The default B-tree leaf fillfactor is 90 %, while pages above the leaf level use a fixed 70 % fillfactor. The v12 header says the leaf fillfactor is applied during index build and when splitting a rightmost page; non-rightmost splits try to divide the data equally, and a page filled entirely with one duplicate value splits at an effective 96 % fillfactor ([nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171)). The split-point chooser adds one v12 case the header does not name: a non-rightmost leaf split whose new item sits at the rightmost point of a localized grouping can also apply the leaf fillfactor instead of splitting 50:50 ([nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L275-L331)).

### Where index scan reads come from

For a plain B-tree scan, `_bt_first` preprocesses the scan keys, descends with `_bt_search`, positions on the target leaf page with `_bt_binsrch`, and loads matching items from that page with `_bt_readpage`. Later calls to `_bt_next` return saved items from the current page until the page is exhausted, then call `_bt_steppage` ([nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L746-L1331), [nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1244-L1321), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381)).

For forward scans, `_bt_steppage` uses the previously saved `nextPage` right link, and `_bt_readnextpage` reads each candidate leaf page with `_bt_getbuf`, skips it and follows its own `btpo_next` if the page is deleted or half-dead, and otherwise hands it to `_bt_readpage`, which stores matching TIDs in the backend-local scan position array ([nbtsearch.c#_bt_steppage-nextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1661-L1690), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [nbtsearch.c#_bt_readpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1383-L1616)). One leaf page per right-link step is therefore one buffer access, whatever the physical block numbers are.

For backward scans, `_bt_readnextpage` uses `_bt_walk_left`, which is more complex because it must handle a left sibling that splits while the scan is in flight and a current page that gets deleted after the scan leaves it ([nbtsearch.c#_bt_readnextpage-backward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1801-L1903), [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2053)).

PostgreSQL 12 issues no prefetch for index pages. Every backend call to `PrefetchBuffer` targets a heap fork: the bitmap heap scan's prefetch iterator, lazy vacuum's truncation scan, and the heap TID horizon helper. The documentation likewise states that `effective_io_concurrency` currently affects only bitmap heap scans ([bufmgr.c#PrefetchBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L521-L530), [nodeBitmapHeapscan.c#prefetch](../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L501-L509), [vacuumlazy.c:2078](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2078), [heapam.c:6956](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6956), [config.sgml#effective_io_concurrency](../../../raw/postgres-12/doc/src/sgml/config.sgml#L2166-L2181)). Nothing inside the server overlaps a backward leaf-chain jump with the next read, so the whole fragmentation penalty is left to kernel read-ahead and the device.

`EXPLAIN (ANALYZE, BUFFERS)` prints node-level shared and local hit, read, dirtied, and written counters plus temp read and written counters from `Instrumentation.bufusage`. It does not split a node's buffers into index pages versus heap pages, and it does not label sequential versus nonsequential reads ([explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2985), [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33)).

### Density impact estimates

These estimates assume the same live index tuple count, the same average tuple width, the same visibility state, and a scan wide enough that leaf-page walking dominates the one-time descent. They estimate leaf pages visited, not heap fetches.

| Average leaf density | Leaf pages versus 90 % baseline | Extra leaf page visits | PostgreSQL-visible I/O effect |
|---:|---:|---:|---|
| 95 % | 0.95x | About 5 % fewer | Possible after compact data or duplicate-heavy effective fill behavior; fewer leaf buffers than 90 % for the same live tuple volume. |
| 90 % | 1.00x | Baseline | Default fresh-build leaf target. |
| 80 % | 1.13x | 12.5 % more | Small but visible for wide scans. |
| 70 % | 1.29x | 28.6 % more | Often enough to change large range-scan buffer counts. |
| 60 % | 1.50x | 50.0 % more | Strong direct impact on range scans and full index scans. |
| 50 % | 1.80x | 80.0 % more | Bloat-sized effect; more planner cost and more executor leaf-page steps. |
| 40 % | 2.25x | 125.0 % more | More than doubles leaf-page work for the same logical key volume. |
| 30 % | 3.00x | 200.0 % more | Severe density loss; index-side page count can dominate. |

The planner path matches this page-count model. For ordinary non-partial indexes, `get_relation_info` sets `IndexOptInfo.pages` from `RelationGetNumberOfBlocks(indexRelation)` and locks `IndexOptInfo.tuples` to the parent table estimate; for partial indexes it calls `estimate_rel_size`, whose index case reports current blocks as pages. `genericcostestimate` then computes `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)` when both counts are greater than one, and charges the index tablespace's random page cost per page, adjusted by the Mackert-Lohman cache model for repeated scans ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L971), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780), [selfuncs.c#genericcostestimate-page-costs](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5782-L5835), [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L786-L878)).

Two caveats on that mapping. `index->pages` is the whole main fork, so it also counts the metapage, internal pages, half-dead pages, and deleted pages; the density-to-page-count translation is exact only when the change is confined to live leaf pages ([pgstatindex.c#index_size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L336-L341), [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407)). And for a one-leaf point probe, density usually does not change the number of leaf pages visited at all. PostgreSQL 12 still adds an explicit B-tree descent charge of `(tree_height + 1) * 50.0 * cpu_operator_cost`, whose comment says the charge exists so that bloated indexes do not appear to have the same search cost as unbloated ones when only a single leaf page is expected ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418)).

### Fragmentation impact estimates

These estimates assume the same density and the same logical key range. Under those assumptions, fragmentation adds no leaf pages. It changes the physical order of the block numbers read while following the logical leaf chain.

Let:

```text
L = leaf pages visited by the logical scan
F = leaf_fragmentation / 100
S = cost of a sequential page fetch
R = cost of a nonsequential page fetch
```

`pgstatindex`'s `F` is an index-wide ratio. For a representative many-page scan, it implies roughly `F * (L - 1)` right-link transitions where the next logical leaf page has a lower physical block number than the current page. It does not say how far those jumps go, whether the remaining forward links form long runs, or whether the operating system or the device can prefetch them ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#leaf_fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).

A simple storage-order model is:

```text
order_cost_multiplier = 1 + F * (R / S - 1)
```

This is not PostgreSQL 12 planner output. It is a sensitivity model built on v12's documented distinction between sequential and nonsequential page fetches, with defaults 1.0 and 4.0 ([config.sgml#seq_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4711), [config.sgml#random_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4770), [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L21-L30)).

Using the default `R / S = 4` as a nonsequential-sensitive model:

| `leaf_fragmentation` | Approximate backward right-link transitions in a representative `L`-page scan | Order-cost multiplier at `R / S = 4` | PostgreSQL buffer counter expectation |
|---:|---:|---:|---|
| 0 % | `0` | 1.00x | Same leaf-page count as density predicts; best physical locality. |
| 10 % | `0.10 * (L - 1)` | 1.30x | Same distinct leaf pages, some order breaks. |
| 25 % | `0.25 * (L - 1)` | 1.75x | Same distinct leaf pages; cold-storage latency can become visible. |
| 50 % | `0.50 * (L - 1)` | 2.50x | Same distinct leaf pages; order cost can exceed a 60-versus-90 density penalty in cold-storage latency. |
| 75 % | `0.75 * (L - 1)` | 3.25x | Same distinct leaf pages; physical-order cost can dominate elapsed time. |
| 100 % | `1.00 * (L - 1)` | 4.00x | Worst-case sensitivity endpoint, though the metric still lacks jump-distance and run-length information. |

When pages are already cached, the docs say setting `random_page_cost` equal to `seq_page_cost` makes sense because there is then no penalty for touching pages out of sequence. Under `R / S = 1` the order-cost multiplier is 1.00x at every fragmentation level, so density dominates the PostgreSQL-visible buffer work ([config.sgml#random_page_cost-cached](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4757-L4768)).

For SSD-like storage, the docs say storage with a low random read cost relative to sequential may be better modeled with a lower `random_page_cost`. At `R / S = 1.2` the same model gives `1 + F * 0.2`, so 50 % fragmentation is only a 1.10x order penalty, which is smaller than the 1.50x density penalty from 60 % versus 90 % leaf density ([config.sgml#random_page_cost-ssd](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4746-L4755)).

### Combined estimate matrix

The combined model for a large cold forward range scan is:

```text
combined_multiplier = (0.90 / density) * (1 + fragmentation * (R / S - 1))
```

Here `density` and `fragmentation` are fractions, such as `0.60` and `0.50`. The first term estimates extra leaf pages. The second term estimates the physical-order penalty. PostgreSQL 12 models the first term directly through physical index pages and does not model the second term at all ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780), [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307)).

At `R / S = 4`:

| Density | Fragmentation 0 % | Fragmentation 25 % | Fragmentation 50 % | Fragmentation 75 % |
|---:|---:|---:|---:|---:|
| 90 % | 1.00x | 1.75x | 2.50x | 3.25x |
| 80 % | 1.13x | 1.97x | 2.81x | 3.66x |
| 70 % | 1.29x | 2.25x | 3.21x | 4.18x |
| 60 % | 1.50x | 2.63x | 3.75x | 4.88x |
| 50 % | 1.80x | 3.15x | 4.50x | 5.85x |
| 40 % | 2.25x | 3.94x | 5.63x | 7.31x |

At `R / S = 1.2`, which represents a much lower random-read penalty:

| Density | Fragmentation 0 % | Fragmentation 25 % | Fragmentation 50 % | Fragmentation 75 % |
|---:|---:|---:|---:|---:|
| 90 % | 1.00x | 1.05x | 1.10x | 1.15x |
| 80 % | 1.13x | 1.18x | 1.24x | 1.29x |
| 70 % | 1.29x | 1.35x | 1.41x | 1.48x |
| 60 % | 1.50x | 1.58x | 1.65x | 1.73x |
| 50 % | 1.80x | 1.89x | 1.98x | 2.07x |
| 40 % | 2.25x | 2.36x | 2.48x | 2.59x |

These tables estimate index-side leaf-page I/O pressure only. They do not estimate heap reads, visibility-map effects in index-only scans, CPU spent evaluating scan keys, kernel read-ahead behavior, controller caches, concurrent buffer churn, or tuple visibility checks outside the B-tree access method.

### Exact-pin measurements

All numbers below come from one isolated PostgreSQL 12.2 server built from this page's `pinned_commit`, with `shared_buffers = 512MB`, `autovacuum = off`, and `pgstattuple` plus `pageinspect` installed. The density and fragmentation fixtures each index 1,000,000 distinct `bigint` keys; the dead-space fixture uses 200,000. Buffer counts come from the second, warm `EXPLAIN (ANALYZE, BUFFERS)` execution of an index-only `count(*)` over the whole key range, so they count buffer accesses rather than device reads.

Density only, two build fillfactors, both freshly built and therefore physically ordered:

| Index | `leaf_pages` | `internal_pages` | `avg_leaf_density` | `leaf_fragmentation` | `tree_level` | `relpages` | Warm buffers | Planner index scan cost |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `fillfactor = 90` | 2733 | 11 | 90.06 | 0 | 2 | 2745 | 2738 | 22647.09 |
| `fillfactor = 60` | 4116 | 16 | 59.90 | 0 | 2 | 4133 | 4121 | 28199.09 |

The predicted multiplier `90.06 / 59.90 = 1.5035` matched the measured leaf-page ratio `4116 / 2733 = 1.5060` and the measured buffer ratio `4121 / 2738 = 1.5051`. Tree height did not change, so the descent charge was identical and the whole cost difference came from the page count.

Fragmentation only, produced by retail insertion in random key order and then removed by a rebuild at a fillfactor chosen to reproduce the same density:

| Index state | `leaf_pages` | `avg_leaf_density` | `leaf_fragmentation` | Backward right links (`pageinspect`) | Mean absolute right-link jump | Right links to `blkno + 1` | Warm buffers |
|---|---:|---:|---:|---:|---:|---:|---:|
| Random-order retail inserts | 3694 | 66.71 | 49.95 | 1845 of 3694 | 1848.4 blocks | 0 | 3697 |
| Rebuilt at `fillfactor = 67` | 3677 | 67.02 | 0 | 0 of 3677 | 1.0 blocks | 3663 | 3680 |

Three things follow. First, `pgstatindex`'s `leaf_fragmentation` reproduced exactly as the share of leaf pages whose `btpo_next` points backward: 1845 / 3694 = 49.95 %. Second, removing that fragmentation left buffer accesses essentially unchanged (3697 to 3680, a 0.5 % move that tracks the 17-page size change), which is what the source predicts because the executor pays one buffer access per right-link step regardless of block order. Third, `leaf_fragmentation` understated the disorder: in the fragmented index not one leaf page pointed at the immediately following block, and the average right-link jump spanned 1848 blocks, while the rebuilt index averaged one block.

Dead space, showing that `avg_leaf_density` reports physical occupancy rather than live-entry density. A 200,000-row table lost 90 % of its rows, with no vacuum in between; the baseline row was measured on a separate, identically built copy of the same fixture:

| State | Live rows | `leaf_pages` | `empty_pages` | `avg_leaf_density` | Index-only scan buffers |
|---|---:|---:|---:|---:|---:|
| Baseline | 200000 | 547 | 0 | 90.00 | 550 |
| After deleting 90 % of rows, before `VACUUM` | 20000 | 547 | 0 | 90.00 | 1438 |
| After `VACUUM` | 20000 | 547 | 0 | 9.26 | 550 |

The index looked perfectly dense at 90.00 while nine out of ten of its entries were dead, and the leaf-page count never dropped: after `VACUUM` removed the dead entries the same 547 leaf pages held 20,000 entries at 9.26 % density. That is the v12 documented failure mode: pages where all but a few keys were deleted stay allocated, and only completely empty pages are reclaimed ([maintenance.sgml#routine-reindex-partly-empty](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874)). The 1438-buffer middle row is inflated by 20,000 heap fetches, because the `DELETE` cleared visibility-map bits and the index-only scan had to visit the heap ([nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L118-L170)).

### How fragmentation arises

During retail insertion, `_bt_insertonpg` splits a page when `PageGetFreeSpace(page) < itemsz`. `_bt_split` acquires the new right page with `_bt_getbuf(rel, P_NEW, BT_WRITE)`, sets the left page's `btpo_next` to the new block, and copies the old right link into the new page's `btpo_next` while pointing its `btpo_prev` back at the original page ([nbtinsert.c#_bt_insertonpg-split-test](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L963-L1002), [nbtinsert.c#_bt_split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1403-L1439)).

When `_bt_getbuf` is asked for `P_NEW`, it first asks `GetFreeIndexPage` for a free page, conditionally locks the candidate, and reinitializes and returns it if `_bt_page_recyclable` agrees; only when the free space map has nothing usable does it extend the relation with `ReadBuffer(rel, P_NEW)` ([nbtpage.c#_bt_getbuf-fsm-reuse](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L832), [nbtpage.c#_bt_getbuf-extend](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L844-L875)). Both outcomes break physical order in different ways: reuse can place a new logical neighbor at an older, lower block, and extension places it at the very end of the relation, far from the page it logically follows. The measured random-insert index above shows the extension case: every right link jumped, on average by 1848 blocks.

VACUUM supplies the reuse path by calling `RecordFreeIndexPage` for each B-tree page that `_bt_page_recyclable` accepts ([nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1173)).

An insert can also recover density in place instead of splitting: when the target page is full and carries the `BTP_HAS_GARBAGE` hint, `_bt_findinsertloc` calls `_bt_vacuum_one_page`, which erases the page's `LP_DEAD` items through `_bt_delitems_delete` ([nbtinsert.c#_bt_findinsertloc-lp-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L752-L761), [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2243-L2288)).

The manual's operational summary matches those mechanics: completely empty B-tree pages are reclaimed for re-use, partly empty pages remain allocated and waste space, and a freshly constructed B-tree index is slightly faster to access because logically adjacent pages are usually also physically adjacent ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L888), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)).

### What both metrics hide

Four blind spots matter when either metric is used to reason about scan I/O.

- Dead entries count as dense. `_bt_killitems` only marks a matched index tuple `LP_DEAD` in place, so its storage keeps counting toward `avg_leaf_density` until an insert on the same page or a VACUUM removes it ([nbtutils.c#_bt_killitems-mark-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1785-L1799), [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2243-L2288)). The dead-space measurement above shows the consequence: 90.00 % density over 90 % dead entries.
- Half-dead pages are invisible to both metrics but still cost reads. `pgstatindex` classifies a page by `P_ISDELETED` first and then by `P_IGNORE`, so a half-dead page lands in `empty_pages` and is excluded from `leaf_pages`, hence from both `avg_leaf_density` and `leaf_fragmentation` ([pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.h#page-flag-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L189-L194)). The first stage of page deletion marks the leaf half-dead but leaves it linked to its siblings, so a scan still reads it and steps right ([nbtpage.c#_bt_mark_page_halfdead](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1645-L1652), [README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L214-L226), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800)).
- Deleted pages are the opposite case. The second deletion stage rewrites the siblings' side links and marks the page `BTP_DELETED`, so scans no longer traverse it, yet it still occupies a block in the main fork and therefore still inflates the `index->pages` the planner prices ([nbtpage.c#_bt_unlink_halfdead_page-sidelinks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1967-L1982), [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407)).
- Both columns can be `NaN`. `avg_leaf_density` is `NaN` when no live leaf page contributed available space, and `leaf_fragmentation` is `NaN` when `leaf_pages` is zero, which is what the regression test observes for an empty index ([pgstatindex.c#avg_leaf_density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351), [pgstatindex.c#leaf_fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [pgstattuple.out#empty-index-NaN](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)).

### Which one matters more

Density matters more when the question is "how many index leaf buffers will PostgreSQL touch?". For wide scans the executor walks more leaf pages when density is lower, and the planner estimates more index pages when the physical index is larger ([nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780)). The exact-pin measurement puts a number on it: 50.5 % more buffer accesses at 59.90 % density than at 90.06 %.

Fragmentation matters more when the question is "how expensive is each of those reads on this storage path?". The executor still follows the logical leaf chain, but `leaf_fragmentation` says some right-link steps go backward in physical block order. PostgreSQL 12 documents nonsequential fetches as more expensive than sequential ones, especially for mechanical storage, while also documenting that fully cached data has no out-of-sequence penalty ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [config.sgml#random_page_cost-mechanical](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4737-L4744), [config.sgml#random_page_cost-cached](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4757-L4768)).

Point lookups usually hide both effects. A unique equality lookup pays one descent and one leaf-page visit; density affects it mainly if bloat increases tree height, and fragmentation has little opportunity to matter because the scan is not walking many leaf links ([nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1244-L1321), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).

Full index scans expose both. When `_bt_first` finds no usable boundary keys, `_bt_endpoint` starts from the first or last leaf page and later `_bt_next` calls continue through the leaf chain. Density controls how many pages are in that chain for the same live tuple volume; fragmentation controls how physically ordered the chain is ([nbtsearch.c#endpoint-start](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L998), [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2136-L2229), [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307)).

Bitmap index scans use the B-tree access method's `btgetbitmap`, which loops with `_bt_first` and `_bt_next` while adding TIDs to the bitmap, so the index side behaves the same way. The heap access pattern after bitmap creation is a separate executor concern, and it is the only part of the plan that v12 prefetches ([nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L286-L342), [nodeBitmapHeapscan.c#prefetch](../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L501-L509)).

Index-only scans still walk the same index pages through `index_getnext_tid`; they avoid heap fetches only when the visibility map says the heap page is all-visible. Density and fragmentation therefore affect index-side I/O even when heap I/O is avoided ([indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L501-L545), [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L118-L170)).

### Settings and apply scope

Only three settings in this analysis are real levers, and none of them needs a restart or a reload. All three are `PGC_USERSET`, so a session or transaction can change them with `SET` or `SET LOCAL`:

| Setting | v12 context | Apply scope | Relevance here |
|---|---|---|---|
| `seq_page_cost` | `PGC_USERSET` | Session or transaction | Sets the `S` in the order-cost model; also settable per tablespace ([guc.c#seq_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3206-L3216), [config.sgml#seq_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4711)). |
| `random_page_cost` | `PGC_USERSET` | Session or transaction | Sets the `R`; the planner reads the index tablespace's value through `get_tablespace_page_costs` ([guc.c#random_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3217-L3227), [config.sgml#random_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4770), [selfuncs.c#genericcostestimate-page-costs](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5782-L5835)). |
| `effective_io_concurrency` | `PGC_USERSET` | Session or transaction | Does not help a fragmented leaf chain; documented as affecting bitmap heap scans only ([guc.c#effective_io_concurrency](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2759-L2775), [config.sgml#effective_io_concurrency](../../../raw/postgres-12/doc/src/sgml/config.sgml#L2166-L2181)). |

Neither page cost changes physical layout; they only change which plan the planner picks. The layout levers are `REINDEX` and the index `fillfactor` reloption, both of which act at build time ([nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)).

### Operational reading

For index scan I/O visible in PostgreSQL counters, compare `leaf_pages`, `index_size`, and `avg_leaf_density` first. A fall from 90 % to 60 % density implies roughly 50 % more leaf-page visits for broad scans, which the exact-pin test reproduced as 50.5 % more buffer accesses ([pgstatindex.c#index_size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L336-L341), [pgstatindex.c#avg_leaf_density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2985)).

For elapsed latency on cold scans, inspect `leaf_fragmentation` next. A 50 % fragmented index does not read 50 % more buffers, as measured above, but it can turn many logical next-page steps into nonsequential block accesses. Severity depends on cache residency, storage random-read cost, kernel read-ahead, and the run structure that `leaf_fragmentation` does not expose ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [config.sgml#random_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4770)).

This survey reads both columns for every ordinary B-tree index. `pgstatindex` reads every block of each index, so it is a diagnostic, not a monitoring query; the `relkind = 'i'` filter keeps partitioned parents out because `IS_INDEX` tests for `RELKIND_INDEX` only, so a `relkind = 'I'` parent fails with `relation "..." is not a btree index`, and the `relpersistence` filter avoids other sessions' temporary indexes, which `pgstatindex` also rejects. Execution needs superuser or `pg_stat_scan_tables` membership. Both timeouts below are `PGC_USERSET`, so they apply at session or transaction scope with no reload or restart ([pgstatindex.c#full-block-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [pgstatindex.c#relkind-am-macros](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L71), [pgstatindex.c#relation-checks](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238), [pgstattuple--1.4--1.5.sql#pgstatindex-grants](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92), [guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)):

```sql
SET /* wiki_leaf_density_vs_fragmentation */ statement_timeout = '10min';
SET /* wiki_leaf_density_vs_fragmentation */ lock_timeout = '5s';

WITH candidate AS MATERIALIZED (
    SELECT c.oid AS indexrelid,
           n.nspname AS schema_name,
           c.relname AS index_name
    FROM pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    JOIN pg_am am ON am.oid = c.relam
    WHERE c.relkind = 'i'
      AND am.amname = 'btree'
      AND c.relpersistence <> 't'
      AND n.nspname NOT IN ('pg_catalog', 'pg_toast', 'information_schema')
)
SELECT /* wiki_leaf_density_vs_fragmentation */
       k.schema_name,
       k.index_name,
       s.index_size / current_setting('block_size')::int AS blocks,
       s.leaf_pages,
       s.empty_pages,
       s.deleted_pages,
       s.avg_leaf_density,
       s.leaf_fragmentation
  FROM candidate k
  CROSS JOIN LATERAL pgstatindex(k.indexrelid::regclass) AS s
 WHERE s.leaf_pages > 0
 ORDER BY s.leaf_fragmentation DESC, s.avg_leaf_density;
```

The `AS MATERIALIZED` is defensive, not stylistic. v12 folds a single-reference, side-effect-free `WITH` query into the parent query by default, and an inlined form leaves the join order free: nothing in the query then guarantees that the access-method and relkind filters are applied before the lateral `pgstatindex` call, and a plan that reached a hash or GiST index first would fail with `relation "..." is not a btree index`. Materializing the candidate list makes the filter unconditional ([queries.sgml#with-materialization](../../../raw/postgres-12/doc/src/sgml/queries.sgml#L2221-L2236), [pgstatindex.c#relation-checks](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238)). Run against a database also holding hash, GiST, GIN, and BRIN indexes plus a partitioned parent index, the query returned only the five ordinary B-trees; an `AS NOT MATERIALIZED` variant happened to return the same rows there, which is exactly the point: the safe outcome should not depend on the plan the planner picks.

For maintenance decisions, PostgreSQL 12 source and documentation define no universal threshold such as "reindex at 60 % density" or "reindex at 40 % fragmentation". The documentation recommends `REINDEX` for B-tree indexes with many empty or nearly-empty pages and notes that a rebuild can improve access speed by restoring physical adjacency ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L888), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)).

### Tests and coverage

The v12 `pgstattuple` regression test checks `pgstatindex` output for an empty B-tree and the error paths for unsupported relation kinds and access methods. It never populates a B-tree, so it never asserts a non-`NaN` density or a nonzero fragmentation ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L9-L113), [pgstattuple.out#empty-index-NaN](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L52), [pgstattuple.out#partition-index-NaN](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L228-L236)).

The core `btree_index` regression test creates a deliberately tall B-tree with `fillfactor = 10` and later exercises multilevel page deletion and FSM page recycling, but it does not compare planner or executor I/O at different `avg_leaf_density` and `leaf_fragmentation` levels ([btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L144-L162)).

No test in the pinned checkout links either column to scan I/O. The measurements in this page were run against a disposable server built from the pin, not from an in-tree test.

## Context Reviewed

- Required wiki navigation: [versions](../../versions.md), [wiki index](../../index.md), [v12/index](../index.md), and recent [log](../../log.md).
- PostgreSQL 12 diagnostic sources: `contrib/pgstattuple/pgstatindex.c`, `contrib/pgstattuple/pgstattuple--1.4--1.5.sql`, `doc/src/sgml/pgstattuple.sgml`, `src/backend/storage/page/bufpage.c`, and `src/include/access/nbtree.h`.
- PostgreSQL 12 B-tree scan and maintenance sources: `nbtsearch.c`, `nbtinsert.c`, `nbtsplitloc.c`, `nbtpage.c`, `nbtutils.c`, `nbtree.c`, and `src/backend/access/nbtree/README`.
- PostgreSQL 12 planner and costing sources: `plancat.c`, `selfuncs.c`, `costsize.c`, `cost.h`, `guc.c`, `config.sgml`, and `indexam.sgml`.
- PostgreSQL 12 measurement and executor sources: `explain.c`, `instrument.h`, `indexam.c`, `nodeIndexonlyscan.c`, `nodeBitmapHeapscan.c`, and `bufmgr.c`.
- Same-checkout docs and tests: `maintenance.sgml`, `ref/reindex.sgml`, `contrib/pgstattuple/sql/pgstattuple.sql`, `contrib/pgstattuple/expected/pgstattuple.out`, and `src/test/regress/sql/btree_index.sql`.
- Exact-pin execution: one isolated PostgreSQL 12.2 server built from `45b88269a353ad93744772791feb6d01bc7e1e42` with `pgstattuple` and `pageinspect` installed, used for the density, fragmentation, and dead-space measurements and for validating the survey query. The server was stopped after testing; its disposable data directory and SQL scripts remain under `.wiki-runtime/`.

## Evidence Map

| Claim | Evidence |
|---|---|
| `avg_leaf_density` is computed from live-leaf free space over per-page available capacity | [pgstatindex.c#leaf-page-accounting](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L300), [pgstatindex.c#avg_leaf_density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597) |
| `leaf_fragmentation` counts live leaf pages whose right link points backward in physical block order, and the docs define it no further | [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#leaf_fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68), [pgstattuple.sgml#leaf_fragmentation-column](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L257-L261) |
| Reading either column costs a full physical index read under `BAS_BULKREAD`, is restricted to `relkind = 'i'` B-tree indexes, rejects other sessions' temporary relations, and is gated on `pg_stat_scan_tables` | [pgstatindex.c:222](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L222), [pgstatindex.c#full-block-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [pgstatindex.c#relkind-am-macros](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L71), [pgstatindex.c#relation-checks](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238), [pgstattuple--1.4--1.5.sql#pgstatindex-grants](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92) |
| Default leaf fillfactor is 90 %, non-leaf 70 %, duplicates 96 %, and v12 can also apply the leaf fillfactor to a non-rightmost split after a new item | [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171), [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L275-L331) |
| Planner B-tree I/O costing uses physical index pages, tuple estimates, and page costs, never `leaf_fragmentation` | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L971), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780), [selfuncs.c#genericcostestimate-page-costs](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5782-L5835), [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L786-L878) |
| A one-leaf probe still pays a tree-height descent charge so bloat is not free | [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418) |
| B-tree scans position once, then walk the leaf chain one buffer per right-link step | [nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1244-L1321), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381), [nbtsearch.c#_bt_steppage-nextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1661-L1690), [nbtsearch.c#_bt_readnextpage-forward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800) |
| Backward scans take the more complex `_bt_walk_left` path | [nbtsearch.c#_bt_readnextpage-backward](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1801-L1903), [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2053) |
| v12 prefetches no index pages; every `PrefetchBuffer` call site is a heap fork, and `effective_io_concurrency` covers bitmap heap scans only | [bufmgr.c#PrefetchBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L521-L530), [nodeBitmapHeapscan.c#prefetch](../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L501-L509), [vacuumlazy.c:2078](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2078), [heapam.c:6956](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6956), [config.sgml#effective_io_concurrency](../../../raw/postgres-12/doc/src/sgml/config.sgml#L2166-L2181) |
| `EXPLAIN BUFFERS` reports node-level buffer counters, not relation-kind or physical-order counters | [explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2985), [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33) |
| Page splits allocate through `_bt_getbuf(P_NEW)`, which reuses an FSM page or extends the relation, and VACUUM feeds the FSM | [nbtinsert.c#_bt_insertonpg-split-test](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L963-L1002), [nbtinsert.c#_bt_split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1403-L1439), [nbtpage.c#_bt_getbuf-fsm-reuse](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L832), [nbtpage.c#_bt_getbuf-extend](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L844-L875), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1173) |
| `LP_DEAD` index tuples keep occupying leaf space until an insert on the page or a VACUUM removes them | [nbtutils.c#_bt_killitems-mark-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1785-L1799), [nbtinsert.c#_bt_findinsertloc-lp-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L752-L761), [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2243-L2288) |
| Half-dead pages stay linked to their siblings and are excluded from both metrics; deleted pages are unlinked but still occupy blocks | [pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.h#page-flag-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L189-L194), [nbtpage.c#_bt_mark_page_halfdead](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1645-L1652), [nbtpage.c#_bt_unlink_halfdead_page-sidelinks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1967-L1982), [README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L214-L226) |
| Only completely empty B-tree pages are reclaimed; pages that keep a few keys stay allocated | [maintenance.sgml#routine-reindex-partly-empty](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57) |
| Fresh B-tree indexes are usually better physically ordered than indexes updated many times | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888) |
| v12 cost constants distinguish sequential from nonsequential page fetches, and both are session-scoped `PGC_USERSET` values | [config.sgml#seq_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4711), [config.sgml#random_page_cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4770), [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L21-L30), [guc.c#seq_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3206-L3216), [guc.c#random_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3217-L3227), [indexam.sgml#index-cost-parameters](../../../raw/postgres-12/doc/src/sgml/indexam.sgml#L1281-L1290) |
| Existing tests do not compare density and fragmentation I/O levels | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L9-L113), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L144-L162) |

## Open Questions

- `leaf_fragmentation` does not encode jump distance or run length, and the exact-pin measurement shows how much that matters: a 49.95 % fragmented index had zero right links to the next physical block and a mean jump of 1848 blocks, so two indexes with equal `leaf_fragmentation` can present very different request patterns to storage ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307)).
- The order-cost and combined tables remain sensitivity models. The measurements in this page were taken on a warm cache, so they confirm the buffer-count claims but not any elapsed-time claim about cold, seek-sensitive storage; PostgreSQL 12 exposes no counter that would isolate that effect ([explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2985), [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33)).
- `EXPLAIN (ANALYZE, BUFFERS)` cannot separate index-relation buffers from heap-relation buffers inside a node, so the measured index-only scans were used precisely to keep heap access out of the numbers ([explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L118-L170)).
- The breakeven point where fragmentation dominates density depends on cache residency, storage hardware, kernel and device read-ahead, tablespace cost settings, and concurrent buffer churn. PostgreSQL 12 derives no such point from `pgstatindex` output ([config.sgml#planner-cost-constants](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4669-L4770), [selfuncs.c#genericcostestimate-page-costs](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5782-L5835)).
- The half-dead page case is source-backed but not measured here, because producing a durable half-dead leaf page requires an interrupted or crashed VACUUM ([README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L214-L226), [nbtpage.c#_bt_mark_page_halfdead](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1645-L1652)).

## Source References

- [pgstatindex.c#relkind-am-macros](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L71)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)
- [pgstattuple--1.4--1.5.sql#pgstatindex-grants](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)
- [pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L189-L266)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)
- [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171)
- [nbtree.h#page-flag-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L189-L194)
- [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L275-L331)
- [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L746-L1331)
- [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381)
- [nbtsearch.c#_bt_readpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1383-L1616)
- [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724)
- [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1726-L1906)
- [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2053)
- [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2136-L2229)
- [nbtinsert.c#_bt_findinsertloc-lp-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L752-L761)
- [nbtinsert.c#_bt_insertonpg](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L931-L1002)
- [nbtinsert.c#_bt_split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1403-L1439)
- [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2243-L2288)
- [nbtutils.c#_bt_killitems-mark-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1785-L1799)
- [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L747-L879)
- [nbtpage.c#_bt_mark_page_halfdead](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1645-L1652)
- [nbtpage.c#_bt_unlink_halfdead_page-sidelinks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1967-L1982)
- [README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L214-L226)
- [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1173)
- [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L286-L342)
- [indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L501-L545)
- [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L118-L170)
- [nodeBitmapHeapscan.c#prefetch](../../../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L501-L509)
- [bufmgr.c#PrefetchBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L521-L530)
- [vacuumlazy.c:2078](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2078)
- [heapam.c:6956](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6956)
- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L418)
- [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L971)
- [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5780)
- [selfuncs.c#genericcostestimate-page-costs](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5782-L5835)
- [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)
- [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L786-L878)
- [cost.h#default-costs](../../../raw/postgres-12/src/include/optimizer/cost.h#L21-L30)
- [guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)
- [guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)
- [guc.c#effective_io_concurrency](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2759-L2775)
- [guc.c#seq_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3206-L3216)
- [guc.c#random_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3217-L3227)
- [explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866)
- [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2985)
- [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33)
- [config.sgml#planner-cost-constants](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4669-L4770)
- [config.sgml#effective_io_concurrency](../../../raw/postgres-12/doc/src/sgml/config.sgml#L2166-L2181)
- [indexam.sgml#index-cost-parameters](../../../raw/postgres-12/doc/src/sgml/indexam.sgml#L1281-L1290)
- [queries.sgml#with-materialization](../../../raw/postgres-12/doc/src/sgml/queries.sgml#L2221-L2236)
- [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L888)
- [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L9-L113)
- [pgstattuple.out#empty-index-NaN](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)
- [pgstattuple.out#partition-index-NaN](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L228-L236)
- [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123)
- [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L144-L162)

## Navigation

- [v12/index](../index.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](explain-analyze-buffers-output.md)
- [versions](../../versions.md)
