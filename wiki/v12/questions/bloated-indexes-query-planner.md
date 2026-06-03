---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, are there mechanisms to penalize bloated indexes in the query planner? If there are, give a comprehensive explanation with examples of types of bloated indexes and how leaf fragmentation or density affects them.

## Answer Up Front

Yes, but the mechanism is indirect. PostgreSQL 12 does not store or cost a planner field named `bloat`, `avg_leaf_density`, or `leaf_fragmentation`. It penalizes a bloated B-tree when the bloat appears as more physical index pages, a higher B-tree level, or a larger cache footprint. `get_relation_info()` fills `IndexOptInfo.pages` from the current index block count for ordinary non-partial indexes and records B-tree height with `_bt_getrootheight()`; `btcostestimate()` and `genericcostestimate()` then turn those values into index scan cost ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).

The important distinction is this:

| Condition | Planner-visible in v12? | Effect |
|---|---:|---|
| More physical index blocks for the same useful keys | Yes | Raises estimated index page work through `index->pages / index->tuples`. |
| Higher B-tree height | Yes | Raises the explicit B-tree descent CPU charge. |
| Low `avg_leaf_density` reported by `pgstatindex` | Not directly | Matters only insofar as it produces more physical pages or height. |
| High `leaf_fragmentation` reported by `pgstatindex` | No | Can slow broad scans on cold storage, but the planner does not read this metric. |
| Empty, deleted, or nearly empty B-tree pages | Partly | Empty/deleted pages still contribute to relation size; nearly empty live pages raise pages per tuple. |

The PostgreSQL 12 manual describes B-tree bloat operationally as indexes with many empty or nearly empty pages and recommends `REINDEX` to reduce that space consumption. It separately says freshly constructed B-tree indexes are slightly faster because logically adjacent pages are usually physically adjacent, which is the runtime issue that `leaf_fragmentation` tries to expose ([ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L889)).

## Planner Mechanisms

### 1. Physical page count enters index cost

For a normal non-partial index, PostgreSQL 12 sets `info->pages = RelationGetNumberOfBlocks(indexRelation)` and `info->tuples = rel->tuples`. For a partial index, it calls `estimate_rel_size()`, whose index case still reports current blocks as `*pages` and estimates tuples from catalog tuple density after discounting the metapage ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1018)).

`genericcostestimate()` estimates the number of index pages touched as a pro-rata fraction of total index pages:

```c
numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
```

It then charges random page cost per touched page for a single scan, or applies `index_pages_fetched()` to model cache effects for repeated scans such as nested-loop inner index scans and scalar-array scans ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L878)).

This means a B-tree with twice as many physical pages for the same estimated useful tuples will usually look more expensive for range scans, bitmap index scans, and broad index-only scans. The effect is weaker for a highly selective point lookup because `ceil()` often keeps the estimated leaf pages at one.

### 2. B-tree height has an explicit bloat charge

After `genericcostestimate()`, `btcostestimate()` adds a B-tree descent CPU charge. It first charges about `log2(index->tuples) * cpu_operator_cost` for comparisons, then adds:

```c
descentCost = (index->tree_height + 1) * 50.0 * cpu_operator_cost;
```

The comment says this prevents bloated indexes from appearing to have the same search cost as unbloated ones when only a single leaf page is expected. `get_relation_info()` records `tree_height` from `_bt_getrootheight()` while it has the B-tree open ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6092-L6116), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417)).

This is the most explicit planner-side bloat penalty in the v12 B-tree code. It is still not a density or fragmentation penalty; it only changes when the B-tree level changes.

### 3. Index size participates in cache modeling

`index_pages_fetched()` estimates cache effects with the Mackert-Lohman formula. It adds the caller's `index_pages` to `root->total_table_pages` when prorating `effective_cache_size`, so a larger index increases the estimated cache competition term used in both index page costing and heap page costing around index scans ([costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L878), [costsize.c#cost_index](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L541-L665)).

The page-cost constants are global or tablespace-level estimates. `seq_page_cost` defaults to 1.0, and `random_page_cost` defaults to 4.0; lowering `random_page_cost` makes index scans look cheaper, raising it makes them look more expensive, but this is not a per-index bloat signal ([config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765)).

## What The Planner Does Not See

`pgstatindex()` is a diagnostic function in `contrib/pgstattuple`, not an input to planner path creation. It scans B-tree blocks, counts leaf/internal/empty/deleted pages, sums leaf free space, and counts backward right-link transitions. The result tuple includes `avg_leaf_density` and `leaf_fragmentation`, but those values are not read by `get_relation_info()`, `genericcostestimate()`, or `btcostestimate()` ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365), [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

`avg_leaf_density` is computed as `100 - free_space / max_avail * 100` across live leaf pages. The free-space input is `PageGetFreeSpace(page)`, and `pgstatindex()` ignores deleted and half-dead pages for that density denominator ([pgstatindex.c#leaf-density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L356)).

`leaf_fragmentation` is computed as `fragments / leaf_pages * 100`, where a fragment is counted when a live leaf page's `btpo_next` is not `P_NONE` and points to a lower physical block number. `BTPageOpaqueData` stores both sibling links, so this metric is about physical order of the logical leaf chain, not free space or tuple density ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L30-L68)).

## Examples Of Bloated Indexes

### Low-density live leaf pages

A B-tree can have many live leaf pages that are only partly full. The docs describe the common shape: completely empty B-tree pages can be reclaimed for reuse, but a page with only a few remaining keys can stay allocated, so patterns that delete most but not all keys from many key ranges produce poor space use ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L874)).

For a fixed live tuple count and tuple width, leaf-page count scales approximately as:

```text
leaf_page_multiplier = baseline_density / observed_density
```

So a 60% dense index needs about `90 / 60 = 1.5x` as many leaf pages as a 90% dense index for the same live entries. The planner does not know the 60% number, but it can see the larger `index->pages`; broad scans then get a larger `numIndexPages` estimate ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

At execution, B-tree scans position once and then walk leaf pages. `btgettuple()` calls `_bt_first()` for the first tuple and `_bt_next()` after that; `_bt_next()` calls `_bt_steppage()` when the current leaf page is exhausted; forward `_bt_readnextpage()` follows the saved right link and reads the next block with `_bt_getbuf()` ([nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1334-L1381), [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1619-L1724), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1800), [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).

### Empty or deleted pages

`pgstatindex()` reports `empty_pages` for pages ignored by B-tree scans, and `deleted_pages` for pages marked deleted. The diagnostic's `index_size` includes leaf, internal, empty, deleted, and metapage blocks, so this kind of bloat appears as physical relation size even though those pages do not hold useful live keys ([pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L264-L344)).

The planner's physical page count does not subtract those diagnostic categories. For ordinary non-partial indexes, it reads the index relation's current block count; for partial indexes, the index branch of `estimate_rel_size()` reports current blocks as pages ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L972)).

This can make a bloated index look more expensive even when many extra pages are not useful scan pages. The model is deliberately coarse: `genericcostestimate()` treats touched index pages as a pro-rata fraction of total pages and says this simplistic method effectively counts leaf pages while ignoring metapage and upper-level overhead ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5750-L5815)).

### Taller B-trees

If extra pages are enough to add another B-tree level, point lookups and narrow scans get a direct cost increase from the `tree_height + 1` descent charge. The root level is stored in the B-tree metapage, and `pgstatindex()` reports it as `tree_level`; `get_relation_info()` reads the root height for planner costing ([nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L76-L91), [pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L239-L249), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).

This is why a unique equality probe is not completely blind to severe bloat: even when the estimated leaf pages remain one, a taller tree raises startup and total cost.

### Physically fragmented leaf chains

Leaf fragmentation is different from density. A fragmented index can have the same number of live leaf pages and the same average density as an unfragmented index, but its logical next leaf pages can be physically out of order. `pgstatindex()` counts a fragment when `btpo_next` points backward to a lower block number ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307)).

The planner does not cost that. The executor can still feel it. Forward scans follow `btpo_next` through `_bt_readnextpage()` and read the named block with `_bt_getbuf()`. If those blocks are not physically adjacent, cold scans can lose sequential locality. The v12 manual states the same operational point: a freshly constructed B-tree is slightly faster because logically adjacent pages are usually physically adjacent ([nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1800), [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L881-L889)).

Page splits and page reuse explain how this can happen. `_bt_split()` allocates a new right page with `_bt_getbuf(rel, P_NEW, BT_WRITE)`, then updates sibling links. `_bt_getbuf(P_NEW)` first asks the free space map for a reusable index page before extending the relation, so a new logical neighbor may be placed at an older physical block ([nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1408-L1437), [nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).

## How Density And Fragmentation Affect Different Queries

| Query shape | Density effect | Fragmentation effect |
|---|---|---|
| Unique point lookup | Usually small unless height rises; estimated leaf pages often remain one. | Usually small because the scan does not walk many leaf links. |
| Selective range scan | More leaf pages for the same key range raises planner cost and executor page steps. | Can add cold-storage latency if the range spans many physically disordered leaves. |
| Wide range scan or ordered index scan | Often large; page count can scale near linearly with density loss. | Can be large on seek-sensitive storage; invisible to planner. |
| Bitmap index scan | B-tree side still uses `_bt_first()` and `_bt_next()` while filling the TID bitmap. | Same index-side locality issue; heap access after bitmap creation is a separate cost. |
| Index-only scan | Same index leaf walk; heap fetches may be avoided by visibility map state. | Same leaf-chain locality issue for the index pages that are read. |

`btgetbitmap()` confirms the bitmap case: it fetches the first matching B-tree tuple and continues scanning by calling `_bt_next()` while adding TIDs to the bitmap ([nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335)). Index-only scans still get TIDs through the index access method; they avoid heap access only when the visibility map says the heap page is all-visible ([indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L502-L545), [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)).

## Practical Interpretation

Use `pgstatindex()` as a diagnosis, not as a direct explanation of planner choices. Low density and many empty/deleted pages are important because they usually increase the physical page count that the planner can see. Fragmentation is important because it can hurt wall-clock scan time, especially on cold or seek-sensitive storage, but it does not change v12 planner cost by itself ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765)).

`REINDEX` is the source-backed repair for physical B-tree bloat. The v12 docs say it rebuilds the index from table data, reduces bloat by writing a new version without dead pages, and can apply changed storage parameters such as fillfactor ([ref/reindex.sgml#reindex-use-cases](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L32-L72)). B-tree fillfactor defaults to 90, can be set from 10 to 100, and is useful differently for static versus heavily updated tables ([create_index.sgml#fillfactor](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L369-L390), [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)).

## Tests And Coverage

The `pgstattuple` regression test checks `pgstatindex()` entry points and empty-index output, but it does not create populated B-trees with non-`NaN` density or nonzero fragmentation assertions ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [pgstattuple.out#empty-index-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L47-L85)).

The core `btree_index` regression test is present in the v12 regression schedule. Its test body creates a deliberately tall B-tree with `fillfactor = 10` and covers B-tree page deletion plus FSM page recycling after `VACUUM`, but it does not compare planner choices for the same logical index at different `avg_leaf_density` or `leaf_fragmentation` levels ([serial_schedule#btree_index](../../../raw/postgres-12/src/test/regress/serial_schedule#L100), [parallel_schedule#btree_index](../../../raw/postgres-12/src/test/regress/parallel_schedule#L71), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L162)).

## Open Questions

- PostgreSQL 12 does not define a universal `avg_leaf_density` or `leaf_fragmentation` threshold for `REINDEX`; the docs describe conditions and tradeoffs, not a percentage cutoff ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L895), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)).
- `leaf_fragmentation` does not encode jump distance, run length, operating-system read-ahead behavior, or cache residency, so it cannot by itself predict elapsed time ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [config.sgml#random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4713-L4765)).
- `EXPLAIN (ANALYZE, BUFFERS)` can show node-level shared/local/temp buffer hits and reads, but the v12 output path does not split those counters into index versus heap pages or sequential versus nonsequential reads ([explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).

## Related Pages

- [v12/index](../index.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](explain-analyze-buffers-output.md)
- [versions](../../versions.md)

## Evidence Map

| Claim | Evidence |
|---|---|
| Planner B-tree bloat penalties are page-count and height based, not direct `pgstatindex` metrics | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116) |
| B-tree page count is read from the current index relation for ordinary indexes and from `estimate_rel_size()` for partial indexes | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1018) |
| `pgstatindex()` computes density, fragmentation, and page category diagnostics outside planner costing | [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365) |
| B-tree range scans walk sibling-linked leaf pages through `_bt_next()`, `_bt_steppage()`, `_bt_readnextpage()`, and `_bt_getbuf()` | [nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1334-L1381), [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1619-L1724), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1800), [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820) |
| B-tree bloat and physical adjacency are documented maintenance concerns | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L889), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55) |
| Fillfactor shapes initial B-tree density and split behavior | [create_index.sgml#fillfactor](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L369-L390), [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171) |
| Existing tests do not compare planner decisions by density or fragmentation level | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [serial_schedule#btree_index](../../../raw/postgres-12/src/test/regress/serial_schedule#L100), [parallel_schedule#btree_index](../../../raw/postgres-12/src/test/regress/parallel_schedule#L71), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L162) |

## Context Reviewed

- Required wiki navigation: [versions](../../versions.md), [wiki index](../../index.md), [v12/index](../index.md), and recent [log](../../log.md).
- PostgreSQL 12 planner and costing sources: `plancat.c`, `selfuncs.c`, `costsize.c`, and `config.sgml`.
- PostgreSQL 12 B-tree sources: `nbtree.c`, `nbtsearch.c`, `nbtpage.c`, `nbtinsert.c`, and `nbtree.h`.
- PostgreSQL 12 diagnostics, docs, and tests: `pgstatindex.c`, `pgstattuple.sql`, `pgstattuple.out`, `maintenance.sgml`, `ref/reindex.sgml`, `ref/create_index.sgml`, `serial_schedule`, and `parallel_schedule`.

## Source References

- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417)
- [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1018)
- [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5680-L5815)
- [selfuncs.c#btcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5920-L6116)
- [costsize.c#cost_index](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L470-L665)
- [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L878)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L30-L68)
- [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L76-L91)
- [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)
- [nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284)
- [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335)
- [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1334-L1381)
- [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1619-L1724)
- [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1800)
- [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)
- [nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1408-L1437)
- [indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L502-L545)
- [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)
- [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L895)
- [ref/reindex.sgml#reindex-use-cases](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L32-L72)
- [create_index.sgml#fillfactor](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L369-L390)
- [config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765)
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)
- [pgstattuple.out#empty-index-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L47-L85)
- [serial_schedule#btree_index](../../../raw/postgres-12/src/test/regress/serial_schedule#L100)
- [parallel_schedule#btree_index](../../../raw/postgres-12/src/test/regress/parallel_schedule#L71)
- [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123)
- [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L162)