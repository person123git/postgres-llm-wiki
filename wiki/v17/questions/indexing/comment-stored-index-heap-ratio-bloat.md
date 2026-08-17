---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The proposal](#the-proposal)
  - [Why COMMENT is a practical slot in core v17](#why-comment-is-a-practical-slot-in-core-v17)
  - [What the ratio actually measures](#what-the-ratio-actually-measures)
  - [Where a fresh ratio is scale-invariant, and where it is not](#where-a-fresh-ratio-is-scale-invariant-and-where-it-is-not)
  - [The hash sawtooth, and a build-time estimate that moves it](#the-hash-sawtooth-and-a-build-time-estimate-that-moves-it)
  - [Measured detection power on seven access methods](#measured-detection-power-on-seven-access-methods)
  - [A 200k-row delete-and-reload cycle test on all seven index types](#a-200k-row-delete-and-reload-cycle-test-on-all-seven-index-types)
  - [Use the main fork, not pg_table_size](#use-the-main-fork-not-pg_table_size)
  - [The detection query](#the-detection-query)
  - [The baseline audit query](#the-baseline-audit-query)
  - [Capture discipline](#capture-discipline)
  - [Baseline durability across maintenance commands](#baseline-durability-across-maintenance-commands)
  - [Locking, permissions, and visibility](#locking-permissions-and-visibility)
  - [Cost of the survey, and the relpages variant](#cost-of-the-survey-and-the-relpages-variant)
  - [Partitioned indexes](#partitioned-indexes)
  - [Fixture thresholds, and what to do with a flagged index](#fixture-thresholds-and-what-to-do-with-a-flagged-index)
  - [Follow-up: drift past a stored baseline versus "reindex when the index is larger than the heap"](#follow-up-drift-past-a-stored-baseline-versus-reindex-when-the-index-is-larger-than-the-heap)
  - [What changed since PostgreSQL 12](#what-changed-since-postgresql-12)
  - [Why this exists: the contrib boundary](#why-this-exists-the-contrib-boundary)
  - [Catalog, generated-header, and extension boundary](#catalog-generated-header-and-extension-boundary)
  - [Tests in the pinned checkout](#tests-in-the-pinned-checkout)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, copy the question from PostgreSQL 12, "Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 12", and review it for PostgreSQL 17.

The copied question is: propose a way to detect bloat in all types of indexes by using the `COMMENT` command to store the ratio between the table heap and the index.

Its follow-up is: how reliable is it to use the index-to-heap size ratio, stored as a comment on the index right after the index is rebuilt or created, as a proxy for index bloat? Let's say the initial ratio is 1, so a maintenance process would recalculate the ratio and get 1.4 as a result, so from 1 to 1.4 is a ~40% increase, and the maintenance process decides to reindex. Compare this to a rule that only reindexes if the index is larger than the heap.

## Answer

Use it as a screening signal, not as a bloat measurement for every index type. In these v17 fixtures the signal was calibratable for B-tree and contrib `bloom`, and — more cautiously — for GiST. It was not calibratable for hash, GIN, BRIN, or SP-GiST. `pg_relation_size()` is the direct core-v17 measurement of an ordinary physical index's current main-fork bytes, regardless of its access method: the one-argument form is a SQL wrapper that resolves to the `'main'` fork ([system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289), [pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7488-L7495), [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371)). An index comment is a practical arbitrary-text field that follows the index through `REINDEX CONCURRENTLY`, but it is not a private monitoring slot: human comments share it, `CREATE TABLE ... LIKE INCLUDING ALL` can copy it, and dump/restore can omit it ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1783), [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1404-L1411), [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L11996-L12000)).

The one-line summary of the mathematics, which drives every result on this page:

```text
drift = current_ratio / baseline_ratio = (index growth factor) / (heap growth factor)
```

The numerator is **index size growth**, not automatically bloat. A rebuild-reclaimable fraction is a different quantity:

```text
reclaimable_fraction = (current_index_bytes - rebuilt_index_bytes)
                       / current_index_bytes
```

The tables below use that fraction where they report rebuild-reclaimable space. The delete-and-reload table instead reports excess bytes relative to the rebuilt size, because that denominator is directly comparable to growth from the initial build.

Two v17-specific results dominate the review, and neither is a threshold-tuning problem:

- **A freshly built index's size is not a property of its own key alone.** A hash index over identical 200,000-row data measured 923 blocks on a narrow heap and 5,122 blocks on a wide, never-analyzed one, because `hashbuild()` sizes the initial bucket array from `estimate_rel_size()` rather than from the real row count ([hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L134-L137), [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L710-L741)). A baseline captured immediately after `CREATE INDEX` can therefore be a 6.2x-inflated reference. See [The hash sawtooth, and a build-time estimate that moves it](#the-hash-sawtooth-and-a-build-time-estimate-that-moves-it).
- **B-tree's same-VACUUM page recycling, added after v12, changes the delete-and-reload result only when another backend is consuming transaction IDs.** On an idle server the reload still grew the index 99.27%; with a concurrent XID consumer the same fixture ended at 745 blocks instead of 1,098 ([nbtpage.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2994-L3055)). See [What changed since PostgreSQL 12](#what-changed-since-postgresql-12).

The follow-up question is answered in [Follow-up: drift past a stored baseline versus "reindex when the index is larger than the heap"](#follow-up-drift-past-a-stored-baseline-versus-reindex-when-the-index-is-larger-than-the-heap), which scores both decision rules against `REINDEX` ground truth over 49 exact-pin cells.

### The proposal

Three moving parts.

1. **Capture.** In a quiescent maintenance window, immediately after the index is built or rebuilt, compute `pg_relation_size(index) / pg_relation_size(table)` and store the bare number as the index's comment. `ANALYZE` the table *before* building the index, and record or otherwise validate the heap's main-fork size at the same time. Ordinary `VACUUM` removes dead tuples and can trim empty pages only from the physical end; it does not compact free space in the middle, so "vacuumed" does not mean "fresh heap" ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2545), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2547-L2660)).
2. **Store.** `COMMENT ON INDEX <index> IS '<number>'` writes one `pg_description` row keyed by `(objoid, classoid, objsubid)` with `objsubid = 0` for a whole relation ([pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L64), [comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L143-L227)). Re-running it replaces the row rather than appending; an empty string deletes it, which the documentation states directly ([comment.sgml#description](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L87-L93)). Measured on the pinned build: two successive comments left exactly one `pg_description` row, and `IS ''` removed it.
3. **Detect.** Periodically recompute the ratio, divide by the stored baseline, and flag indexes whose drift exceeds a locally calibrated, per-access-method threshold. Read the baseline with `obj_description(indexrelid, 'pg_class')`, a SQL-language builtin that selects the `objsubid = 0` row ([system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301)). Treat a zero-byte heap as an explicit measurement state, not as a row to discard.

### Why COMMENT is a practical slot in core v17

The main storage choices have different lifecycle costs:

| Keyed by | Survives `REINDEX` | Survives `REINDEX CONCURRENTLY` | Survives `pg_dump`/restore |
|---|---|---|---|
| `pg_index.indexrelid` in your own table | yes | **no** — the OID changes | no — OIDs are not stable across restore |
| `relfilenode` in your own table | **no** | no | no |
| Logical name in your own table | yes, if kept in sync with rename/schema changes | yes | yes, if the baseline table is included |
| Index comment | yes | yes | by default; `--no-comments` disables dump or restore |

The `REINDEX CONCURRENTLY` case is the interesting one. `index_concurrently_swap()` explicitly rewrites the `pg_description` row's `objoid` from the old index to the new one, under the header comment `Move comment if any` ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1783)). Upstream regression coverage asserts exactly this behavior for both `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` ([create_index.sql#comments-preserved](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L940-L948), [create_index.out#comments-preserved](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2300-L2323)). By default `pg_dump` emits the index comment, but `pg_dump --no-comments` and `pg_restore --no-comments` deliberately omit it ([pg_dump.c#dumpIndex-comment](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L17085-L17093), [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L11996-L12000), [pg_backup_archiver.c#restore-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_backup_archiver.c#L180-L184)).

There is no access-method-neutral arbitrary-text index reloption in v17 to use instead. The index API delegates options to the selected AM's `amoptions` parser, and validated parsers reject an unrecognized parameter ([amapi.h#IndexAmRoutine-amoptions](../../../../raw/postgres-17/src/include/access/amapi.h#L276-L281), [reloptions.c#unrecognized-parameter](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L1470-L1480), [reloptions.c#index_reloptions](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L2058-L2072)). A dedicated baseline table remains a valid alternative. It avoids commandeering the human comment and can store the index and heap components separately, but it must track OID changes, renames, schema moves, drops, and restores.

### What the ratio actually measures

Since `current_ratio / baseline_ratio` expands to `(idx1/idx0) / (heap1/heap0)`, the drift number conflates two independent quantities. When the heap growth factor is 1, drift reduces to index growth. It still does not prove bloat: hash bucket allocation, GIN pending-list state, key distribution, and access-method build behavior can change index size without creating rebuild-reclaimable space.

Three structural facts about v17 fix the shape of both factors:

- **Ordinary index VACUUM does not shrink a main fork.** `RelationTruncate()` is the main-fork truncation entry point ([storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L288-L306)). Its active index call is the `TRUNCATE` path, which empties and rebuilds indexes under `AccessExclusiveLock` ([heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3056-L3101)). The apparent SP-GiST VACUUM call is compiled out under `#ifdef NOT_USED` because it is unsafe against concurrent inserts ([spgvacuum.c#truncate-disabled](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L884-L901)). `REINDEX` recreates an index, while `VACUUM FULL` and `CLUSTER` swap in a rewritten heap and rebuild all of its indexes ([index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3609), [cluster.c#finish_heap_swap](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1438-L1470)).
- **Index VACUUM returns pages to the free space map instead.** B-tree ([nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059)), GiST ([gistvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L229-L242)), GIN ([ginvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L754-L794)), and SP-GiST ([spgvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L845-L876)) all call `RecordFreeIndexPage`/`IndexFreeSpaceMapVacuum`. Reclaimed space is reusable but invisible to file size, because `RecordFreeIndexPage` only records the block in the FSM ([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L49-L56)).
- **The heap can shrink.** `lazy_truncate_heap()` drops trailing empty pages, so a heap that loses its newest rows can get smaller while ordinary index VACUUM keeps each index's main-fork block count. It can also skip truncation because too few trailing pages qualify or because it cannot obtain the conditional `AccessExclusiveLock` ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2545), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2547-L2660)).

That asymmetry makes drift sensitive to both index growth and heap movement. Neither direction is a bloat verdict by itself.

### Where a fresh ratio is scale-invariant, and where it is not

A stored baseline is only meaningful if a *freshly built* index keeps the same ratio to its heap as the table grows. Measured on the pinned 17.10 build: one table per access method, identical 40-byte payload column, index built from scratch at each of three scales.

| AM | ratio at 100k rows | at 400k | at 1.6M | drift over 16x growth |
|---|---|---|---|---|
| gist | 0.399688 | 0.399974 | 0.399968 | **1.0007** |
| btree | 0.143451 | 0.142857 | 0.142671 | **0.9946** |
| bloom (contrib) | 0.102911 | 0.102171 | 0.102015 | **0.9913** |
| hash | 0.267152 | 0.212661 | 0.212837 | 0.7967 |
| gin | 1.066008 | 1.066424 | 0.799773 | 0.7502 |
| spgist | 0.286383 | 0.515924 | 0.652584 | **2.2787** |
| brin | 0.001559 | 0.000390 | 0.000097 | **0.0625** |

Read the last column as "how much a perfectly healthy index's ratio moved for reasons that have nothing to do with bloat". Four access methods are structurally disqualified in this fixture:

- **BRIN.** The index stayed at exactly 3 blocks at all three measured scales. A BRIN index summarizes `pages_per_range` heap pages per tuple, defaulting to 128 ([brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-17/src/include/access/brin.h#L39-L45)), while each revmap page holds many range pointers ([brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-17/src/include/access/brin_page.h#L93-L96)). That does **not** make BRIN size constant at every scale: the revmap computes additional pages from the number of covered heap ranges and extends as needed ([brin_revmap.c#range-to-revmap-page](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L40-L44), [brin_revmap.c#brinRevmapExtend](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L112-L120)). The tested ratio therefore decayed 16x inside one size plateau; at larger scales it changes in steps. Either shape defeats one fixed drift threshold.
- **SP-GiST.** The fresh ratio more than doubled over 16x growth: 1,082 index blocks at 200k rows but 20,080 at 1.6M, against a heap that grew strictly linearly. The indexed column is `point(g, (g * 37) % 1000)`, whose x-range grows with the row count while y stays inside 0-999, so the quad tree deepens faster than the data. This is a property of this fixture's key distribution, not a defect; the point stands that a stored ratio cannot distinguish it from bloat.
- **GIN.** In this fixture, which held three array elements per row drawn from a widening domain, the fresh ratio fell 25% at the largest scale. The build path groups each extracted key with its sorted heap-TID list before inserting the entries, so row count alone does not determine the representation ([ginbulk.c#ginGetBAEntry](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L250-L274), [gininsert.c#ginbuild-entry-insert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L389-L415)).
- **Hash.** See the next section.

The measured GiST, B-tree, and contrib `bloom` ratios stayed within 0.1%, 0.6%, and 0.9% respectively over the 16x growth. That stability is the precondition the scheme needs.

### The hash sawtooth, and a build-time estimate that moves it

Hash sizing is a step function by construction. `_hash_init()` picks the initial bucket count from the estimated tuple count divided by the fill factor, rounded up to a whole splitpoint ([hashpage.c#_hash_init_metabuffer-buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526)), with a default fill factor of 75 ([hash.h#HASH_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/hash.h#L296)). `_hash_expandtable()` then allocates a whole splitpoint's worth of buckets at once when the split point advances ([hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L783-L800)).

Freshly built hash and B-tree indexes over the same data, measured on the pinned build:

| rows | hash index bytes | hash ratio | btree ratio |
|---|---|---|---|
| 100,000 | 4,210,688 | 0.267152 | 0.143451 |
| 150,000 | 4,210,688 | 0.178163 | 0.143501 |
| 200,000 | 6,733,824 | 0.213673 | 0.143228 |
| 300,000 | 8,404,992 | 0.177816 | 0.142981 |
| 450,000 | 16,572,416 | 0.233765 | 0.142940 |
| 600,000 | 16,793,600 | 0.177658 | 0.142820 |
| 900,000 | 33,030,144 | 0.232956 | 0.142766 |

The healthy hash ratio oscillates between 0.1777 and 0.2672, a 1.50x swing with no rebuild-reclaimable space introduced by the workload, while B-tree moves 0.5% over the same 9x range.

**The estimate that feeds that calculation is not the row count.** `hashbuild()` calls `estimate_rel_size()` and hands the result to `_hash_init()` ([hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L134-L137)). For a heap that has never been vacuumed or analyzed, `reltuples` is `-1`, so the estimator falls back to deriving a density from the *estimated* tuple width ([plancat.c#estimate_rel_size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1065-L1080), [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L710-L741)). Three 200,000-row tables with an identical `k` column, differing only in an unindexed `pad` column:

| fixture | planner row estimate | heap blocks | fresh hash blocks | `maxbucket` | fresh ratio |
|---|---|---|---|---|---|
| `pad` 40, no `ANALYZE` | 253,440 | 2,112 | 923 | 895 | 0.437027 |
| `pad` 400, no `ANALYZE` | **1,336,320** | 11,136 | **5,122** | 5,119 | 0.459950 |
| `pad` 400, `ANALYZE` first | 200,000 | 11,136 | **822** | 767 | 0.073815 |

The wide, never-analyzed heap produced a hash index **6.2x larger** than the same data analyzed first, and `ANALYZE` followed by `REINDEX` brought it back to 822 blocks. A baseline captured immediately after `CREATE INDEX` on that table records `0.459950` as "healthy", and the later, genuinely-better rebuild reads as a 84% *drop* in drift. `ANALYZE` before the build, and before the capture.

### Measured detection power on seven access methods

Each fixture was one 200,000-row heap carrying all seven indexes — the six core access methods plus contrib `bloom` — on an isolated 17.10 server built from the pinned checkout with `autovacuum = off`. After the workload, `REINDEX TABLE` supplied a reference index size. "Index reclaimable fraction" is `(current index - rebuilt index) / current index`. These are operational rebuild comparisons, not access-method-independent definitions of bloat. `idx_factor` and `heap_factor` are the two components of drift.

**Index-heavy churn** (five rounds updating 20% of rows' indexed keys, `VACUUM` after each round):

| AM | baseline | current | drift | idx_factor | heap_factor | index reclaimable fraction |
|---|---|---|---|---|---|---|
| btree | 0.143228 | 0.285529 | **1.994** | 2.392 | 1.200 | 0.582 |
| spgist | 0.281258 | 0.519497 | **1.847** | 2.216 | 1.200 | 0.513 |
| gin | 1.066285 | 1.603986 | **1.504** | 1.805 | 1.200 | 0.446 |
| gist | 0.399792 | 0.571924 | **1.431** | 1.717 | 1.200 | 0.537 |
| hash | 0.213673 | 0.249783 | 1.169 | 1.403 | 1.200 | 0.297 |
| bloom | 0.102417 | 0.102470 | 1.001 | 1.201 | 1.200 | 0.167 |
| brin | 0.000780 | 0.000650 | 0.833 | 1.000 | 1.200 | 0.000 |

Because these updates changed indexed keys, each successful update required a new index entry. PostgreSQL collects every indexed attribute into the HOT-safety bitmap, and `heap_update()` permits a heap-only tuple (HOT) update only when no such attribute changed ([relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5267-L5282), [heapam.c#HOT-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4141-L4155)).

Drift understates index growth here — 1.994 against a 2.392x index factor for B-tree — because the heap grew 20% at the same time. Where index and heap grow by the same factor, as with `bloom` (1.201 against 1.200), drift is 1.001 even though a rebuild would reclaim 16.7% of the current index file.

**Ordinary 4x ascending growth with no maintenance** — the baseline for "what does normal look like":

| AM | drift | index reclaimable fraction |
|---|---|---|
| spgist | 2.100 | 0.000 |
| hash | 1.103 | 0.102 |
| btree | 0.997 | 0.000 |
| bloom | 0.997 | 0.001 |
| gin | 0.981 | -0.020 |
| gist | 0.923 | -0.084 |
| brin | 0.250 | 0.000 |

For B-tree, `bloom`, GIN and GiST this fixture moved drift by less than 8%. SP-GiST reached 2.100 with a 0.000 reclaimable fraction: a maximal false positive produced entirely by key distribution. Negative reclaimable fractions for GiST and GIN are not arithmetic errors — rebuilding those two produced a *larger* index than the incrementally built one, so "REINDEX shrinks it" is not universal.

**The three heap-side failure modes.** First, **shared retained space is invisible**: a scattered delete of 80% of rows followed by `VACUUM` moved neither file, so drift is exactly 1.000 on all seven access methods while the index reclaimable fraction runs from 0.000 (BRIN) to 0.800 (GIN, GiST). Second, **heap growth masks index retained space**: widening every row from 40 to 400 bytes grew the heap 4.466x while indexes grew at most 2.9x, driving drift *down* to 0.447 for B-tree against a 49.9% reclaimable fraction. Third, **heap truncation fakes an index-growth alarm**: deleting the newest 75% of rows by `id` let `lazy_truncate_heap()` drop trailing heap pages while ordinary index VACUUM kept every index block, so drift landed at 3.999 on all seven access methods — correct for six of them, a false positive on BRIN, whose reclaimable fraction is 0.000.

### A 200k-row delete-and-reload cycle test on all seven index types

This cycle is the scheme's best case. The heap ends at exactly the size it started, so the heap growth factor is 1.000 and drift collapses to pure index growth. One table, 200,000 rows, seven indexes, `autovacuum = off`, every index key a strictly increasing function of the generated series so the reload's keys are all greater than the first load's.

The heap measured 3,847 blocks before the cycle and 3,847 after it, so the ratio column carries the index change alone. Sizes are in 8 KB blocks; `rebuilt` is the `REINDEX TABLE` reference.

| AM | initial | final | index size % | baseline ratio | current ratio | ratio % | rebuilt | excess vs rebuilt % |
|---|---|---|---|---|---|---|---|---|
| btree | 551 | 1098 | **+99.27** | 0.143228 | 0.285417 | **+99.27** | 551 | +99.27 |
| gist | 1538 | 2914 | **+89.47** | 0.399792 | 0.757473 | **+89.47** | 1538 | +89.47 |
| gin | 4102 | 7495 | **+82.72** | 1.066285 | 1.948271 | **+82.72** | 4102 | +82.72 |
| spgist | 1082 | 1091 | +0.83 | 0.281258 | 0.283598 | +0.83 | 1082 | +0.83 |
| bloom | 394 | 394 | 0.00 | 0.102417 | 0.102417 | 0.00 | 394 | 0.00 |
| brin | 3 | 3 | 0.00 | 0.000780 | 0.000780 | 0.00 | 3 | 0.00 |
| hash | 822 | 822 | 0.00 | 0.213673 | 0.213673 | 0.00 | 815 | +0.86 |

Three readings matter:

- **`ratio_pct` equals `index_size_pct` on all seven rows, digit for digit.** That is the arithmetic of a heap that returns to its starting size. It makes drift an index-growth signal, not by itself a proof that the growth is reclaimable.
- **`ratio_pct` equals `excess_vs_rebuilt_pct` on six of seven rows.** Hash is the exception. `hash_metapage_info` reported `maxbucket = 767` and `ovflpoint = 11` for the initial, final, and rebuilt index alike; they differed only in overflow pages, which is the 7-block gap. The bucket count is fixed by the estimated tuple count at build time ([hashpage.c#_hash_init_metabuffer-buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526)).
- **BRIN reported 0.00% on a workload that deleted and reinserted every row.** `brinbulkdelete()` is a documented no-op — "there are no per-heap-tuple index tuples in BRIN indexes, there's not a lot we can do here" ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1284-L1301)) — so BRIN's 0.00% here is a correct reading, unlike its false positives elsewhere on this page.

**Mid-cycle, the ratio is undefined.** Between the delete and the reload the heap was **0 bytes**: `DELETE` emptied every page and `lazy_truncate_heap()` dropped all 3,847 of them, while all seven indexes kept every block. The ratio is therefore `NULL` for all seven indexes. The detection query below preserves those rows and reports `heap main fork is zero`.

#### Why the cycle splits the access methods: the recyclability gate

Rerunning the identical cycle with two and three post-`DELETE` `VACUUM` passes isolates the cause. Only the `VACUUM` count changed:

| AM | 1 VACUUM | 2 VACUUMs | 3 VACUUMs |
|---|---|---|---|
| btree | **+99.27%** | 0.00% | 0.00% |
| gist | **+89.47%** | 0.00% | 0.00% |
| gin | **+82.72%** | **+82.72%** | **+82.72%** |
| spgist | +0.83% | +0.83% | +0.83% |
| hash, brin, bloom | 0.00% | 0.00% | 0.00% |

Three groups, and each one is visible in source.

**B-tree and GiST: deleted, but not yet recyclable.** Both gate FSM reuse on the global visibility horizon. `BTPageIsRecyclable()` returns true only for a page that is deleted *and* whose `safexid` passes `GlobalVisCheckRemovableFullXid()` ([nbtree.h#BTPageIsRecyclable](../../../../raw/postgres-17/src/include/access/nbtree.h#L290-L318)); `gistPageRecyclable()` applies the same test to the page's delete XID ([gistutil.c#gistPageRecyclable](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L885-L908)). In this fixture the `VACUUM` that deleted the pages could not recycle them in the same pass: `btvacuumpage()` took its `else if (P_ISDELETED(opaque))` branch — commented "Already deleted page ... Can't recycle yet" — instead of calling `RecordFreeIndexPage()` ([nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179)), and GiST's cleanup has the same two-branch shape ([gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L298-L309)).

The measurement confirms it: after the single post-delete `VACUUM` the FSM fork of both indexes was **0 bytes**, while SP-GiST, `bloom` and BRIN each had a 24,576-byte FSM. Hash also showed 0 bytes, for the opposite reason — it never uses the index FSM at all, as below. `pgstatindex` on the B-tree tells the whole story in three snapshots:

| phase | blocks | internal | leaf | deleted | avg_leaf_density |
|---|---|---|---|---|---|
| after build | 551 | 3 | 547 | 0 | 90.00 |
| after DELETE + 1 VACUUM | 551 | 2 | 1 | 547 | 0.05 |
| after reload | 1098 | 3 | 547 | 547 | 90.00 |

The final 1,098 blocks are exactly `1 metapage + 3 internal + 547 leaf + 547 deleted`. Every leaf page from the first load is still allocated and still dead weight, and the reload had to extend the relation for 547 brand-new leaves.

This is not a consequence of the reload's keys being ascending. `_bt_getbuf()` takes whatever block `GetFreeIndexPage()` hands back ([nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L925)), and that function returns the first page the FSM reports with at least half a block free ([indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L38-L46)). Free-page reuse is key-order agnostic. With a second `VACUUM` before the reload the pages reached the FSM in time, the inserts reused them, and both indexes ended at *exactly* their initial 551 and 1,538 blocks.

**v17's same-VACUUM recycling does fire — but only with a concurrent XID consumer.** Since v14, `btvacuumscan()` calls `_bt_pendingfsm_finalize()` at the end of the scan to place pages deleted during that very VACUUM into the FSM, if the horizon has moved by then ([nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059), [nbtpage.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2994-L3055)). The function's own debugging aid states the condition plainly: "the optimization will never be effective without some other backend concurrently consuming an XID" ([nbtpage.c#pendingfsm-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3012-L3021)). Repeating the one-VACUUM arm on a B-tree while a second session ran `txid_current()` in a loop:

| arm | index FSM after 1 VACUUM | blocks after reload | growth |
|---|---|---|---|
| idle server (as above) | 0 bytes | 1,098 | +99.27% |
| concurrent XID consumer | 24,576 bytes | **745** | **+35.2%** |

Partial, not complete: `_bt_pendingfsm_finalize()` stops at the first page whose `safexid` is still too new, because pages are added in `safexid` order ([nbtpage.c#pendingfsm-loop](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3034-L3052)). A monitoring baseline therefore cannot assume a fixed post-cycle index size even on identical data and an identical VACUUM count — it depends on other sessions.

**GIN: no recovery in the measured one-, two-, or three-pass arms.** GIN uses a visibility recyclability gate ([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)), but that was not the limiting path here. `ginVacuumEntryPage()` does not remove an entry tuple merely because its posting list becomes empty; when `nitems` reaches 0 it sets `plist = NULL, plistsize = 0` and puts the tuple back on the page ([ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L513-L564)). Those retained entry tuples kept the measured entry pages nonempty and out of the FSM. A `gin_metapage_info` probe measured it directly: after deleting all 200,000 rows and running `VACUUM` three times, GIN still reported **600,000 entries across 4,101 entry pages** in a 4,102-block index with a 0-byte FSM. The reload then grew the file to 7,495 blocks.

**SP-GiST, hash, BRIN, and bloom: different reuse paths.** SP-GiST's final decision to put a new or empty non-root page in the FSM has no separate transaction test ([spgvacuum.c#spgvacuumpage](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L621-L680)). That does not make all SP-GiST cleanup ungated: a redirect tuple becomes a removable placeholder only when its XID precedes the visibility horizon ([spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L493-L560)). The measured SP-GiST file retained nine extra blocks (+0.83%) in every arm. Contrib `bloom` records new or deleted pages directly ([blvacuum.c#blvacuumcleanup](../../../../raw/postgres-17/contrib/bloom/blvacuum.c#L169-L215)). Hash does not use the index FSM: `_hash_freeovflpage()` clears a bit in the index's own bitmap and `_hash_getovflpage()` reuses a free overflow page from that map ([hashovfl.c#_hash_freeovflpage-bitmap](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L556-L590), [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L318-L332)). BRIN has no per-row entries to remove ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1284-L1301)).

**What the cycle changes about the recommendation.** Two operational points sharpen. First, for B-tree and GiST a second `VACUUM` before the reload prevented the measured growth — a fixture result, not a universal prescription, since a held transaction horizon can delay recycling and another `VACUUM` has its own I/O cost. Once the reload extended the files, a later `VACUUM` could make old pages reusable for future inserts but could not return their current main-fork blocks to the operating system. Second, **do not normalize unexplained growth by re-capturing it.** Capture a new baseline only after a deliberate rebuild or after proving the changed allocation is the intended healthy reference.

### Use the main fork, not pg_table_size

`pg_table_size()` sums every fork plus the TOAST relation and its index ([dbsize.c#calculate_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L424-L452), [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L378-L422)). TOAST traffic can move independently of an index on a narrow key, so this denominator can overwhelm the signal. Measured on the pinned build, moving 20% of rows to TOAST while churning indexed keys:

| denominator | heap factor | drift, btree | drift, gin | drift, spgist |
|---|---|---|---|---|
| `pg_relation_size` (main fork) | 1.699 | 2.110 | 1.110 | 1.776 |
| `pg_table_size` (all forks + TOAST) | 24.451 | 0.147 | 0.077 | 0.123 |

The main-fork denominator flagged the B-tree's 3.586x index growth; the `pg_table_size` denominator reported it as a 6.8x improvement. Use `pg_relation_size` on both sides.

One caveat that made this fixture harder to build than expected: a first attempt used `repeat('z', 3000)`, which pglz compressed to a 44-byte value that never left the heap — measured `pg_column_size` of 44 and a 0-byte TOAST relation. Wide-but-compressible columns do not produce TOAST traffic.

### The detection query

Read-only. Verified against the pinned 17.10 build.

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

WITH candidates AS MATERIALIZED (
    SELECT /* wiki_index_bloat_ratio_candidates */
           i.indexrelid,
           i.indrelid,
           n.nspname AS schema_name,
           tc.relname AS table_name,
           ic.relname AS index_name,
           am.amname AS index_am,
           CASE WHEN d.description ~ '^[0-9]+([.][0-9]+)?$'
                THEN d.description::numeric END AS baseline_ratio
    FROM pg_index i
    JOIN pg_class ic ON ic.oid = i.indexrelid
    JOIN pg_class tc ON tc.oid = i.indrelid
    JOIN pg_namespace n ON n.oid = ic.relnamespace
    JOIN pg_am am ON am.oid = ic.relam
    LEFT JOIN pg_description d
           ON d.objoid = i.indexrelid
          AND d.classoid = 'pg_class'::regclass
          AND d.objsubid = 0
    WHERE ic.relkind = 'i'
), measured AS MATERIALIZED (
    SELECT /* wiki_index_bloat_ratio_measure */
           c.*,
           pg_relation_size(c.indexrelid) AS index_bytes,
           pg_relation_size(c.indrelid) AS heap_bytes
    FROM candidates c
    WHERE c.baseline_ratio > 0
)
SELECT /* wiki_index_bloat_ratio_drift */
       m.schema_name,
       m.table_name,
       m.index_name,
       m.index_am,
       m.baseline_ratio,
       CASE WHEN m.index_bytes IS NULL THEN 'index unavailable'
            WHEN m.heap_bytes IS NULL THEN 'heap unavailable'
            WHEN m.heap_bytes = 0 THEN 'heap main fork is zero'
            WHEN m.index_bytes = 0 THEN 'index main fork is zero'
            ELSE 'ok'
       END AS measurement_state,
       round(m.index_bytes::numeric / nullif(m.heap_bytes, 0), 6)
           AS current_ratio,
       round((m.index_bytes::numeric / nullif(m.heap_bytes, 0))
             / m.baseline_ratio, 3) AS drift,
       pg_size_pretty(m.index_bytes) AS index_size,
       pg_size_pretty(m.heap_bytes) AS heap_main_size,
       s.n_live_tup,
       s.n_dead_tup
FROM measured m
LEFT JOIN pg_stat_all_tables s ON s.relid = m.indrelid
ORDER BY CASE WHEN m.index_bytes IS NULL OR m.heap_bytes IS NULL
                        OR m.heap_bytes = 0 OR m.index_bytes = 0
                     THEN 0 ELSE 1 END,
         drift DESC NULLS LAST;
COMMIT;
```

Details that matter:

- The `CASE` guard is not decoration. A bare `description::numeric` fails the whole query when an index carries a human comment, and PostgreSQL does not promise that a `WHERE` filter runs before a cast; the documentation names `CASE` as the supported way to force evaluation order ([syntax.sgml#expression-evaluation](../../../../raw/postgres-17/doc/src/sgml/syntax.sgml#L2535-L2560)). The accepted storage format is a positive unsigned decimal such as `0.143228`.
- The materialized `measured` common table expression calls `pg_relation_size` once per side and retains zero or unavailable relations. `pg_relation_size` opens each relation separately with `AccessShareLock`, returns `NULL` if a concurrently dropped relation can no longer be opened, `stat()`s its segment files, and then closes it ([dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371), [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343)). The two byte counts are not one atomic snapshot; run the survey in a quiet period if concurrent extension would make the ratio misleading.
- `relkind = 'i'` excludes partitioned indexes, which are `'I'` and have no storage ([pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L163-L175), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L197-L202)).
- `n_live_tup` and `n_dead_tup` are estimated row counts, not heap-compaction or bloat measurements ([monitoring.sgml#estimated-tuples](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L3812-L3828)). Use them as context only. They require `track_counts`, a `PGC_SUSET` setting that a superuser can change for a session or transaction; it defaults to on ([guc_tables.c#track_counts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1412-L1420)).
- `statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so `SET LOCAL` here is transaction scope with no restart or reload ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2612-L2621), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2623-L2632)).
- `pg_index.indexrelid` and `pg_index.indrelid` are the index and table OIDs ([pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L29-L36)).

### The baseline audit query

The comment is a shared, human-visible slot, so a monitoring scheme built on it needs a second query that reports which indexes have no usable baseline. Read-only, verified against the pinned build:

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

WITH comments AS MATERIALIZED (
    SELECT /* wiki_index_baseline_comments */
           n.nspname AS schema_name,
           ic.relname AS index_name,
           am.amname AS index_am,
           d.description,
           CASE WHEN d.description ~ '^[0-9]+([.][0-9]+)?$'
                THEN d.description::numeric END AS baseline_ratio
    FROM pg_index i
    JOIN pg_class ic ON ic.oid = i.indexrelid
    JOIN pg_namespace n ON n.oid = ic.relnamespace
    JOIN pg_am am ON am.oid = ic.relam
    LEFT JOIN pg_description d
           ON d.objoid = i.indexrelid
          AND d.classoid = 'pg_class'::regclass
          AND d.objsubid = 0
    WHERE ic.relkind = 'i'
      AND n.nspname !~ '^pg_'
      AND n.nspname <> 'information_schema'
)
SELECT /* wiki_index_baseline_audit */
       c.schema_name,
       c.index_name,
       c.index_am,
       CASE WHEN c.description IS NULL THEN 'no baseline stored'
            WHEN c.baseline_ratio IS NULL THEN 'comment is not a positive decimal'
            ELSE 'baseline is zero'
       END AS problem,
       left(coalesce(c.description, ''), 40) AS comment_prefix
FROM comments c
WHERE c.baseline_ratio IS NULL OR c.baseline_ratio <= 0
ORDER BY 1, 2;
COMMIT;
```

On the test server this reported the `_ccnew` leftover of an interrupted `REINDEX INDEX CONCURRENTLY` as `no baseline stored`, alongside partition child indexes whose baselines were destroyed by the hazard described below.

### Capture discipline

Since the payload is a bare number with no metadata, every guarantee has to live in the capture procedure. Writes are described here rather than filed as production SQL:

- **`ANALYZE` the table before building the index and capturing the baseline.** On a never-analyzed heap, `reltuples` is `-1` and the build-time size estimate is derived from an estimated tuple width, which produced a 6.2x-oversized hash index in the measurement above ([plancat.c#estimate_rel_size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1065-L1080), [hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L134-L137)).
- Capture immediately after building or rebuilding the index, but only when the heap main-fork size is also a trusted reference. Ordinary `VACUUM` does not make the heap compact; it can leave internal free space in place and may skip tail truncation ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2545)). Prefer a freshly loaded table or a deliberate heap rewrite when defining a compact baseline, or store the heap bytes separately so later movement remains visible.
- Store enough decimal precision that a small valid ratio cannot round to zero. Six decimal places are used only to present the fixture tables on this page; they are not a safe universal storage format.
- Capture again after a deliberate reset of either component, such as `REINDEX`, `VACUUM FULL`, or `CLUSTER`, and for each newly created or attached leaf index. Never re-capture merely to silence unexplained drift.
- Treat `n_live_tup` and `n_dead_tup` as estimates, not proof that a heap is compact ([monitoring.sgml#estimated-tuples](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L3812-L3828)).
- Refuse to overwrite a non-numeric comment without a human decision. A numeric human comment is indistinguishable from a baseline, so reserve the field by policy or use a dedicated table.
- Audit after `CREATE TABLE ... LIKE INCLUDING ALL`: PostgreSQL copies the source indexes and their comments, so the new indexes inherit numeric ratios measured against a different heap ([parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1404-L1411)). Measured: a clone's `clone_k_idx` carried the source's `0.267216`.

### Baseline durability across maintenance commands

Measured on the pinned 17.10 build, tracking the index OID, its `relfilenode`, and the stored comment:

| Operation | index OID | `relfilenode` | baseline |
|---|---|---|---|
| `REINDEX INDEX` | unchanged | changed | kept |
| `REINDEX INDEX CONCURRENTLY` | **changed** (17187 → 17195) | changed | kept |
| `VACUUM FULL` on the table | unchanged | changed | kept |
| `CLUSTER` | unchanged | changed | kept |
| `ALTER INDEX ... RENAME` | unchanged | unchanged | kept |
| `ALTER TABLE ... ALTER COLUMN TYPE`, plain table | **changed** (17195 → 17210) | changed | kept |
| `ALTER TABLE ... ALTER COLUMN TYPE`, partitioned table | changed | changed | **child baselines destroyed** |
| `CREATE TABLE ... LIKE INCLUDING ALL` | new | new | **copied from the source index** |
| `DROP INDEX` | — | — | `pg_description` row deleted |
| `REINDEX INDEX CONCURRENTLY` on a temporary index | unchanged | changed | kept |

The partitioned-table row is a genuine data-loss hazard, and its shape changed in v17. A parent index commented `parent-baseline` with children commented `0.111111` and `0.222222` ended up with **both children carrying no comment at all** after one `ALTER TABLE ... ALTER COLUMN TYPE`; the parent kept its own. In PostgreSQL 12 the same test propagated the *parent's* comment onto both children, and the regression suite labelled its own expected output as wrong. Commit `fee8cb9473462e023dcf0b41212ca3890ddc28d6` ("Use generateClonedIndexStmt to propagate CREATE INDEX to partitions", first released in 17.1) removed that propagation, and the v17 expected output records blank child comments with no apology ([alter_table.sql#partitioned-comments](../../../../raw/postgres-17/src/test/regress/sql/alter_table.sql#L1499-L1516), [alter_table.out#partitioned-comments](../../../../raw/postgres-17/src/test/regress/expected/alter_table.out#L2217-L2249)). For this scheme the practical consequence is unchanged in severity and better in kind: a lost baseline is detectable by [the baseline audit query](#the-baseline-audit-query), whereas a silently wrong inherited number is not.

Default dump/restore preserves comments, but either `pg_dump --no-comments` or `pg_restore --no-comments` omits them ([pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L11996-L12000), [pg_backup_archiver.c#restore-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_backup_archiver.c#L180-L184)). `DROP INDEX` removing the comment is expected behavior, documented as comments being dropped with their object ([comment.sgml#description](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L87-L93)).

### Locking, permissions, and visibility

`CommentObject()` resolves the target through `get_object_address()` with `ShareUpdateExclusiveLock` and then demands ownership ([comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L78)). For an index, that ownership check runs through `check_object_ownership()` ([objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2382-L2400)), and the accepted relkinds come from `get_relation_by_qualified_name()` ([objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L1333-L1350)). The grammar production is `COMMENT ON object_type_any_name any_name IS comment_text` ([gram.y#CommentStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L7049-L7058)). v17 documents the lock level explicitly, which v12 did not ([comment.sgml#lock-level](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L99)).

Measured consequences on the pinned build:

- Inside an explicit transaction, `COMMENT ON INDEX` held a `ShareUpdateExclusiveLock` on the **target index** and a `RowExclusiveLock` on `pg_description`, with no target lock on its heap. `CreateComments()` opens `pg_description` with `RowExclusiveLock` and keeps it to transaction end ([comment.c#CreateComments-catalog-lock](../../../../raw/postgres-17/src/backend/commands/comment.c#L174-L226)).
- A non-owner got `ERROR:  must be owner of index big_k`.
- The same non-owner read the baseline back successfully. Comments have no read privilege gate at all, which the documentation states explicitly and warns about ([comment.sgml#notes](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L291-L300)).

The conflict table shows that `ShareUpdateExclusiveLock` conflicts with another lock of the same mode plus `ShareLock`, `ShareRowExclusiveLock`, `ExclusiveLock`, and `AccessExclusiveLock` ([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L109)). A capture therefore serializes against another capture and the conflicting index-maintenance/DDL paths.

The baseline is world-readable, so it must never encode anything sensitive. In verbose mode, `psql`'s `\di+` selects `obj_description(c.oid, 'pg_class')` into its `Description` column, making the machine value look like an ordinary human comment ([describe.c#listTables-description](../../../../raw/postgres-17/src/bin/psql/describe.c#L4000-L4004)). Measured: `\di+ pr.big_k` printed `0.123456` in `Description`.

### Cost of the survey, and the relpages variant

With 300 indexes in one database on the pinned build, the size-based detection query ran in 2.4-4.0 ms on a warm, idle server. Each `pg_relation_size` call probes segment files until the first missing segment, so the work scales with the number of measured relations plus their segment files ([dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343)).

A catalog-only variant that reads `pg_class.relpages` instead of calling `pg_relation_size` ran in 0.7-1.0 ms and matched the live file size for all 300 freshly analyzed indexes — zero mismatches. It is not equivalent. `VACUUM` and `ANALYZE` update relation statistics through `vac_update_relstats()` ([vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1400-L1440)), and `CREATE INDEX` and `REINDEX` update `relpages` through `index_update_stats()` ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2829)). Ordinary DML does not keep the field current. After inserting 100,000 rows without analyzing, the measured `relpages` value still said 3 blocks against a live 445. Use `pg_relation_size` for detection; use a `relpages` approximation only when its update history is known.

### Partitioned indexes

A partitioned index is relkind `'I'` and has no storage: `pg_relation_size()` returned 0 for the parent while the leaf index returned 8192 on the same test ([pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L163-L175), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L197-L202)). Commenting the parent is allowed, but a parent-level ratio is meaningless. Combined with the `ALTER COLUMN TYPE` child-comment loss above, the practical rule is to store baselines only on leaf indexes, and to re-capture every leaf baseline after any partitioned-table column type change.

### Fixture thresholds, and what to do with a flagged index

These values separate the workloads on this page. They are fixture-calibration points, not production defaults:

| AM | usable in these fixtures? | observed separating drift | why |
|---|---|---|---|
| btree | yes | 1.30 | ascending 4x growth reached 0.997; five 20%-key-churn rounds reached 1.994 |
| bloom | yes | 1.30 | 0.997 under normal growth |
| gist | with care | 1.35 | fresh ratio itself is stable (1.0007 over 16x), but growth drove drift to 0.923 |
| hash | no | — | healthy sawtooth spans 1.50x, and the fresh size depends on the heap's estimated row count |
| gin | no | — | healthy ratio fell 25% over 16x growth |
| brin | no | — | index size is nearly constant; ratio decays with table size |
| spgist | no | — | fresh ratio more than doubled over 16x growth with 0.0% reclaimable |

A flagged index means only "the index grew faster than its heap since the baseline." Before acting, compare the current index and heap bytes with their captured components or maintenance history, check whether the heap recently shrank or widened, and use an access-method-specific diagnostic where one exists. If the investigation justifies a rebuild, choose plain or concurrent `REINDEX` according to the required lock and availability tradeoff; `VACUUM FULL` and `CLUSTER` are broader heap rewrites that also rebuild the table's indexes ([index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3609), [cluster.c#finish_heap_swap](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1438-L1470)). Capture a new baseline after the chosen rebuild.

The scheme's blind spot cannot be fixed by tuning the threshold: in the scattered-delete fixture, ordinary `VACUUM` left drift at exactly 1.000 while a rebuild reclaimed up to 80% of the index file. Do not make this ratio the installation's only retained-space monitor.

### Follow-up: drift past a stored baseline versus "reindex when the index is larger than the heap"

The stored-ratio drift rule is unreliable but usable; "reindex when the index is larger than the heap" is worse on every measurement below and should not be used at all. Scored over 49 exact-pin cells — seven workloads times seven access methods, with `REINDEX TABLE` supplying ground truth — `drift >= 1.40` caught 13 of the 25 genuinely rebuild-worthy indexes with 3 false alarms, while `index > heap` caught 6 with 2 false alarms. Neither rule is a bloat measurement. Both read the same two numbers, so the second rule is not an independent check on the first.

The maintenance process in question rebuilds with `REINDEX INDEX CONCURRENTLY`. That does not change any score: on an idle table, `REINDEX INDEX CONCURRENTLY` and plain `REINDEX INDEX` produced byte-identical files on all seven access methods. What it does change is the cost of a false positive and the failure modes of an automated loop; see [What changes when the rebuild is REINDEX INDEX CONCURRENTLY](#what-changes-when-the-rebuild-is-reindex-index-concurrently).

The decisive objection to the absolute rule is arithmetic, not empirical. Because `drift = current_ratio / baseline_ratio`, the condition `index_bytes > heap_bytes` is identical to:

```text
current_ratio > 1   <=>   drift > 1 / baseline_ratio
```

So "the index is larger than the heap" *is* a drift rule. Its threshold is just not a number anyone chose: it is the reciprocal of whatever ratio the index happened to have on the day it was built.

#### The question's own numbers

The follow-up posits a baseline ratio of 1. That case was built directly: 200,000 rows, seven indexes on one heap, `pad` sized so the freshly built GIN index lands on **1.004900** (4,102 index blocks against a 4,082-block heap). Indexed keys were then re-written in 10% slices with a `VACUUM` after each round.

At a baseline ratio of exactly 1, the two rules collapse onto the same axis and the absolute rule becomes the *more* trigger-happy of the two — it fires at `drift > 1.00`, so any growth at all trips it:

| churn | GIN index blocks | current ratio | drift | rebuild reclaims | `drift >= 1.40` | `index > heap` |
|---|---|---|---|---|---|---|
| 0% (just built) | 4102 | 1.004900 | 1.000 | **0.0%** | no | **yes** |
| 10% | 4660 | 1.037862 | 1.033 | 12.0% | no | yes |
| 30% | 5206 | 1.159465 | 1.154 | 21.2% | no | yes |
| 50% | 6025 | 1.341871 | 1.335 | 31.9% | no | yes |
| 60% | 6163 | 1.372606 | 1.366 | 33.4% | no | yes |

The first row is the whole argument. The absolute rule ordered a rebuild of an index that had just been built and had 0.0% reclaimable space, and it kept ordering one at every later step, so it carried no information about the index's condition at any point. In this v17 run the drift rule made the opposite error on the same index: it never reached 1.40, so it stayed silent all the way to a 33.4% reclaimable fraction. Neither rule tracked the truth on GIN.

The rebuild reference in that table is a second index of the same definition built on the same live table and then dropped, which is the same work `REINDEX` performs without destroying the state under test ([index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3609)).

#### What a drift of 1.4 was actually worth

Same fixture, all seven access methods. "First crossing" is the first churn round at which drift reached 1.40:

| AM | first crossing | drift there | rebuild reclaims there | outcome |
|---|---|---|---|---|
| spgist | 30% churn | 1.406 | 26.8% | fired, modest payoff |
| btree | 50% churn | 1.541 | 41.0% | fired, real payoff |
| gin | never (1.366 at 60%) | 1.366 | **33.4%** | missed |
| gist | never (1.235 at 60%) | 1.235 | **33.7%** | missed |
| hash | never (1.103 plateau) | 1.103 | 17.9% | missed |
| bloom | never (1.001 flat) | 1.001 | 9.2% | missed |
| brin | never (0.909) | 0.909 | 0.0% | correctly quiet |

So the answer to "I saw 1.4, should I reindex?" is: in this fixture, yes — a rebuild reclaimed 27-41% of the index file wherever the threshold fired. The failure is on the other side. GIN and GiST accumulated 33.4% and 33.7% reclaimable space without ever reaching the threshold, and `bloom`'s index grew from 394 to 434 blocks in round 1 while the heap grew from 4,082 to 4,490 blocks in the same round, so the two factors cancelled.

#### Day zero: the absolute rule's threshold is set by heap row width

The reason `index > heap` behaves so erratically is that a freshly built index's ratio is a property of the *table's row width*, not of the index's health. Three tables, 200,000 rows each, identical index definitions, differing only in a `pad` column that no index references:

| AM | index blocks | ratio, heap 2858 blk | ratio, heap 3847 blk | ratio, heap 13334 blk |
|---|---|---|---|---|
| gin | 4102 | **1.435269** | **1.066285** | 0.307635 |
| gist | 1538 | 0.538139 | 0.399792 | 0.115344 |
| spgist | 1082 | 0.378586 | 0.281258 | 0.081146 |
| hash | 2562 / 822 | 0.296711 | 0.213673 | 0.192140 |
| btree | 551 | 0.192792 | 0.143228 | 0.041323 |
| bloom | 394 | 0.137859 | 0.102417 | 0.029549 |
| brin | 3 | 0.001050 | 0.000780 | 0.000225 |

Every index block count is identical down the column except hash, whose build-time estimate depends on the heap itself, as shown above. For the other six, only the denominator moved. That follows from the tuple layouts: an index tuple carries an 8-byte header holding just the heap TID and a length/flags word, followed by the indexed attributes only ([itup.h#IndexTupleData](../../../../raw/postgres-17/src/include/access/itup.h#L35-L59), [itup.h#IndexInfoFindDataOffset](../../../../raw/postgres-17/src/include/access/itup.h#L99-L110)), whereas a heap tuple carries a 23-byte header plus every column in the row ([htup_details.h#HeapTupleHeaderData](../../../../raw/postgres-17/src/include/access/htup_details.h#L153-L188)). Widening an unindexed column grows the heap and leaves the index untouched.

Converting each fresh ratio into the drift threshold the absolute rule implies, `1 / baseline_ratio`:

| AM | narrow heap | medium heap | wide heap |
|---|---|---|---|
| gin | **0.70** | **0.94** | 3.25 |
| gist | 1.86 | 2.50 | 8.67 |
| spgist | 2.64 | 3.56 | 12.32 |
| hash | 3.37 | 4.68 | 5.20 |
| btree | 5.19 | 6.98 | 24.20 |
| bloom | 7.25 | 9.76 | 33.84 |
| brin | 952.67 | 1282.33 | 4444.67 |

On one table the rule demands anywhere from a 0.94x to a 1282x change before it reacts, a spread of `1.066285 / 0.000780` = 1367x that is driven entirely by access method and row width. The two GIN cells below 1.00 are indexes the rule condemns before any workload runs. BRIN's column is unreachable in practice: its file stayed at 3 blocks at every scale measured on this page.

A related trap for *both* rules: the baseline is only stable once the table is large. A fresh B-tree on `(k)` over a `pad`-40 table measures:

| rows | heap blocks | index blocks | fresh ratio |
|---|---|---|---|
| 0 | 0 | 1 | undefined |
| 1 | 1 | 2 | 2.000000 |
| 10 | 1 | 2 | 2.000000 |
| 100 | 2 | 2 | 1.000000 |
| 1,000 | 11 | 5 | 0.454545 |
| 10,000 | 104 | 30 | 0.288462 |
| 100,000 | 1031 | 276 | 0.267701 |
| 1,000,000 | 10310 | 2745 | 0.266246 |

An empty index is one metapage block, written last by the build ([nbtree.h#BTREE_METAPAGE](../../../../raw/postgres-17/src/include/access/nbtree.h#L148), [nbtsort.c#_bt_uppershutdown-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1153-L1166)). A baseline captured at 1,000 rows and re-checked at 1,000,000 shows drift of 0.586 — a 41% *decrease* with no bloat anywhere — purely because the metapage and internal levels amortize away. Capture the baseline on a table that is already at working size.

#### The scored fixture matrix

Seven workloads, each a single 200,000-row heap carrying all seven indexes, each with the baseline captured immediately after `CREATE INDEX`, then `REINDEX TABLE` for ground truth. Labelling an index "rebuild-worthy" when the rebuild reclaims at least 25% of its current file:

| rule | true positive | false positive | false negative | true negative |
|---|---|---|---|---|
| `drift >= 1.40` | **13** | 3 | 12 | 21 |
| `index > heap` | 6 | 2 | **19** | 22 |

The 49 cells, condensed. "Rebuild-worthy AMs" lists the access methods whose rebuild reclaimed at least 25%; the drift column is the drift range over just those access methods, and the last column is which access methods `index > heap` flagged, whether or not they were rebuild-worthy:

| fixture | heap change | rebuild-worthy AMs | drift over those AMs | `index > heap` flagged |
|---|---|---|---|---|
| ascending growth 4x | 3847 -> 15385 blk | none | — (0.250-2.100 overall) | gin (**false alarm**) |
| random growth 4x | 3847 -> 15385 blk | btree | 1.307 | gin (**false alarm**) |
| key churn, 5 x 20% | 3847 -> 4616 blk | btree, gin, gist, hash, spgist | 1.169-1.994 | gin |
| scattered delete 80% | unchanged | btree, gist, gin, spgist, hash, bloom | **1.000 on all seven** | gin |
| heap tail truncate 75% | 3847 -> 962 blk | btree, gist, gin, spgist, hash, bloom | 3.999 on all seven | gin, gist, spgist |
| widen rows 40->400 | 3847 -> 17180 blk | btree, spgist, hash, bloom | **0.447-0.657** | none |
| delete-all + reload | unchanged | btree, gist, gin | 1.827-1.993 | gin |

Four readings:

- **The absolute rule's positives are almost all one index.** It flagged 7 of GIN's 7 cells, 1 of GiST's, 1 of SP-GiST's, and none of the other four access methods' 28 cells. GIN was over the line at build time, so on that index the rule is a constant "yes" that happened to be right five-sevenths of the time. Both of its false positives are also GIN.
- **The absolute rule never fires for B-tree, hash, `bloom`, or BRIN, at any bloat level in these fixtures.** In the scattered-delete fixture a rebuild reclaimed 79.7% of both the B-tree and the `bloom` index, whose current ratios were 0.143228 and 0.102417; the rule stayed silent on both.
- **Two fixtures defeat the drift rule outright, and the absolute rule barely rescues them.** A scattered 80% delete followed by `VACUUM` moved neither file, so drift is exactly 1.000 on all seven access methods against reclaimable fractions up to 80.0%; the absolute rule caught only the GIN index there, 1 of the 6 that were rebuild-worthy. Widening every row drove drift *down* to 0.447-0.657 while a rebuild would have reclaimed 49.9-65.9%, and the absolute rule caught none of those four.
- **The drift rule's false positives are BRIN and SP-GiST.** BRIN reaches 3.999 in the tail-truncation fixture against a 0.000 reclaimable fraction, because `brinbulkdelete()` removes nothing ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1284-L1301)); SP-GiST reaches 2.100 in both growth fixtures, also against 0.000, purely from key distribution.

#### The 40% threshold is not the weak part

Sweeping the drift threshold over the same 49 cells, with ground truth fixed at "a rebuild reclaims >= 25%":

| threshold | TP | FP | FN | TN |
|---|---|---|---|---|
| 1.10 | 15 | 5 | 10 | 19 |
| 1.20 | 14 | 3 | 11 | 21 |
| 1.30 | 14 | 3 | 11 | 21 |
| **1.40** | **13** | **3** | **12** | **21** |
| 1.50 | 12 | 3 | 13 | 21 |
| 1.75 | 11 | 3 | 14 | 21 |
| 2.00 | 6 | 3 | 19 | 21 |

Everything from 1.20 to 1.50 scores within one cell. The rule's misses are structural, not a tuning error: they are the fixtures where the two file sizes move together or where the index side does not move at all. Moving the ground-truth threshold instead leaves the ranking unchanged — at 10%, 25% and 40% reclaimable the drift rule scores 13/3 true/false positives against the absolute rule's 6/2, and at 50% it is 9/7 against 4/4.

The sharpest single statement of the ratio's unreliability is the band around drift 1.00. Of the 19 cells with drift between 0.90 and 1.10, the true reclaimable fraction ranges from **-8.4%** (a GiST index that `REINDEX` made *larger*) to **80.0%**. A drift reading of "no change" is consistent with the entire ground-truth range.

#### Why drift moves at all: decompose before acting

Every drift number is a quotient of two independent factors, and a large drift, a large index-side change, and a large reclaimable fraction are three different things. Measured `idx_factor` and `heap_factor` from the matrix:

| case | idx_factor | heap_factor | drift | reclaimable |
|---|---|---|---|---|
| B-tree key churn: real index growth, heap also grew | 2.392 | 1.200 | 1.994 | 58.2% |
| BRIN heap truncation: index never moved, heap shrank | 1.000 | 0.250 | **3.999** | **0.0%** |
| `bloom` key churn: index grew 20%, heap grew 20% | 1.201 | 1.200 | **1.001** | 16.7% |
| SP-GiST ascending growth: index outgrew the heap legitimately | 8.396 | 3.999 | **2.100** | **0.0%** |

The middle rows are maximal drift readings produced entirely by the denominator or by key distribution. The `bloom` row is a null drift reading produced by two real changes that cancelled.

A bare stored ratio cannot separate these, because it discards the two components at capture time. If this scheme is kept, store the index and heap byte counts rather than their quotient, and treat drift as a trigger to look at the components rather than as a decision.

#### Comparing both rules from the stored baseline

Read-only. Verified against the pinned 17.10 build, including an index whose comment is human text rather than a number:

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

WITH candidates AS MATERIALIZED (
    SELECT /* wiki_index_rule_compare_candidates */
           i.indexrelid,
           i.indrelid,
           n.nspname AS schema_name,
           ic.relname AS index_name,
           am.amname AS index_am,
           CASE WHEN d.description ~ '^[0-9]+([.][0-9]+)?$'
                THEN d.description::numeric END AS baseline_ratio
    FROM pg_index i
    JOIN pg_class ic ON ic.oid = i.indexrelid
    JOIN pg_namespace n ON n.oid = ic.relnamespace
    JOIN pg_am am ON am.oid = ic.relam
    LEFT JOIN pg_description d
           ON d.objoid = i.indexrelid
          AND d.classoid = 'pg_class'::regclass
          AND d.objsubid = 0
    WHERE ic.relkind = 'i'
      AND n.nspname !~ '^pg_'
      AND n.nspname <> 'information_schema'
), measured AS MATERIALIZED (
    SELECT /* wiki_index_rule_compare_measure */
           c.*,
           pg_relation_size(c.indexrelid) AS index_bytes,
           pg_relation_size(c.indrelid) AS heap_bytes
    FROM candidates c
)
SELECT /* wiki_index_rule_compare */
       m.schema_name,
       m.index_name,
       m.index_am,
       m.baseline_ratio,
       round(m.index_bytes::numeric / nullif(m.heap_bytes, 0), 6) AS current_ratio,
       round((m.index_bytes::numeric / nullif(m.heap_bytes, 0))
             / nullif(m.baseline_ratio, 0), 3) AS drift,
       round(1.0 / nullif(m.baseline_ratio, 0), 2) AS index_gt_heap_needs_drift,
       (m.index_bytes::numeric / nullif(m.heap_bytes, 0))
             / nullif(m.baseline_ratio, 0) >= 1.4 AS fires_drift_40pct,
       m.index_bytes > m.heap_bytes AS fires_index_gt_heap,
       pg_size_pretty(m.index_bytes) AS index_size,
       pg_size_pretty(m.heap_bytes) AS heap_main_size
FROM measured m
ORDER BY drift DESC NULLS LAST;
COMMIT;
```

`index_gt_heap_needs_drift` is the point of the query: it prints, per index, the drift the absolute rule is silently demanding. An index carrying a human comment comes back with `NULL` in the baseline and both rule columns while the query still succeeds; the `CASE` guard is what makes that safe ([syntax.sgml#expression-evaluation](../../../../raw/postgres-17/doc/src/sgml/syntax.sgml#L2535-L2560)).

#### What changes when the rebuild is REINDEX INDEX CONCURRENTLY

The scoring above is unaffected and the baseline survives, but an automated `REINDEX INDEX CONCURRENTLY` (RIC) loop carries costs and failure modes a plain-`REINDEX` loop does not.

**The ground truth does not move.** Two byte-identical 200,000-row tables carrying all seven indexes were put through the same deterministic five-round key churn, then one was rebuilt with `REINDEX INDEX CONCURRENTLY` and the other with plain `REINDEX INDEX`:

| AM | churned blocks | after RIC | after plain `REINDEX` | same | reclaimed |
|---|---|---|---|---|---|
| btree | 1318 | 551 | 551 | yes | 58.2% |
| gist | 2640 | 1222 | 1222 | yes | 53.7% |
| spgist | 2398 | 1169 | 1169 | yes | 51.3% |
| gin | 7404 | 4102 | 4102 | yes | 44.6% |
| hash | 1153 | 810 | 810 | yes | 29.7% |
| bloom | 473 | 394 | 394 | yes | 16.7% |
| brin | 3 | 3 | 3 | yes | 0.0% |

Identical on all seven, so every reclaimable fraction on this page describes what RIC would recover too. That equality was measured on a quiescent table; RIC's validation step inserts any tuples the concurrent build missed, so a table taking writes during the rebuild can finish larger.

**The baseline survives, and the index OID does not.** All seven comments came through the rebuild intact while every OID changed, because `index_concurrently_swap()` rewrites the `pg_description` row's `objoid` onto the new index ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1783)). Measured OIDs moved 58584 → 58603 for B-tree and similarly for the other six, with every comment preserved. This is the one place RIC is strictly better than a side table keyed on `indexrelid`, which the changed OIDs would break. It also means the process must **overwrite the comment after every rebuild**: the surviving value is the pre-rebuild baseline, which is now stale by exactly the amount that was just reclaimed.

**A false positive is cheaper, but not cheap.** RIC takes `ShareUpdateExclusiveLock` rather than the `ShareLock` on the table and `AccessExclusiveLock` on the index that plain `REINDEX INDEX` takes ([indexcmds.c#reindex-lockmode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2872), [indexcmds.c#RIC-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3793-L3796), [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3609)), so ordinary DML keeps running. Four costs remain:

- **Peak storage is old plus new.** Phase 1 creates a `_ccnew` copy ([indexcmds.c#create-copy](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3849-L3858)) and the old index is dropped only in the final phase ([indexcmds.c#RIC-drop-old](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4257-L4263)). A rebuild fired on a false positive temporarily *doubles* the very allocation the rule was trying to reduce.
- **It blocks VACUUM on that table for the whole run.** Lazy VACUUM also takes `ShareUpdateExclusiveLock` on the relation ([vacuum.c#vacuum_rel-lockmode](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2094-L2104)), and `ShareUpdateExclusiveLock` conflicts with itself ([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L109)). Every unnecessary rebuild suppresses the maintenance that actually limits retained space.
- **It waits on other sessions.** RIC has four `WaitForLockersMultiple` points — two at `ShareLock` ([indexcmds.c:3973](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3973), [indexcmds.c:4034](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4034)) and two at `AccessExclusiveLock` before set-dead and drop ([indexcmds.c:4200](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4200), [indexcmds.c:4234](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4234)) — plus the snapshot wait. A long-running transaction stalls the loop.
- **It cannot run inside a transaction block.** Measured: `ERROR:  REINDEX CONCURRENTLY cannot run inside a transaction block`. A maintenance job that wraps its work in `BEGIN`/`COMMIT` cannot issue it at all.

**Four cases silently or loudly break the loop.** Measured on the pinned build:

| case | v17 result |
|---|---|
| `REINDEX INDEX CONCURRENTLY` on an exclusion-constraint index | `ERROR:  concurrent index creation for exclusion constraints is not supported` ([index.c#exclusion-error](../../../../raw/postgres-17/src/backend/catalog/index.c#L1334-L1341)) |
| `REINDEX TABLE CONCURRENTLY` on a table that has one | `WARNING:  cannot reindex exclusion constraint index "pr.ex_r_excl" concurrently, skipping` ([indexcmds.c#exclusion-skip](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3571-L3576)) |
| `REINDEX INDEX CONCURRENTLY` on a temporary index | succeeds, but silently runs the **non-concurrent** path ([indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L616)); measured: the comment survived and the OID did not change |
| `REINDEX INDEX CONCURRENTLY` on a system catalog index | `ERROR:  cannot reindex system catalogs concurrently` ([indexcmds.c#catalog-error](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3662-L3666)) |

The exclusion-index skip is the dangerous one for this scheme. The index is never rebuilt, so its drift keeps climbing and the monitor keeps flagging it forever, once per cycle, with only a `WARNING` to show for it.

**A failed RIC leaves an index with no baseline.** A `REINDEX INDEX CONCURRENTLY` on a 3,000,000-row table, interrupted by `statement_timeout = '90ms'`, produced:

| index | `indisvalid` | `indisready` | `indislive` | blocks | comment |
|---|---|---|---|---|---|
| `big_k` | t | t | t | 8228 | `0.123456` |
| `big_k_ccnew` | f | f | t | 0 | *(none)* |

The tracked index kept its validity and its baseline, so the monitor is not corrupted. But the `_ccnew` leftover has no comment, because the swap that would have moved one never ran. [The baseline audit query](#the-baseline-audit-query) reported it as `no baseline stored`, and it must be dropped by hand. Note also that the leftover is invisible to the ratio: [the detection query](#the-detection-query) measures each index against the heap separately, so a 0-byte or partially built `_ccnew` never inflates the tracked index's own ratio.

#### What to do with these two rules

- **Do not adopt "reindex when the index is larger than the heap."** It is a drift rule with an arbitrary per-index threshold, it is unreachable for narrow-key indexes on wide tables, and it condemns healthy GIN indexes on the day they are built.
- **Keep the drift rule only as a screen**, with the per-access-method calibration in [Fixture thresholds, and what to do with a flagged index](#fixture-thresholds-and-what-to-do-with-a-flagged-index). It fired on 16 of the 49 cells and 13 of those were rebuild-worthy, so a reading of 1.4 is worth investigating; silence is worth nothing, since 12 rebuild-worthy indexes never reached it.
- **Never let either rule fire a rebuild on its own, even with `REINDEX INDEX CONCURRENTLY`.** RIC keeps writes running, but an unnecessary one still doubles peak index storage, blocks VACUUM on the table for its whole duration, and waits on other sessions at four lock-wait points; see [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md).
- **Re-capture the baseline as the last step of every rebuild, and handle the cases RIC refuses.** The comment survives the swap, so an un-refreshed baseline is the pre-rebuild value. Give the loop an explicit path for exclusion-constraint indexes and a step that finds and drops `_ccnew` leftovers from failed runs.
- **Confirm with a rebuild-relative measurement before acting.** For B-tree, hash and GiST that is `pgstattuple`; the contrib coverage gap for GIN, SP-GiST and BRIN is exactly why this scheme was proposed, and it is not closed by choosing a different threshold on the same two numbers ([pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L240-L308)).
- **A rebuilt index is not a maximally dense index.** `REINDEX` fills B-tree leaf pages to the default fillfactor of 90, not 100 ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L200), [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L733-L760)), while the heap's default fillfactor is 100 ([rel.h#HEAP_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/utils/rel.h#L349)).

### What changed since PostgreSQL 12

Every item below is attributed to a commit that is an ancestor of this page's pin, with the first release tag that contains it taken from the v17 checkout's own tag set. The `COMMENT`, `pg_description`, and `pg_relation_size` paths themselves carry no behavioral change in that range — their history in this window is copyright updates and catalog-declaration refactoring.

| Change | Commit | First release | Effect on this scheme |
|---|---|---|---|
| B-tree deduplication | `0d861bbb702` | 13.0 | A fresh B-tree over duplicate-heavy keys is smaller, so baselines captured on v12 and v17 are not comparable. |
| `pg_class.reltuples` is `-1` before the first VACUUM/ANALYZE | `3d351d916b2` | 14.0 | Makes the never-analyzed build-time estimate explicit; this is the path that inflated the hash index 6.2x above. |
| B-tree bottom-up index deletion | `d168b666823` | 14.0 | Non-HOT version churn grows a B-tree more slowly, so the drift signal on churn workloads is weaker than on v12. |
| B-tree recycles pages deleted during the same VACUUM | `9dd963ae253` | 14.0 | The delete-and-reload growth can disappear after one VACUUM instead of two — but only with a concurrent XID consumer, as measured above. |
| VACUUM bypasses index vacuuming for few dead tuples | `5100010ee4d` | 14.0 | A VACUUM can leave index entries in place, so "I vacuumed" no longer implies the index side was touched. |
| VACUUM wraparound failsafe | `1e55e7d1755` | 14.0 | Under wraparound pressure VACUUM skips index vacuuming entirely, decoupling drift from maintenance activity. |
| Cumulative statistics moved to shared memory | `5891c7a8ed8` | 15.0 | `n_live_tup`/`n_dead_tup` are no longer collector-file readings; `stats_fetch_consistency` now governs snapshot behavior ([monitoring.sgml#stats-fetch-consistency](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L252-L269)). |
| `COMMENT` lock level documented | `b2a76bb7d05` | 15.0 | The `SHARE UPDATE EXCLUSIVE` lock the capture takes is now stated in the reference page ([comment.sgml#lock-level](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L99)). |
| Parent index comments no longer propagate to partitions | `fee8cb94734` | 17.1 | The v12 data-loss shape (children silently overwritten with the parent's baseline) is gone; children now lose their comment instead, which the audit query detects. |

Two v12-era statements no longer hold and were removed rather than rewritten:

- The v12 page's GiST empty-page handling predates `4e514c6180f` ("Delete empty pages in each pass during GIST VACUUM", first released in 13.0), so GiST cleanup behavior in the cycle test is not the v12 behavior.
- The v12 page described statistics-collector lag of 500 ms and a per-transaction snapshot. In v17 that text is gone from the documentation, replaced by the shared-memory statistics model and `stats_fetch_consistency` ([monitoring.sgml#stats-fetch-consistency](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L252-L269)).

### Why this exists: the contrib boundary

The reason a home-grown ratio is attractive at all is that v17 has no core function that reports per-index bloat, and the contrib tools do not expose one uniform result for every access method. The generic `pgstattuple(regclass)` dispatcher handles heaps plus B-tree, hash, and GiST indexes; it rejects GIN, SP-GiST, BRIN, unknown index AMs, and invalid indexes ([pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L240-L308)). The extension does have a separate `pgstatginindex(regclass)`, but that reads only the GIN metapage version, pending-page count, and pending-tuple count — not a live/dead/free-space census ([pgstatindex.c#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L521-L580)). Installing the extension needs superuser: its control file does not set `superuser = false` or `trusted = true`, and the control-file parser defaults `superuser` to true ([pgstattuple.control](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5), [extension.c#read_extension_control_file](../../../../raw/postgres-17/src/backend/commands/extension.c#L785-L800)).

`pg_relation_size` has neither extension restriction: for an ordinary physical index, it measures the main-fork files without dispatching to the index access method ([dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371)). That uniform allocation measurement — not uniform bloat semantics — is the real argument for this design. Custom access methods were not tested.

One access-method-specific size confounder worth naming: GIN keeps a pending list. `gin_pending_list_limit` is a cleanup trigger, not a hard size bound: after the threshold is exceeded, insertion requests a non-forced cleanup that can contend with another cleanup process ([ginfast.c#pending-list-cleanup-trigger](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L449-L472)). The setting defaults to 4 MB and is `PGC_USERSET`, so it has session/transaction scope, and GIN also accepts it as a per-index reloption ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3577-L3587), [ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L602-L626)). Measured in the cycle test: after the reload the GIN index carried 443 pending pages, which is allocation the ratio counts but a rebuild removes.

### Catalog, generated-header, and extension boundary

The proposal writes an existing catalog; it does not add a catalog column or change the server build. `pg_description.h` defines the catalog and includes generated `pg_description_d.h`; it also declares the catalog's TOAST table and its unique index through the BKI macros that `genbki.pl` consumes ([pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L71)). `src/include/catalog/Makefile` lists `pg_description.h` in `CATALOG_HEADERS` and feeds that list to the `bki-stamp` rule that runs `genbki.pl` ([catalog/Makefile#CATALOG_HEADERS](../../../../raw/postgres-17/src/include/catalog/Makefile#L20-L45), [catalog/Makefile#bki-stamp](../../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143)). The generated `_d.h` file is therefore a build artifact, not a missing source dependency in the pinned checkout.

At runtime, `COMMENT` writes the existing `pg_description` row through `CreateComments()` ([comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L143-L227)). Detection needs no extension. The `bloom` measurements do cross into contrib and require that extension, while `pgstattuple` and `pageinspect` appear only as validation aids in this page's experiments.

### Tests in the pinned checkout

- Comment survival across `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` is directly covered ([create_index.sql#comments-preserved](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L940-L948), [create_index.out#comments-preserved](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2300-L2323)).
- Basic `COMMENT ON INDEX` creation, `NULL` deletion, and empty-string deletion are covered ([create_index.sql#index-comments](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L45-L51), [create_index.out#index-comments](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L31-L45)).
- `CREATE TABLE ... LIKE INCLUDING ALL` copying source-index comments is covered ([create_table_like.sql#index-comments-copied](../../../../raw/postgres-17/src/test/regress/sql/create_table_like.sql#L139-L146)).
- The partitioned-index `ALTER COLUMN TYPE` comment outcome is covered, and unlike v12 the v17 expected output records blank child comments as the intended result ([alter_table.sql#partitioned-comments](../../../../raw/postgres-17/src/test/regress/sql/alter_table.sql#L1499-L1516), [alter_table.out#partitioned-comments](../../../../raw/postgres-17/src/test/regress/expected/alter_table.out#L2217-L2249)).
- `ALTER TABLE ... ALTER COLUMN TYPE` on a plain table preserving index comments is covered ([alter_table.sql#plain-comments](../../../../raw/postgres-17/src/test/regress/sql/alter_table.sql#L2145-L2159)).
- There is **no** upstream test that compares index size to heap size, none that exercises a comment as machine-readable data, and none that asserts an index's block count after a delete-and-reload cycle. Everything in the measurement tables above was produced on a purpose-built 17.10 server, not by the regression suite.

## Context Reviewed

- `COMMENT` path end to end: grammar production, `CommentObject`, `CreateComments`, `DeleteComments`, `GetComment`, the `pg_description` catalog definition, and the `obj_description` SQL builtin in `system_functions.sql`.
- Object resolution and privileges for indexes: `get_object_address`, `get_relation_by_qualified_name`, `check_object_ownership`, and the lock conflict table.
- Comment lifecycle across DDL: `index_concurrently_swap`, `RelationTruncateIndexes`, `CREATE TABLE ... LIKE` comment cloning through `IndexStmt.idxcomment`, default and `--no-comments` dump/restore behavior, and the `create_index`, `create_table_like`, and `alter_table` regression suites.
- Size measurement: `pg_relation_size` one- and two-argument forms, the `system_functions.sql` wrapper, relation opening and missing-relation behavior, `calculate_relation_size`, `calculate_table_size`, `calculate_toast_table_size`, `pg_table_size`, `pg_indexes_size`, and the writers of `pg_class.relpages` relevant to the catalog-only alternative.
- Build-time sizing inputs: `hashbuild` -> `estimate_rel_size` -> `_hash_init` -> `_hash_init_metabuffer`, `table_block_relation_estimate_size`'s never-vacuumed fallback, and `get_rel_data_width`.
- Heap baseline behavior: ordinary VACUUM's conditional tail truncation in v17's rewritten `vacuumlazy.c`, and the heap/index rebuild boundaries in `REINDEX`, `VACUUM FULL`, and `CLUSTER`.
- Per-access-method storage behavior: default fill factors; hash splitpoint allocation and the overflow bitmap; BRIN pages-per-range, revmap capacity, and revmap extension; and the main-FSM, GIN-entry, SP-GiST redirect, contrib `bloom`, and BRIN bulk-delete paths.
- The v17 recyclability gates: `BTPageIsRecyclable`, `gistPageRecyclable`, `GinPageIsRecyclable`, `GlobalVisCheckRemovableFullXid`/`GlobalVisCheckRemovableXid`, and the `_bt_pendingfsm_init`/`_bt_pendingfsm_add`/`_bt_pendingfsm_finalize` same-VACUUM recycling path with its documented dependence on a concurrently advancing XID horizon.
- Update behavior: the indexed-attribute bitmap and HOT decision in `RelationGetIndexAttrBitmap()` and `heap_update()`.
- Catalog and build boundary: `pg_description.h`, generated `pg_description_d.h`, the catalog BKI make rules, and the DECLARE_TOAST/DECLARE_UNIQUE_INDEX_PKEY macros.
- Adjacent observability: `pg_stat_all_tables`, v17's shared-memory statistics and `stats_fetch_consistency`, `psql` verbose index display, `track_counts`, both query timeouts, GIN's global and per-index pending-list cleanup settings, and the `pgstattuple` access-method support matrix with its extension privilege gate.
- `REINDEX INDEX CONCURRENTLY` as the maintenance process's rebuild command: `ExecReindex` dispatch and its temporary-relation downgrade, `ReindexRelationConcurrently` per-relkind branches, the `ShareUpdateExclusiveLock` table/index/session locks against `reindex_index`'s `ShareLock`/`AccessExclusiveLock` pair, `index_concurrently_create_copy` and the final old-index drop that bracket peak storage, the four `WaitForLockersMultiple` calls, the exclusion-constraint error and skip, and the system-catalog error.
- Source history in the pinned checkout for every claim in [What changed since PostgreSQL 12](#what-changed-since-postgresql-12), each verified as an ancestor of the pin with its first containing release tag.
- Pinned checkout `raw/postgres-17/` at commit `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11, `REL_17_11-7-g786db8dcf16`); repinned from `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on 2026-08-17. Every measured number on this page was produced on the 17.10 server built from that previous pin and was **not** re-measured for the repin. The one commit in the range that touches a path used here is `28269fed661`, which adds the `INDEX_CREATE_DEFERRABLE` flag to `index_concurrently_create_copy` and a `reindex-conc-index-built` injection point ([index.c:1348-1352](../../../../raw/postgres-17/src/backend/catalog/index.c#L1348-L1352), [indexcmds.c:4015](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4015)). No fixture here creates a `DEFERRABLE` constraint, so the `REINDEX INDEX CONCURRENTLY`-versus-plain-`REINDEX` size equality, the four RIC refusal cases, and the interrupted-RIC `_ccnew` observation are unaffected. The one anchor correction made during the repin is the `psql \di+` `Description` citation, which now points at the `obj_description(c.oid, 'pg_class')` line ([describe.c:4002](../../../../raw/postgres-17/src/bin/psql/describe.c#L4002)).
- Exact-pin measurements, all on one isolated 17.10 server built from the pinned checkout with `autovacuum = off`: a three-scale fresh-ratio invariance sweep over seven access methods; a day-zero sweep over three heap row widths; a seven-point hash/B-tree sawtooth sweep; a hash build-time estimate probe with and without `ANALYZE`; a fresh-B-tree sweep from 0 to 1,000,000 rows; a 49-cell matrix of seven workloads times seven access methods with `REINDEX TABLE` ground truth; drift-threshold and ground-truth-threshold sensitivity sweeps; a stepwise churn fixture tuned to a GIN baseline ratio of 1.004900; a delete-and-reload cycle in one-, two-, and three-VACUUM arms with `pgstatindex`, `gin_metapage_info` and `hash_metapage_info` probes; a repeat of the one-VACUUM arm against a concurrent XID consumer; a TOAST denominator fixture; a 300-index survey timing comparison against the `relpages` variant; a nine-operation comment durability matrix; lock, ownership and read-visibility probes; a RIC-versus-plain-`REINDEX` size comparison over seven access methods; the four RIC refusal cases; and a `statement_timeout`-interrupted RIC on a 3,000,000-row table.

## Evidence Map

| Claim | Evidence |
|---|---|
| `COMMENT ON INDEX` takes `ShareUpdateExclusiveLock` on the index and requires ownership | [comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L78), [objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2382-L2400), [objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L1333-L1350), [gram.y#CommentStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L7049-L7058), [comment.sgml#lock-level](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L99) |
| One comment per index, stored as a single replaceable `pg_description` row with `objsubid = 0`; empty string deletes it | [pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L64), [comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L143-L227), [comment.sgml#description](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L87-L93) |
| The baseline is readable by any user in the database and has no privilege gate | [comment.sgml#notes](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L291-L300), [system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301) |
| The comment follows the index through `REINDEX CONCURRENTLY`, which changes the index OID | [index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1783), [create_index.sql#comments-preserved](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L940-L948), [create_index.out#comments-preserved](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2300-L2323) |
| `ALTER TABLE ... ALTER COLUMN TYPE` on a partitioned table leaves child index comments empty in v17, and the fix that changed this is in 17.1 | [alter_table.sql#partitioned-comments](../../../../raw/postgres-17/src/test/regress/sql/alter_table.sql#L1499-L1516), [alter_table.out#partitioned-comments](../../../../raw/postgres-17/src/test/regress/expected/alter_table.out#L2217-L2249) |
| Default dump/restore preserves baselines; `--no-comments` can omit them | [pg_dump.c#dumpIndex-comment](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L17085-L17093), [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L11996-L12000), [pg_backup_archiver.c#restore-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_backup_archiver.c#L180-L184) |
| `CREATE TABLE ... LIKE INCLUDING ALL` copies source-index comments into new indexes | [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1404-L1411), [create_table_like.sql#index-comments-copied](../../../../raw/postgres-17/src/test/regress/sql/create_table_like.sql#L139-L146) |
| `pg_relation_size(regclass)` is a SQL wrapper onto the `'main'` fork that opens the relation and `stat()`s each segment | [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289), [pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7488-L7495), [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343), [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371) |
| `pg_table_size` adds every fork plus TOAST and its index, which destroys the ratio | [dbsize.c#calculate_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L424-L452), [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L378-L422), [dbsize.c#pg_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L486-L503) |
| A fresh hash index is sized from an estimate, not the row count, so a wide never-analyzed heap inflates it | [hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L134-L137), [plancat.c#estimate_rel_size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1065-L1080), [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L710-L741), [hashpage.c#_hash_init_metabuffer-buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526) |
| Ordinary index VACUUM does not truncate the main fork; heap VACUUM can conditionally trim empty tail pages | [storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L288-L306), [vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2545), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2547-L2660), [spgvacuum.c#truncate-disabled](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L884-L901) |
| B-tree, GiST, GIN, and SP-GiST VACUUM record reusable pages in the index FSM rather than shrinking the file | [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059), [gistvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L229-L242), [ginvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L754-L794), [spgvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L845-L876), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L49-L56) |
| B-tree and GiST gate free-page reuse on the global visibility horizon | [nbtree.h#BTPageIsRecyclable](../../../../raw/postgres-17/src/include/access/nbtree.h#L290-L318), [gistutil.c#gistPageRecyclable](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L885-L908), [nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179), [gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L298-L309) |
| v17 can recycle B-tree pages deleted during the same VACUUM, but only if the XID horizon advances during it | [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059), [nbtpage.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2994-L3055), [nbtpage.c#pendingfsm-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3012-L3021) |
| B-tree free-page reuse is key-order agnostic: `_bt_getbuf` takes whatever block the FSM reports with at least half a page free | [nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L925), [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L38-L46) |
| GIN retains an entry tuple when its posting list becomes empty; its cleanup separately gates recyclable pages on the horizon | [ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L513-L564), [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829) |
| SP-GiST records new or empty non-root pages in the FSM, but redirect-to-placeholder cleanup has a horizon gate | [spgvacuum.c#spgvacuumpage](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L621-L680), [spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L493-L560) |
| Contrib `bloom` records new or deleted pages in the FSM | [blvacuum.c#blvacuumcleanup](../../../../raw/postgres-17/contrib/bloom/blvacuum.c#L169-L215) |
| Hash does not use the index FSM at all; it frees and reuses overflow pages through its own bitmap pages | [hashovfl.c#_hash_freeovflpage-bitmap](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L556-L590), [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L318-L332) |
| BRIN removes nothing on delete, so its size does not respond to a delete-and-reload cycle | [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1284-L1301) |
| Updating an indexed attribute prevents a HOT update and requires index maintenance | [relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5267-L5282), [heapam.c#HOT-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4141-L4155) |
| Hash index size is a step function of estimated tuples, allocated a whole splitpoint at a time at fillfactor 75 | [hashpage.c#_hash_init_metabuffer-buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526), [hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L783-L800), [hash.h#HASH_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/hash.h#L296) |
| BRIN revmap size changes in steps as the number of summarized heap ranges crosses page capacity | [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-17/src/include/access/brin.h#L39-L45), [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-17/src/include/access/brin_page.h#L93-L96), [brin_revmap.c#brinRevmapExtend](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L112-L120) |
| GIN's pending-list limit triggers a non-forced cleanup and is also a per-index reloption | [ginfast.c#pending-list-cleanup-trigger](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L449-L472), [ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L602-L626) |
| Default fill factors differ per access method, so fresh ratios differ per access method | [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L200), [hash.h#HASH_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/hash.h#L296), [spgist_private.h#SPGIST_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/spgist_private.h#L495), [gist_private.h#GIST_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/gist_private.h#L480) |
| An index tuple stores an 8-byte header plus the indexed attributes only, while a heap tuple stores a 23-byte header plus every column | [itup.h#IndexTupleData](../../../../raw/postgres-17/src/include/access/itup.h#L35-L59), [itup.h#IndexInfoFindDataOffset](../../../../raw/postgres-17/src/include/access/itup.h#L99-L110), [htup_details.h#HeapTupleHeaderData](../../../../raw/postgres-17/src/include/access/htup_details.h#L153-L188) |
| A rebuilt B-tree is filled to fillfactor 90, not 100, while the heap default fillfactor is 100 | [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L200), [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L733-L760), [rel.h#HEAP_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/utils/rel.h#L349) |
| A freshly built B-tree over no rows occupies one metapage block, written as the final build step | [nbtree.h#BTREE_METAPAGE](../../../../raw/postgres-17/src/include/access/nbtree.h#L148), [nbtsort.c#_bt_uppershutdown-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1153-L1166) |
| An unnecessary `REINDEX INDEX` costs a table `ShareLock` plus an index `AccessExclusiveLock`; RIC instead takes `ShareUpdateExclusiveLock` | [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3609), [indexcmds.c#reindex-lockmode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2872), [indexcmds.c#RIC-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3793-L3796) |
| RIC holds the old and new index at once: it creates a `_ccnew` copy and drops the old one only at the end | [indexcmds.c#create-copy](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3849-L3858), [indexcmds.c#RIC-drop-old](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4257-L4263) |
| RIC's `ShareUpdateExclusiveLock` on the heap conflicts with lazy VACUUM's own | [vacuum.c#vacuum_rel-lockmode](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2094-L2104), [lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L109) |
| RIC waits for lockers four times, twice at `ShareLock` and twice at `AccessExclusiveLock` | [indexcmds.c:3973](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3973), [indexcmds.c:4034](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4034), [indexcmds.c:4200](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4200), [indexcmds.c:4234](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4234) |
| RIC errors on exclusion-constraint and system-catalog indexes, skips exclusion indexes with a `WARNING` at table level, and silently downgrades for temporary relations | [index.c#exclusion-error](../../../../raw/postgres-17/src/backend/catalog/index.c#L1334-L1341), [indexcmds.c#exclusion-skip](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3571-L3576), [indexcmds.c#catalog-error](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3662-L3666), [indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L616) |
| A `CASE` guard is the documented way to force evaluation order around the numeric cast | [syntax.sgml#expression-evaluation](../../../../raw/postgres-17/doc/src/sgml/syntax.sgml#L2535-L2560) |
| Partitioned indexes are relkind `'I'` and have no storage | [pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L163-L175), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L197-L202) |
| `relpages` is updated by VACUUM/ANALYZE and by index-build paths, but ordinary DML can leave it stale | [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1400-L1440), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2829) |
| `n_live_tup` and `n_dead_tup` are estimates, and v17 governs their snapshot behavior with `stats_fetch_consistency` | [monitoring.sgml#estimated-tuples](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L3812-L3828), [monitoring.sgml#stats-fetch-consistency](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L252-L269) |
| `pg_description_d.h` and the BKI input are generated from catalog headers during the build | [pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L71), [catalog/Makefile#CATALOG_HEADERS](../../../../raw/postgres-17/src/include/catalog/Makefile#L20-L45), [catalog/Makefile#bki-stamp](../../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143) |
| Generic `pgstattuple(regclass)` covers heaps, B-tree, hash, and GiST; separate `pgstatginindex` reports only GIN pending-list metadata; extension installation needs superuser | [pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L240-L308), [pgstatindex.c#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L521-L580), [pgstattuple.control](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5), [extension.c#read_extension_control_file](../../../../raw/postgres-17/src/backend/commands/extension.c#L785-L800) |
| Query timeouts and `track_counts` used above are session/transaction scope | [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2612-L2621), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2623-L2632), [guc_tables.c#track_counts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1412-L1420) |
| `psql \di+` prints the stored baseline in its `Description` column | [describe.c#listTables-description](../../../../raw/postgres-17/src/bin/psql/describe.c#L4000-L4004) |

## Open Questions

- Every number in the measurement tables comes from one purpose-built 17.10 server with synthetic fixtures (200k-1.6M rows, one narrow payload column, a fixed set of seven index definitions). The per-access-method drift thresholds are calibrated to those fixtures, not to any production workload. Different key widths, key distributions, fill factors, or table shapes will move them.
- The fixtures on this page are not the same fixtures the PostgreSQL 12 page used, and its numbers are not citable here. No table on this page should be read as a v12-to-v17 measured delta. The only version-to-version claims made here are in [What changed since PostgreSQL 12](#what-changed-since-postgresql-12), and each is sourced from the v17 checkout's own commit history, not from comparing measurements.
- The SP-GiST fresh-ratio non-invariance (drift 2.2787 over 16x growth, 0.0% reclaimable) was traced to the fixture's `point(g, (g * 37) % 1000)` key distribution by inspection, not to a specific code path in `spgdoinsert.c`. Whether a different point distribution restores scale invariance was not measured.
- The GiST and GIN cases where `REINDEX` produced a *larger* index than incremental insertion were observed but not traced to the responsible build path in source; they are reported as measurements only.
- The concurrent-XID B-tree experiment used a loop of separate `txid_current()` connections, which is not a realistic workload shape. How much of the 547-page backlog a real concurrent workload would recycle, and how the result varies with VACUUM duration, was not measured. The reported 745-block figure is one run and was not repeated.
- The `work_mem`-bounded size of the `_bt_pendingfsm_init` buffer means a VACUUM that deletes very many pages can stop tracking some of them entirely. That bound was not exercised: the fixture's 547 deleted pages are far below the default limit.
- The scheme was not tested against `CREATE ACCESS METHOD` custom index AMs, only against the six core access methods plus contrib `bloom`.
- The 300-index survey timings were taken on a warm, idle, single-user server with all relation files in page cache. Cold-cache `stat()` cost on an installation with tens of thousands of relations was not measured.
- The delete-and-reload cycle's idle arms ran with `autovacuum = off` and no concurrent transactions. On a real server autovacuum decides the `VACUUM` count, so the one-, two- and three-pass arms bracket the behavior rather than predict it.
- The 13/3/12/21 and 6/2/19/22 scorecards depend on a chosen ground-truth label, "a rebuild reclaims at least 25% of the current index file". That threshold is a labelling convention for this page, not a v17 definition of bloat; the checkout defines no such quantity. The reported sweep at 10%, 25%, 40% and 50% shows the ranking of the two rules is stable across those choices, but the absolute counts are not.
- The seven workloads are a chosen sample of failure shapes, not a workload distribution. The random-growth fixture randomizes only the `k` column, so its GiST, SP-GiST, GIN, `bloom` and BRIN rows are close to the ascending-growth fixture by construction; only the B-tree and hash rows are an independent observation of random-key insertion.
- The stepwise scenario's ground truth is a second index built on the live table rather than an actual `REINDEX`. The two perform the same build, but the probe was not compared against a real `REINDEX` on the same state at every step, and the probe index's own build competed for the same buffer cache.
- The hash build-time estimate finding was measured with `text` payload columns and no extended statistics. Which column types and statistics states produce the largest over-estimate, and whether the same effect changes the fresh size of any non-hash access method, was not measured.
- The `REINDEX INDEX CONCURRENTLY` versus plain `REINDEX INDEX` size equality was measured on a quiescent single-user table; under concurrent writes RIC can produce a larger file, and that case was not measured.
- The interrupted-RIC leftover was caught early enough to hold 0 blocks. How much space a `_ccnew` holds when the interruption lands mid-build, and whether an automated loop's retries accumulate several of them, were not measured.
- Whether reindexing a leftover `_ccnew` by name repairs or duplicates it was not tested.
- No upstream test in the pinned checkout compares `REINDEX CONCURRENTLY` output size against plain `REINDEX` output size, so the size-equality table has no regression coverage behind it.
- The v17 partitioned-child comment behavior was verified against the pinned checkout's regression expectations and reproduced on the test server, but whether any later 17-branch minor release changes it again cannot be checked from this pin.

## Source References

- [comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L39-L131)
- [comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L143-L227)
- [comment.c#DeleteComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L326-L380)
- [gram.y#CommentStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L7049-L7058)
- [pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L71)
- [catalog/Makefile#CATALOG_HEADERS](../../../../raw/postgres-17/src/include/catalog/Makefile#L20-L45)
- [catalog/Makefile#bki-stamp](../../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143)
- [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)
- [system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301)
- [pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7488-L7495)
- [objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L1333-L1350)
- [objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2382-L2400)
- [index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1783)
- [index.c#exclusion-error](../../../../raw/postgres-17/src/backend/catalog/index.c#L1334-L1341)
- [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2829)
- [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3609)
- [indexcmds.c#temp-fallback](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L605-L616)
- [indexcmds.c#reindex-lockmode](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2872)
- [indexcmds.c#ReindexRelationConcurrently](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3451-L3466)
- [indexcmds.c#exclusion-skip](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3571-L3576)
- [indexcmds.c#catalog-error](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3662-L3666)
- [indexcmds.c#RIC-locks](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3793-L3796)
- [indexcmds.c#create-copy](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3849-L3858)
- [indexcmds.c#RIC-drop-old](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4257-L4263)
- [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1400-L1440)
- [vacuum.c#vacuum_rel-lockmode](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2094-L2104)
- [cluster.c#finish_heap_swap](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1438-L1470)
- [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3056-L3101)
- [storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L288-L306)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371)
- [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L378-L422)
- [dbsize.c#calculate_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L424-L452)
- [dbsize.c#pg_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L486-L503)
- [dbsize.c#pg_indexes_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L505-L522)
- [vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2545)
- [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2547-L2660)
- [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059)
- [nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179)
- [nbtree.h#BTPageIsRecyclable](../../../../raw/postgres-17/src/include/access/nbtree.h#L290-L318)
- [nbtree.h#BTREE_METAPAGE](../../../../raw/postgres-17/src/include/access/nbtree.h#L148)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L200)
- [nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L925)
- [nbtpage.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2994-L3055)
- [nbtpage.c#pendingfsm-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3012-L3021)
- [nbtpage.c#pendingfsm-loop](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3034-L3052)
- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L733-L760)
- [nbtsort.c#_bt_uppershutdown-metapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1153-L1166)
- [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L38-L46)
- [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L49-L56)
- [gistvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L229-L242)
- [gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L298-L309)
- [gistutil.c#gistPageRecyclable](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L885-L908)
- [ginvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L754-L794)
- [ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L513-L564)
- [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)
- [ginbulk.c#ginGetBAEntry](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L250-L274)
- [gininsert.c#ginbuild-entry-insert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L389-L415)
- [ginfast.c#pending-list-cleanup-trigger](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L449-L472)
- [ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L602-L626)
- [spgvacuum.c#fsm](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L845-L876)
- [spgvacuum.c#spgvacuumpage](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L621-L680)
- [spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L493-L560)
- [spgvacuum.c#truncate-disabled](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L884-L901)
- [spgist_private.h#SPGIST_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/spgist_private.h#L495)
- [gist_private.h#GIST_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/gist_private.h#L480)
- [blvacuum.c#blvacuumcleanup](../../../../raw/postgres-17/contrib/bloom/blvacuum.c#L169-L215)
- [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1284-L1301)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-17/src/include/access/brin.h#L39-L45)
- [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-17/src/include/access/brin_page.h#L93-L96)
- [brin_revmap.c#brinRevmapExtend](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L112-L120)
- [hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L134-L137)
- [hashpage.c#_hash_init_metabuffer-buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526)
- [hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L783-L800)
- [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L318-L332)
- [hashovfl.c#_hash_freeovflpage-bitmap](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L556-L590)
- [hash.h#HASH_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/hash.h#L296)
- [plancat.c#estimate_rel_size](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1065-L1080)
- [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L710-L741)
- [relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L5267-L5282)
- [heapam.c#HOT-decision](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4141-L4155)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/utils/rel.h#L349)
- [itup.h#IndexTupleData](../../../../raw/postgres-17/src/include/access/itup.h#L35-L59)
- [itup.h#IndexInfoFindDataOffset](../../../../raw/postgres-17/src/include/access/itup.h#L99-L110)
- [htup_details.h#HeapTupleHeaderData](../../../../raw/postgres-17/src/include/access/htup_details.h#L153-L188)
- [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L29-L36)
- [pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L163-L175)
- [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L197-L202)
- [amapi.h#IndexAmRoutine-amoptions](../../../../raw/postgres-17/src/include/access/amapi.h#L276-L281)
- [reloptions.c#unrecognized-parameter](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L1470-L1480)
- [reloptions.c#index_reloptions](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L2058-L2072)
- [lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L109)
- [guc_tables.c#track_counts](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1412-L1420)
- [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2612-L2621)
- [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2623-L2632)
- [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3577-L3587)
- [pg_dump.c#dumpIndex-comment](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L17085-L17093)
- [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L11996-L12000)
- [pg_backup_archiver.c#restore-no-comments](../../../../raw/postgres-17/src/bin/pg_dump/pg_backup_archiver.c#L180-L184)
- [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1404-L1411)
- [describe.c#listTables-description](../../../../raw/postgres-17/src/bin/psql/describe.c#L4000-L4004)
- [pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L240-L308)
- [pgstatindex.c#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L521-L580)
- [pgstattuple.control](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [extension.c#read_extension_control_file](../../../../raw/postgres-17/src/backend/commands/extension.c#L785-L800)
- [comment.sgml#description](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L87-L93)
- [comment.sgml#lock-level](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L99)
- [comment.sgml#notes](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L291-L300)
- [syntax.sgml#expression-evaluation](../../../../raw/postgres-17/doc/src/sgml/syntax.sgml#L2535-L2560)
- [monitoring.sgml#stats-fetch-consistency](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L252-L269)
- [monitoring.sgml#estimated-tuples](../../../../raw/postgres-17/doc/src/sgml/monitoring.sgml#L3812-L3828)
- [create_index.sql#index-comments](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L45-L51)
- [create_index.out#index-comments](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L31-L45)
- [create_index.sql#comments-preserved](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L940-L948)
- [create_index.out#comments-preserved](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2300-L2323)
- [create_table_like.sql#index-comments-copied](../../../../raw/postgres-17/src/test/regress/sql/create_table_like.sql#L139-L146)
- [alter_table.sql#partitioned-comments](../../../../raw/postgres-17/src/test/regress/sql/alter_table.sql#L1499-L1516)
- [alter_table.out#partitioned-comments](../../../../raw/postgres-17/src/test/regress/expected/alter_table.out#L2217-L2249)
- [alter_table.sql#plain-comments](../../../../raw/postgres-17/src/test/regress/sql/alter_table.sql#L2145-L2159)

## Navigation

- [v17/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](create-index-concurrently.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md)
- [PostgreSQL 17 Contrib Extensions (unverified)](../server-administration/contrib-extensions.md)
