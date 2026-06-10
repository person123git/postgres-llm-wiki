---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: claude-fable-5 2026-06-10T11:04:20Z
---

# Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, provide a comprehensive analysis of the impact on queries that scan indexes with a leaf density of 60% vs 90%?

## Answer Up Front

A B-tree index with 60% average leaf density requires roughly 50% more leaf pages than the same index at 90% density, assuming the same live index tuple count and tuple sizes.

- The planner does not read `avg_leaf_density`, but it does cost B-tree paths from physical index size. For ordinary non-partial indexes, `get_relation_info` sets `index->pages` from `RelationGetNumberOfBlocks(indexRelation)`; for partial indexes it calls `estimate_rel_size`, which reports current blocks and estimates tuples from catalog density. `genericcostestimate` then scales `numIndexPages` directly from `index->pages / index->tuples`.
- At execution, scans that cover a key range will visit more B-tree leaf pages. After `_bt_first` positions the scan, `_bt_next` uses `_bt_steppage` and `_bt_readnextpage`; forward scans follow the saved `btpo_next` link and read the next leaf with `_bt_getbuf`, while backward scans use `_bt_walk_left`.
- Extra leaf pages add buffer lookup, pin, content-lock, and `_bt_readpage` tuple-checking work. The first page is positioned with `_bt_binsrch`; later forward pages usually start at `P_FIRSTDATAKEY`, so they are scanned for matches rather than binary-searched again.
- Point lookups (equality on a unique index) usually touch one leaf page either way; the visible cost difference is small, though `btcostestimate` still adds an explicit `50 * cpu_operator_cost` per descended B-tree level to keep bloated indexes from looking free.
- Wide range scans, `ORDER BY` scans, and full-index operations (e.g. `COUNT(*)` via index-only scan) see near-linear cost increase with the extra leaf pages.
- PostgreSQL 12 source and docs do not define a universal "bad" density threshold. The docs recommend `REINDEX` when B-tree indexes contain many empty or nearly-empty pages, and the right operational threshold is workload-specific.

All behavioral claims below are derived from the v12 pinned checkout under `raw/postgres-12/`.

## What "Leaf Density" Means

PostgreSQL 12 does not store a runtime "leaf density" value. The diagnostic is produced on demand by `pgstatindex` (from the `pgstattuple` contrib module).

`avg_leaf_density` is computed as:

```
100 - (sum(PageGetFreeSpace(leaf)) / sum(pd_special - SizeOfPageHeaderData for leaf)) * 100
```

over every live leaf page that `pgstatindex_impl` counts. The v12 code uses `PageGetFreeSpace`, not `PageGetExactFreeSpace`, so it reserves space for one line pointer in the reported free-space value ([pgstatindex.c#leaf-density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L356), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L581-L597)).

- `BTREE_DEFAULT_FILLFACTOR` is 90; `BTREE_NONLEAF_FILLFACTOR` is 70.
- Newly created or freshly reindexed B-trees target ~90% fill on leaf pages (rightmost-page and single-value special cases differ).
- Over time, inserts, updates that move rows, and page splits that do not perfectly repack cause the average to drop. Deletes do not shrink leaf pages directly: later index scans mark known-dead index tuples `LP_DEAD` via `_bt_killitems`, and an insertion into a full leaf page (`_bt_vacuum_one_page`) or VACUUM removes them, but the reclaimed space is not always reused on the original page ([nbtutils.c#_bt_killitems](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1687-L1814), [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2243-L2288)).

The fillfactor comment in `nbtree.h` says the leaf fillfactor is applied during index build and rightmost-page splits, while non-rightmost splits usually divide data equally and single-value splits use a different effective fillfactor ([nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)). The split-location code matches that: rightmost leaf splits apply the relation fillfactor, while other leaf splits fall back to a 50:50 split unless a special case applies ([nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L98-L104), [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330)).

Thus 90% -> 60% is a plausible symptom of B-tree space inefficiency, but it is not itself a stored planner input.

## Planner Cost Impact (Higher Estimated I/O)

The planner's index cost estimation for btree routes through `btcostestimate` → `genericcostestimate` (both in `selfuncs.c`).

The important sizing input is `IndexOptInfo.pages`. For a normal non-partial index, PostgreSQL 12 sets it to the index relation's current block count and sets `IndexOptInfo.tuples` to the parent table tuple estimate. For a partial index, it calls `estimate_rel_size`, which still reports current blocks as `*pages` but estimates tuple count from prior `pg_class` density after discounting the metapage ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026)).

Key sizing step:

```c
if (index->pages > 1 && index->tuples > 1)
    numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
```

For a fixed number of qualifying index tuples, a 1.5x increase in `index->pages` produces a 1.5x increase in estimated `numIndexPages`, before cache adjustment. That number feeds the Mackert-Lohman cache model and the index tablespace page cost ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L877)). The proportional result is approximate because total index pages include the metapage, internal pages, deleted pages, and empty pages; it is closest when the page-count change is dominated by live leaf pages.

In addition, `btcostestimate` explicitly adds a CPU descent charge to make bloated indexes visibly more expensive even for point queries. The charge is `(index->tree_height + 1) * 50.0 * cpu_operator_cost` per scan and per scalar-array-operation expansion ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)). If lower density adds enough physical pages to increase B-tree height, `get_relation_info` records the higher `_bt_getrootheight` value and this charge rises too ([plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417)).

Result: when two otherwise similar indexes are viable, the planner should cost the physically smaller one lower. It may choose a different plan (sequential scan, bitmap plan, or a different index) once the larger index's page cost exceeds the alternatives.

## Executor Scan Path Impact (More Real Work per Qualifying Tuple)

All B-tree scan variants use the B-tree access method callbacks registered in `bthandler`: `btgettuple` for plain and index-only scans, and `btgetbitmap` for bitmap index scans ([nbtree.c#bthandler-callbacks](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L128-L148)).

1. `btgettuple` calls `_bt_first` for the first item, then `_bt_next` for later items ([nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284)).
2. `_bt_first` descends with `_bt_search`, positions precisely on the first leaf with `_bt_binsrch`, then loads matching items from that leaf with `_bt_readpage` ([nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328)).
3. `_bt_next` advances inside the current leaf page. When the current page is exhausted, it calls `_bt_steppage`, which calls `_bt_readnextpage` ([nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381), [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724)).
4. Forward `_bt_readnextpage` follows the saved `btpo_next` page number and reads the next page with `_bt_getbuf`; backward scans call `_bt_walk_left` because left-link navigation must handle concurrent splits and deletion cases ([nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [nbtsearch.c#backward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1813-L1905), [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2053)).

The forward page step is:

```c
so->currPos.buf = _bt_getbuf(rel, blkno, BT_READ);
...
if (_bt_readpage(scan, dir, P_FIRSTDATAKEY(opaque)))
    break;
```

Each leaf page in the result set therefore costs:
- A shared-buffer lookup/read, buffer pin, and B-tree read lock through `_bt_getbuf`.
- `_bt_readpage` checks items against the scan keys, copies matching TIDs into backend-local `BTScanPos.items`, and can stop early when `_bt_checkkeys` proves no later page can match.
- A later lock/pin drop through `_bt_drop_lock_and_maybe_pin` or `_bt_relbuf`, depending on the path.

A 60% dense index therefore causes wide scans to execute the page-step and read-page work roughly 50% more often for the same logical key range, when the only important change is leaf density. The extra work can show up as higher elapsed CPU time and more shared-buffer hits/reads, but PostgreSQL 12 does not label those counters as "index leaf" work.

Backward scans follow the left links instead of right links; the page count effect is identical.

Bitmap scans (`btgetbitmap`) use the same `_bt_first` / `_bt_next` loop to populate the TID bitmap, so the index-side page-walk effect is the same ([nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L340)).

Index-only scans still walk the same index pages through `index_getnext_tid`; they avoid heap fetches only when the visibility map says the heap page is all-visible ([indexam.c#index_getnext_tid](../../../raw/postgres-12/src/backend/access/index/indexam.c#L502-L545), [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)).

## Buffer Manager and Caching Effects

- More leaf pages enlarge the working set competing for cache space; the planner's cache model explicitly counts index pages as part of the cache competition term ([costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L877)).
- A freshly built B-tree usually has logically adjacent pages physically adjacent too; the PostgreSQL 12 manual explicitly says an updated B-tree can become slower to access for that reason and that periodic reindexing can improve speed ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889)).
- Each new shared-buffer page access goes through the buffer mapping hash. Even a buffer hit takes the partition lock, performs `BufTableLookup`, pins the buffer, and releases the lock. In v12 the mapping table has `NUM_BUFFER_PARTITIONS = 128` ([bufmgr.c#BufferAlloc-hit](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1011-L1035), [buf_internals.h#BufMappingPartitionLock](../../../raw/postgres-12/src/include/storage/buf_internals.h#L120-L131), [lwlock.h#NUM_BUFFER_PARTITIONS](../../../raw/postgres-12/src/include/storage/lwlock.h#L107-L114)).

## Point Query vs. Range Query Difference

- Equality probe on a unique or high-cardinality leading column: typically one leaf page visit regardless of density; the descent cost can still rise if tree height rises ([nbtsearch.c#_bt_first-position](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1245-L1328), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).
- Range scan, `ORDER BY` using the index, or prefix scan on a multi-column index: the number of leaf pages visited grows with the number of leaf pages that contain qualifying keys, because `_bt_next` steps page-by-page after exhausting each current leaf ([nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1333-L1381), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800)).
- Full-index scan (e.g. `SELECT count(*) FROM t` when only an index is needed): with no usable boundary key, `_bt_first` starts from an endpoint and `_bt_next` continues page-by-page, so the index-side work is directly tied to total leaf page count ([nbtsearch.c#endpoint-start](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L990), [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2137-L2228)).

## Tests and Coverage

The v12 `pgstattuple` regression test checks `pgstatindex` on an empty primary-key B-tree through the regclass/text/name entry points, but it does not populate a B-tree and assert non-`NaN` `avg_leaf_density` values ([pgstattuple.sql#empty-btree-pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L17-L37), [pgstattuple.out#empty-btree-pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

The core `btree_index` regression test creates a deliberately tall B-tree with `fillfactor = 10`, and later covers B-tree page deletion and FSM page recycling, but it does not compare 60% and 90% leaf density planner costs or executor buffer counts ([btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L119-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L161)).

## Open Questions

- v12 `EXPLAIN (ANALYZE, BUFFERS)` prints buffer counters from each plan node's `Instrumentation.bufusage`, and `show_buffer_usage` labels them only as shared/local/temp hit/read/dirtied/written plus optional I/O timing. It does not split a node's buffers into index-relation versus heap-relation pages ([explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2985)).
- No GUC or per-index planner knob says "cost this B-tree as if leaf density were 70%". `fillfactor` affects build/split behavior, not planner costing; planner costing uses physical pages, tuple estimates, selectivity, correlation, and cost GUCs.
- The exact breakeven point where a bloated index loses to a seqscan depends on `random_page_cost`, `cpu_tuple_cost`, correlation, and buffer hit rate; those are workload-specific and not modeled from `pgstatindex` output in the v12 planner.
- There is no source-backed universal `avg_leaf_density` threshold for `REINDEX`. PostgreSQL 12 docs recommend `REINDEX` for indexes with many empty or nearly-empty pages, but not for a specific percentage cutoff ([ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)).

## Related Pages

- [v12/index](../index.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](explain-analyze-buffers-output.md)
- [versions](../../versions.md)

## Evidence Map

| Claim | Evidence |
|-------|----------|
| Default leaf target is 90%, non-leaf 70% | [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L169-L171) |
| `avg_leaf_density` formula and `PageGetFreeSpace` usage (reserves one line pointer) | [pgstatindex.c#leaf-accumulation and result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L356), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L581-L597) |
| Planner page and tuple inputs for ordinary and partial indexes | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026) |
| Planner scales `numIndexPages` directly from `index->pages / index->tuples` | [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815) |
| Explicit bloat penalty comment and 50× cpu_operator_cost descent charge | [selfuncs.c#btcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116) |
| B-tree `btgettuple` uses `_bt_first` then `_bt_next`; `btgetbitmap` uses `_bt_first` and `_bt_next` too | [nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284), [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L340) |
| Forward leaf walking uses `_bt_steppage` / `_bt_readnextpage` / `_bt_getbuf`; backward walking uses `_bt_walk_left` | [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1724), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1747-L1800), [nbtsearch.c#backward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1813-L1905), [nbtsearch.c#_bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1930-L2053) |
| `_bt_readpage` checks page items and saves matching TIDs into `BTScanPos` | [nbtsearch.c#_bt_readpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1383-L1616) |
| `btvacuumcleanup` / `btvacuumscan` supply index tuple and page counts that VACUUM can write to `pg_class` | [nbtree.c#btvacuumcleanup](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L897-L940), [nbtree.c#btvacuumscan-stats](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1015-L1099), [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1772-L1817) |
| EXPLAIN BUFFERS reports node-level shared/local/temp counters, not relation-kind counters | [explain.c#plan-buffer-usage](../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2985) |
| v12 tests cover empty `pgstatindex` and a fillfactor/tall-B-tree case, but not 60% vs 90% cost behavior | [pgstattuple.sql#empty-btree-pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L17-L37), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L119-L123) |

## Context Reviewed

- Required wiki navigation: [versions](../../versions.md), [wiki index](../../index.md), [v12/index](../index.md), and the recent [log](../../log.md).
- PostgreSQL 12 B-tree sources: `nbtree.c`, `nbtsearch.c`, `nbtpage.c`, `nbtinsert.c`, `nbtutils.c`, `nbtsplitloc.c`, `nbtsort.c`, and `nbtree.h`.
- Planner and costing sources: `plancat.c`, `selfuncs.c`, and `costsize.c`.
- Measurement and storage sources: `explain.c`, `bufmgr.c`, `buf_internals.h`, `lwlock.h`, `vacuumlazy.c`, and `indexfsm.c`.
- Same-checkout docs and tests: `maintenance.sgml`, `ref/reindex.sgml`, `pgstattuple.sgml`, `contrib/pgstattuple/sql/pgstattuple.sql`, and `src/test/regress/sql/btree_index.sql`.

## Source References

- [nbtree.h#fillfactor constants](../../../raw/postgres-12/src/include/access/nbtree.h#L168-L172)
- [nbtree.h#BTPageOpaqueData and page macros](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L196)
- [nbtsort.c#build-fillfactor](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L719-L735)
- [nbtsplitloc.c#split-fillfactor](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L98-L104)
- [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330)
- [nbtsearch.c#_bt_search and _bt_moveright](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L92-L323)
- [nbtsearch.c#_bt_first and _bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L746-L1381)
- [nbtsearch.c#_bt_readpage and _bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1383-L1724)
- [nbtsearch.c#_bt_readnextpage and _bt_walk_left](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1726-L2053)
- [nbtsearch.c#_bt_endpoint](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L2137-L2228)
- [nbtpage.c#_bt_relandgetbuf and buffer helpers](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L890-L918)
- [nbtree.c#btgettuple and btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L342)
- [nbtree.c#btvacuumcleanup and btvacuumscan stats](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L890-L1099)
- [plancat.c#get_relation_info index sizing](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407)
- [plancat.c#estimate_rel_size index sizing](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026)
- [selfuncs.c#genericcostestimate and btcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5679-L6223)
- [costsize.c#cost_index](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L475-L748)
- [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L877)
- [bufpage.c#PageGetFreeSpace / PageGetExactFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L581-L647)
- [pgstatindex.c#pgstatindex_impl (density math)](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)
- [bufmgr.c#ReleaseAndReadBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1506-L1553)
- [bufmgr.c#LockBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607)
- [bufmgr.c#BufferAlloc hit path](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1011-L1035)
- [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2985)
- [maintenance.sgml#routine reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L889)
- [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)
- [pgstattuple.sql#pgstatindex regression](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L17-L37)
- [btree_index.sql#fillfactor regression](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L119-L123)
