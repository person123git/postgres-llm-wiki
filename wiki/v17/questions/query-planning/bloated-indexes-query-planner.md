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

## Answer

Yes, but every mechanism is indirect. PostgreSQL 17 stores no planner field called `bloat`, `avg_leaf_density`, or `leaf_fragmentation`, and `pg_class` carries only `relpages`, `reltuples`, and `relallvisible` ([pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)). Bloat is penalized only when it shows up as one of exactly four planner inputs:

| Planner input | Filled by | Penalizes bloat? |
|---|---|---|
| `IndexOptInfo.pages` | live block count from the storage manager (non-partial) or `estimate_rel_size()` (partial) | Yes, this is the main channel |
| `IndexOptInfo.tree_height` | `_bt_getrootheight()`, B-tree only | Yes, one explicit charge per extra level |
| `index_pages` passed into cache and parallel-worker modeling | the same `pages` value | Yes, second-order |
| `ceil(index->pages * 0.3333333)` descent clamp | the same `pages` value | Yes, and this channel is new in v17 |

`get_relation_info()` fills all of them while it has the index open ([plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)), and the access method's `amcostestimate` turns them into cost ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)).

The consequences are sharply uneven, and measurements on an isolated server built from this exact pin make that concrete:

- A **broad scan** is penalized in direct proportion to page count. A 1,000,000-row index at 90.06% leaf density (2,745 blocks) costs `28480.42` for a full index-only scan; its byte-for-byte logical twin at 9.62% density (26,411 blocks) costs `123144.43`.
- A **point lookup** is almost blind to bloat. The same two indexes both cost exactly `4.44`. The only bloat signal available to a single-leaf-page search is tree height, worth `(tree_height + 1) * 50 * cpu_operator_cost`, which is `0.125` per extra level at default settings.
- **Leaf fragmentation is worth exactly zero.** A 1,101-block index at 49.73% `leaf_fragmentation` and a 744-block index at 0% differ in cost by `1428.00`, which is precisely `(1101 - 744) * random_page_cost`. There is no residual for fragmentation.
- **`avg_leaf_density` and planner cost can point in opposite directions.** Two 2,745-block indexes over the same 100,000 rows cost an identical `12730.42`, while `pgstatindex` reports 9.27% density for one and a healthy-looking 89.18% for the other (the second hides 2,465 deleted pages).

The v17 manual describes B-tree bloat operationally as an index that "contains many empty or nearly-empty pages" and recommends `REINDEX` ([ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)), notes that a page keeps its space when "all but a few index keys on a page have been deleted" ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1034)), and separately says a freshly built B-tree is slightly faster because logically adjacent pages are usually physically adjacent ([maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1048)). That last effect is exactly what `leaf_fragmentation` exposes and exactly what the planner ignores.

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

For B-trees only, the height comes from the metapage while the index is open; every other AM is left at `-1` ([plancat.c#get_relation_info-tree-height](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L500)). Partitioned indexes have no storage, so all three fields are zeroed ([plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)).

### 1. Physical page count enters index cost

`genericcostestimate()` estimates touched index pages as a pro-rata share of the whole index, and its comment states it counts only leaf pages, ignoring the metapage and upper levels:

```c
if (index->pages > 1 && index->tuples > 1)
    numIndexPages = ceil(numIndexTuples * index->pages / index->tuples);
else
    numIndexPages = 1.0;
```

([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6716-L6731))

For a single scan it charges `spc_random_page_cost` per touched page ([selfuncs.c#genericcostestimate-single-scan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6779-L6786)); for repeated scans it first runs the page count through the Mackert-Lohman cache model ([selfuncs.c#genericcostestimate-mackert-lohman](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6738-L6778)).

This is the dominant penalty, and it scales linearly with bloat for anything that reads a meaningful fraction of the index. Two guards blunt it: the `index->pages > 1 && index->tuples > 1` test floors a tiny index at one page, and `ceil()` collapses every selective lookup to one page regardless of how bloated the index is.

### 2. B-tree height carries an explicit anti-bloat charge

`btcostestimate()` adds two CPU charges after delegating to `genericcostestimate()`. The first is roughly `log2(N)` comparisons ([selfuncs.c#btcostestimate-log2-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7074-L7090)). The second exists specifically to stop bloated indexes from looking free:

```c
descentCost = (index->tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost;
```

([selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7092-L7105))

The in-tree comment is explicit: "if we had no such charge at all, bloated indexes would appear to have the same search cost as unbloated ones, at least in cases where only a single leaf page is expected to be visited." `DEFAULT_PAGE_CPU_MULTIPLIER` is `50.0` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), so each extra level costs `50 * cpu_operator_cost` = `0.125` at the default `cpu_operator_cost` of `0.0025`.

Two caveats matter. First, this is a *level* charge, not a density charge: it changes only when the B-tree gains or loses a level. Second, the planner's height is the **fast-root** level, not the true root level. `_bt_getrootheight()` returns `btm_fastlevel` ([nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717)), and page deletion can lower `btm_fastlevel` in place ([nbtpage.c#fastroot-update](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2565-L2659)); the nbtree README explains the fast-root idea and states that tree height can never *decrease* by page deletion alone ([README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381)). `pgstatindex` reports `btm_level` as `tree_level` ([pgstatindex.c#metapage](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265)), so `pgstatindex.tree_level` and the planner's `tree_height` are not guaranteed to be the same number.

### 3. Index pages enter cache modeling and parallel worker counts

`index_pages_fetched()` prorates `effective_cache_size` across "all the tables in the query and the index currently under consideration":

```c
total_pages = root->total_table_pages + index_pages;
b = (double) effective_cache_size * T / total_pages;
```

([costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951))

`root->total_table_pages` counts only non-dummy *table* pages ([allpaths.c#total_table_pages](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L183-L216)), so the index's own page count is added separately at every call site. `cost_index()` passes `index->pages` on all four heap-fetch estimates ([costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747)), `genericcostestimate()` passes it for repeated index scans, and `compute_bitmap_pages()` passes `get_indexpath_pages()` for repeated bitmap scans ([costsize.c#compute_bitmap_pages](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6463-L6476)). A bloated index therefore claims a larger notional share of cache for itself while shrinking the share available to the heap.

The same `index_pages` value chooses parallel workers. `cost_index()` hands it to `compute_parallel_worker()` ([costsize.c#cost_index-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L749-L772)), which rejects a parallel path outright below `min_parallel_index_scan_size` and otherwise ramps the worker count by powers of three ([allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279)). Bloat can therefore change the *shape* of a plan, not only its price.

### 4. v17 only: index pages cap the ScalarArrayOp descent estimate

`btcostestimate()` clamps the number of estimated array descents to one third of the index's physical pages:

```c
num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333));
num_sa_scans = Max(num_sa_scans, 1);
```

([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7020-L7041))

`num_sa_scans` then multiplies both descent charges and the per-tuple CPU cost, and is handed to `genericcostestimate()` through `GenericCosts` ([selfuncs.c#btcostestimate-genericcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7066-L7072)). The clamp was introduced with v17's native `ScalarArrayOpExpr` execution and does not exist in `REL_12_0`; see [v17: index pages now cap ScalarArrayOp descents](#v17-index-pages-now-cap-scalararrayop-descents). Its effect on bloat is counter-intuitive but real: a bloated index has a *higher* cap, so it is charged for more descents than its dense twin on the identical query.

### Which access methods share these mechanisms

Exactly four AMs route through `genericcostestimate()`: B-tree, hash, GiST and SP-GiST ([selfuncs.c#genericcostestimate-callers](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7072)). GIN and BRIN have their own models and never call it; both source comments say their search behavior is "completely different from other index types" ([selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L7670), [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8051-L8060)).

The height charge is B-tree-only in the sense that only B-trees supply a *measured* height. GiST and SP-GiST fill the `-1` themselves by assuming a fanout of 100 and taking `log100(index->pages)`, then apply the same formula "calculated the same as for btrees" ([selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7255-L7307), [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7310-L7362)); their descent charge therefore tracks physical page count rather than real height. `hashcostestimate()` adds no descent charge at all ([selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7212-L7252)).

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

([pgstatindex.c#page-classification](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372))

So `avg_leaf_density` answers "how full are the leaf pages that still hold data", and `leaf_fragmentation` answers "how often does the right-sibling link point backwards". Neither answers "how many blocks is the planner charged for", which is the only question the cost model asks. The measurement section shows a case where `avg_leaf_density` reads 89.18% on an index that is ten times larger than it needs to be.

## Types Of Bloated Indexes

### Low-density live leaf pages

The classic case the manual describes: scattered deletions leave every leaf page allocated but nearly empty ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1034)). Measured: deleting 90% of 1,000,000 rows with `id % 10 <> 0` and vacuuming twice left the index at **2,745 blocks with 2,733 live leaf pages, zero deleted pages, and 9.27% `avg_leaf_density`**. This is the one bloat shape that both `avg_leaf_density` and the planner agree on, because low density here means high `pages / tuples`.

Split policy sets the ceiling on density. A rightmost leaf split uses the index fillfactor, an internal split uses `BTREE_NONLEAF_FILLFACTOR`, an ordinary leaf split uses `0.50`, and an all-duplicates page uses `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtsplitloc.c#single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L399-L413)). Those constants are `90`, `70`, and `96` ([nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202)), and the per-index `fillfactor` reloption takes `ShareUpdateExclusiveLock` because it only affects later inserts ([reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)).

### Deleted and half-dead pages

When a leaf page becomes completely empty, VACUUM can delete it, but the page stays in the relation as a tombstone with its sibling links intact, labelled with a `safexid` ([nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2648)). It becomes recyclable through the FSM only once no scan can still hold a reference ([README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441)). Ordinary VACUUM does not shorten the index fork, so those blocks remain inside `RelationGetNumberOfBlocks()` and remain charged.

Measured: deleting a *contiguous* 90% (`id > 100000`) let VACUUM empty whole pages, giving **2,745 blocks made of 276 live leaf pages plus 2,465 deleted pages**, and `avg_leaf_density` of **89.18%**. A second VACUUM did not shrink the fork. `REINDEX` cut it to 276 blocks. This is the shape where `avg_leaf_density` is actively misleading and the planner is right.

### Extra tree levels

Because a taller tree means more descent pages, the level charge fires. Measured: 50,000 rows produced a 139-block index at `fastlevel = 1`, while the same 50,000 rows with `fillfactor = 10` produced a 1,323-block index at `fastlevel = 2`. Point-lookup cost differed by exactly `0.125`. Note the asymmetry the README calls out: the height can grow with bloat but never shrinks through deletion alone, only through a rebuild.

### Physically fragmented leaf chains

`leaf_fragmentation` counts leaf pages whose right sibling lives at a lower block number. It rises when pages split in the middle of the key space and the new right half is appended at the end of the file, which is what random-order insertion produces. The manual ties this to real runtime cost ([maintenance.sgml#fresh-index](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1048)). The planner never reads it. Measured contribution to cost: exactly `0.00`.

### Version-churn duplicates from non-HOT UPDATEs

An `UPDATE` that modifies any indexed column writes a new index entry in *every* index, including indexes whose own columns did not change ([btree.sgml#version-churn](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L638-L655)). Those entries are logically unchanged duplicates. Since v14, nbtree attacks them with bottom-up index deletion passes triggered when a version-churn page split is anticipated ([btree.sgml#bottom-up-deletion](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L678), [nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L307-L320), [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785)).

Measured on the pin: five whole-table non-HOT `UPDATE` rounds over 200,000 rows grew an unrelated index from 169 to **583 blocks**. Repeating the identical workload with a long-lived `REPEATABLE READ` snapshot open in another session, which is the condition the README names as blocking deletion, grew it to **1,174 blocks** instead.

### Duplicate-heavy indexes with deduplication disabled

Deduplication merges duplicate leaf tuples into posting lists, lazily, at the point a page would otherwise split ([btree.sgml#deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L800), [README#notes-about-deduplication](../../../../raw/postgres-17/src/backend/access/nbtree/README#L904-L948), [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L45-L70)). It is on by default and can be turned off per index with the `deduplicate_items` reloption ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)).

Measured: 1,000,000 rows over 100 distinct keys built **852 blocks** with `deduplicate_items = on` and **2,749 blocks** with it off, a 3.2x difference in the planner's `pages` input for identical logical content.

### Bloat that VACUUM deliberately skipped

Two v14-era escape hatches let index bloat accumulate without any VACUUM touching it:

- The 2% bypass. `lazy_vacuum()` skips index vacuuming when fewer than `BYPASS_THRESHOLD_PAGES` (2% of `rel_pages`) hold `LP_DEAD` items and the TID store is under 32MB ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1947)).
- The wraparound failsafe, which makes the ongoing VACUUM bypass all further index vacuuming ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2340)).

`vacuum_index_cleanup = off` has the same effect by request, and the manual warns it "may also lead to severely bloated indexes if table modifications are frequent" ([ref/create_table.sgml#vacuum_index_cleanup](../../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1558-L1575)).

### Non-B-tree bloat

The manual states plainly that "the potential for bloat in non-B-tree indexes has not been well researched" and recommends monitoring physical size ([maintenance.sgml#non-btree-bloat](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1036-L1040)). Hash, GiST and SP-GiST still get the page-count penalty through `genericcostestimate()`. GIN and BRIN do not: their cost models are separate, and for GIN the pending list is a bloat source with its own cost consequences, covered in [How a GIN Index Becomes Bloated in PostgreSQL 17 (unverified)](../indexing/gin-index-bloat.md).

## How Density And Fragmentation Affect Different Queries

| Query shape | Sensitivity to extra pages | Sensitivity to extra levels | Sensitivity to fragmentation |
|---|---|---|---|
| Equality point lookup on a unique or highly selective key | None: `ceil()` keeps `numIndexPages` at 1 | Full: the only signal | None |
| Range scan / broad index-only scan | Linear in `pages / tuples` | One charge per level | None |
| Bitmap index scan feeding a `BitmapAnd` | Linear, and can get the index dropped by `choose_bitmap_and()` ([indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287)) | One charge per level | None |
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

Six row versions exist per logical row after five rounds. The blocked run grew roughly 6.9x, close to storing every version; the unblocked run grew 3.4x. The difference is the work bottom-up index deletion was able to do, and the README names an old snapshot holding up cleanup as exactly the condition that defeats it ([README#deduplication-in-unique-indexes](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). Note also that the unblocked run's density reads 97.31%, above its 90 fillfactor, because the single-value split strategy and deduplication pack duplicate-heavy pages tighter.

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

`eb5c4e953bb` "Extract the multiplier for CPU process cost of index page into a macro" (2023-01-08, first in `REL_16_0`) replaced three literal `50.0` occurrences with `DEFAULT_PAGE_CPU_MULTIPLIER`. `REL_12_0` already charged `(index->tree_height + 1) * 50.0 * cpu_operator_cost` in `btcostestimate`, `gistcostestimate` and `spgcostestimate`. The value and the behavior are unchanged; only the spelling moved ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).

The companion commit `cd9479af2af` "Improve GIN cost estimation" (also first in `REL_16_0`) newly applied the same per-page CPU multiplier inside `gincostestimate()` ([selfuncs.c#gincostestimate-page-charges](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7954)), which changes how GIN bloat is priced but does not touch the B-tree path.

### v17: index pages now cap ScalarArrayOp descents

`5bf748b86bc` "Enhance nbtree ScalarArrayOp execution." (2024-04-06, first in `REL_17_0`) is the only commit since v12 that added a *new* way for a bloated B-tree to be charged more. It introduced the clamp `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))`, moved `num_sa_scans` into `GenericCosts` so `btcostestimate()` can hand its own estimate to `genericcostestimate()`, and reworded the descent comments from "per SA scan" to "per estimated SA index descent" ([selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7020-L7041)).

In `REL_12_0` there was no clamp: `num_sa_scans` was simply the product of array lengths, computed inside `genericcostestimate()`. The measured consequence in v17 is that `index->pages` now sets the ceiling on descent charges, so the dense and bloated twins in fixture I diverge by 3.25x on a ten-element `= ANY` where v12's formula had no page-count input at that point at all.

The related commit `9391f71523b` "Teach estimate_array_length() to use statistics where available." (also `REL_17_0`) changed how the *unclamped* array length is estimated, which feeds the same variable.

### v16: partitioned indexes are zeroed out

`3c569049b7b` "Allow left join removals and unique joins on partitioned tables" (2023-01-09, first in `REL_16_0`) added the `RELKIND_PARTITIONED_INDEX` guard that sets `pages = 0`, `tuples = 0.0`, `tree_height = -1` for partitioned indexes ([plancat.c#get_relation_info-partitioned](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L502-L508)). This is the only structural change to `get_relation_info()`'s index-size block since `REL_12_0`.

### v14: reltuples turns negative for never-analyzed relations

`3d351d916b2` "Redefine pg_class.reltuples to be -1 before the first VACUUM or ANALYZE." (2020-08-30, first in `REL_14_0`) changed `estimate_rel_size()`'s index branch from `if (relpages > 0)` to `if (reltuples >= 0 && relpages > 0)` ([plancat.c#estimate_rel_size-density](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1116-L1146)), and the catalog header now documents `-1` as "unknown" ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). A never-analyzed partial index therefore falls through to the attribute-width density fallback in v17 instead of computing a density from a zero tuple count. Non-partial indexes are unaffected, because they never consult `reltuples`.

### v13: deduplication

`0d861bbb702` "Add deduplication to nbtree." (2020-02-26, first in `REL_13_0`) introduced posting-list tuples, the lazy pre-split deduplication pass, and the `deduplicate_items` reloption, none of which exist in `REL_12_0`'s `reloptions.c`. It does not change any cost formula; it changes how many pages a duplicate-heavy index needs, and therefore what the unchanged formula is fed. Fixture N measures 852 blocks against 2,749 for the same data with the feature disabled.

### v14: bottom-up index deletion

`d168b666823` "Enhance nbtree index tuple deletion." (2021-01-13, first in `REL_14_0`) added `_bt_bottomupdel_pass()`. The v17 documentation states the boundary directly: "Prior to PostgreSQL 14, the only category of B-Tree deletion was simple deletion" ([btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703)), and the README says bottom-up index deletion "was added to PostgreSQL 14" ([README#added-to-postgresql-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988)). The docs claim it is possible for such an index's on-disk size to "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)); fixture P measures a 3.4x growth under a deliberately harsh five-round whole-table churn with no VACUUM, against 6.9x when an old snapshot blocks deletion.

### v14: faster recycling of deleted pages

`9dd963ae253` "Recycle nbtree pages deleted during same VACUUM." (2021-03-21, first in `REL_14_0`). The README describes the change in its own words: before v14 VACUUM placed only *previously* deleted pages in the FSM, and "PostgreSQL 14 added the ability for VACUUM to consider if it's possible to recycle newly deleted pages at the end of the full index scan where the page deletion took place" ([README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424)). This shortens the window in which deleted pages are dead weight, but it never shrinks the fork, so the planner keeps paying for them until a rebuild. Fixture M measures 2,465 deleted pages still inside a 2,745-block index after two VACUUMs.

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
| `random_page_cost` | `DEFAULT_RANDOM_PAGE_COST` (4.0) | Multiplies every touched index page ([guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3685-L3695)) | session/transaction |
| `cpu_operator_cost` | `DEFAULT_CPU_OPERATOR_COST` (0.0025) | Scales the height charge and the log2 descent charge ([guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3718-L3728)) | session/transaction |
| `effective_cache_size` | `DEFAULT_EFFECTIVE_CACHE_SIZE` | Splits notional cache between heap and index pages ([guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3507-L3517)) | session/transaction |
| `min_parallel_index_scan_size` | 512kB ([guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3530-L3539)) | Threshold and ramp for index-driven worker counts | session/transaction |

Two per-index storage parameters change the physical layout rather than its price, and both take `ShareUpdateExclusiveLock` because they apply only to later inserts: `fillfactor` ([reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)) and `deduplicate_items` ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)). Changing either only affects existing pages after a rebuild, which is one of the scenarios `REINDEX` documents ([ref/reindex.sgml#storage-parameter](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L66-L71)).

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
| `IndexOptInfo` | `pages`, `tuples`, `tree_height` | The complete set of size inputs available to any `amcostestimate` ([pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128)) |
| `RelOptInfo` | `pages`, `tuples`, `allvisfrac` | Parent-table estimates; `tuples` is copied into a non-partial index's `tuples` ([pathnodes.h#RelOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L941-L944)) |
| `PlannerInfo` | `total_table_pages` | Table-only page total used to prorate `effective_cache_size` ([pathnodes.h#total_table_pages](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L483-L484)) |
| `GenericCosts` | `numIndexPages`, `num_sa_scans` | Shared cost scratchpad; `num_sa_scans` became an input in v17 |
| `BTMetaPageData` | `btm_level`, `btm_fastlevel` | True root level versus the fast-root level the planner uses ([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)) |
| `LVRelState` | `consider_bypass_optimization`, `do_index_vacuuming` | Controls whether VACUUM touches indexes at all ([vacuumlazy.c#LVRelState](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L148-L156)) |
| `pg_class` | `relpages`, `reltuples`, `relallvisible` | The only physical-size catalog columns; no density or fragmentation column exists ([pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69)) |

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

Symbols: [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508), [costsize.c#cost_index](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L621), [selfuncs.c#genericcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6625-L6827), [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951), [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279), [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287), [indxpath.c#bitmap_scan_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1526), [pathnode.c#compare_path_costs_fuzzily](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L164).

On the write side, the code that decides how many pages exist runs entirely inside nbtree: [nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334) chooses split points, [nbtinsert.c#delete-then-dedup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2770-L2785) tries bottom-up deletion and then deduplication before splitting, and [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2659) marks pages deleted and can lower the fast-root level.

## Build, Generated-Header, And Extension Boundary

- Nothing on this path depends on a generated parser or catalog artifact at cost time. `pg_class.relpages` / `reltuples` / `relallvisible` come from the hand-written catalog header `src/include/catalog/pg_class.h`, whose `BKI_DEFAULT(-1)` on `reltuples` encodes the v14 redefinition ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66)). The `pg_class` struct that `estimate_rel_size()` reads is materialized through the normal `genbki.pl` header generation, so a change to that column's type or default would require a catalog version bump, not just a recompile.
- `DEFAULT_PAGE_CPU_MULTIPLIER` is a private `#define` inside `selfuncs.c` ([selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)), not an exported symbol and not a GUC. An extension cannot change the height charge except by changing `cpu_operator_cost`.
- `pathnodes.h` deliberately types `IndexOptInfo.amcostestimate` weakly to avoid including `amapi.h`, so `cost_index()` casts it before calling ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). A custom index AM therefore participates in bloat pricing entirely through its own `amcostestimate`, and gets `tree_height = -1` unless it is a B-tree.
- The two supported hooks on this path are `get_relation_stats_hook` and `get_index_stats_hook`, which `btcostestimate()` consults for correlation statistics only ([selfuncs.c#btcostestimate-stats-hooks](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7115-L7170)). Neither can substitute a page count or a tree height.
- Every density and fragmentation metric on this page comes from `contrib`, not core: `pgstatindex` in `pgstattuple` and `bt_metap()` in `pageinspect`. Core SQL exposes physical size (`pg_relation_size`) and the catalog columns, and nothing else about index page structure.
- Within the 17.x series, `036decbba2a` "pgstattuple: Improve reports generated for indexes (hash, gist, btree)" (first in `REL_17_7`, backpatched through 13) added a `BTPageOpaqueData` size check to `pgstattuple`'s B-tree handling and made zero pages count as free space. The pin at 17.10 contains it. It changes `pgstattuple()` output, not `pgstatindex()` density or fragmentation.

## Tests And Explicit Test Absence

- **No test exercises the bloat charge.** `src/test` contains no reference to `tree_height`, `btcostestimate`, or `genericcostestimate` anywhere. There is no regression test that asserts a bloated index is costed higher than an unbloated one.
- **The only in-tree `pgstatindex` test uses an empty index**, so it asserts nothing about density or fragmentation. It creates `test (a int primary key, b int[])` with no rows and expects `avg_leaf_density` and `leaf_fragmentation` to be `NaN` ([sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)). The `NaN` comes from the `max_avail > 0` and `leaf_pages > 0` guards ([pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
- The nbtree features that limit bloat do have tests, but they test correctness, not size: deduplication and deletion are covered through `src/test/regress/sql/btree_index.sql` and `contrib/amcheck`, neither of which asserts page counts.
- All measurements on this page were therefore produced ad hoc on an isolated exact-pin server, not by any in-tree test.

## Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead

### Short answer

PostgreSQL 17 discards a GIN index at three separate gates, and only the third one is about cost. A GIN index that clears gates 1 and 2 still loses to a B-tree on the same column for ordinary comparison predicates, because `gincostestimate()` charges `random_page_cost` for every pending, entry and data page it expects to touch and adds a `50 * cpu_operator_cost` charge per page on top ([selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L8049)), while `genericcostestimate()` charges the B-tree only a pro-rata share of `index->pages` plus cheap CPU descent ([selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6716-L6731)).

| Gate | Where | What makes GIN lose | Recovery |
|---|---|---|---|
| 1. Clause matching | `match_clause_to_indexcol()` | The query operator is not in the GIN index's operator family and no planner support function rewrites it, so no `IndexClause` and therefore no GIN path is ever built ([indxpath.c#match_clause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2203-L2269), [indxpath.c#match_opclause_to_indexcol](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2386-L2500)) | Use a matching operator, or add the operators with `contrib/btree_gin` |
| 2. Plan shape | `build_index_paths()` / `get_index_paths()` | GIN has no `amgettuple`, no ordering, no `amcanreturn`, no null search, no native array search and no parallelism, so it cannot produce a plain `Index Scan`, satisfy `ORDER BY` pathkeys, feed an `Index Only Scan`, serve `IS NULL`, or run in parallel ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89), [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767)) | None. These are AM properties, not costs |
| 3. Cost | `gincostestimate()` versus `btcostestimate()`, then `add_path()` / `choose_bitmap_and()` | GIN's page charges are all at `random_page_cost` and include the whole pending list, as startup cost ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7881-L7885), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7975-L7979)) | Clean the pending list, `VACUUM`, or drop `fastupdate` |

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
| `amcanorder` / `amcanorderbyop` | `false` / `false` ([ginutil.c:44](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L44)) | `true` / `false` | `get_relation_info()` fills `sortopfamily` only for `BTREE_AM_OID` or another `amcanorder` AM ([plancat.c#get_relation_info-sortopfamily](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L340-L419)), so `index_is_ordered` is false for GIN and `useful_pathkeys` stays `NIL` ([indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944)) |
| `amcanreturn` | `NULL` ([ginutil.c:70](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L70)) | `btcanreturn` | `index_can_return()` returns false when `amcanreturn` is `NULL` ([indexam.c#index_can_return](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L780-L797)), so every `canreturn[i]` is false and `check_index_only()` fails ([indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800)) |
| `amsearchnulls` | `false` ([ginutil.c:51](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L51)) | `true` | `match_clause_to_indexcol()` accepts a `NullTest` only when `index->amsearchnulls`, so `IS NULL` never reaches GIN ([indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266)) |
| `amsearcharray` | `false` ([ginutil.c:50](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L50)) | `true` | A `ScalarArrayOpExpr` is omitted from plain paths and re-offered only as a bitmap path ([indxpath.c#build_index_paths-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L862-L885), [indxpath.c#get_index_paths-nonnative-saop](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L753-L766)), and `counts.arrayScans` multiplies the GIN estimate ([selfuncs.c#gincost_scalararrayopexpr](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7549-L7659)) |
| `amcanparallel` | `false` ([ginutil.c:55](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L55)) | `true` | No partial GIN path; `build_index_paths()` gates parallel paths on `index->amcanparallel` ([indxpath.c#build_index_paths-parallel](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L975-L1002)) |

Three consequences follow, and none can be reversed by tuning:

- **No plain index scan.** The AM developer documentation explains why: `amgetbitmap` returns tuples in a bitmap that "doesn't have any specific ordering", "Ordering operators will never be supplied for such a scan", and "there is no provision for index-only scans with `amgetbitmap`, since there is no way to return the contents of index tuples" ([indexam.sgml#amgetbitmap](../../../../raw/postgres-17/doc/src/sgml/indexam.sgml#L991-L1010)). Two upstream test comments say the same operationally: "GIN currently supports only bitmap scans, not plain indexscans" and "GIN only supports bitmapscan, so no need to test plain indexscan" ([create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268), [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)).
- **No sorted output.** "Of the index types currently supported by PostgreSQL, only B-tree can produce sorted output — the other index types return matching rows in an unspecified, implementation-dependent order." ([indices.sgml#ordering](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L530-L538)). Even a bitmap plan built from a B-tree loses order, because the bitmap is laid out in physical order ([indices.sgml#bitmap-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L643-L656)).
- **No index-only scan.** "As a counterexample, GIN indexes cannot support index-only scans because each index entry typically holds only part of the original data value" ([indices.sgml#index-only-scans](../../../../raw/postgres-17/doc/src/sgml/indices.sgml#L1125-L1136)), matching the `amcanreturn` contract.

The upstream `amutils` regression test asserts exactly this property matrix for `gin` versus `btree`: `orderable`, `returnable`, `search_array` and `search_nulls` are all `f` for GIN while `bitmap_scan` is `t` and `index_scan` is `f`, and `can_order`, `can_unique`, `can_exclude`, `can_include` are all `f` ([amutils.out#column-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L96-L108), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129), [amutils.out#am-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L152-L157)).

### Gate 3: cost, and why GIN loses on the same column

`cost_index()` calls the AM's `amcostestimate` through the `IndexOptInfo` function pointer, so GIN and B-tree paths for the same clause are priced by different code ([costsize.c#cost_index-amcostestimate](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L610-L621)). `gincostestimate()` builds its estimate like this:

1. Read the metapage counters with `ginGetStats()`. Only `nPendingPages` is current; the rest are as of the last `VACUUM` ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7696-L7710), [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)).
2. Seed the startup page count with the **entire pending list**: `entryPagesFetched = numPendingPages` ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7881-L7885)).
3. Add `ceil(counts.searchEntries * rint(pow(numEntryPages, 0.15)))` entry pages, plus a proportional share of entry and data pages for partial-match keys ([selfuncs.c#gincostestimate-entrypages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7887-L7913)).
4. Charge about `log2(numEntries)` comparisons per search entry for the entry-tree descent ([selfuncs.c#gincostestimate-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7918-L7934)).
5. Charge `DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost`, that is `50 * cpu_operator_cost`, for every entry and data page ([selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7954), [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).
6. Charge all pending and entry pages at `random_page_cost` as **startup** cost, "because logically-close pages could be far apart on disk" ([selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7975-L7979)).
7. Add scan-time data pages, taking the larger of the per-entry estimate and a selectivity-derived floor of `ceil(indexSelectivity * numTuples / (BLCKSZ / 3))`, again at `random_page_cost`, then per-qual CPU with no descent-height charge and no ordering support ([selfuncs.c#gincostestimate-datapages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7981-L8028), [selfuncs.c#gincostestimate-qualcost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8030-L8047)).

`btcostestimate()` instead delegates to `genericcostestimate()`, which prorates `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)`, then adds a `log2(index->tuples)` comparison charge and the `(tree_height + 1) * 50 * cpu_operator_cost` descent charge described in [2. B-tree height carries an explicit anti-bloat charge](#2-b-tree-height-carries-an-explicit-anti-bloat-charge).

That asymmetry is the whole story for a selective equality lookup, and it survives GIN being the physically smaller index. Reproducing both closed forms in SQL from the catalog and the GIN metapage matched `EXPLAIN` to the cent for `n = 42` (30 matching rows out of 300,000, `numEntryPages` 278, `numEntries` 10,000, `nDataPages` 0):

- GIN: 2 entry pages plus 1 data page at `random_page_cost`, predicted total `12.9725`, printed `12.97`.
- B-tree: `ceil(30 * 280 / 300000) = 1` page at `4.0`, plus `ceil(log2(300000)) = 19` comparisons, plus `(1 + 1) * 50 * 0.0025 = 0.25`, plus `30 * (0.005 + 0.0025)`, predicted total `4.5225`, printed `4.52`.

The losing path is then dropped by ordinary path pruning. `add_path()` compares candidates with `compare_path_costs_fuzzily()` at `STD_FUZZ_FACTOR = 1.01` and refuses or removes a dominated path ([pathnode.c#add_path](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L419-L453), [pathnode.c#STD_FUZZ_FACTOR](../../../../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L42-L47)). Because a GIN path is only ever a bitmap input, the decisive filter is usually `choose_bitmap_and()`: it first keeps only the cheapest path in each group of paths using identical clause sets, sorts the survivors by index access cost, and then adds a further index to the AND group only when `bitmap_and_cost_est()` reports a lower total ([indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [indxpath.c#choose_bitmap_and-accept-reject](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1418-L1489), [indxpath.c#bitmap_and_cost_est](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1560-L1571)). A GIN index whose own scan cost exceeds the saving it produces is therefore dropped from the bitmap tree entirely, and its predicate reappears as a `Filter` above the surviving B-tree bitmap scan.

### A bloated GIN index loses to a B-tree

This is the case that connects the follow-up back to this page's subject. GIN bloat in the `fastupdate` pending list is charged to the planner in full and immediately, because `nPendingPages` is the one metapage counter `gincostestimate()` treats as current ([selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7696-L7710)). The manual states the runtime consequence: "searches must scan the list of pending entries in addition to searching the regular index, and so a large list of pending entries will slow searches significantly" ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L521-L529)).

Measured at the pin on a 200,000-row table with a `tsvector` GIN index (`fastupdate = on`) and a B-tree index on an `int` category column, querying `tsv @@ to_tsquery('simple','zebracorn') AND cat = 7`:

| Pending pages (`pgstatginindex`) | GIN blocks | GIN scan cost for `tsv @@ …` | Plan chosen | Plan total cost |
|---:|---:|---:|---|---:|
| 0 | 228 | `13.80` | `BitmapAnd` of GIN and B-tree | `163.08` |
| 982 | 1,210 | `4187.83` | B-tree bitmap scan only; `tsv @@ …` demoted to `Filter` | `4095.28` |
| 0 after `gin_clean_pending_list()` | 1,250 | `18.59` | `BitmapAnd` of GIN and B-tree | `244.20` |

The middle row's `4187.83` comes from a separate `EXPLAIN` of the GIN qual on its own, because the chosen two-clause plan no longer contains that scan node.

`gin_clean_pending_list()` returned exactly `982`, matching `pgstatginindex` ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1085), [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)).

Note the direction of the sign: unlike the B-tree page-count penalty in [1. Physical page count enters index cost](#1-physical-page-count-enters-index-cost), this is not a mild pro-rata increase. Every pending page is added to **startup** cost for every scan ([selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7975-L7979)), so it also disqualifies the GIN path from cheap-startup plans such as `LIMIT`.

### Stale GIN metapage statistics

`gincostestimate()` trusts the last-`VACUUM` counters only when the index has not grown too much. It requires `nTotalPages <= numPages`, `nTotalPages > numPages / 4`, `nEntryPages > 0` and `nEntries > 0`; otherwise it invents statistics from the live block count — 90% entry pages, the rest data pages, and 100 entries per entry page — after clamping the page count to at least 10 ([selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7712-L7766)). The source comment names the 4X cutoff and calls the 100-entries figure "rather bogus".

Two details matter operationally. First, `numPages` comes from `index->pages`, which `get_relation_info()` reads live with `RelationGetNumberOfBlocks()`, so index growth reaches the cost model before any `ANALYZE` ([plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486)). Second, `numPendingPages` is discarded when it is not smaller than `numPages`, which is a sanity guard, not a cost reduction ([selfuncs.c#gincostestimate-pending-guard](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7723-L7726)).

Measured: a 2,000-row table's GIN index sat at 2 blocks with metapage counters `(nTotalPages, nEntryPages, nDataPages, nEntries) = (2, 1, 0, 100)` and priced `n = 42` at `8.64`. Adding 398,000 rows without `VACUUM` grew it to 1,369 live blocks while the metapage stayed at 2, so `1369 > 2 * 4` put the estimate on the invented branch (1,232 entry pages, 137 data pages, 123,200 entries) and the cost rose to `17.10`. A later `VACUUM` rewrote the metapage to `(1369, 1368, 0, 200000)` and the cost stayed `17.10` to two decimal places.

The same staleness is visible after a manual pending-list drain, which is a trap worth naming: `gin_clean_pending_list()` moves entries into the tree and grows the fork but does **not** refresh `nTotalPages`. In the measurement below, a freshly vacuumed index read `(324, 200, 123, 1001)` at 324 live blocks, and after the drain it was 1,346 live blocks with the metapage still reading `324`, so the cost model was still using invented statistics.

### The keyless full-index path on a partial GIN index

GIN sets `amoptionalkey = true` ([ginutil.c:49](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L49)), so `build_index_paths()` does not bail out when no clause matches the first index column ([indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897)), and a partial index whose predicate is proven still yields a path through `useful_predicate` ([indxpath.c#build_index_paths-generate](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L954-L1003)). `gincostestimate()` then prices that clauseless path as a whole-index scan: when `fullIndexScan` is set or `indexQuals == NIL`, it sets `searchEntries = numEntries`, "as if every key in the index had been listed in the query" ([selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7851-L7876)). The same branch fires when an attribute has a full scan but no normal scan, which is how `GIN_SEARCH_MODE_ALL` reaches the estimate ([gin.h#GIN_SEARCH_MODE](../../../../raw/postgres-17/src/include/access/gin.h#L34-L37), [selfuncs.c#gincost_pattern-searchmode](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7453-L7488)).

Measured: a **10-block** partial GIN index with 1,000 entries priced `WHERE id <= 5000` (its own predicate, no GIN-indexable clause) at `4430.35`, and the two-`random_page_cost` probe recovered exactly `1001.00` charged pages — 100 times the index's physical size, because `ceil(1000 * rint(pow(9, 0.15))) = 1000`. Adding `AND n = 42` dropped the same index's cost to `8.55`.

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
- **One multicolumn GIN instead of a `BitmapAnd`.** The `btree_gin` documentation says that "for queries that test both a GIN-indexable column and a B-tree-indexable column, it might be more efficient to create a multicolumn GIN index that uses one of these operator classes than to create two separate indexes that would have to be combined via bitmap ANDing" ([btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)). Measured: a `gin (tsv, cat)` index priced the two-column predicate at `21.55` and won the plan at `75.23`, against `183.18` for the `BitmapAnd` of the separate GIN and B-tree indexes at a plan cost of `236.85`.
- **Write amortization.** The pending list exists to make GIN insertion cheap, at the documented cost of slower searches ([gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)).

### GIN exact-pin measurements

All numbers in this follow-up come from one isolated server built from the pinned checkout (`PostgreSQL 17.10 on x86_64-pc-linux-gnu`), with `autovacuum = off`, `shared_buffers = 256MB` and default planner cost settings (`random_page_cost = 4`, `cpu_operator_cost = 0.0025`, `cpu_index_tuple_cost = 0.005`). `btree_gin` supplied the int4 GIN opclass, `pgstattuple` supplied `pgstatginindex`, and `pageinspect` supplied `gin_metapage_info()` and `bt_metap()`.

| Fixture | Contents | GIN index | B-tree index |
|---|---|---|---|
| S | `t_both`, 300,000 rows, `n = i % 10000` | `t_both_n_gin`, `btree_gin` int4 opclass, **279 blocks**, `nEntryPages` 278, `nEntries` 10,000, `nDataPages` 0 | `t_both_n_bt`, **280 blocks**, `fastlevel` 1, 90.31% `avg_leaf_density` |
| D | `docs`, 200,000 then 300,000 then 400,000 rows | `docs_tsv_gin` on `tsvector`, `fastupdate = on` | `docs_cat_bt` on `cat = i % 20`, 171 blocks |
| T | `t_stale`, 2,000 then 400,000 rows | `t_stale_gin`, `fastupdate = off`, never re-vacuumed | none |
| P | `t_part`, 100,000 rows | `t_part_gin`, partial `WHERE id <= 5000`, 10 blocks, 1,000 entries | none |
| B | `t_bool`, 100,000 boolean rows | `t_bool_gin`, `btree_gin` bool opclass | none |

#### Same table, same statistics, both indexes

Each row below is two `EXPLAIN` runs on fixture S, with the *other* index dropped inside a rolled-back transaction, so `pg_statistic`, `reltuples` and every selectivity estimate are identical. `enable_seqscan` was off so that a missing index path is visible as a `disable_cost`-priced sequential scan.

| Predicate | GIN | B-tree | Row estimate (both) |
|---|---:|---:|---:|
| `n = 42` | `12.97` | `4.52` | 30 |
| `n BETWEEN 100 AND 200` | `67.15` | `44.31` | 3,201 |
| `n < 20` | `28.63` | `8.89` | 612 |
| `n IN (1,2,3)` | `30.10` | `13.57` | 90 |
| `ORDER BY n LIMIT 10` | no index path: `Sort` over `Seq Scan`, `Limit` at `10000011688.92` | `Limit` at `0.78` | 10 |
| `SELECT n WHERE n = 42` | no index-only scan: `Bitmap Heap Scan` at `122.86` | `Index Only Scan` at `4.82` | 30 |
| `n IS NULL` | no index path: `Seq Scan` at `10000005206.00` | `Index Scan` at `8.31` | 1 |

The three "no index path" rows are priced by `disable_cost = 1.0e10` ([costsize.c:130](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L130)), which is what a forced-off sequential scan costs when it is the only path available. They are gate-2 outcomes, not cost losses. With every `enable_*` setting at its default, the planner chose the B-tree for `n = 42`, `n BETWEEN 100 AND 200` and `n < 20` alike.

#### The page charge, isolated

Running the same query at `random_page_cost = 4` and `random_page_cost = 1` isolates the page count, because every other GIN charge is a CPU charge that does not scale with it:

| Case | Cost at `rpc = 4` | Cost at `rpc = 1` | Difference / 3 | Reconciliation |
|---|---:|---:|---:|---|
| Fixture S, `n = 42` | `12.97` | `3.97` | `3.00` | 2 entry pages + 1 data page |
| Fixture D state A, vacuumed, `nTotalPages` 324 = 324 live blocks | `14.50` | `5.50` | `3.00` | trusted stats: `ceil(rint(200^0.15)) = 2` entry pages + 1 data page |
| Fixture D state B, 982 pending pages, 1,306 live blocks | `4188.59` | `1233.59` | `985.00` | invented stats (`1306 > 324 * 4`): 982 pending + 2 entry + 1 data page |
| Fixture T, stale metapage, 1,369 live blocks | `17.10` | `5.10` | `4.00` | invented stats: 3 entry pages + 1 data page |
| Fixture P, keyless partial index, 10 live blocks | `4430.35` | `1427.35` | `1001.00` | `searchEntries = numEntries = 1000` entry pages + 1 data page |

Fixture D state B is the sharpest result: a `BitmapAnd` plan that cost `250.66` while the metapage and the fork agreed rose to `4501.99` once 982 pending pages appeared, an 18x increase driven entirely by pages that hold no tree structure at all. In the earlier run recorded in the table above, the same condition removed the GIN index from the `BitmapAnd` outright.

#### Boolean column

Fixture B priced `WHERE i`, `WHERE i = true` and `WHERE i IS TRUE` identically at `33.79` for the GIN index scan, each with `Index Cond: (i = true)`. So a bare boolean `Var` is not a gate-1 rejection in v17. `i IS TRUE` additionally left `Filter: (i IS TRUE)` on the heap node while still using the index.

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

It reported `live_blocks = 339`, `pending_pages = 246`, `pending_tuples = 50000`, `pending_pct_of_index = 72.57`. `pgstatginindex` requires `pgstattuple` and reads only the metapage ([pgstatindex.c#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L476-L495)).

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

The `Bitmap Index Scan` costs were `2030.46` and `1121.46`, so `(2030.46 - 1121.46) / 3 = 303.00` pages, of which 246 were pending list. Divide by three because the two runs differ by exactly `3.0` per charged page.

### GIN settings that move the boundary

| Setting | v17 default | Role in the GIN-versus-B-tree decision | Apply scope |
|---|---|---|---|
| `random_page_cost` | 4.0 | Multiplies every pending, entry and data page GIN expects to touch ([guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3685-L3695)) | session/transaction (`PGC_USERSET`) |
| `cpu_operator_cost` | 0.0025 | Scales GIN's entry-tree descent and its `50 *` per-page CPU charge, and the B-tree's height charge ([guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3718-L3728)) | session/transaction (`PGC_USERSET`) |
| `gin_pending_list_limit` | 4MB | Caps how large a pending list gets before an insert drains it, so it caps the startup penalty ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584)) | session/transaction (`PGC_USERSET`) |
| `enable_bitmapscan` | on | Turning it off removes GIN's only plan shape entirely | session/transaction |

Two per-index storage parameters change the physical shape rather than its price, and both take `AccessExclusiveLock`: `fastupdate` (default on) and a per-index `gin_pending_list_limit` override ([reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L130), [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)). `gin_clean_pending_list()` drains the list on demand and takes `RowExclusiveLock` on the index ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1085)).

### GIN key data structures

| Structure | Field | Role |
|---|---|---|
| `GinQualCounts` | `partialEntries`, `exactEntries`, `searchEntries`, `arrayScans`, `attHasFullScan`, `attHasNormalScan` | The whole per-qual working set `gincostestimate()` derives from the index quals ([selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7369-L7377)) |
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

Symbols: [indxpath.c#create_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L234-L310), [indxpath.c#get_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L709-L767), [indxpath.c#build_index_paths](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L804-L1003), [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287-L1489), [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L8049), [selfuncs.c#gincost_pattern](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7379-L7491), [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89).

### GIN tests and explicit test absence

- **No in-tree test compares a GIN plan against a B-tree plan on the same column**, and none exercises `gincostestimate()`. `src/test` contains no reference to `gincostestimate`.
- The `btree_gin` regression suite sets `enable_seqscan = off` and uses `EXPLAIN (COSTS OFF)` throughout, so it asserts plan *shape* only, never cost ([bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9), [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)).
- What *is* covered is the gate-2 property matrix, by `amutils` ([amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129)), and the bitmap-only restriction, by comments and expected plans in `create_index` and `tsearch` ([create_index.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L264-L268), [tsearch.sql#gin-bitmap-only](../../../../raw/postgres-17/src/test/regress/sql/tsearch.sql#L225-L230)).
- Every measurement in this follow-up was therefore produced ad hoc on an isolated exact-pin server.

## Open Questions

- The one-cent gaps between observed and predicted costs (`28480.42` vs `28480.43`, `8226.42` vs `8226.43`, `9654.42` vs `9654.43`) are consistent with rounding inside the verification SQL rather than in the planner, because the exact-match cases (`123144.43`) use the same expression. Reproducing the prediction in C `double` arithmetic would settle it.
- Fixture P's growth ratios (3.4x unblocked, 6.9x blocked) are a single measurement of one deliberately harsh workload with autovacuum off. The documentation's stronger claim, that some indexes "never increase by even one single page/block despite constant version churn" ([btree.sgml#bottom-up-effectiveness](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L704-L720)), was not reproduced here and would need a gentler, steady-state workload to test.
- Mechanism 3 (the `effective_cache_size` split between heap and index pages) was demonstrated only indirectly: fixture A's nested-loop inner index-only scan priced at `3.83` per loop dense against `4.37` bloated. Isolating the heap-side share of that difference from the index-side share was not attempted, because `EXPLAIN` does not break out the two components.
- Whether GIN's v16 `DEFAULT_PAGE_CPU_MULTIPLIER` charges make GIN bloat pricing behave qualitatively like the B-tree page charge was not investigated here; the GIN cost model is separate and is covered only to the extent needed to scope this page.
- The claim that a custom index AM gets `tree_height = -1` unless `relam == BTREE_AM_OID` is read straight from `get_relation_info()`, but no custom-AM fixture was built to confirm what such an AM's `amcostestimate` then does with it.
- `pgstatindex.tree_level` (`btm_level`) and the planner's `tree_height` (`btm_fastlevel`) were equal in every fixture measured here, because all fixtures were freshly built or rebuilt. A fixture that forces a fast root to diverge from the true root was not constructed.
- The follow-up's `BitmapAnd` outcome under pending-list bloat was not stable across two runs of the same fixture family. At 982 pending pages, the first run dropped the GIN index and demoted its qual to a `Filter`; a second run on a larger, separately analyzed table kept the GIN index in the `BitmapAnd` while the plan cost rose 18x. Both are `choose_bitmap_and()` outcomes driven by the surviving B-tree's own cost and row estimate, but the boundary between them was not characterized.
- Fixture T's `VACUUM` refreshed the GIN metapage from `(2, 1, 0, 100)` to `(1369, 1368, 0, 200000)` and the `n = 42` cost stayed `17.10` to two decimal places. The invented and trusted branches happened to produce the same charged page count here, so this fixture does not separate them; a fixture where the two branches diverge visibly was not built.
- The GIN entry-page estimate `ceil(searchEntries * rint(pow(numEntryPages, 0.15)))` was reconciled arithmetically in every measured case, but `rint()`'s banker's rounding at exact `.5` boundaries was not exercised.
- Whether `contrib/btree_gin`'s partial-match path (`gincost_pattern()` charging `partialEntries += 100` per key) systematically over- or under-charges a range predicate was not investigated; the measurements report only the resulting costs.

## Related Pages

- [How a GIN Index Becomes Bloated in PostgreSQL 17, and How to Measure It (unverified)](../indexing/gin-index-bloat.md) - GIN's separate bloat mechanisms and cost model in the same version.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)](../indexing/pgstatindex-sample-variant-proposal.md) - why `pgstatindex` has to read every block to produce the metrics discussed here.
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](../indexing/reindex-index-concurrently.md) - the online rebuild that resets every input on this page.
- [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](../indexing/partial-indexes-pros-cons.md) - more on the partial-index costing path that behaves differently here.
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](../../../v12/questions/query-planning/bloated-indexes-query-planner.md) - the same question, and the same GIN-versus-B-tree follow-up, answered against the v12 pin.

## Evidence Map

| Claim | Evidence |
|---|---|
| Planner has no bloat/density/fragmentation field | [pg_class.h#relpages](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L69), [pathnodes.h#IndexOptInfo](../../../../raw/postgres-17/src/include/nodes/pathnodes.h#L1120-L1128) |
| `pages` comes from the live block count for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486); measured: forged `relpages = 1` left cost at `25528.42` |
| `tuples` is the parent table's estimate for non-partial indexes | [plancat.c#get_relation_info-index-size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L486) |
| Partial indexes use `pg_class` density | [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160); measured: forged `reltuples = 20` moved cost `24140.12` -> `23140.29` |
| Page count enters cost pro-rata | [selfuncs.c#genericcostestimate-numIndexPages](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6716-L6731); measured `28480.42` vs `123144.43` |
| Height charge exists to stop bloated indexes looking free | [selfuncs.c#btcostestimate-page-descent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7092-L7105) |
| Height charge is `50 * cpu_operator_cost` per level | [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145); measured startup gap exactly `50.00` at `cpu_operator_cost = 1` |
| Planner height is the fast-root level | [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L663-L717), [README#page-deletion-and-tree-height](../../../../raw/postgres-17/src/backend/access/nbtree/README#L362-L381) |
| Index pages enter cache modeling | [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951), [costsize.c#cost_index-heap-fetches](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L670-L747) |
| Index pages choose parallel workers | [allpaths.c#compute_parallel_worker](../../../../raw/postgres-17/src/backend/optimizer/path/allpaths.c#L4202-L4279); measured 4 vs 6 workers |
| v17 clamps SAOP descents to `ceil(pages/3)` | [selfuncs.c#btcostestimate-saop-clamp](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7020-L7041); measured plateau at 3 vs 19 |
| Only bt/hash/gist/spgist use `genericcostestimate` | [selfuncs.c#genericcostestimate-callers](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7072); [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L7670), [selfuncs.c#brincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L8051-L8060) |
| GiST/SP-GiST estimate height from page count | [selfuncs.c#gistcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7255-L7307), [selfuncs.c#spgcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7310-L7362) |
| Hash charges no descent cost | [selfuncs.c#hashcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7212-L7252) |
| `avg_leaf_density`/`leaf_fragmentation` ignore deleted pages | [pgstatindex.c#page-classification](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372); measured 89.18% on a 2,465-deleted-page index |
| Fragmentation contributes zero cost | measured gap `1428.00` = `357 * 4.0` with 49.73% vs 0% fragmentation |
| Deleted pages stay in the fork | [nbtpage.c#page-deleted](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2628-L2648), [README#placing-deleted-pages-in-the-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/README#L383-L441); measured 2,745 blocks after two VACUUMs |
| Split policy sets density ceilings | [nbtsplitloc.c#fillfactor-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L280-L334), [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L200-L202) |
| Bloat can drop an index from a `BitmapAnd` | [indxpath.c#choose_bitmap_and](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1287); measured `c = 7` demoted to `Filter` |
| VACUUM can skip index vacuuming | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1947), [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2287-L2340) |
| `index_pages_fetched` / `cost_index` unchanged since `REL_12_0` | byte-identical file text at `REL_12_0` and the pin |
| SAOP clamp is new in v17 | `5bf748b86bc`, first release tag `REL_17_0`; absent from `REL_12_0` `btcostestimate` |
| `50.0` -> macro is cosmetic and v16 | `eb5c4e953bb` diff, first release tag `REL_16_0`; `REL_12_0` already had `* 50.0 *` |
| Deduplication is v13 | `0d861bbb702`, first release tag `REL_13_0`; `deduplicate_items` absent from `REL_12_0` `reloptions.c`; measured 852 vs 2,749 blocks |
| Bottom-up deletion is v14 | `d168b666823`, first release tag `REL_14_0`; [btree.sgml#simple-vs-bottom-up](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L679-L703), [README#added-to-postgresql-14](../../../../raw/postgres-17/src/backend/access/nbtree/README#L980-L988); measured 583 vs 1,174 blocks |
| Newly deleted pages recyclable in the same VACUUM is v14 | `9dd963ae253`, first release tag `REL_14_0`; [README#postgresql-14-fsm-change](../../../../raw/postgres-17/src/backend/access/nbtree/README#L403-L424) |
| `reltuples = -1` is v14 | `3d351d916b2`, first release tag `REL_14_0`; [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L65-L66) |
| No test covers the bloat charge | no `src/test` match for `tree_height`, `btcostestimate`, `genericcostestimate` |
| The `pgstatindex` test uses an empty index | [sql/pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52) |
| All four cost GUCs are session-scoped | [guc_tables.c#random_page_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3685-L3695), [guc_tables.c#cpu_operator_cost](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3718-L3728), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3507-L3517), [guc_tables.c#min_parallel_index_scan_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3530-L3539) |
| GIN is discarded at clause matching when the operator is not in its opfamily | [indxpath.c#match_opclause_to_indexcol-op_in_opfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2433-L2459); core GIN families carry no `<`/`<=`/`>=`/`>` ([pg_amop.dat#gin-array_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1232-L1244), [pg_amop.dat#gin-jsonb_ops](../../../../raw/postgres-17/src/include/catalog/pg_amop.dat#L1593-L1611)) |
| `btree_gin` adds strategies 1-5 but is documented not to outperform B-tree | [btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69), [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33) |
| A bare boolean `Var` still matches a GIN bool opclass in v17 | [indxpath.c#IsBooleanOpfamily](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2271-L2286), [indxpath.c#match_boolean_index_clause](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2288-L2384), [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98); measured `33.79` for `i`, `i = true` and `i IS TRUE` |
| GIN yields no plain index scan, no pathkeys, no index-only scan, no `IS NULL`, no native array scan, no parallel scan | [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89), [indxpath.c#get_index_paths-submit](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L740-L751), [indxpath.c#build_index_paths-pathkeys](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L905-L944), [indxpath.c#check_index_only](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L1730-L1800), [indxpath.c#match_clause_to_indexcol-nulltest](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L2251-L2266), [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129); measured as `disable_cost` sequential scans |
| GIN charges every pending, entry and data page at `random_page_cost` plus `50 * cpu_operator_cost` | [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7954), [selfuncs.c#gincostestimate-random-page-cost](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7975-L7979); measured `12.97` GIN vs `4.52` B-tree on identical statistics, both predicted to the cent |
| The pending list is charged to startup cost in full | [selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7881-L7885); measured 982 pending pages moving the GIN scan from `13.80` to `4187.83` and out of the `BitmapAnd` |
| `gin_clean_pending_list()` drains the list but leaves `nTotalPages` stale | [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1085), [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7712-L7766); measured 1,346 live blocks against a metapage still reading 324 |
| Stats older than 4X growth are replaced by invented ones | [selfuncs.c#gincostestimate-scale-or-invent](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7712-L7766); measured 1,369 live blocks against a metapage reading 2, charged 4 pages |
| A keyless partial GIN path is priced as a whole-index scan | [selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7851-L7876), [indxpath.c#build_index_paths-amoptionalkey](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L887-L897); measured `1001.00` charged pages on a 10-block index |
| GIN rejects unique, `INCLUDE`, exclusion and `CLUSTER` | [indexcmds.c#DefineIndex-am-checks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L860-L879), [cluster.c#cluster_rel-amclusterable](../../../../raw/postgres-17/src/backend/commands/cluster.c#L517-L522); all four messages reproduced |
| A multicolumn GIN can beat a `BitmapAnd` of GIN + B-tree | [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33); measured `21.55` versus `183.18` |
| No test compares GIN and B-tree plan choice, and none covers `gincostestimate` | no `src/test` match for `gincostestimate`; `btree_gin` tests use `EXPLAIN (COSTS OFF)` ([bool.sql#enable_seqscan-off](../../../../raw/postgres-17/contrib/btree_gin/sql/bool.sql#L1-L9)) |

## Context Reviewed

Planner and cost path: `src/backend/optimizer/util/plancat.c` (`get_relation_info`, `estimate_rel_size`), `src/backend/optimizer/path/costsize.c` (`cost_index`, `index_pages_fetched`, `cost_bitmap_heap_scan`, `compute_bitmap_pages`, `get_indexpath_pages`), `src/backend/optimizer/path/allpaths.c` (`total_table_pages`, `compute_parallel_worker`), `src/backend/optimizer/path/indxpath.c` (`choose_bitmap_and`, `bitmap_scan_cost_est`), `src/backend/optimizer/util/pathnode.c` (`compare_path_costs_fuzzily`), `src/backend/utils/adt/selfuncs.c` (`genericcostestimate`, `btcostestimate`, `hashcostestimate`, `gistcostestimate`, `spgcostestimate`, `gincostestimate`, `brincostestimate`, `add_predicate_to_index_quals`).

nbtree: `nbtpage.c` (`_bt_getrootheight`, fast-root update, page deletion), `nbtsplitloc.c` (`_bt_findsplitloc` fillfactor policy and single-value strategy), `nbtdedup.c` (`_bt_dedup_pass`, `_bt_bottomupdel_pass`), `nbtinsert.c` (pre-split deletion and deduplication), `nbtsort.c`, `src/include/access/nbtree.h`, and the nbtree `README` sections on page deletion, tree height, FSM placement, simple deletion, bottom-up deletion, split policy, and deduplication.

VACUUM: `src/backend/access/heap/vacuumlazy.c` (`BYPASS_THRESHOLD_PAGES`, `lazy_vacuum`, `lazy_check_wraparound_failsafe`, `LVRelState`).

Headers and catalogs: `src/include/nodes/pathnodes.h` (`IndexOptInfo`, `RelOptInfo`, `PlannerInfo`), `src/include/catalog/pg_class.h`, `src/backend/access/common/reloptions.c`, `src/backend/utils/misc/guc_tables.c`.

Contrib: `contrib/pgstattuple/pgstatindex.c` plus its `sql/` and `expected/` regression files, `contrib/pageinspect` `bt_metap()` definitions.

Documentation: `ref/reindex.sgml`, `maintenance.sgml` (routine reindexing), `glossary.sgml` (Bloat), `btree.sgml` (version churn, bottom-up deletion, deduplication), `ref/create_table.sgml` (`vacuum_index_cleanup`), `ref/create_index.sgml`, `indices.sgml`, `pgstattuple.sgml`.

History: `git log -L` on `genericcostestimate`, `btcostestimate`, `get_relation_info`, `index_pages_fetched`, and `cost_index` bounded by `REL_12_0..HEAD`; full-file diffs of those functions against `REL_12_0`; `git tag --contains` for every attributed commit; `contrib/pgstattuple` and `src/backend/access/nbtree` history since `REL_12_0`.

Empirical: one isolated PostgreSQL 17.10 server built from the pin, eight fixture scripts covering density, real deletion bloat, height, fragmentation, catalog forgery, plan flips, `BitmapAnd` pruning, parallel worker counts, the v17 SAOP clamp, version churn with and without a held snapshot, and deduplication.

Follow-up (GIN versus B-tree) additions. Path generation and gating: `src/backend/optimizer/path/indxpath.c` (`create_index_paths`, `match_restriction_clauses_to_index`, `match_clause_to_indexcol`, `match_opclause_to_indexcol`, `match_saopclause_to_indexcol`, `match_boolean_index_clause`, `IsBooleanOpfamily`, `get_index_clause_from_support`, `get_index_paths`, `build_index_paths`, `check_index_only`, `choose_bitmap_and`, `bitmap_and_cost_est`), `src/backend/optimizer/util/plancat.c` (the AM capability-flag copy and the `sortopfamily` branches), `src/backend/optimizer/util/pathnode.c` (`add_path`, `STD_FUZZ_FACTOR`), `src/backend/access/index/indexam.c` (`index_can_return`), `src/backend/optimizer/path/costsize.c` (`disable_cost`).

GIN internals: `src/backend/utils/adt/selfuncs.c` (`gincostestimate`, `gincost_pattern`, `gincost_opexpr`, `gincost_scalararrayopexpr`, `GinQualCounts`), `src/backend/access/gin/ginutil.c` (`ginhandler`, `ginGetStats`, `ginUpdateStats`), `src/backend/access/gin/ginfast.c` (`gin_clean_pending_list`), `src/include/access/gin.h` (`GinStatsData`, `GIN_SEARCH_MODE_*`), `src/include/access/ginblock.h` (`GinMetaPageData`).

Catalogs, errors and settings: `src/include/catalog/pg_amop.dat` (the four core GIN opfamilies plus the B-tree and hash `jsonb` families), `src/include/catalog/pg_opfamily.h` (`IsBuiltinBooleanOpfamily`), `src/backend/commands/indexcmds.c` (unique/`INCLUDE`/multicolumn/exclusion AM checks), `src/backend/commands/cluster.c` (`amclusterable`), `src/backend/access/common/reloptions.c` (`fastupdate`, per-index `gin_pending_list_limit`), `src/backend/utils/misc/guc_tables.c` (`gin_pending_list_limit`).

Contrib, tests and docs: `contrib/btree_gin` (`btree_gin--1.0.sql` operator classes, `sql/bool.sql`, `expected/bool.out`), `contrib/pgstattuple/pgstatindex.c` (`pgstatginindex`), `src/test/regress/expected/amutils.out`, `src/test/regress/sql/create_index.sql`, `src/test/regress/sql/tsearch.sql`, `doc/src/sgml/indices.sgml`, `doc/src/sgml/indexam.sgml`, `doc/src/sgml/gin.sgml`, `doc/src/sgml/btree-gin.sgml`, `doc/src/sgml/pgtrgm.sgml`.

Follow-up history: `git log -L` on `gincostestimate` bounded by `REL_12_0..HEAD` returned exactly two commits, `cd9479af2af` and `4b754d6c16e`, whose first release tags by `git tag --contains` are `REL_16_0` and `REL_13_0`. The follow-up describes only v17 behavior and makes no cross-version claim.

Follow-up empirical: one isolated PostgreSQL 17.10 server built from the pin with `btree_gin`, `pgstattuple` and `pageinspect` installed, eight scripts covering same-column GIN-versus-B-tree costing on identical statistics, closed-form reconciliation of both cost models, all six plan-shape cases, `fastupdate` pending-list bloat and drain, the 4X stale-metapage fallback, the keyless partial-index path, the boolean-column case, the live AM property matrix, the four `CREATE INDEX`/`CLUSTER` rejections, a multicolumn-GIN comparison, and verbatim execution of the two filed diagnostic blocks. Test objects were dropped, the server was stopped, and its data directory was removed.

## Source References

- [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L508)
- [plancat.c#estimate_rel_size-index](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1079-L1160)
- [selfuncs.c#genericcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6625-L6827)
- [selfuncs.c#btcostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L6869-L7210)
- [selfuncs.c:145](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)
- [costsize.c#cost_index](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L549-L621)
- [costsize.c#index_pages_fetched](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L860-L951)
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
- [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1880-L1947)
- [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L167)
- [pgstatindex.c#density-and-fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L352-L372)
- [ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1048)
- [glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)
- [btree.sgml#bottom-up-deletion](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L656-L733)
- [expected/pgstattuple.out#NaN](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)
- [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L8049)
- [selfuncs.c#gincost_pattern](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7379-L7491)
- [selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7369-L7377)
- [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89)
- [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)
- [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1027-L1085)
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
- [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584)
- [btree_gin--1.0.sql#int4_ops](../../../../raw/postgres-17/contrib/btree_gin/btree_gin--1.0.sql#L56-L69)
- [btree-gin.sgml#caveats](../../../../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L33)
- [gin.sgml#fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L529)
- [indexam.sgml#amgetbitmap](../../../../raw/postgres-17/doc/src/sgml/indexam.sgml#L991-L1010)
- [pgtrgm.sgml#index-support](../../../../raw/postgres-17/doc/src/sgml/pgtrgm.sgml#L413-L425)
- [pgstatindex.c#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L476-L495)
- [amutils.out#index-properties](../../../../raw/postgres-17/src/test/regress/expected/amutils.out#L122-L129)
- [bool.out#gin-bool-equality](../../../../raw/postgres-17/contrib/btree_gin/expected/bool.out#L89-L98)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [log](../../../log.md)
