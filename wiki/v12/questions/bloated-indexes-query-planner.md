---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: claude-fable-5 2026-06-10T10:15:55Z
---

# Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, are there mechanisms to penalize bloated indexes in the query planner? If there are, give a comprehensive explanation with examples of types of bloated indexes and how leaf fragmentation or density affects them.

## Answer Up Front

Yes, but the mechanism is indirect. PostgreSQL 12 does not store or cost a planner field named `bloat`, `avg_leaf_density`, or `leaf_fragmentation`. It penalizes a bloated B-tree when the bloat appears as more physical index pages, a higher B-tree level, or a larger cache footprint. `get_relation_info()` fills `IndexOptInfo.pages` from the current index block count for ordinary non-partial indexes and records B-tree height with `_bt_getrootheight()`; `btcostestimate()` and `genericcostestimate()` then turn those values into index scan cost ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)). The block-count read itself is not a B-tree page scan: `RelationGetNumberOfBlocks()` asks the storage manager for the main-fork relation length, and the magnetic-disk storage manager computes that from segment file sizes through `FileSize()` / `lseek(SEEK_END)` ([bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199), [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2791-L2811), [md.c#mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L698-L755), [fd.c#FileSize](../../../raw/postgres-12/src/backend/storage/file/fd.c#L2039-L2053)).

These effects are not all unique to B-trees. The physical-page-count penalty flows through the shared `genericcostestimate()` helper, so it also covers hash, GiST, and SP-GiST indexes; GIN and BRIN use separate cost models. The height penalty measured from the index itself is B-tree-only: GiST and SP-GiST charge the same descent formula but estimate the height from the physical page count, and hash charges no descent cost. Every `pgstatindex` metric in this page is B-tree-only ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [selfuncs.c#gistcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6283-L6317), [selfuncs.c#hashcostestimate-no-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6237-L6260), [selfuncs.c#gincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6672), [selfuncs.c#brincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6985)).

The important distinction is this:

| Condition | Planner-visible in v12? | Effect |
|---|---:|---|
| More physical index blocks for the same useful keys | Yes | Raises estimated index page work through `index->pages / index->tuples` for ordinary non-partial indexes. |
| Higher B-tree height | Yes | Raises the explicit B-tree descent CPU charge. |
| Low `avg_leaf_density` reported by `pgstatindex` | Not directly | Matters only insofar as it produces more physical pages or height. |
| High `leaf_fragmentation` reported by `pgstatindex` | No | Can slow broad scans on cold storage, but the planner does not read this metric. |
| Empty, deleted, or nearly empty B-tree pages | Partly | Empty/deleted pages still contribute to relation size; nearly empty live pages raise pages per tuple. |

The PostgreSQL 12 manual describes B-tree bloat operationally as indexes with many empty or nearly empty pages and recommends `REINDEX` to reduce that space consumption. It separately says freshly constructed B-tree indexes are slightly faster because logically adjacent pages are usually physically adjacent, which is the runtime issue that `leaf_fragmentation` tries to expose ([ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L889)).

## Planner Mechanisms

The planner reaches every index cost estimate through the index access method's `amcostestimate` support function. `get_relation_info()` copies `amcostestimate` from the AM handler into the `IndexOptInfo`, and `cost_index()` casts and calls it for each candidate index scan. PostgreSQL 12's B-tree handler points `amcostestimate` at `btcostestimate`, which is where the height-based charge in section 2 lives ([nbtree.c#bthandler](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L134), [plancat.c#amcostestimate](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L276-L277), [costsize.c#cost_index-amcostestimate](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L544-L548)).

The physical-page-count penalty generalizes across the non-GIN, non-BRIN index types. `btcostestimate()`, `hashcostestimate()`, `gistcostestimate()`, and `spgcostestimate()` all delegate to the shared `genericcostestimate()` helper, so the `index->pages / index->tuples` page-count effect in section 1 applies to B-tree, hash, GiST, and SP-GiST indexes alike. `gincostestimate()` and `brincostestimate()` compute cost with their own models and never call `genericcostestimate()`, so this page's page-count formula does not describe GIN or BRIN bloat ([selfuncs.c#btcostestimate-genericcost](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6084), [selfuncs.c#hashcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6226-L6235), [selfuncs.c#gistcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6270-L6281), [selfuncs.c#spgcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6327-L6338), [selfuncs.c#gincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6672), [selfuncs.c#brincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6985)).

The height charge in section 2 uses a height measured from the index only for B-trees. `get_relation_info()` records a real `tree_height` from `_bt_getrootheight()` for B-tree indexes and sets `tree_height = -1` for every other access method ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417)). `gistcostestimate()` and `spgcostestimate()` fill in that `-1` themselves: each assumes a fanout of 100, estimates the height as `log100(index->pages)`, and adds the same `(index->tree_height + 1) * 50.0 * cpu_operator_cost` charge, which its comment calls "calculated the same as for btrees". Their descent charge therefore tracks the physical page count rather than a height read from the index ([selfuncs.c#gistcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6283-L6317), [selfuncs.c#spgcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6340-L6374)). `hashcostestimate()` adds no descent charge at all; its comment says hash search goes directly to the target bucket ([selfuncs.c#hashcostestimate-no-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6237-L6260)). Every `pgstatindex` metric discussed below is B-tree-only, since `pgstatindex()` reads B-tree page structures.

### 1. Physical page count enters index cost

For a normal non-partial index, PostgreSQL 12 sets `info->pages = RelationGetNumberOfBlocks(indexRelation)` and `info->tuples = rel->tuples`. For a partial index, it calls `estimate_rel_size()`, whose index case still reports current blocks as `*pages` and estimates tuples from catalog tuple density after discounting the metapage ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026)).

That difference matters. In an ordinary non-partial index, extra blocks with the same parent-table tuple estimate directly increase `index->pages / index->tuples`. In a partial index, the current page count still enters costing, but the tuple estimate can scale from the index's recorded tuple density and current block count, so the page-count penalty is less direct ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L394-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L990-L1027)).

#### How `RelationGetNumberOfBlocks(indexRelation)` gets the number

`RelationGetNumberOfBlocks(reln)` is only a macro wrapper around `RelationGetNumberOfBlocksInFork(reln, MAIN_FORKNUM)`, and `MAIN_FORKNUM` is the main relation fork. For indexes, `RelationGetNumberOfBlocksInFork()` opens storage-manager state if needed and returns `smgrnblocks(relation->rd_smgr, forkNum)` ([bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199), [relpath.h#ForkNumber](../../../raw/postgres-12/src/include/common/relpath.h#L40-L46), [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2791-L2811)).

That storage-manager open is lightweight. `RelationOpenSmgr` attaches an `SMgrRelation` only when `rd_smgr` is null, and `smgropen()` looks up or creates the `SMgrRelation` object without opening the underlying relation file. The real file-length work begins when `smgrnblocks()` dispatches through the storage-manager table; PostgreSQL 12's built-in table maps `smgr_nblocks` to `mdnblocks()` for the magnetic-disk storage manager ([rel.h#RelationOpenSmgr](../../../raw/postgres-12/src/include/utils/rel.h#L472-L479), [smgr.c#smgropen](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L139-L178), [smgr.c#smgrsw](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L40-L83), [smgr.c#smgrnblocks](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L630-L640)).

`mdnblocks()` opens the first segment with `mdopen()` if that segment is not already open. `mdopen()` itself opens only the first segment and is a no-op when that segment is already open. `mdnblocks()` then starts from the last open segment, calls `_mdnblocks()` for that segment, and returns `segno * RELSEG_SIZE + nblocks` as soon as it finds a segment smaller than `RELSEG_SIZE`. If a segment is exactly `RELSEG_SIZE`, `mdnblocks()` tries to open the next segment; a missing next segment means the relation length is exactly the number of full segments already found ([md.c#mdopen](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L428-L465), [md.c#mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L698-L755)).

`_mdnblocks()` does not read relation blocks. It calls `FileSize(seg->mdfd_vfd)`, checks for an error, and divides the byte length by `BLCKSZ`, ignoring any partial trailing block. `FileSize()` opens the virtual file descriptor if needed and returns `lseek(VfdCache[file].fd, 0, SEEK_END)`. `_mdfd_openseg()` opens later segment files when `mdnblocks()` has to discover more full segments, and assertion-enabled builds can add extra `_mdnblocks()` checks there; those checks are still file-size probes, not page reads ([md.c#_mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1229-L1242), [fd.c#FileSize](../../../raw/postgres-12/src/backend/storage/file/fd.c#L2039-L2053), [md.c#_mdfd_openseg](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1056-L1089)).

So the PostgreSQL relation-page I/O for `info->pages = RelationGetNumberOfBlocks(indexRelation)` is zero: this path invokes the `smgr_nblocks` callback, not the `smgr_read` callback, and it does not bring B-tree pages into shared buffers. At the operating-system level, the common already-open single-segment case is a file-size seek on one segment descriptor plus function-call, switch, arithmetic, and virtual-file-descriptor overhead. If the backend has not opened the relation file yet, add an `SMgrRelation` hash lookup or creation and a `PathNameOpenFile()` for the first segment. If the index spans multiple relation segment files, a first discovery pass can open and size each full segment until it reaches the final partial segment or a missing next segment. Relation segment files are capped by `RELSEG_SIZE * BLCKSZ`, and the source comment says the default segment limit is 1GB, so the loop scales with segment files, not with B-tree pages or tuples ([smgr.c#smgrsw](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L40-L83), [md.c#mdopen](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L428-L465), [md.c#mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L698-L755), [pg_config.h.in#RELSEG_SIZE](../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)).

The CPU cost is therefore `O(1)` for a normal index whose last open segment is the only segment that needs checking, and `O(number_of_segment_files_probed)` for a large multi-segment relation or first-time segment discovery. It is not `O(index_pages)` and not `O(index_tuples)`. Wall time can still include filesystem metadata latency: a cold directory entry, inode, or file descriptor open can make the kernel perform metadata I/O, but PostgreSQL is not transferring index page contents for this planner field ([md.c#mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L698-L755), [fd.c#FileSize](../../../raw/postgres-12/src/backend/storage/file/fd.c#L2039-L2053)).

`genericcostestimate()` estimates the number of index pages touched as a pro-rata fraction of total index pages:

```c
if (index->pages > 1 && index->tuples > 1)
    numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
else
    numIndexPages = 1.0;
```

It then charges random page cost per touched page for a single scan, or applies `index_pages_fetched()` to model cache effects for repeated scans such as nested-loop inner index scans and scalar-array scans ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L878)).

This means an ordinary non-partial B-tree with twice as many physical pages for the same estimated useful tuples will usually look more expensive for range scans, bitmap index scans, and broad index-only scans. The effect is weaker for a highly selective point lookup because `ceil()` often keeps the estimated leaf pages at one, and a tiny index of one page or one tuple is floored to a single page by the `index->pages > 1 && index->tuples > 1` guard ([selfuncs.c#genericcostestimate-guard](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5777-L5780)).

### 2. B-tree height has an explicit bloat charge

After `genericcostestimate()`, `btcostestimate()` adds a B-tree descent CPU charge. It first charges about `log2(index->tuples) * cpu_operator_cost` for comparisons ([selfuncs.c#btcostestimate-descent-comparisons](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6097-L6102)), then adds:

```c
descentCost = (index->tree_height + 1) * 50.0 * cpu_operator_cost;
```

The comment says this prevents bloated indexes from appearing to have the same search cost as unbloated ones when only a single leaf page is expected. `get_relation_info()` records `tree_height` from `_bt_getrootheight()` while it has the B-tree open ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417)).

The planner height is the fast-root height. `_bt_getrootheight()` returns the cached metapage's `btm_fastlevel`, and its comment says this represents the levels a search descends through. `pgstatindex()` reports `btm_level` as `tree_level`, so after B-tree page deletion creates a fast root, `pgstatindex.tree_level` and planner `tree_height` need not be the same number ([nbtpage.c#_bt_getrootheight](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L575-L626), [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110), [pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L239-L249)).

This is the most explicit planner-side bloat penalty in the v12 B-tree code. It is still not a density or fragmentation penalty; it only changes when the B-tree level changes.

### 3. Index size participates in cache modeling

`index_pages_fetched()` estimates cache effects with the Mackert-Lohman formula. It adds the caller's `index_pages` to `root->total_table_pages` when prorating `effective_cache_size`, so a larger index increases the estimated cache competition term used in both index page costing and heap page costing around index scans ([costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L878), [costsize.c#cost_index](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L470-L665)).

The page-cost constants are global or tablespace-level estimates. `seq_page_cost` defaults to 1.0, and `random_page_cost` defaults to 4.0; lowering `random_page_cost` makes index scans look cheaper, raising it makes them look more expensive, but this is not a per-index bloat signal ([config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765)). `seq_page_cost`, `random_page_cost`, and `effective_cache_size` are all `PGC_USERSET` (`user`-context) GUCs, so a session can change them with `SET` for the current session or transaction; none needs a server restart or a config reload ([guc.c#seq-random-page-cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3207-L3227), [guc.c#effective-cache-size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3108-L3113)).

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

`pgstatindex()` reports `deleted_pages` for pages marked deleted and `empty_pages` for the remaining `P_IGNORE` case, which is the half-dead state in this code path. The diagnostic's `index_size` includes leaf, internal, empty, deleted, and metapage blocks, so this kind of bloat appears as physical relation size even though those pages do not hold useful live keys ([pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L264-L344), [nbtree.h#P_IGNORE](../../../raw/postgres-12/src/include/access/nbtree.h#L186-L195)).

The planner's physical page count does not subtract those diagnostic categories. For ordinary non-partial indexes, it reads the index relation's current block count; for partial indexes, the index branch of `estimate_rel_size()` reports current blocks as pages ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L972)).

This can make a bloated index look more expensive even when many extra pages are not useful scan pages. The model is deliberately coarse: `genericcostestimate()` treats touched index pages as a pro-rata fraction of total pages and says this simplistic method effectively counts leaf pages while ignoring metapage and upper-level overhead ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

This page-count inflation is not always permanent. A deleted B-tree page that has aged past its recyclable horizon can be handed back out by a later page split, because `_bt_getbuf(rel, P_NEW)` asks `GetFreeIndexPage()` for a free page before extending the relation. So under ongoing insert load some deleted pages are reused rather than left as permanent dead weight in `index->pages` — the same FSM reuse described in the fragmented leaf-chains example below ([nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).

### Taller B-trees

If extra pages are enough to add another planner-visible B-tree search level, point lookups and narrow scans get a direct cost increase from the `tree_height + 1` descent charge. The B-tree metapage stores both `btm_level` and `btm_fastlevel`; `pgstatindex()` reports `btm_level` as `tree_level`, while planner costing uses `_bt_getrootheight()` and therefore `btm_fastlevel` ([nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110), [pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L239-L249), [nbtpage.c#_bt_getrootheight](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L575-L626), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).

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

## Proposed Reindex Candidate Heuristic

Treat `avg_leaf_density` and `leaf_fragmentation` as triage signals, not as automatic `REINDEX` triggers. PostgreSQL 12 exposes both fields through `pgstatindex(regclass)` (the installed default extension version 1.5 re-creates the function with the same output columns), and `pgstatindex_impl()` computes them by scanning B-tree pages; the planner still does not read either value, and the v12 manuals describe REINDEX-worthy B-tree bloat and physical-adjacency benefits without defining percentage cutoffs ([pgstattuple--1.4.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L73), [pgstattuple--1.4--1.5.sql#pgstatindex-regclass-15](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92), [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L895), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)).

Use a size and workload gate before acting on either metric. The signal matters most when the index is large, growing faster than the live key set, or used by range scans, ordered index scans, bitmap index scans, or index-only scans that walk many leaf pages. Those are the cases where PostgreSQL's page-count costing and executor leaf-page traversal make extra leaf pages or poor physical order visible in cost or wall-clock time ([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1800), [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335), [nodeIndexonlyscan.c#visibility-map-check](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L121-L170)).

### Ordinary Non-Partial Indexes

For an ordinary non-partial B-tree, low density is the cleaner REINDEX signal. `get_relation_info()` sets `info->pages` to the current index block count and `info->tuples` to the parent table's tuple estimate, so extra pages for the same useful key population feed directly into the `index->pages / index->tuples` term used by `genericcostestimate()` ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).

| Signal | Proposed candidate band | Interpretation for ordinary indexes |
|---|---|---|
| Low `avg_leaf_density` alone | Watch below about 70%; strong candidate below about 50-60% when the index is large or scan-heavy. | A default-built B-tree targets 90% leaf fillfactor, so 70% density implies roughly `90 / 70 = 1.29x` as many leaf pages as a 90% baseline, and 60% implies `1.5x`. This is a practical threshold, not a PostgreSQL-defined rule ([create_index.sgml#fillfactor](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L369-L390), [pgstatindex.c#leaf-density](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L356)). |
| High `leaf_fragmentation` alone | Watch around 30%; strong candidate above about 50% only when broad leaf-chain scans are latency-sensitive. | The metric counts backward physical jumps in the logical leaf chain, so it points at lost locality rather than wasted space. A rebuilt B-tree can help because freshly built B-trees usually place logically adjacent pages physically adjacent, but the metric does not encode jump distance or cache residency ([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L881-L889)). |
| Low density and high fragmentation together | Strong candidate even if neither metric is individually extreme, especially for repeated range or ordered scans. | Low density increases the page population the planner can see through `index->pages`; fragmentation can add storage-locality cost that the planner does not see. This combination is where `REINDEX` can plausibly improve both estimated cost and runtime locality ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)). |

### Partial Indexes

For a partial B-tree, use the same `pgstatindex()` metrics, but be more conservative before calling it a planner-cost problem. `get_relation_info()` does not set `info->tuples` to the parent table estimate for partial indexes. It calls `estimate_rel_size()`, which still reports the current index block count as `pages` but estimates tuples from the partial index's own catalog tuple density after discounting the metapage; `get_relation_info()` then clamps that tuple estimate to the parent table size ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026)).

That means a low `avg_leaf_density` partial index is still a REINDEX candidate when it is large or scan-heavy, but the planner penalty is less direct than for an ordinary index. The strongest partial-index candidates are those where low density or many empty/deleted pages coincide with a large current `index_size`, a stable or shrinking predicate-matching row set, and observed plans or `EXPLAIN (ANALYZE, BUFFERS)` output showing expensive scans. High `leaf_fragmentation` remains a runtime-locality signal rather than a planner input, so it matters most for partial indexes that support broad ordered or range scans over the predicate slice ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L889), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).

Keep small indexes and point-lookup-only indexes low in the queue unless height, relation size, or observed latency changed. `REINDEX` rewrites the index and normally takes an `ACCESS EXCLUSIVE` lock; `REINDEX CONCURRENTLY` reduces the lock level but is still maintenance work, so the better candidate is an index whose diagnostic signal lines up with size, workload shape, and observed query cost or buffer behavior ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L890-L895), [ref/reindex.sgml#reindex-use-cases](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L32-L72), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).

## Tests And Coverage

The `pgstattuple` regression test checks `pgstatindex()` entry points and empty-index output, but it does not create populated B-trees with non-`NaN` density or nonzero fragmentation assertions ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [pgstattuple.out#empty-index-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L47-L85)).

The core `btree_index` regression test is present in the v12 regression schedule. Its test body creates a deliberately tall B-tree with `fillfactor = 10` and covers B-tree page deletion plus FSM page recycling after `VACUUM`, but it does not compare planner choices for the same logical index at different `avg_leaf_density` or `leaf_fragmentation` levels ([serial_schedule#btree_index](../../../raw/postgres-12/src/test/regress/serial_schedule#L100), [parallel_schedule#btree_index](../../../raw/postgres-12/src/test/regress/parallel_schedule#L71), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L162)).

## Open Questions

- PostgreSQL 12 does not define a universal `avg_leaf_density` or `leaf_fragmentation` threshold for `REINDEX`; the proposed candidate bands above are operational triage values, while the docs describe conditions and tradeoffs rather than a percentage cutoff ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L895), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55)).
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
| The planner calls each index's `amcostestimate`; the page-count penalty is shared by B-tree, hash, GiST, and SP-GiST via `genericcostestimate()`; GIN and BRIN use separate cost models; the measured tree-height charge is B-tree-only, GiST and SP-GiST reuse the descent formula with a height estimated as `log100(index->pages)`, and hash adds no descent charge | [nbtree.c#bthandler](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L134), [plancat.c#amcostestimate](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L276-L277), [costsize.c#cost_index-amcostestimate](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L544-L548), [selfuncs.c#btcostestimate-genericcost](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6084), [selfuncs.c#gincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6672), [selfuncs.c#brincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6985), [plancat.c#get_relation_info-tree-height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L417), [selfuncs.c#gistcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6283-L6317), [selfuncs.c#spgcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6340-L6374), [selfuncs.c#hashcostestimate-no-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6237-L6260) |
| B-tree page count is read from the current index relation for ordinary indexes and from `estimate_rel_size()` for partial indexes | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026) |
| Planner B-tree height uses `_bt_getrootheight()` / `btm_fastlevel`; `pgstatindex.tree_level` reports `btm_level` | [nbtpage.c#_bt_getrootheight](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L575-L626), [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110), [pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L239-L249) |
| `RelationGetNumberOfBlocks()` obtains the main-fork length through storage-manager segment size checks, not a B-tree page scan | [bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199), [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2791-L2811), [smgr.c#smgrnblocks](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L630-L640), [md.c#mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L698-L755), [md.c#_mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1229-L1242), [fd.c#FileSize](../../../raw/postgres-12/src/backend/storage/file/fd.c#L2039-L2053) |
| `pgstatindex()` computes density, fragmentation, and page category diagnostics outside planner costing | [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365) |
| A reindex-candidate heuristic can use `avg_leaf_density` and `leaf_fragmentation`, but PostgreSQL 12 defines no source-level cutoff; ordinary and partial indexes also differ in how `get_relation_info()` estimates tuples for page-count costing | [pgstattuple--1.4.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L73), [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L895), [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L407), [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026) |
| B-tree range scans walk sibling-linked leaf pages through `_bt_next()`, `_bt_steppage()`, `_bt_readnextpage()`, and `_bt_getbuf()` | [nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284), [nbtsearch.c#_bt_next](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1334-L1381), [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1619-L1724), [nbtsearch.c#forward-readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1800), [nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820) |
| B-tree bloat and physical adjacency are documented maintenance concerns | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L861-L889), [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L49-L55) |
| Fillfactor shapes initial B-tree density and split behavior | [create_index.sgml#fillfactor](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L369-L390), [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171) |
| Existing tests do not compare planner decisions by density or fragmentation level | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [serial_schedule#btree_index](../../../raw/postgres-12/src/test/regress/serial_schedule#L100), [parallel_schedule#btree_index](../../../raw/postgres-12/src/test/regress/parallel_schedule#L71), [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123), [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L162) |

## Context Reviewed

- Required wiki navigation: [versions](../../versions.md), [wiki index](../../index.md), [v12/index](../index.md), and recent [log](../../log.md).
- PostgreSQL 12 planner, costing, and block-count sources: `plancat.c`, `selfuncs.c`, `costsize.c`, `bufmgr.h`, `bufmgr.c`, `rel.h`, `relpath.h`, `smgr.c`, `md.c`, `fd.c`, `pg_config.h.in`, and `config.sgml`.
- Index access method cost-estimate dispatch and per-AM cost functions: the `amcostestimate` wiring in `plancat.c` and `costsize.c`, the B-tree handler in `nbtree.c`, and `btcostestimate`, `hashcostestimate`, `gistcostestimate`, `spgcostestimate`, `gincostestimate`, and `brincostestimate` in `selfuncs.c` to confirm which cost functions delegate to `genericcostestimate()` and how each handles the descent charge (B-tree measured fast-root height, GiST/SP-GiST synthetic `log100(pages)` height, hash none).
- PostgreSQL 12 B-tree sources: `nbtree.c`, `nbtsearch.c`, `nbtpage.c`, `nbtinsert.c`, and `nbtree.h`.
- PostgreSQL 12 diagnostics, docs, and tests: `pgstatindex.c`, `pgstattuple--1.4.sql`, `pgstattuple.sql`, `pgstattuple.out`, `maintenance.sgml`, `ref/reindex.sgml`, `ref/create_index.sgml`, `serial_schedule`, and `parallel_schedule`.

## Source References

- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417)
- [plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L1026)
- [plancat.c#amcostestimate](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L276-L277)
- [bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199)
- [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2791-L2811)
- [relpath.h#ForkNumber](../../../raw/postgres-12/src/include/common/relpath.h#L40-L46)
- [rel.h#RelationOpenSmgr](../../../raw/postgres-12/src/include/utils/rel.h#L472-L479)
- [smgr.c#smgropen](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L139-L178)
- [smgr.c#smgrsw](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L40-L83)
- [smgr.c#smgrnblocks](../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L630-L640)
- [md.c#mdopen](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L428-L465)
- [md.c#mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L698-L755)
- [md.c#_mdfd_openseg](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1056-L1089)
- [md.c#_mdnblocks](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1229-L1242)
- [fd.c#FileSize](../../../raw/postgres-12/src/backend/storage/file/fd.c#L2039-L2053)
- [pg_config.h.in#RELSEG_SIZE](../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)
- [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5680-L5815)
- [selfuncs.c#btcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5920-L6116)
- [selfuncs.c#hashcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6226-L6235)
- [selfuncs.c#hashcostestimate-no-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6237-L6260)
- [selfuncs.c#gistcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6270-L6281)
- [selfuncs.c#gistcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6283-L6317)
- [selfuncs.c#spgcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6327-L6338)
- [selfuncs.c#spgcostestimate-descent](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6340-L6374)
- [selfuncs.c#gincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6672)
- [selfuncs.c#brincostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6985)
- [costsize.c#cost_index](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L470-L665)
- [costsize.c#index_pages_fetched](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L787-L878)
- [pgstattuple--1.4.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L73)
- [pgstattuple--1.4--1.5.sql#pgstatindex-regclass-15](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L365)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L30-L68)
- [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110)
- [nbtree.h#P_IGNORE](../../../raw/postgres-12/src/include/access/nbtree.h#L186-L195)
- [nbtree.h#fillfactor](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)
- [nbtree.c#bthandler](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L134)
- [nbtree.c#btgettuple](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L213-L284)
- [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335)
- [nbtpage.c#_bt_getrootheight](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L575-L626)
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
- [guc.c#seq-random-page-cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3207-L3227)
- [guc.c#effective-cache-size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3108-L3113)
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)
- [pgstattuple.out#empty-index-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L47-L85)
- [serial_schedule#btree_index](../../../raw/postgres-12/src/test/regress/serial_schedule#L100)
- [parallel_schedule#btree_index](../../../raw/postgres-12/src/test/regress/parallel_schedule#L71)
- [btree_index.sql#tall-fillfactor](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L110-L123)
- [btree_index.sql#page-recycling](../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L147-L162)
