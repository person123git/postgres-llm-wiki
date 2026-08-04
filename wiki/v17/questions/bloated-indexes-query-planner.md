---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
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
  - [Leaf fragmentation contributes exactly zero](#leaf-fragmentation-contributes-exactly-zero)
  - [Two indexes with the same cost and opposite avg_leaf_density](#two-indexes-with-the-same-cost-and-opposite-avg_leaf_density)
  - [The planner reads the live block count, not pg_class.relpages](#the-planner-reads-the-live-block-count-not-pg_classrelpages)
  - [Bloat changes plans](#bloat-changes-plans)
  - [Bloat changes parallel worker counts](#bloat-changes-parallel-worker-counts)
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
- [Open Questions](#open-questions)
- [Related Pages](#related-pages)
- [Evidence Map](#evidence-map)
- [Context Reviewed](#context-reviewed)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, are there mechanisms to penalize bloated indexes in the query planner? If there are, give a comprehensive explanation with examples of types of bloated indexes and how leaf fragmentation or density affects them, and what changed since PostgreSQL 12.

## Answer

Yes, but every mechanism is indirect. PostgreSQL 17 stores no planner field called `bloat`, `avg_leaf_density`, or `leaf_fragmentation`, and `pg_class` carries only `relpages`, `reltuples`, and `relallvisible` ([pg_class.h#relpages](../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)). Bloat is penalized only when it shows up as one of exactly four planner inputs:

| Planner input | Filled by | Penalizes bloat? |
|---|---|---|
| `IndexOptInfo.pages` | live block count from the storage manager (non-partial) or `estimate_rel_size()` (partial) | Yes, this is the main channel |
| `IndexOptInfo.tree_height` | `_bt_getrootheight()`, B-tree only | Yes, one explicit charge per extra level |
| `index_pages` passed into cache and parallel-worker modeling | the same `pages` value | Yes, second-order |
| `ceil(index->pages * 0.3333333)` descent clamp | the same `pages` value | Yes, and this channel is new in v17 |

`get_relation_info()` fills all of them while it has the index open ([plancat.c#get_relation_info](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)), and the access method's `amcostestimate` turns them into cost ([costsize.c#cost_index-amcostestimate](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)).

The consequences are sharply uneven, and measurements on an isolated server built from this exact pin make that concrete:

- A **broad scan** is penalized in direct proportion to page count. A 1,000,000-row index at 90.06% leaf density (2,745 blocks) costs `28480.42` for a full index-only scan; its byte-for-byte logical twin at 9.62% density (26,411 blocks) costs `123144.43`.
- A **point lookup** is almost blind to bloat. The same two indexes both cost exactly `4.44`. The only bloat signal available to a single-leaf-page search is tree height, worth `(tree_height + 1) * 50 * cpu_operator_cost`, which is `0.125` per extra level at default settings.
- **Leaf fragmentation is worth exactly zero.** A 1,101-block index at 49.73% `leaf_fragmentation` and a 744-block index at 0% differ in cost by `1428.00`, which is precisely `(1101 - 744) * random_page_cost`. There is no residual for fragmentation.
- **`avg_leaf_density` and planner cost can point in opposite directions.** Two 2,745-block indexes over the same 100,000 rows cost an identical `12730.42`, while `pgstatindex` reports 9.27% density for one and a healthy-looking 89.18% for the other (the second hides 2,465 deleted pages).

The v17 manual describes B-tree bloat operationally as an index that "contains many empty or nearly-empty pages" and recommends `REINDEX` ([ref/reindex.sgml#bloated](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)), notes that a page keeps its space when "all but a few index keys on a page have been deleted" ([maintenance.sgml#routine-reindex](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1034)), and separately says a freshly built B-tree is slightly faster because logically adjacent pages are usually physically adjacent ([maintenance.sgml#fresh-index](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1048)). That last effect is exactly what `leaf_fragmentation` exposes and exactly what the planner ignores.

Since PostgreSQL 12, the cost formulas themselves barely moved: `index_pages_fetched()` and `cost_index()` are byte-identical to their `REL_12_0` text, and the bloat-relevant core of `genericcostestimate()` is unchanged. What changed is (a) one new v17 penalty channel through `index->pages`, and (b) a great deal of nbtree work in v13 and v14 that reduces how much bloat exists to be penalized in the first place.

## Planner Mechanisms

### How the planner obtains its index-size inputs

`get_relation_info()` opens each index, copies the AM's `amcostestimate` into the `IndexOptInfo` ([plancat.c#amcostestimate](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L331-L332)), then fills the size estimates. For a non-partial, non-partitioned index it takes the current block count directly from the storage manager and locks the tuple estimate to the parent table's:

```c
if (info->indpred == NIL)
{
    info->pages = RelationGetNumberOfBlocks(indexRelation);
    info->tuples = rel->tuples;
}
```

([plancat.c#get_relation_info-index-size](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486))

Three consequences follow directly:

1. **Bloat is visible immediately, without `ANALYZE`.** `pages` is a live `smgrnblocks()` answer, not `pg_class.relpages`. Measured below: forging `pg_class.relpages` to `1` leaves the cost unchanged.
2. **Removing index entries does not lower `tuples`.** For a non-partial index, `tuples` is the *table's* row estimate, so the `pages / tuples` ratio rises purely as pages accumulate.
3. **Partial indexes take a different path.** `estimate_rel_size()` still reports live blocks as `*pages`, but derives `*tuples` from `pg_class` tuple density after discounting the metapage ([plancat.c#estimate_rel_size-index](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)). A partial index therefore inherits stale-statistics behavior that an ordinary index does not.

For B-trees only, the height comes from the metapage while the index is open; every other AM is left at `-1` ([plancat.c#get_relation_info-tree-height](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500)). Partitioned indexes have no storage, so all three fields are zeroed ([plancat.c#get_relation_info-partitioned](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)).

### 1. Physical page count enters index cost

`genericcostestimate()` estimates touched index pages as a pro-rata share of the whole index, and its comment states it counts only leaf pages, ignoring the metapage and upper levels:

```c
if (index->pages > 1 && index->tuples > 1)
    numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
else
    numIndexPages = 1.0;
```

([selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6716-L6731))

For a single scan it charges `spc_random_page_cost` per touched page ([selfuncs.c#genericcostestimate-single-scan](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6779-L6786)); for repeated scans it first runs the page count through the Mackert-Lohman cache model ([selfuncs.c#genericcostestimate-mackert-lohman](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6738-L6778)).

This is the dominant penalty, and it scales linearly with bloat for anything that reads a meaningful fraction of the index. Two guards blunt it: the `index->pages > 1 && index->tuples > 1` test floors a tiny index at one page, and `ceil()` collapses every selective lookup to one page regardless of how bloated the index is.

### 2. B-tree height carries an explicit anti-bloat charge

`btcostestimate()` adds two CPU charges after delegating to `genericcostestimate()`. The first is roughly `log2(N)` comparisons ([selfuncs.c#btcostestimate-log2-descent](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7074-L7090)). The second exists specifically to stop bloated indexes from looking free:

```c
descentCost = (index->tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost;
```

([selfuncs.c#btcostestimate-page-descent](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7092-L7105))

The in-tree comment is explicit: "if we had no such charge at all, bloated indexes would appear to have the same search cost as unbloated ones, at least in cases where only a single leaf page is expected to be visited." `DEFAULT_PAGE_CPU_MULTIPLIER` is `50.0` ([selfuncs.c:145](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), so each extra level costs `50 * cpu_operator_cost` = `0.125` at the default `cpu_operator_cost` of `0.0025`.

Two caveats matter. First, this is a *level* charge, not a density charge: it changes only when the B-tree gains or loses a level. Second, the planner's height is the **fast-root** level, not the true root level. `_bt_getrootheight()` returns `btm_fastlevel` ([nbtpage.c#_bt_getrootheight](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)), and page deletion can lower `btm_fastlevel` in place ([nbtpage.c#fastroot-update](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)); the nbtree README explains the fast-root idea and states that tree height can never *decrease* by page deletion alone ([README#page-deletion-and-tree-height](../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)). `pgstatindex` reports `btm_level` as `tree_level` ([pgstatindex.c#metapage](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265)), so `pgstatindex.tree_level` and the planner's `tree_height` are not guaranteed to be the same number.

### 3. Index pages enter cache modeling and parallel worker counts

`index_pages_fetched()` prorates `effective_cache_size` across "all the tables in the query and the index currently under consideration":

```c
total_pages = root->total_table_pages + index_pages;
b = (double) effective_cache_size * T / total_pages;
```

([costsize.c#index_pages_fetched](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951))

`root->total_table_pages` counts only non-dummy *table* pages ([allpaths.c#total_table_pages](../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216)), so the index's own page count is added separately at every call site. `cost_index()` passes `index->pages` on all four heap-fetch estimates ([costsize.c#cost_index-heap-fetches](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)), `genericcostestimate()` passes it for repeated index scans, and `compute_bitmap_pages()` passes `get_indexpath_pages()` for repeated bitmap scans ([costsize.c#compute_bitmap_pages](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476)). A bloated index therefore claims a larger notional share of cache for itself while shrinking the share available to the heap.

The same `index_pages` value chooses parallel workers. `cost_index()` hands it to `compute_parallel_worker()` ([costsize.c#cost_index-parallel](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)), which rejects a parallel path outright below `min_parallel_index_scan_size` and otherwise ramps the worker count by powers of three ([allpaths.c#compute_parallel_worker](../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)). Bloat can therefore change the *shape* of a plan, not only its price.

### 4. v17 only: index pages cap the ScalarArrayOp descent estimate

`btcostestimate()` clamps the number of estimated array descents to one third of the index's physical pages:

```c
num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333));
num_sa_scans = Max(num_sa_scans, 1);
```

([selfuncs.c#btcostestimate-saop-clamp](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7020-L7041))

`num_sa_scans` then multiplies both descent charges and the per-tuple CPU cost, and is handed to `genericcostestimate()` through `GenericCosts` ([selfuncs.c#btcostestimate-genericcost](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7066-L7072)). The clamp was introduced with v17's native `ScalarArrayOpExpr` execution and does not exist in `REL_12_0`; see [v17: index pages now cap ScalarArrayOp descents](#v17-index-pages-now-cap-scalararrayop-descents). Its effect on bloat is counter-intuitive but real: a bloated index has a *higher* cap, so it is charged for more descents than its dense twin on the identical query.

### Which access methods share these mechanisms

Exactly four AMs route through `genericcostestimate()`: B-tree, hash, GiST and SP-GiST ([selfuncs.c#genericcostestimate-callers](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7072)). GIN and BRIN have their own models and never call it; both source comments say their search behavior is "completely different from other index types" ([selfuncs.c#gincostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L7670), [selfuncs.c#brincostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8051-L8060)).

The height charge is B-tree-only in the sense that only B-trees supply a *measured* height. GiST and SP-GiST fill the `-1` themselves by assuming a fanout of 100 and taking `log100(index->pages)`, then apply the same formula "calculated the same as for btrees" ([selfuncs.c#gistcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7255-L7307), [selfuncs.c#spgcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7310-L7362)); their descent charge therefore tracks physical page count rather than real height. `hashcostestimate()` adds no descent charge at all ([selfuncs.c#hashcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7212-L7252)).

Every `pgstatindex` metric discussed on this page is B-tree-only: the function reads B-tree page structures and refuses other AMs.

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

([pgstatindex.c#page-classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372))

So `avg_leaf_density` answers "how full are the leaf pages that still hold data", and `leaf_fragmentation` answers "how often does the right-sibling link point backwards". Neither answers "how many blocks is the planner charged for", which is the only question the cost model asks. The measurement section shows a case where `avg_leaf_density` reads 89.18% on an index that is ten times larger than it needs to be.

## Types Of Bloated Indexes

### Low-density live leaf pages

The classic case the manual describes: scattered deletions leave every leaf page allocated but nearly empty ([maintenance.sgml#routine-reindex](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1034)). Measured: deleting 90% of 1,000,000 rows with `id % 10 <> 0` and vacuuming twice left the index at **2,745 blocks with 2,733 live leaf pages, zero deleted pages, and 9.27% `avg_leaf_density`**. This is the one bloat shape that both `avg_leaf_density` and the planner agree on, because low density here means high `pages / tuples`.

Split policy sets the ceiling on density. A rightmost leaf split uses the index fillfactor, an internal split uses `BTREE_NONLEAF_FILLFACTOR`, an ordinary leaf split uses `0.50`, and an all-duplicates page uses `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#fillfactor-policy](../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtsplitloc.c#single-value](../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L399-L413)). Those constants are `90`, `70`, and `96` ([nbtree.h#fillfactors](../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202)), and the per-index `fillfactor` reloption takes `ShareUpdateExclusiveLock` because it only affects later inserts ([reloptions.c#btree-fillfactor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)).

### Deleted and half-dead pages

When a leaf page becomes completely empty, VACUUM can delete it, but the page stays in the relation as a tombstone with its sibling links intact, labelled with a `safexid` ([nbtpage.c#page-deleted](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2648)). It becomes recyclable through the FSM only once no scan can still hold a reference ([README#placing-deleted-pages-in-the-fsm](../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)). Ordinary VACUUM does not shorten the index fork, so those blocks remain inside `RelationGetNumberOfBlocks()` and remain charged.

Measured: deleting a *contiguous* 90% (`id > 100000`) let VACUUM empty whole pages, giving **2,745 blocks made of 276 live leaf pages plus 2,465 deleted pages**, and `avg_leaf_density` of **89.18%**. A second VACUUM did not shrink the fork. `REINDEX` cut it to 276 blocks. This is the shape where `avg_leaf_density` is actively misleading and the planner is right.

### Extra tree levels

Because a taller tree means more descent pages, the level charge fires. Measured: 50,000 rows produced a 139-block index at `fastlevel = 1`, while the same 50,000 rows with `fillfactor = 10` produced a 1,323-block index at `fastlevel = 2`. Point-lookup cost differed by exactly `0.125`. Note the asymmetry the README calls out: the height can grow with bloat but never shrinks through deletion alone, only through a rebuild.

### Physically fragmented leaf chains

`leaf_fragmentation` counts leaf pages whose right sibling lives at a lower block number. It rises when pages split in the middle of the key space and the new right half is appended at the end of the file, which is what random-order insertion produces. The manual ties this to real runtime cost ([maintenance.sgml#fresh-index](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1048)). The planner never reads it. Measured contribution to cost: exactly `0.00`.

### Version-churn duplicates from non-HOT UPDATEs

An `UPDATE` that modifies any indexed column writes a new index entry in *every* index, including indexes whose own columns did not change ([btree.sgml#version-churn](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L638-L655)). Those entries are logically unchanged duplicates. Since v14, nbtree attacks them with bottom-up index deletion passes triggered when a version-churn page split is anticipated ([btree.sgml#bottom-up-deletion](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L678), [nbtdedup.c#_bt_bottomupdel_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L307-L320), [nbtinsert.c#delete-then-dedup](../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785)).

Measured on the pin: five whole-table non-HOT `UPDATE` rounds over 200,000 rows grew an unrelated index from 169 to **583 blocks**. Repeating the identical workload with a long-lived `REPEATABLE READ` snapshot open in another session, which is the condition the README names as blocking deletion, grew it to **1,174 blocks** instead.

### Duplicate-heavy indexes with deduplication disabled

Deduplication merges duplicate leaf tuples into posting lists, lazily, at the point a page would otherwise split ([btree.sgml#deduplication](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800), [README#notes-about-deduplication](../../../raw/postgres-17/src/backend/access/nbtree/README#L904-L948), [nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L45-L70)). It is on by default and can be turned off per index with the `deduplicate_items` reloption ([reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)).

Measured: 1,000,000 rows over 100 distinct keys built **852 blocks** with `deduplicate_items = on` and **2,749 blocks** with it off, a 3.2x difference in the planner's `pages` input for identical logical content.

### Bloat that VACUUM deliberately skipped

Two v14-era escape hatches let index bloat accumulate without any VACUUM touching it:

- The 2% bypass. `lazy_vacuum()` skips index vacuuming when fewer than `BYPASS_THRESHOLD_PAGES` (2% of `rel_pages`) hold `LP_DEAD` items and the TID store is under 32MB ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1947)).
- The wraparound failsafe, which makes the ongoing VACUUM bypass all further index vacuuming ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2340)).

`vacuum_index_cleanup = off` has the same effect by request, and the manual warns it "may also lead to severely bloated indexes if table modifications are frequent" ([ref/create_table.sgml#vacuum_index_cleanup](../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1558-L1575)).

### Non-B-tree bloat

The manual states plainly that "the potential for bloat in non-B-tree indexes has not been well researched" and recommends monitoring physical size ([maintenance.sgml#non-btree-bloat](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1036-L1040)). Hash, GiST and SP-GiST still get the page-count penalty through `genericcostestimate()`. GIN and BRIN do not: their cost models are separate, and for GIN the pending list is a bloat source with its own cost consequences, covered in [How a GIN Index Becomes Bloated in PostgreSQL 17 (unverified)](gin-index-bloat.md).

## How Density And Fragmentation Affect Different Queries

| Query shape | Sensitivity to extra pages | Sensitivity to extra levels | Sensitivity to fragmentation |
|---|---|---|---|
| Equality point lookup on a unique or highly selective key | None: `ceil()` keeps `numIndexPages` at 1 | Full: the only signal | None |
| Range scan / broad index-only scan | Linear in `pages / tuples` | One charge per level | None |
| Bitmap index scan feeding a `BitmapAnd` | Linear, and can get the index dropped by `choose_bitmap_and()` ([indxpath.c#choose_bitmap_and](../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287)) | One charge per level | None |
| Nested-loop inner index scan | Sub-linear: Mackert-Lohman damps repeated fetches | Charged once per loop | None |
| `= ANY (array)` on a B-tree | Linear, plus the v17 descent clamp raises the descent count | Charged once per estimated descent | None |
| Parallel index or index-only scan | Threshold and worker count both move | One charge per level | None |
| GIN / BRIN | Separate cost models | Not applicable | None |

The practical shape of this: bloat mostly hurts plans that were already reading a lot of the index, and it barely registers on the selective lookups that dominate OLTP traffic. That is why a badly bloated index can keep serving primary-key lookups with an almost unchanged plan and price while quietly wrecking reporting queries over the same table.

## Exact-Pin Measurements

### Fixtures and method

All numbers below come from one isolated server built from the pinned checkout (`PostgreSQL 17.10 on x86_64-pc-linux-gnu`), with `autovacuum = off`, `shared_buffers = 256MB`, and otherwise default planner settings (`random_page_cost = 4`, `seq_page_cost = 1`, `cpu_tuple_cost = 0.01`, `cpu_index_tuple_cost = 0.005`, `cpu_operator_cost = 0.0025`, `effective_cache_size = 4GB`). `pgstattuple` supplied `pgstatindex`, `pageinspect` supplied `bt_metap()` so the planner's fast-root height could be read directly, and `pg_visibility` confirmed visibility-map state.

Method notes that matter for reading the numbers:

- Each comparison uses **two separate tables with identical contents**, so every query has exactly one candidate index and no index-choice tie-breaking is involved.
- Comparisons use `Index Only Scan` on tables vacuum-frozen to 100% all-visible, so the heap component of `cost_index()` is zero and the reported cost is the pure index cost.
- `enable_seqscan` and `enable_bitmapscan` were disabled where a specific scan type had to be priced; they were reset for the plan-choice tests.

| Fixture | Contents | Dense index | Bloated twin |
|---|---|---|---|
| A | 1,000,000 rows, `int` key | `a_dense_idx`, 2,745 blocks, 90.06% density, `fastlevel` 2 | `a_sparse_idx`, `fillfactor = 10`, 26,411 blocks, 9.62% density, `fastlevel` 2 |
| B | 1,000,000 rows, then `DELETE` of 90% by `id % 10 <> 0` | after `REINDEX`: 276 blocks, 89.83%, `fastlevel` 1 | before: 2,745 blocks, 9.27%, `fastlevel` 2 |
| G | 300,000 rows | `g_seq_idx`, 744 blocks, 99.89% density, 0% fragmentation | `g_frag_idx` built by random-order insert, 1,101 blocks, 67.45% density, **49.73% fragmentation** |
| H | 50,000 rows | `h_l1_idx`, 139 blocks, `fastlevel` 1 | `h_l2_idx`, 1,323 blocks, `fastlevel` 2 |
| I | 2,000 rows | `i_small_idx`, 8 blocks | `i_big_idx`, 55 blocks |
| M | 1,000,000 rows, then `DELETE` of a contiguous 90% | after `REINDEX`: 276 blocks | 2,745 blocks = 276 live leaves + **2,465 deleted pages**, 89.18% density |
| N | 1,000,000 rows over 100 distinct keys | `deduplicate_items = on`, 852 blocks | `deduplicate_items = off`, 2,749 blocks |
| P | 200,000 rows, five whole-table non-HOT `UPDATE` rounds | no blocking snapshot: 169 -> 583 blocks | `REPEATABLE READ` snapshot held: 169 -> 1,174 blocks |

### Index cost is a closed form in pages, tuples and tree height

Fixture A, whole-index scan (`WHERE id > 0`, selectivity exactly 1 so both row estimates are 1,000,000):

| Index | Blocks | Density | `fastlevel` | Observed total cost | Predicted from `(pages, tuples, fastlevel)` |
|---|---:|---:|---:|---:|---:|
| `a_dense_idx` | 2,745 | 90.06% | 2 | `28480.42` | `28480.43` |
| `a_sparse_idx` | 26,411 | 9.62% | 2 | `123144.43` | `123144.43` |

The prediction is `pages * random_page_cost + qual_op_cost + tuples * (cpu_index_tuple_cost + qual_op_cost) + ceil(log2(tuples)) * cpu_operator_cost + (fastlevel + 1) * 50 * cpu_operator_cost + rows * cpu_tuple_cost`, computed in SQL from the catalog and `bt_metap()`. It matches to the cent (the one-cent gap on the dense row is rounding in the SQL expression, not in the planner). A 9.62x larger index costs 4.32x more.

Fixture A, 10% range scan: `3136.70` dense against `12502.38` bloated.

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

With `cpu_operator_cost = 1` the startup costs are `116.00` and `166.00`. Both decompose exactly: `ceil(log2(50000)) = 16` comparison charges plus `(fastlevel + 1) * 50`. The gap is `50.00`, exactly `DEFAULT_PAGE_CPU_MULTIPLIER`. At default settings the same gap is `0.125`, so a 9.5x larger index raises a point-lookup cost by about 1.5%.

### Leaf fragmentation contributes exactly zero

Fixture G, both tables frozen to 100% all-visible so the index-only scan pays no heap cost:

| Index | Blocks | Density | Fragmentation | `fastlevel` | Observed | Predicted |
|---|---:|---:|---:|---:|---:|---:|
| `g_seq_idx` | 744 | 99.89% | **0.00%** | 2 | `8226.42` | `8226.43` |
| `g_frag_idx` | 1,101 | 67.45% | **49.73%** | 2 | `9654.42` | `9654.43` |

The observed gap is `9654.42 - 8226.42 = 1428.00`. The page-count gap is `1101 - 744 = 357`, and `357 * random_page_cost = 357 * 4.0 = 1428.00`. The residual attributable to a 49.73-point fragmentation difference is `0.00`, and the closed-form prediction, which has no fragmentation term, reproduces both costs.

### Two indexes with the same cost and opposite avg_leaf_density

Fixtures B and M both end at 2,745 blocks over 100,000 surviving rows, and both price a whole-index scan at exactly `12730.42`, dropping to `2854.29` after `REINDEX`. Their `pgstatindex` output could hardly be more different:

| Fixture | Deletion pattern | Blocks | Live leaf pages | Deleted pages | `avg_leaf_density` | Whole-index cost |
|---|---|---:|---:|---:|---:|---:|
| B | scattered (`id % 10 <> 0`) | 2,745 | 2,733 | 0 | **9.27%** | `12730.42` |
| M | contiguous (`id > 100000`) | 2,745 | 276 | **2,465** | **89.18%** | `12730.42` |

A monitoring rule that flags low `avg_leaf_density` catches fixture B and completely misses fixture M, even though the planner is charged the same 2,745 pages in both and a rebuild recovers 10x in both. Physical size against row count is the reliable signal; `avg_leaf_density` alone is not.

### The planner reads the live block count, not pg_class.relpages

A 200,000-row table with a `fillfactor = 10` index occupying 5,285 blocks priced its whole-index scan at `25528.42`. Forging the catalog on the scratch server:

```text
UPDATE pg_class SET relpages = 1, reltuples = 200000 WHERE relname = 'b_stale_idx';
-- pg_class.relpages = 1, real size = 5285 blocks
Index Only Scan using b_stale_idx on b_stale  (cost=0.42..25528.42 rows=200000 width=4)
```

The cost did not move, confirming that `get_relation_info()` used `RelationGetNumberOfBlocks()`. A **partial** index behaves differently, because `estimate_rel_size()` derives its tuple count from `pg_class` density: forging `reltuples` from 200,000 to 20 on an otherwise identical partial index moved the cost from `24140.12` to `23140.29` and the startup cost from `0.42` to `0.39`.

### Bloat changes plans

Fixture A with all `enable_*` settings at their defaults, selecting a non-indexed column so a heap fetch is required:

| Selectivity | Dense index | Bloated twin |
|---|---|---|
| ~25% (`id BETWEEN 1 AND 250000`) | `Index Scan using a_dense_idx` at `9731.06` | `Seq Scan` at `22353.00`; the index path is rejected |
| ~12% (`id BETWEEN 1 AND 120000`) | `Index Scan` at `4706.74` | `Index Scan` at `16026.44` |

Bloat also removes an index from a `BitmapAnd`. With two healthy indexes on a 500,000-row table, `a = 5 AND c = 7` produced a `BitmapAnd` over both (total `328.41`, the `c` bitmap costing `276.30`). Bloating only the `c` index from 428 to 3,801 blocks (89.59% to 10.37% density) made the planner drop it and demote `c = 7` to a `Filter`:

```text
Bitmap Heap Scan on l3  (cost=6.29..722.82 rows=13 width=0)
  Recheck Cond: (a = 5)
  Filter: (c = 7)
  ->  Bitmap Index Scan on l3_a  (cost=0.00..6.29 rows=249 width=0)
```

### Bloat changes parallel worker counts

With `max_parallel_workers_per_gather = 8`, `max_parallel_workers = 8`, `min_parallel_table_scan_size = 0` and `enable_seqscan = off`, fixture A's parallel index-only scans differed only in the index:

| Index | Blocks | Workers planned |
|---|---:|---:|
| `a_dense_idx` | 2,745 | 4 |
| `a_sparse_idx` | 26,411 | 6 |

Both counts follow `compute_parallel_worker()`'s powers-of-three ramp from `min_parallel_index_scan_size` (default 512kB, that is 64 blocks). Bloat bought two extra workers for no extra useful data.

### The v17 ScalarArrayOp descent clamp

Fixture I, with `cpu_operator_cost = 1` so each estimated descent is worth about 116 cost units and the clamp is directly readable. The clamp is `ceil(pages * 0.3333333)`: `3` for the 8-block index, `19` for the 55-block index.

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

Fixture P: a table with indexes on `payload` and `tag`, where `UPDATE ... SET payload = payload + 1` is non-HOT and therefore writes a logically unchanged duplicate into the `tag` index on every round. No VACUUM ran.

| Run | `tag` index blocks after 5 rounds | Density | Fragmentation | `fastlevel` | Bitmap index scan cost |
|---|---:|---:|---:|---:|---:|
| No blocking snapshot | 583 (from 169) | 97.31% | 62.46% | 1 -> 2 | `26.35` |
| `REPEATABLE READ` snapshot held elsewhere | 1,174 (from 169) | 86.73% | 57.03% | 2 | `30.30` |

Six row versions exist per logical row after five rounds. The blocked run grew roughly 6.9x, close to storing every version; the unblocked run grew 3.4x. The difference is the work bottom-up index deletion was able to do, and the README names an old snapshot holding up cleanup as exactly the condition that defeats it ([README#deduplication-in-unique-indexes](../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). Note also that the unblocked run's density reads 97.31%, above its 90 fillfactor, because the single-value split strategy and deduplication pack duplicate-heavy pages tighter.

### Deduplication

Fixture N, 1,000,000 rows over 100 distinct keys, two indexes on the same column:

| Index | Blocks | Density | `WHERE k = 5` | `WHERE k > 0` |
|---|---:|---:|---:|---:|
| `deduplicate_items = on` | 852 | 89.70% | `13438.87` | `33945.86` |
| `deduplicate_items = off` | 2,749 | 90.16% | `13506.87` | `41461.86` |

Both gaps are pure page arithmetic. For `k > 0`, touched pages are `ceil(990367 * 852 / 1000000) = 844` against `ceil(990367 * 2749 / 1000000) = 2723`, and `(2723 - 844) * 4.0 = 7516.00`, exactly the observed `41461.86 - 33945.86`. For `k = 5`, `(26 - 9) * 4.0 = 68.00`, exactly the observed gap. Note that `avg_leaf_density` is ~90% in both cases: it cannot see that one index stores 3.2x more pages for the same information.

## What Changed Since PostgreSQL 12

Every attribution below is anchored to the pinned v17 checkout's own history, which contains the `REL_12_0` through `REL_17_0` tags. A commit is assigned to the first release tag that contains it.

### The cost code that did not change at all

Comparing the `REL_12_0` file text with the pin:

- `index_pages_fetched()` is byte-identical.
- `cost_index()` is byte-identical.
- The bloat-relevant core of `genericcostestimate()` is unchanged: the `numIndexPages` prorating formula, the `index->pages > 1 && index->tuples > 1` guard, the Mackert-Lohman call, and the single-scan `numIndexPages * spc_random_page_cost` charge. The only diff in that function is the v17 change to accept a caller-supplied `num_sa_scans`.
- Exactly five commits touched `btcostestimate()` since `REL_12_0`, and only two are functional: `5bf748b86bc` and `9391f71523b`. The remaining three are a typo fix (`950d4a2cb1d`), a `MemSet`-to-struct-initializer change (`9fd45870c14`), and the macro extraction (`eb5c4e953bb`).

So a reader who knows the v12 model already knows most of the v17 model.

### v16: the 50x page charge became a macro

`eb5c4e953bb` "Extract the multiplier for CPU process cost of index page into a macro" (2023-01-08, first in `REL_16_0`) replaced three literal `50.0` occurrences with `DEFAULT_PAGE_CPU_MULTIPLIER`. `REL_12_0` already charged `(index->tree_height + 1) * 50.0 * cpu_operator_cost` in `btcostestimate`, `gistcostestimate` and `spgcostestimate`. The value and the behavior are unchanged; only the spelling moved ([selfuncs.c:145](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).

The companion commit `cd9479af2af` "Improve GIN cost estimation" (also first in `REL_16_0`) newly applied the same per-page CPU multiplier inside `gincostestimate()` ([selfuncs.c#gincostestimate-page-charges](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7954)), which changes how GIN bloat is priced but does not touch the B-tree path.

### v17: index pages now cap ScalarArrayOp descents

`5bf748b86bc` "Enhance nbtree ScalarArrayOp execution." (2024-04-06, first in `REL_17_0`) is the only commit since v12 that added a *new* way for a bloated B-tree to be charged more. It introduced the clamp `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))`, moved `num_sa_scans` into `GenericCosts` so `btcostestimate()` can hand its own estimate to `genericcostestimate()`, and reworded the descent comments from "per SA scan" to "per estimated SA index descent" ([selfuncs.c#btcostestimate-saop-clamp](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7020-L7041)).

In `REL_12_0` there was no clamp: `num_sa_scans` was simply the product of array lengths, computed inside `genericcostestimate()`. The measured consequence in v17 is that `index->pages` now sets the ceiling on descent charges, so the dense and bloated twins in fixture I diverge by 3.25x on a ten-element `= ANY` where v12's formula had no page-count input at that point at all.

The related commit `9391f71523b` "Teach estimate_array_length() to use statistics where available." (also `REL_17_0`) changed how the *unclamped* array length is estimated, which feeds the same variable.

### v16: partitioned indexes are zeroed out

`3c569049b7b` "Allow left join removals and unique joins on partitioned tables" (2023-01-09, first in `REL_16_0`) added the `RELKIND_PARTITIONED_INDEX` guard that sets `pages = 0`, `tuples = 0.0`, `tree_height = -1` for partitioned indexes ([plancat.c#get_relation_info-partitioned](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)). This is the only structural change to `get_relation_info()`'s index-size block since `REL_12_0`.

### v14: reltuples turns negative for never-analyzed relations

`3d351d916b2` "Redefine pg_class.reltuples to be -1 before the first VACUUM or ANALYZE." (2020-08-30, first in `REL_14_0`) changed `estimate_rel_size()`'s index branch from `if (relpages > 0)` to `if (reltuples >= 0 && relpages > 0)` ([plancat.c#estimate_rel_size-density](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)), and the catalog header now documents `-1` as "unknown" ([pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). A never-analyzed partial index therefore falls through to the attribute-width density fallback in v17 instead of computing a density from a zero tuple count. Non-partial indexes are unaffected, because they never consult `reltuples`.

### v13: deduplication

`0d861bbb702` "Add deduplication to nbtree." (2020-02-26, first in `REL_13_0`) introduced posting-list tuples, the lazy pre-split deduplication pass, and the `deduplicate_items` reloption, none of which exist in `REL_12_0`'s `reloptions.c`. It does not change any cost formula; it changes how many pages a duplicate-heavy index needs, and therefore what the unchanged formula is fed. Fixture N measures 852 blocks against 2,749 for the same data with the feature disabled.

### v14: bottom-up index deletion

`d168b666823` "Enhance nbtree index tuple deletion." (2021-01-13, first in `REL_14_0`) added `_bt_bottomupdel_pass()`. The v17 documentation states the boundary directly: "Prior to PostgreSQL 14, the only category of B-Tree deletion was simple deletion" ([btree.sgml#simple-vs-bottom-up](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703)), and the README says bottom-up index deletion "was added to PostgreSQL 14" ([README#added-to-postgresql-14](../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). The docs claim it is possible for such an index's on-disk size to "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)); fixture P measures a 3.4x growth under a deliberately harsh five-round whole-table churn with no VACUUM, against 6.9x when an old snapshot blocks deletion.

### v14: faster recycling of deleted pages

`9dd963ae253` "Recycle nbtree pages deleted during same VACUUM." (2021-03-21, first in `REL_14_0`). The README describes the change in its own words: before v14 VACUUM placed only *previously* deleted pages in the FSM, and "PostgreSQL 14 added the ability for VACUUM to consider if it's possible to recycle newly deleted pages at the end of the full index scan where the page deletion took place" ([README#postgresql-14-fsm-change](../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424)). This shortens the window in which deleted pages are dead weight, but it never shrinks the fork, so the planner keeps paying for them until a rebuild. Fixture M measures 2,465 deleted pages still inside a 2,745-block index after two VACUUMs.

### v14: VACUUM can skip index vacuuming entirely

Two `REL_14_0` commits created bloat paths that do not exist in v12: `5100010ee4d` "Teach VACUUM to bypass unnecessary index vacuuming." (the 2% `BYPASS_THRESHOLD_PAGES` rule) and `1e55e7d1755` "Add wraparound failsafe to VACUUM.". A third, `3499df0dee8` "Support disabling index bypassing by VACUUM." (2021-06-18, `REL_14_0`), added the tri-valued `auto` handling so `INDEX_CLEANUP`/`vacuum_index_cleanup` can force cleanup back on. The `vacuum_index_cleanup` reloption itself already existed in `REL_12_0` as a plain boolean.

### Since-v12 summary table

| Change | First release | Effect on planner-visible bloat |
|---|---|---|
| `0d861bbb702` deduplication | 13 | Fewer pages for duplicate-heavy indexes; measured 3.2x |
| `3d351d916b2` `reltuples = -1` | 14 | Different fallback for never-analyzed partial indexes |
| `d168b666823` bottom-up index deletion | 14 | Fewer pages under non-HOT `UPDATE` churn; measured ~2x |
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
| `random_page_cost` | `DEFAULT_RANDOM_PAGE_COST` (4.0) | Multiplies every touched index page ([guc_tables.c#random_page_cost](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3685-L3695)) | session/transaction |
| `cpu_operator_cost` | `DEFAULT_CPU_OPERATOR_COST` (0.0025) | Scales the height charge and the log2 descent charge ([guc_tables.c#cpu_operator_cost](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3718-L3728)) | session/transaction |
| `effective_cache_size` | `DEFAULT_EFFECTIVE_CACHE_SIZE` | Splits notional cache between heap and index pages ([guc_tables.c#effective_cache_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3507-L3517)) | session/transaction |
| `min_parallel_index_scan_size` | 512kB ([guc_tables.c#min_parallel_index_scan_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3530-L3539)) | Threshold and ramp for index-driven worker counts | session/transaction |

Two per-index storage parameters change the physical layout rather than its price, and both take `ShareUpdateExclusiveLock` because they apply only to later inserts: `fillfactor` ([reloptions.c#btree-fillfactor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)) and `deduplicate_items` ([reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)). Changing either only affects existing pages after a rebuild, which is one of the scenarios `REINDEX` documents ([ref/reindex.sgml#storage-parameter](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L66-L71)).

## Practical Interpretation

- **Rank rebuild candidates by physical size against useful rows, not by `avg_leaf_density`.** The planner is charged for blocks, and fixture M shows a 10x-oversized index reporting 89.18% density.
- **Do not expect a bloated index to be abandoned by OLTP queries.** Point lookups are charged only the height difference: `0.125` per level at default settings, about 1.5% on a typical lookup.
- **Expect plan changes on the analytical side.** Fixture A flipped to a sequential scan at 25% selectivity, and a bloated index was dropped from a `BitmapAnd` entirely.
- **Treat `leaf_fragmentation` as a runtime concern only.** It has real I/O consequences the manual acknowledges, and exactly zero cost-model consequences.
- **Watch for `= ANY (...)` regressions on v17 specifically.** The descent clamp means bloat is now charged through a channel that did not exist before v17.
- **A bloated partial index is priced through a different path.** Its tuple estimate comes from `pg_class`, so its cost can be wrong in either direction until it is analyzed.

## Key Data Structures

| Structure | Field | Role |
|---|---|---|
| `IndexOptInfo` | `pages`, `tuples`, `tree_height` | The complete set of size inputs available to any `amcostestimate` ([pathnodes.h#IndexOptInfo](../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)) |
| `RelOptInfo` | `pages`, `tuples`, `allvisfrac` | Parent-table estimates; `tuples` is copied into a non-partial index's `tuples` ([pathnodes.h#RelOptInfo](../../../raw/postgres-17/src/include/nodes/pathnodes.h#L941-L944)) |
| `PlannerInfo` | `total_table_pages` | Table-only page total used to prorate `effective_cache_size` ([pathnodes.h#total_table_pages](../../../raw/postgres-17/src/include/nodes/pathnodes.h#L483-L484)) |
| `GenericCosts` | `numIndexPages`, `num_sa_scans` | Shared cost scratchpad; `num_sa_scans` became an input in v17 |
| `BTMetaPageData` | `btm_level`, `btm_fastlevel` | True root level versus the fast-root level the planner uses ([nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)) |
| `LVRelState` | `consider_bypass_optimization`, `do_index_vacuuming` | Controls whether VACUUM touches indexes at all ([vacuumlazy.c#LVRelState](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L148-L156)) |
| `pg_class` | `relpages`, `reltuples`, `relallvisible` | The only physical-size catalog columns; no density or fragmentation column exists ([pg_class.h#relpages](../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)) |

## Caller And Callee Boundary

```text
query_planner
  └─ get_relation_info                       plancat.c
       ├─ RelationGetNumberOfBlocks          -> smgrnblocks (non-partial index)
       ├─ estimate_rel_size                  -> pg_class density (partial index)
       └─ _bt_getrootheight                  -> btm_fastlevel (B-tree only)
  └─ make_one_rel -> set_base_rel_sizes -> root->total_table_pages
  └─ create_index_paths                      indxpath.c
       ├─ cost_index                         costsize.c
       │    ├─ amcostestimate  ==  btcostestimate / hash / gist / spgist / gin / brin
       │    │    └─ genericcostestimate      (bt, hash, gist, spgist only)
       │    │         └─ index_pages_fetched (repeated scans)
       │    ├─ index_pages_fetched           (heap fetches, 4 call sites)
       │    └─ compute_parallel_worker       allpaths.c
       ├─ choose_bitmap_and                  indxpath.c
       │    └─ bitmap_scan_cost_est -> cost_bitmap_heap_scan -> compute_bitmap_pages
       └─ add_path -> compare_path_costs_fuzzily   pathnode.c
```

Symbols: [plancat.c#get_relation_info](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508), [costsize.c#cost_index](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L621), [selfuncs.c#genericcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6625-L6827), [costsize.c#index_pages_fetched](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951), [allpaths.c#compute_parallel_worker](../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279), [indxpath.c#choose_bitmap_and](../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287), [indxpath.c#bitmap_scan_cost_est](../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526), [pathnode.c#compare_path_costs_fuzzily](../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L164).

On the write side, the code that decides how many pages exist runs entirely inside nbtree: [nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334) chooses split points, [nbtinsert.c#delete-then-dedup](../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785) tries bottom-up deletion and then deduplication before splitting, and [nbtpage.c#page-deleted](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659) marks pages deleted and can lower the fast-root level.

## Build, Generated-Header, And Extension Boundary

- Nothing on this path depends on a generated parser or catalog artifact at cost time. `pg_class.relpages` / `reltuples` / `relallvisible` come from the hand-written catalog header `src/include/catalog/pg_class.h`, whose `BKI_DEFAULT(-1)` on `reltuples` encodes the v14 redefinition ([pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). The `pg_class` struct that `estimate_rel_size()` reads is materialized through the normal `genbki.pl` header generation, so a change to that column's type or default would require a catalog version bump, not just a recompile.
- `DEFAULT_PAGE_CPU_MULTIPLIER` is a private `#define` inside `selfuncs.c` ([selfuncs.c:145](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), not an exported symbol and not a GUC. An extension cannot change the height charge except by changing `cpu_operator_cost`.
- `pathnodes.h` deliberately types `IndexOptInfo.amcostestimate` weakly to avoid including `amapi.h`, so `cost_index()` casts it before calling ([costsize.c#cost_index-amcostestimate](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). A custom index AM therefore participates in bloat pricing entirely through its own `amcostestimate`, and gets `tree_height = -1` unless it is a B-tree.
- The two supported hooks on this path are `get_relation_stats_hook` and `get_index_stats_hook`, which `btcostestimate()` consults for correlation statistics only ([selfuncs.c#btcostestimate-stats-hooks](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7115-L7170)). Neither can substitute a page count or a tree height.
- Every density and fragmentation metric on this page comes from `contrib`, not core: `pgstatindex` in `pgstattuple` and `bt_metap()` in `pageinspect`. Core SQL exposes physical size (`pg_relation_size`) and the catalog columns, and nothing else about index page structure.
- Within the 17.x series, `036decbba2a` "pgstattuple: Improve reports generated for indexes (hash, gist, btree)" (first in `REL_17_7`, backpatched through 13) added a `BTPageOpaqueData` size check to `pgstattuple`'s B-tree handling and made zero pages count as free space. The pin at 17.10 contains it. It changes `pgstattuple()` output, not `pgstatindex()` density or fragmentation.

## Tests And Explicit Test Absence

- **No test exercises the bloat charge.** `src/test` contains no reference to `tree_height`, `btcostestimate`, or `genericcostestimate` anywhere. There is no regression test that asserts a bloated index is costed higher than an unbloated one.
- **The only in-tree `pgstatindex` test uses an empty index**, so it asserts nothing about density or fragmentation. It creates `test (a int primary key, b int[])` with no rows and expects `avg_leaf_density` and `leaf_fragmentation` to be `NaN` ([sql/pgstattuple.sql#pgstatindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)). The `NaN` comes from the `max_avail > 0` and `leaf_pages > 0` guards ([pgstatindex.c#density-and-fragmentation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
- The nbtree features that limit bloat do have tests, but they test correctness, not size: deduplication and deletion are covered through `src/test/regress/sql/btree_index.sql` and `contrib/amcheck`, neither of which asserts page counts.
- All measurements on this page were therefore produced ad hoc on an isolated exact-pin server, not by any in-tree test.

## Open Questions

- The one-cent gaps between observed and predicted costs (`28480.42` vs `28480.43`, `8226.42` vs `8226.43`, `9654.42` vs `9654.43`) are consistent with rounding inside the verification SQL rather than in the planner, because the exact-match cases (`123144.43`) use the same expression. Reproducing the prediction in C `double` arithmetic would settle it.
- Fixture P's growth ratios (3.4x unblocked, 6.9x blocked) are a single measurement of one deliberately harsh workload with autovacuum off. The documentation's stronger claim, that some indexes "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)), was not reproduced here and would need a gentler, steady-state workload to test.
- Mechanism 3 (the `effective_cache_size` split between heap and index pages) was demonstrated only indirectly: fixture A's nested-loop inner index-only scan priced at `3.83` per loop dense against `4.37` bloated. Isolating the heap-side share of that difference from the index-side share was not attempted, because `EXPLAIN` does not break out the two components.
- Whether GIN's v16 `DEFAULT_PAGE_CPU_MULTIPLIER` charges make GIN bloat pricing behave qualitatively like the B-tree page charge was not investigated here; the GIN cost model is separate and is covered only to the extent needed to scope this page.
- The claim that a custom index AM gets `tree_height = -1` unless `relam == BTREE_AM_OID` is read straight from `get_relation_info()`, but no custom-AM fixture was built to confirm what such an AM's `amcostestimate` then does with it.
- `pgstatindex.tree_level` (`btm_level`) and the planner's `tree_height` (`btm_fastlevel`) were equal in every fixture measured here, because all fixtures were freshly built or rebuilt. A fixture that forces a fast root to diverge from the true root was not constructed.

## Related Pages

- [How a GIN Index Becomes Bloated in PostgreSQL 17, and How to Measure It (unverified)](gin-index-bloat.md) - GIN's separate bloat mechanisms and cost model in the same version.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)](pgstatindex-sample-variant-proposal.md) - why `pgstatindex` has to read every block to produce the metrics discussed here.
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md) - the online rebuild that resets every input on this page.
- [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](partial-indexes-pros-cons.md) - more on the partial-index costing path that behaves differently here.
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](../../v12/questions/bloated-indexes-query-planner.md) - the same question answered against the v12 pin.

## Evidence Map

| Claim | Evidence |
|---|---|
| Planner has no bloat/density/fragmentation field | [pg_class.h#relpages](../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69), [pathnodes.h#IndexOptInfo](../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128) |
| `pages` comes from the live block count for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486); measured: forged `relpages = 1` left cost at `25528.42` |
| `tuples` is the parent table's estimate for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486) |
| Partial indexes use `pg_class` density | [plancat.c#estimate_rel_size-index](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160); measured: forged `reltuples = 20` moved cost `24140.12` -> `23140.29` |
| Page count enters cost pro-rata | [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6716-L6731); measured `28480.42` vs `123144.43` |
| Height charge exists to stop bloated indexes looking free | [selfuncs.c#btcostestimate-page-descent](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7092-L7105) |
| Height charge is `50 * cpu_operator_cost` per level | [selfuncs.c:145](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145); measured startup gap exactly `50.00` at `cpu_operator_cost = 1` |
| Planner height is the fast-root level | [nbtpage.c#_bt_getrootheight](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717), [README#page-deletion-and-tree-height](../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381) |
| Index pages enter cache modeling | [costsize.c#index_pages_fetched](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951), [costsize.c#cost_index-heap-fetches](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747) |
| Index pages choose parallel workers | [allpaths.c#compute_parallel_worker](../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279); measured 4 vs 6 workers |
| v17 clamps SAOP descents to `ceil(pages/3)` | [selfuncs.c#btcostestimate-saop-clamp](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7020-L7041); measured plateau at 3 vs 19 |
| Only bt/hash/gist/spgist use `genericcostestimate` | [selfuncs.c#genericcostestimate-callers](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7072); [selfuncs.c#gincostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L7670), [selfuncs.c#brincostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8051-L8060) |
| GiST/SP-GiST estimate height from page count | [selfuncs.c#gistcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7255-L7307), [selfuncs.c#spgcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7310-L7362) |
| Hash charges no descent cost | [selfuncs.c#hashcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7212-L7252) |
| `avg_leaf_density`/`leaf_fragmentation` ignore deleted pages | [pgstatindex.c#page-classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372); measured 89.18% on a 2,465-deleted-page index |
| Fragmentation contributes zero cost | measured gap `1428.00` = `357 * 4.0` with 49.73% vs 0% fragmentation |
| Deleted pages stay in the fork | [nbtpage.c#page-deleted](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2648), [README#placing-deleted-pages-in-the-fsm](../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441); measured 2,745 blocks after two VACUUMs |
| Split policy sets density ceilings | [nbtsplitloc.c#fillfactor-policy](../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtree.h#fillfactors](../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202) |
| Bloat can drop an index from a `BitmapAnd` | [indxpath.c#choose_bitmap_and](../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287); measured `c = 7` demoted to `Filter` |
| VACUUM can skip index vacuuming | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1947), [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2340) |
| `index_pages_fetched` / `cost_index` unchanged since `REL_12_0` | byte-identical file text at `REL_12_0` and the pin |
| SAOP clamp is new in v17 | `5bf748b86bc`, first release tag `REL_17_0`; absent from `REL_12_0` `btcostestimate` |
| `50.0` -> macro is cosmetic and v16 | `eb5c4e953bb` diff, first release tag `REL_16_0`; `REL_12_0` already had `* 50.0 *` |
| Deduplication is v13 | `0d861bbb702`, first release tag `REL_13_0`; `deduplicate_items` absent from `REL_12_0` `reloptions.c`; measured 852 vs 2,749 blocks |
| Bottom-up deletion is v14 | `d168b666823`, first release tag `REL_14_0`; [btree.sgml#simple-vs-bottom-up](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703), [README#added-to-postgresql-14](../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988); measured 583 vs 1,174 blocks |
| Newly deleted pages recyclable in the same VACUUM is v14 | `9dd963ae253`, first release tag `REL_14_0`; [README#postgresql-14-fsm-change](../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424) |
| `reltuples = -1` is v14 | `3d351d916b2`, first release tag `REL_14_0`; [pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66) |
| No test covers the bloat charge | no `src/test` match for `tree_height`, `btcostestimate`, `genericcostestimate` |
| The `pgstatindex` test uses an empty index | [sql/pgstattuple.sql#pgstatindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52) |
| All four cost GUCs are session-scoped | [guc_tables.c#random_page_cost](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3685-L3695), [guc_tables.c#cpu_operator_cost](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3718-L3728), [guc_tables.c#effective_cache_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3507-L3517), [guc_tables.c#min_parallel_index_scan_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3530-L3539) |

## Context Reviewed

Planner and cost path: `src/backend/optimizer/util/plancat.c` (`get_relation_info`, `estimate_rel_size`), `src/backend/optimizer/path/costsize.c` (`cost_index`, `index_pages_fetched`, `cost_bitmap_heap_scan`, `compute_bitmap_pages`, `get_indexpath_pages`), `src/backend/optimizer/path/allpaths.c` (`total_table_pages`, `compute_parallel_worker`), `src/backend/optimizer/path/indxpath.c` (`choose_bitmap_and`, `bitmap_scan_cost_est`), `src/backend/optimizer/util/pathnode.c` (`compare_path_costs_fuzzily`), `src/backend/utils/adt/selfuncs.c` (`genericcostestimate`, `btcostestimate`, `hashcostestimate`, `gistcostestimate`, `spgcostestimate`, `gincostestimate`, `brincostestimate`, `add_predicate_to_index_quals`).

nbtree: `nbtpage.c` (`_bt_getrootheight`, fast-root update, page deletion), `nbtsplitloc.c` (`_bt_findsplitloc` fillfactor policy and single-value strategy), `nbtdedup.c` (`_bt_dedup_pass`, `_bt_bottomupdel_pass`), `nbtinsert.c` (pre-split deletion and deduplication), `nbtsort.c`, `src/include/access/nbtree.h`, and the nbtree `README` sections on page deletion, tree height, FSM placement, simple deletion, bottom-up deletion, split policy, and deduplication.

VACUUM: `src/backend/access/heap/vacuumlazy.c` (`BYPASS_THRESHOLD_PAGES`, `lazy_vacuum`, `lazy_check_wraparound_failsafe`, `LVRelState`).

Headers and catalogs: `src/include/nodes/pathnodes.h` (`IndexOptInfo`, `RelOptInfo`, `PlannerInfo`), `src/include/catalog/pg_class.h`, `src/backend/access/common/reloptions.c`, `src/backend/utils/misc/guc_tables.c`.

Contrib: `contrib/pgstattuple/pgstatindex.c` plus its `sql/` and `expected/` regression files, `contrib/pageinspect` `bt_metap()` definitions.

Documentation: `ref/reindex.sgml`, `maintenance.sgml` (routine reindexing), `glossary.sgml` (Bloat), `btree.sgml` (version churn, bottom-up deletion, deduplication), `ref/create_table.sgml` (`vacuum_index_cleanup`), `ref/create_index.sgml`, `indices.sgml`, `pgstattuple.sgml`.

History: `git log -L` on `genericcostestimate`, `btcostestimate`, `get_relation_info`, `index_pages_fetched`, and `cost_index` bounded by `REL_12_0..HEAD`; full-file diffs of those functions against `REL_12_0`; `git tag --contains` for every attributed commit; `contrib/pgstattuple` and `src/backend/access/nbtree` history since `REL_12_0`.

Empirical: one isolated PostgreSQL 17.10 server built from the pin, eight fixture scripts covering density, real deletion bloat, height, fragmentation, catalog forgery, plan flips, `BitmapAnd` pruning, parallel worker counts, the v17 SAOP clamp, version churn with and without a held snapshot, and deduplication.

## Source References

- [plancat.c#get_relation_info](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)
- [plancat.c#estimate_rel_size-index](../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)
- [selfuncs.c#genericcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6625-L6827)
- [selfuncs.c#btcostestimate](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6869-L7210)
- [selfuncs.c:145](../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)
- [costsize.c#cost_index](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L621)
- [costsize.c#index_pages_fetched](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951)
- [costsize.c#compute_bitmap_pages](../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6420-L6518)
- [allpaths.c#compute_parallel_worker](../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)
- [indxpath.c#choose_bitmap_and](../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287)
- [pathnodes.h#IndexOptInfo](../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)
- [pg_class.h#relpages](../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)
- [nbtpage.c#_bt_getrootheight](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)
- [nbtpage.c#page-deleted](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659)
- [nbtsplitloc.c#fillfactor-policy](../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334)
- [nbtdedup.c#_bt_bottomupdel_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L307-L320)
- [nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)
- [README#page-deletion-and-tree-height](../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)
- [README#placing-deleted-pages-in-the-fsm](../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)
- [README#bottom-up-deletion](../../../raw/postgres-17/src/backend/access/nbtree/README#L557-L619)
- [vacuumlazy.c#lazy_vacuum-bypass](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1947)
- [reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)
- [pgstatindex.c#density-and-fragmentation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372)
- [ref/reindex.sgml#bloated](../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)
- [maintenance.sgml#routine-reindex](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1048)
- [glossary.sgml#Bloat](../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)
- [btree.sgml#bottom-up-deletion](../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L733)
- [expected/pgstattuple.out#NaN](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)

## Navigation

- [v17/index](../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../codebase-navigation-guide.md)
- [wiki index](../../index.md)
- [versions](../../versions.md)
- [log](../../log.md)
