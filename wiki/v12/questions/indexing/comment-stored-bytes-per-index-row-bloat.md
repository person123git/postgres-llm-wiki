---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Calibrating a COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What the ratio measures, input by input](#what-the-ratio-measures-input-by-input)
  - [Who writes an index's reltuples, and what it counts](#who-writes-an-indexs-reltuples-and-what-it-counts)
  - [Why index bytes are a high-water mark](#why-index-bytes-are-a-high-water-mark)
  - [Calibration fixture and workloads](#calibration-fixture-and-workloads)
  - [The measured drift-versus-reclaim matrix](#the-measured-drift-versus-reclaim-matrix)
  - [Calibrated thresholds per index type](#calibrated-thresholds-per-index-type)
  - [GIN: the denominator changes units at every rebuild](#gin-the-denominator-changes-units-at-every-rebuild)
  - [BRIN: the denominator counts ranges, not rows](#brin-the-denominator-counts-ranges-not-rows)
  - [B-tree partial indexes: what the swap fixes](#b-tree-partial-indexes-what-the-swap-fixes)
  - [Refresh discipline: what VACUUM (ANALYZE) does not update](#refresh-discipline-what-vacuum-analyze-does-not-update)
  - [The noise floor, by predicate selectivity](#the-noise-floor-by-predicate-selectivity)
  - [Two bloat shapes, and only one repays a rebuild](#two-bloat-shapes-and-only-one-repays-a-rebuild)
  - [Bloat the planner charges for even when no page is read](#bloat-the-planner-charges-for-even-when-no-page-is-read)
  - [Collateral damage: a partial index's predicate column blocks HOT for the whole table](#collateral-damage-a-partial-indexs-predicate-column-blocks-hot-for-the-whole-table)
  - [Reading reltuples without losing digits](#reading-reltuples-without-losing-digits)
  - [Tested capture and detection SQL](#tested-capture-and-detection-sql)
  - [The runbook, executed on the pin](#the-runbook-executed-on-the-pin)
  - [Operating rules, failure modes, and cheaper fixes](#operating-rules-failure-modes-and-cheaper-fixes)
  - [Caller, callee, data structures, and build boundaries](#caller-callee-data-structures-and-build-boundaries)
  - [Test coverage in the pinned tree](#test-coverage-in-the-pinned-tree)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Record a stable post-build baseline for every index in its PostgreSQL index comment using `COMMENT ON INDEX`. Calculate the baseline as **index size in bytes divided by the table's estimated live-row count**, producing bytes per live row. During maintenance, recalculate the ratio and compare it with the stored baseline; an increase indicates that the index is consuming more space per live row. For each index type and workload, determine the percentage increase that serves as the best proxy for triggering a reindex while minimizing unnecessary rebuilds and missed harmful bloat. Calibrate this threshold using realistic insert, update, and delete workloads, then rebuild each index and measure the actual space reclaimed and query-performance improvement. Include index-specific factors, such as the GIN pending list, when evaluating the accuracy of the proxy.

Target version: PostgreSQL 12.

Follow-up: Add a section with a comprehensive analysis for B-tree partial indexes. Also include this in the analysis: Partial indexes: the denominator is wrong by construction.

Follow-up: replace live rows by the index `pg_class.reltuples`.

## Answer

### Verdict

Divide by the **index's own** `pg_class.reltuples`, not the table's. The swap costs nothing on ordinary indexes, fixes partial indexes, and disqualifies two access methods that the table denominator only appeared to support.

Three measured facts decide it, all from one isolated PostgreSQL 12.2 server built from the pin:

1. **On ordinary indexes the swap changes nothing.** Across all 192 plain-`ANALYZE` observations of a 12-workload x 8-index matrix, an index's `reltuples` equalled its table's exactly, because `compute_index_stats()` skips any index that is neither partial nor expression-bearing and leaves `tupleFract` at 1.0 ([analyze.c#skip-plain-index](../../../../raw/postgres-12/src/backend/commands/analyze.c#L730-L732), [analyze.c#tupleFract-init](../../../../raw/postgres-12/src/backend/commands/analyze.c#L437-L438), [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)). Drift therefore matched to four decimals in all 84 non-BRIN cells when both denominators were read at the same instant; the only two cells where the recommended `VACUUM (ANALYZE)` discipline changed a number are GIN indexes whose *numerator* moved, because that VACUUM flushed a pending list and extended the fork from 15,228,928 to 26,435,584 bytes and from 109,379,584 to 114,466,816 bytes.
2. **On partial indexes it is the difference between a working screen and a broken one.** Over 13 partial-B-tree cells the table denominator was right on 8 of 13; the index denominator was right on all 10 cells where it is defined, and the 3 undefined cells are exactly the indexes whose own `reltuples` is `0`, which a separate empty-index rule catches with more certainty than any drift number.
3. **Two access methods do not count rows at all.** After a build, a GIN index's `reltuples` is the number of extracted *entries* ([gininsert.c#indtuples](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L260-L272)), and a BRIN index's is the number of *range summary tuples* ([brin.c#brinbuild-idxtuples](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L726-L742)). Measured: one 20-keys-per-row GIN index reported 4,200,000 rows for a 400,000-row table, so a baseline captured after its rebuild made the very next comparison read drift `10.50` with zero bloat. One 24,576-byte BRIN index reported `0.0123` and `279.2727` bytes per index row at different times without changing a single byte, a factor of 22,705.

Calibrated result on the 96-cell matrix, with harmful defined as "`REINDEX` reclaims at least 25% of the index":

| Access method | Trigger | Benign max / harmful min | Verdict for the index denominator |
|---|---|---|---|
| B-tree | +30% (`drift >= 1.30`) | 1.0000 / 1.4991 | usable; widest clean band |
| hash | +30% | 1.1398 / 1.3978 | usable; the band is only 0.26 wide |
| contrib `bloom` | +30% | 1.2513 / 2.0000 | usable |
| GiST | +30% | 1.0000 / 1.0057 | usable with one blind spot: in-domain churn |
| SP-GiST | +30% | 1.0000 / 1.0000 | usable with the same blind spot |
| GIN | none | 1.1795 / 1.0899 with `fastupdate = on`, 1.0007 / 1.0000 with it off | **do not use**: bands inverted, and the unit changes at every rebuild |
| BRIN | none | 2.0000 / no harmful cell | **do not use**: the denominator counts ranges |

Scored as a classifier over all 96 cells, `drift >= 1.30` gave 42 true positives, 1 false positive, 4 false negatives and 49 true negatives (94.8%). Dropping BRIN's 12 cells leaves 42 / 0 / 4 / 38 (95.2%). All four false negatives are the same shape: churn that reinserts keys drawn from the *existing* key domain, where the index never grows but a rebuild still reclaims 26% to 56%.

Store the baseline, alert on it, and require a second, access-method-specific confirmation before rebuilding. Never wire the ratio straight to `REINDEX`.

### What the ratio measures, input by input

```text
bpr      = pg_relation_size(index_oid) / <that index's pg_class.reltuples>
drift    = bpr_now / bpr_baseline
increase = (drift - 1) * 100
```

**Numerator.** The one-argument `pg_relation_size(regclass)` is a SQL-language wrapper that passes `'main'`, and the C implementation opens the relation under `AccessShareLock` and `stat()`s every segment of that one fork ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)). It therefore measures *allocated* bytes, including pages that are empty, deleted, or recorded as free in the index free space map. It excludes the index FSM fork, which `pg_table_size` and `pg_total_relation_size` would include ([dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467)).

**Denominator.** `pg_class.reltuples` is a `float4` documented in the catalog header as "# of tuples (not always up-to-date)" ([pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)). An index has its own `pg_class` row and therefore its own `reltuples`, written by a different set of code paths from the table's. Which paths, and what they count, is the whole subject of the next section.

**Ratio semantics.** The value you must store is the measured baseline, for example `22.609920`, not the literal `1.0`. The `1.0` is the normalized starting drift, `bpr_baseline / bpr_baseline`. Storing `1.0` throws away the scale that later comparisons need.

The core reason the ratio is only a screen: PostgreSQL 12 exposes no access-method callback that reports reclaimable space. `IndexAmRoutine` carries build, insert, bulk-delete, cleanup, cost, options, validate, and scan callbacks and nothing about bloat or free space ([amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L233)). Any generic screen is inferring waste from allocation.

### Who writes an index's reltuples, and what it counts

Three writers touch an index's `reltuples`, and they disagree about what a "tuple" is.

| Writer | Value written | Path |
|---|---|---|
| `CREATE INDEX`, `REINDEX`, `REINDEX ... CONCURRENTLY` | `IndexBuildResult.index_tuples`, exactly as the access method counted it | [index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987), [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680), [index.c#index_update_stats-reltuples](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2786) |
| plain `ANALYZE` | `ceil(tupleFract * totalrows)`, where `tupleFract` is 1.0 unless the index is partial | [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629), [analyze.c#tupleFract-estimate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L817-L822) |
| `VACUUM`, including the VACUUM half of `VACUUM (ANALYZE)` | `IndexBulkDeleteResult.num_index_tuples`, but only when the access method says the count is exact | [vacuumlazy.c#lazy_cleanup_index](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1798-L1815) |

`ANALYZE` deliberately steps aside inside VACUUM so the exact count survives: "Vacuum always scans all indexes, so if we're part of VACUUM ANALYZE, don't overwrite the accurate count already inserted by VACUUM" ([analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L612)).

What each access method reports differs, and `genam.h` warns that it may: "Some index AMs may compute num_index_tuples by reference to num_heap_tuples, in which case they should copy the estimated_count field" ([genam.h#IndexBulkDeleteResult](../../../../raw/postgres-12/src/include/access/genam.h#L55-L81)).

| Access method | Build result counts | VACUUM cleanup reports | Refreshes on every VACUUM? |
|---|---|---|---|
| B-tree | one per indexed heap tuple ([nbtsort.c#btbuild-result](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L351-L355)) | leaf items counted by the scan, clamped to the heap count ([nbtree.c#btvacuumscan-counts](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L967-L973), [nbtree.c#btvacuumcleanup-clamp](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L929-L941)) | no: `btvacuumcleanup()` returns NULL when `_bt_vacuum_needs_cleanup()` declines ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L927)) |
| hash | one per indexed heap tuple ([hash.c#hashbuild-result](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L179-L188)) | tuples counted by `hashbulkdelete()`, or dead-reckoned from the metapage ([hash.c#hashbulkdelete-count](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L574-L632)) | no: `hashvacuumcleanup()` returns NULL when no bulk delete ran ([hash.c#hashvacuumcleanup](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L640-L656)) |
| GiST | one per indexed heap tuple ([gistbuild.c#gistbuild-result](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L228-L236)) | leaf items counted by the scan, clamped to the heap count ([gistvacuum.c#gistvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L100-L145)) | yes: it scans when no bulk delete ran |
| SP-GiST | one per indexed heap tuple ([spginsert.c#spgbuild-result](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L145-L149)) | leaf items counted by the scan, clamped to the heap count ([spgvacuum.c#spgvacuumcleanup](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L931-L969)) | yes |
| contrib `bloom` | one per indexed heap tuple ([blinsert.c#blbuild-result](../../../../raw/postgres-12/contrib/bloom/blinsert.c#L155-L159)) | items counted page by page ([blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L168-L216)) | yes |
| GIN | one per extracted **entry**: `buildstate->indtuples += nentries` ([gininsert.c#indtuples](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L260-L272), [gininsert.c#ginbuild-result](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L419-L427)) | the **heap** tuple count, under an explicit XXX: "we always report the heap tuple count as the number of index entries. This is bogus if the index is partial" ([ginvacuum.c#ginvacuumcleanup-heap-count](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L723-L731)) | yes, unless VACUUM skipped heap pages |
| BRIN | one per **range summary tuple** ([brin.c#brinbuild-idxtuples](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L726-L742)) | summarized plus existing ranges, both counted into the same variable ([brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L790-L815), [brin.c#brinsummarize-counts](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1299-L1312)) | yes |

Measured on a fresh 200,000-row fixture, one index per access method, reading `reltuples` after the build, after a plain `ANALYZE`, and after a `VACUUM (ANALYZE)`:

| Index | Bytes | After build | After plain `ANALYZE` | After `VACUUM (ANALYZE)` |
|---|---|---|---|---|
| btree | 4,513,792 | 200,000 | 200,000 | 200,000 |
| hash | 6,733,824 | 200,000 | 200,000 | 200,000 |
| gist | 19,980,288 | 200,000 | 200,000 | 200,000 |
| spgist | 21,897,216 | 200,000 | 200,000 | 200,000 |
| gin | 11,206,656 | 200,000 | 200,000 | 200,000 |
| gin, `fastupdate = off` | 11,206,656 | 200,000 | 200,000 | 200,000 |
| brin | 24,576 | **40** | **200,000** | **39** |
| bloom | 3,227,648 | 200,000 | 200,000 | 200,000 |

GIN agrees at 200,000 only because this fixture stores one array element per row, so entries and rows coincide. BRIN disagrees immediately, and its two exact writers disagree with each other by one: `brinvacuumcleanup()` passes `include_partial = false`, so the partial range at the end of the table that the build counted is not counted again ([brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L790-L815), [brin.c#brinsummarize-counts](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1299-L1312)).

### Why index bytes are a high-water mark

No core index access method shrinks its own main fork in PostgreSQL 12. VACUUM returns reusable index pages to the index free space map instead, through `RecordFreeIndexPage()` ([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L56)):

| Access method | What VACUUM does with emptied pages |
|---|---|
| B-tree | records recyclable pages in the FSM and vacuums the FSM if any were found ([nbtree.c#recyclable-page](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1183), [nbtree.c#IndexFreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1085-L1095)) |
| GiST | unlinks empty leaf pages from the tree in a second stage, then records them ([gistvacuum.c#delete-empty-pages](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L115-L129), [gistvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L303-L312)) |
| SP-GiST | records new or empty pages ([spgvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L655-L665)) |
| GIN | records recyclable posting-tree pages and freed pending-list pages ([ginvacuum.c#recyclable](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L786), [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667)) |
| contrib `bloom` | records new or deleted pages ([blvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L195-L217)) |
| hash | no `RecordFreeIndexPage()` call exists under `src/backend/access/hash`; a freed overflow page is reinitialized as `LH_UNUSED_PAGE` and tracked in the hash index's own overflow bitmap ([hashovfl.c#reinit-freed-overflow](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L593-L611)) |
| BRIN | records free space on its own regular pages during insertion and vacuums the FSM at cleanup ([brin.c#FreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1490-L1500), [brin_pageops.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/brin/brin_pageops.c#L455-L466)) |

SP-GiST contains the only index-truncation code, and it is compiled out: the block is wrapped in `#ifdef NOT_USED` with the comment "disabled because it's unsafe due to possible concurrent inserts ... Note that btree doesn't do this either" ([spgvacuum.c#disabled-truncation](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)). For indexes, the only `RelationTruncate()` caller left is `TRUNCATE TABLE`, which resets every index to zero blocks ([heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3164-L3199)).

Consequence for the metric: with a stable row count, bytes per index row can only rise or stay flat, no matter how much of the index is reusable free space. "More space per index row" is literally true and still does not mean "wasted space a rebuild can recover".

### Calibration fixture and workloads

Everything below was measured on one isolated server built from the pinned checkout (`PostgreSQL 12.2 on x86_64-pc-linux-gnu`), configured with `autovacuum = off`, `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`, so that only explicit `VACUUM` and `ANALYZE` moved statistics.

The fixture is one table per workload, each with 200,000 rows and eight indexes, one per access method, each on its own column, plus a non-indexed `payload` column for HOT tests:

```sql
CREATE INDEX i_btree   ON t USING btree  (k_btree);
CREATE INDEX i_hash    ON t USING hash   (k_hash);
CREATE INDEX i_gist    ON t USING gist   (k_gist);     -- point
CREATE INDEX i_spgist  ON t USING spgist (k_spgist);   -- point
CREATE INDEX i_gin     ON t USING gin    (k_gin);      -- int[], fastupdate on (default)
CREATE INDEX i_ginoff  ON t USING gin    (k_gin2) WITH (fastupdate = off);
CREATE INDEX i_brin    ON t USING brin   (k_brin);
CREATE INDEX i_bloom   ON t USING bloom  (k_bloom1, k_bloom2);
```

Sequence per workload: build, capture the build-time denominator, `ANALYZE`, capture, `VACUUM (ANALYZE)`, capture the baseline, run the workload, `ANALYZE`, capture, `VACUUM (ANALYZE)`, capture the current state, `REINDEX TABLE`, capture the rebuilt size. Ground truth is `reclaim_pct = (bytes_before - bytes_after) / bytes_before`, measured against the last pre-rebuild capture. Reported drift uses the `VACUUM (ANALYZE)` captures at both ends, which is the refresh this page recommends.

| Workload | Definition | Observed table counters |
|---|---|---|
| `w_noop` | nothing after the build | 200,000 ins |
| `w_grow_seq` | append 200,000 rows, ascending keys | 400,000 ins |
| `w_grow_10x` | append 1,800,000 rows, ascending keys | 2,000,000 ins |
| `w_grow_rand` | append 200,000 rows, keys drawn from the existing domain | 400,000 ins |
| `w_hot_ff40` | heap `fillfactor = 40`, two full-table updates of the non-indexed column | 400,000 upd, **400,000 HOT** |
| `w_hot_upd` | default fillfactor, two full-table updates of the non-indexed column | 400,000 upd, **0 HOT** |
| `w_idx_upd` | two full-table updates that change every indexed column | 400,000 upd, 0 HOT |
| `w_del_range` | delete the 100,000 lowest ids, then `VACUUM` | 100,000 del |
| `w_del_scatter` | delete every second id, then `VACUUM` | 100,000 del |
| `w_del_novac` | the same scattered delete, no `VACUUM` | 100,000 del, 100,000 dead |
| `w_churn` | 4 rounds of insert 50,000 / delete 50,000, `VACUUM` per round, keys advance | 400,000 ins, 200,000 del |
| `w_churn2` | the same churn with keys kept inside the original domain | 400,000 ins, 200,000 del |

`w_hot_ff40` versus `w_hot_upd` is the one comparison that isolates HOT. Identical statements against a heap with and without free space produced 400,000 HOT updates and 0 HOT updates respectively, read from `pg_stat_all_tables.n_tup_hot_upd` ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)). "Update-heavy" on its own says nothing about index growth.

`w_del_novac` has no `VACUUM (ANALYZE)` capture, because vacuuming it would make it `w_del_scatter`; its cell uses the plain-`ANALYZE` capture.

### The measured drift-versus-reclaim matrix

Each cell is `drift / reclaim_pct`, drift computed against the index's own `reltuples`. Negative reclaim means the rebuild produced a **larger** index.

| Workload | btree | hash | gist | spgist | gin (fastupdate on) | gin (off) | brin | bloom |
|---|---|---|---|---|---|---|---|---|
| `w_noop` | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 |
| `w_grow_seq` | 0.9973 / 0.00 | 1.0894 / 8.65 | 0.9457 / -5.74 | 0.9989 / 0.00 | 1.1795 / 15.22 | 1.0000 / 0.00 | 0.5000 / 0.00 | 0.9987 / 0.13 |
| `w_grow_10x` | 0.9956 / -0.02 | 1.1398 / 12.54 | 0.9022 / -10.84 | 0.9984 / 0.00 | 1.0214 / 2.15 | 0.9995 / 0.00 | 0.1333 / 0.00 | 0.9959 / 0.03 |
| `w_grow_rand` | 0.9991 / 0.09 | 1.0888 / 5.64 | 0.5098 / -43.31 | 0.5004 / -0.04 | 0.6795 / 11.14 | 0.5000 / -20.76 | 0.5000 / 0.00 | 0.9987 / 0.13 |
| `w_hot_ff40` | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 |
| `w_hot_upd` | 3.9873 / 74.92 | 3.0973 / 67.71 | 1.9881 / 49.70 | 2.0112 / 50.28 | 1.3772 / 27.39 | 1.0007 / 0.07 | 0.3333 / 0.00 | 2.9924 / 66.58 |
| `w_idx_upd` | 3.9873 / 74.92 | 3.1496 / 68.25 | 1.8918 / 47.14 | 1.9993 / 49.98 | 2.3765 / 57.92 | 2.0000 / 50.00 | 0.3333 / 0.00 | 2.9924 / 66.58 |
| `w_del_range` | 2.0000 / 49.91 | 2.0000 / 37.47 | 2.0000 / 50.02 | 2.0000 / 49.98 | 2.0000 / 49.93 | 2.0000 / 49.93 | 1.0000 / 0.00 | 2.0000 / 49.75 |
| `w_del_scatter` | 2.0000 / 49.91 | 2.0000 / 37.47 | 2.0000 / 50.02 | 2.0000 / 49.98 | 2.0000 / 49.93 | 2.0000 / 49.93 | 1.0000 / 0.00 | 2.0000 / 49.75 |
| `w_del_novac` | 2.0000 / 49.91 | 2.0000 / 37.47 | 2.0000 / 50.02 | 2.0000 / 49.98 | 2.0000 / 49.93 | 2.0000 / 49.93 | 2.0000 / 0.00 | 2.0000 / 49.75 |
| `w_churn` | 1.4991 / 33.29 | 1.4002 / 29.19 | 1.4453 / 30.84 | 1.9978 / 50.19 | 2.0899 / 52.15 | 2.0000 / 50.00 | 0.8125 / 0.00 | 1.2513 / 20.08 |
| `w_churn2` | 1.9964 / 49.82 | 1.3978 / 26.28 | 1.0057 / 26.34 | 1.0000 / 56.15 | 1.0899 / 34.41 | 1.0000 / 28.51 | 0.8125 / 0.00 | 1.2513 / 20.08 |

What the matrix shows:

- **The three delete rows are identical at `2.0000` for the seven row-counting indexes** while reclaim spans 26.28% to 50.02%. Bytes never shrank and the row count halved, so drift is arithmetically `1 / 0.5` regardless of the access method. `w_del_novac` proves the same drift arrives without VACUUM, because plain `ANALYZE` alone moves the denominator.
- **`w_hot_ff40` is the perfect true negative.** 400,000 HOT updates left every index byte-identical: drift `1.0000`, reclaim 0.00%, on all eight.
- **Both update rows are strong true positives** for every access method except BRIN, whose bytes never move because `brinbulkdelete()` is a no-op ([brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)) and the updates fall inside already-summarized ranges, and `gin (off)` in `w_hot_upd`, which absorbed the new entries into existing posting lists.
- **Growth rows expose the false-positive risk.** Hash reached `1.1398` on healthy 10x growth with 12.54% reclaimable, because `hashbuild()` sizes the initial bucket count from `estimate_rel_size()` and then allocates whole splitpoint batches ([hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L108-L160), [hashpage.c#splitpoint-batch](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L779-L800)). BRIN fell to `0.1333` on the same growth.
- **`w_churn2` is where the signal fails.** Reinserting keys inside the original domain lets new entries land in pages the deleted entries vacated, so GiST read `1.0057`, SP-GiST and `gin (off)` read exactly `1.0000`, and rebuilds still reclaimed 26.34%, 56.15% and 28.51%. Four of the matrix's four false negatives at `1.30` are these cells.
- **GiST's rebuilds are not reproducible in size.** `gistchoose()` breaks equal-penalty ties with `random()` ([gistutil.c#gistchoose-random](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L502-L537)), which is why three growth rows show a rebuild landing 5.74% to 43.31% *larger* than the index it replaced.

### Calibrated thresholds per index type

Scoring all 96 cells against candidate triggers, with harmful defined as `reclaim_pct >= 25` (46 of the 96 cells are harmful):

| Trigger | TP | FP | FN | TN | Accuracy |
|---|---|---|---|---|---|
| 1.05 | 43 | 7 | 3 | 43 | 89.6% |
| 1.10 | 42 | 5 | 4 | 45 | 90.6% |
| 1.20 | 42 | 3 | 4 | 47 | 92.7% |
| **1.30** | **42** | **1** | **4** | **49** | **94.8%** |
| 1.40 | 40 | 1 | 6 | 49 | 92.7% |
| 1.50 | 37 | 1 | 9 | 49 | 89.6% |
| 1.75 | 37 | 1 | 9 | 49 | 89.6% |
| 2.00 | 32 | 1 | 14 | 49 | 84.4% |

Excluding BRIN's 12 cells, `1.30` scores 42 / 0 / 4 / 38 (95.2%) and `1.20` scores 42 / 2 / 4 / 36 (92.9%). The single false positive at `1.30` over the full set is BRIN's `w_del_novac` cell at `2.0000` with 0.00% reclaimable. The four false negatives are the `w_churn2` cells named above.

Per access method, the useful number is the **separating band** between the highest benign drift and the lowest harmful drift, because the trigger must sit inside it:

| Access method | Highest benign drift (workload) | Lowest harmful drift (workload) | Recommended trigger |
|---|---|---|---|
| btree | 1.0000 (`w_noop`) | 1.4991 (`w_churn`) | 1.30 |
| hash | 1.1398 (`w_grow_10x`, 12.54% reclaimable) | 1.3978 (`w_churn2`) | 1.30 |
| bloom | 1.2513 (`w_churn`, 20.08%) | 2.0000 (`w_del_*`) | 1.30 |
| gist | 1.0000 (`w_noop`) | 1.0057 (`w_churn2`, 26.34%) | 1.30, accepting the `w_churn2` miss |
| spgist | 1.0000 (`w_noop`) | 1.0000 (`w_churn2`, 56.15%) | 1.30, accepting the `w_churn2` miss |
| gin, `fastupdate = on` | 1.1795 (`w_grow_seq`, 15.22%) | 1.0899 (`w_churn2`) | none: band inverted |
| gin, `fastupdate = off` | 1.0007 (`w_hot_upd`, 0.07%) | 1.0000 (`w_churn2`) | none: band inverted |
| brin | 2.0000 (`w_del_novac`, 0%) | no harmful cell in 12 workloads | none |

Two honest limits. First, each band rests on 12 observations from one fixture shape, so the endpoints are this fixture's extremes, not a distribution. Second, "harmful" is a policy choice: at `reclaim_pct >= 50` the hash and bloom bands move, and an absolute floor in bytes relabels every small index.

### GIN: the denominator changes units at every rebuild

GIN is the access method the question calls out, and under the index denominator it fails in a new way: the number stored in `reltuples` means different things at different times.

`ginbuild()` counts one per extracted key, not one per row ([gininsert.c#indtuples](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L260-L272)). `ginvacuumcleanup()` overwrites it with the heap tuple count ([ginvacuum.c#ginvacuumcleanup-heap-count](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L723-L731)). Plain `ANALYZE` overwrites it with the table's row estimate. Measured on a table that starts with 200,000 single-key rows and then receives 200,000 rows carrying 20 keys each:

| Phase | Bytes | Index `reltuples` | Table `reltuples` | bpr |
|---|---|---|---|---|
| build, 200,000 rows x 1 key | 11,206,656 | 200,000 | 200,000 | 56.0333 |
| after the 20-key rows, plain `ANALYZE` | 35,782,656 | 400,000 | 400,000 | 89.4566 |
| after `VACUUM (ANALYZE)` | 35,782,656 | 400,000 | 400,000 | 89.4566 |
| after `REINDEX` | 32,047,104 | **4,200,000** | 400,000 | **7.6303** |
| after the next plain `ANALYZE` | 32,047,104 | 400,000 | 400,000 | 80.1178 |

A baseline recaptured immediately after that rebuild stores `7.6303`. The next maintenance window reads `80.1178` and reports drift `10.50` on an index nothing has touched. The rebuild itself reclaimed 10.44%, which the pre-rebuild drift of `1.5965` did flag, so the metric was not useless before the rebuild; it became meaningless after it.

The pending list is the second GIN-specific factor. In PostgreSQL 12 `fastupdate` defaults to **on** ([gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-12/src/include/access/gin_private.h#L23-L39)), so ordinary inserts land in a pending list and are merged only when the list outgrows `gin_pending_list_limit`, a `PGC_USERSET` GUC with a 4 MB default that a per-index reloption of the same name can override ([ginfast.c#needCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L462), [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [ginutil.c#gin-reloptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L604-L613)). Every scan must read the whole pending list first, because a pending entry can belong anywhere in the tree ([ginget.c#scanPendingInsert](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1750-L1760), [ginget.c#gingetbitmap-order](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1866-L1884)).

Measured on 200,000 single-key rows plus 200,000 more inserted with `gin_pending_list_limit = '1GB'` so nothing auto-flushed, probing `WHERE k @> ARRAY[350000]`, best of five:

| Phase | Bytes | bpr | drift | Probe blocks | Probe time |
|---|---|---|---|---|---|
| baseline | 11,206,656 | 56.0333 | 1.0000 | — | — |
| 200,000 rows in the pending list | 15,228,928 | 38.0723 | **0.6795** | 497 | 4.043 ms |
| after `gin_clean_pending_list()` | 26,435,584 | 66.0890 | **1.1795** | 6 | 0.006 ms |
| after `REINDEX` | 22,413,312 | 56.0333 | 1.0000 | 5 | 0.007 ms |

`pageinspect` confirmed 491 pending pages holding 200,000 pending tuples before the cleanup and 0 after. The pending list made the ratio fall 32% while making the probe 674x slower; the cheap correct fix, `gin_clean_pending_list()`, then raised the ratio 74% while restoring the query ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074)). Both moves are the wrong sign.

Do not screen GIN with this metric. Screen it with the pending-list counters from `pageinspect`, and remember that `ginGetStats()` documents everything except `nPendingPages` and `ginVersion` as being "as of the last VACUUM" ([ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L630-L655), [ginvacuum.c#ginUpdateStats](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L784)).

### BRIN: the denominator counts ranges, not rows

BRIN must be excluded, and the index denominator makes the case unarguable. Measured on a 1,000,000-row table, then 1,000,000 more rows, with the index's byte count never changing:

| Phase | Bytes | Index `reltuples` | bpr |
|---|---|---|---|
| build | 24,576 | 89 | 276.1348 |
| plain `ANALYZE` | 24,576 | 1,000,000 | 0.0246 |
| `VACUUM (ANALYZE)` | 24,576 | 88 | 279.2727 |
| +1,000,000 rows, plain `ANALYZE` | 24,576 | 2,000,000 | 0.0123 |
| `VACUUM (ANALYZE)` | 24,576 | 177 | 138.8475 |
| `REINDEX` | 24,576 | 178 | 138.0674 |

The same 24,576-byte index reports values 22,705x apart depending on which command ran last. Under the table denominator BRIN was merely uninformative; under the index denominator it is actively hostile, so the shipped detection query refuses it by access method rather than by threshold.

The underlying behavior is unchanged from the table-denominator analysis: BRIN's size tracks heap page ranges, `brinbulkdelete()` does nothing ([brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)), rows appended past the last summarized range are not summarized because `brininsert()` returns without acting when the revmap has no entry for the range ([brin.c#brininsert-unsummarized](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L134-L142), [brin.c#brininsert-break](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L212-L218)), and a scan must return **all** pages of an unsummarized range whatever the scan keys ([brin.c#bringetbitmap](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L355-L366)). The correct action for a BRIN index is `VACUUM` or `brin_summarize_new_values()`, since `brinvacuumcleanup()` exists precisely to summarize unsummarized ranges ([brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L786-L800), [brin.c#brin_summarize_new_values](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L850-L860)).

### B-tree partial indexes: what the swap fixes

A partial index holds entries only for rows that satisfy its predicate; the table's live-row count counts every row ([pg_index.h#indpred](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L56-L57), [indices.sgml#indexes-partial](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L757-L772)). The two populations are filtered at three separate places, none of which touches the table's row count:

- The build scan counts every live heap tuple into the heap's `reltuples` *before* testing the predicate, then skips non-matching rows for the index callback ([heapam_handler.c#reltuples-count](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1395-L1400), [heapam_handler.c#partial-index-discard](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1607-L1615)).
- `ExecInsertIndexTuples()` evaluates `ii_Predicate` per index and skips the index update when it is false ([execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L334-L353)).
- `ANALYZE` re-tests the predicate over its sample and derives a per-index fraction for exactly this reason ([analyze.c#compute_index_stats-predicate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L771-L777)).

So the two inputs move independently. Any row inserted, deleted, or updated outside the predicate changes the table denominator and cannot change the numerator. Growth outside the predicate hides real bloat; deletion outside it manufactures fake bloat.

The measurement: one isolated 12.2 server, `autovacuum = off`, one table per workload with 1,000,000 rows, every tenth row carrying `st = 'open'` and `done_at IS NULL` (100,000 qualifying), heap 14,085 pages. Three B-tree indexes per table: `_full` on `(k)` (2,745 pages, 22,487,040 bytes), `_part` on `(k) WHERE st = 'open'` and `_null` on `(k) WHERE done_at IS NULL` (276 pages, 2,260,992 bytes each). `drift_t` divides by the table's `reltuples`, `drift_i` by the index's own, both refreshed with `VACUUM (ANALYZE)`; ground truth is the `REINDEX` size delta.

| Workload | What it does | `drift_t` | `drift_i` | reclaim |
|---|---|---|---|---|
| `p_noop` | nothing | 1.0000 | 0.9814 | 0.00% |
| `p_grow_out` | +3,000,000 rows outside the predicate (4x table) | **0.2500** | 1.0000 | 0.00% |
| `p_grow_in` | +900,000 rows inside the predicate (10x index) | **5.2346** | 0.9946 | 0.00% |
| `p_churn_in` | 5 rounds: delete 20,000 inside, insert 20,000 inside | 1.3986 | 1.3986 | 28.50% |
| `p_drain` | every qualifying row updated out of the predicate | **1.0000** | undefined (0 rows) | 99.64% |
| `p_queue` | 5 rounds: enqueue 100,000, dequeue by update | 1.9903 | undefined (0 rows) | 99.88% |
| `p_transit_in` (`_part`) | 100,000 rows updated into the `st` predicate | 2.9891 | 1.4946 | 33.21% |
| `p_transit_in` (`_null`) | the same statement, this predicate untouched | 1.0000 | 1.0000 | 0.00% |
| `p_del_in` | every qualifying row deleted | **1.1111** | undefined (0 rows) | 99.64% |
| `p_del_out` | 500,000 non-qualifying rows deleted | **2.0000** | 1.0000 | 0.00% |
| `p_hot_payload` | 2 full-table HOT payload updates, heap `fillfactor = 40` | 1.0000 | 1.0163 | 0.00% |
| `p_pred_col` | 1,800,000 updates of the predicate column, staying outside it | 1.0000 | 1.0000 | 0.00% |
| `q2_scatter` | half the qualifying rows deleted, scattered across the key range | **1.0526** | 2.0000 | 49.64% |

`_part` and `_null` agreed in every workload except `p_transit_in`, where only the `st` predicate gained rows. Bold cells are the table denominator's wrong answers. Scored at 1.30 with the page's harmful line of 25%:

| Denominator | TP | FP | FN | TN | Accuracy |
|---|---|---|---|---|---|
| table `reltuples` | 3 | 2 | 3 | 5 | 8/13 = 61.5% |
| index `reltuples` | 3 | 0 | 0 | 7 | 10/10 defined = 100% |

The three cells with no defined `drift_i` are the three indexes whose own `reltuples` reached `0`. That is not a gap: an empty index with a 2,260,992-byte fork is a certainty, not an estimate, and the detection query has an explicit rule for it. The index denominator's separating band is wide — highest benign 1.0163 (`p_hot_payload`), lowest harmful 1.3986 (`p_churn_in`) — so any trigger in [1.02, 1.39] is exact on this fixture. 1.30 is the right pick because it is already calibrated for full B-tree indexes and because the denominator carries its own noise, quantified below.

### Refresh discipline: what VACUUM (ANALYZE) does not update

`VACUUM (ANALYZE)` is the correct refresh, because its VACUUM half writes the access method's own count and its `ANALYZE` half deliberately skips indexes ([analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L612), [vacuumlazy.c#lazy_cleanup_index](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1798-L1815)). It is not a guarantee. When no heap tuple was removed, `btvacuumcleanup()` asks `_bt_vacuum_needs_cleanup()`, which compares the *heap's* tuple count against `btm_last_cleanup_num_heap_tuples` scaled by `vacuum_cleanup_index_scale_factor`, and returns NULL if the answer is no ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L927), [nbtree.c#_bt_vacuum_needs_cleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L787-L846)). A NULL result means `lazy_cleanup_index()` writes nothing, so whatever a previous plain `ANALYZE` estimated survives.

Measured with `VACUUM (VERBOSE, ANALYZE)` on a 1,000,000-row table whose partial index holds exactly 100,000 rows:

| Step | Heap rows | True index rows | What VACUUM printed | Index `reltuples` |
|---|---|---|---|---|
| after build | 1,000,000 | 100,000 | — | 100,000 (exact) |
| first `VACUUM (ANALYZE)` | 1,000,000 | 100,000 | `index "vs_part" now contains 100000 row versions in 276 pages` | 100,000 |
| plain `ANALYZE` | 1,000,000 | 100,000 | — | 99,734 (-0.27%) |
| second `VACUUM (ANALYZE)`, no DML | 1,000,000 | 100,000 | **no index line at all** | 99,734, unchanged |
| +50,000 qualifying rows, plain `ANALYZE` | 1,050,000 | 150,000 | — | 150,710 (+0.47%) |
| `VACUUM (ANALYZE)` after 5% heap growth | 1,050,000 | 150,000 | **no index line at all** | 150,710, unchanged |
| +150,000 qualifying rows, `VACUUM (ANALYZE)` | 1,200,000 | 300,000 | `now contains 300000 row versions in 825 pages` | 300,000 (exact) |
| reloption set to 0, +10,000 rows, `VACUUM (ANALYZE)` | 1,210,000 | 310,000 | `now contains 310000 row versions in 852 pages` | 310,000 (exact) |

The partial index's own population had grown 50% when the second-to-last `VACUUM` declined to look at it, because the heap had grown 5%. A fresh index metapage starts at `btm_last_cleanup_num_heap_tuples = -1.0`, which is why the first `VACUUM` always scans ([nbtpage.c#_bt_initmetapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L46-L65)). To force the exact refresh, set the B-tree reloption `vacuum_cleanup_index_scale_factor = 0` on the index, which takes `ShareUpdateExclusiveLock` ([reloptions.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L423-L431)); the GUC of the same name is `PGC_USERSET`, so a session or transaction setting is enough for a maintenance script ([guc.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3423-L3431)).

Hash has the same hole for a different reason: `hashvacuumcleanup()` returns NULL whenever `hashbulkdelete()` did not run, which is every VACUUM that finds no dead tuples ([hash.c#hashvacuumcleanup](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L640-L656)). GiST, SP-GiST, bloom, GIN and BRIN all scan when no bulk delete ran, so they always refresh.

Two consequences for the metric. A denominator you believe is exact may be a sample. And the exactness of `VACUUM (ANALYZE)` is a property of the access method plus the workload, not of the command.

### The noise floor, by predicate selectivity

`tupleFract` is measured on the same sample used for column statistics: 300 rows per unit of `attstattarget`, so 30,000 rows at the default `default_statistics_target = 100` ([analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700), [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994); `PGC_USERSET`, so session or transaction scope, as is the per-column `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS`). A selective predicate leaves few sampled rows to divide by, and the error grows accordingly. Six consecutive plain `ANALYZE` runs on one 1,000,000-row table with four partial indexes, true counts confirmed by `VACUUM (ANALYZE)`:

| Predicate selects | True index rows | Plain `ANALYZE` results | Error band |
|---|---|---|---|
| 10% | 100,000 | 98,834 / 104,034 / 100,234 / 102,500 / 98,000 / 99,200 | -2.00% to +4.03% |
| 1% | 10,000 | 9,900 / 10,200 / 10,434 / 10,167 / 9,434 / 10,634 | -5.66% to +6.34% |
| 0.1% | 1,000 | 1,034 / 767 / 900 / 1,000 / 667 / 767 | -33.30% to +3.40% |
| 0.001% | 10 | 0 / 0 / 67 / 34 / 0 / 0 | -100% or +570% |

The table's own `reltuples` was exactly 1,000,000 in all six runs, because `ANALYZE` samples up to `targrows` *blocks* and takes all of them when the relation has fewer, which 12,346 heap pages is ([sampling.c#BlockSampler_Init](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L51)). Three rules follow. Do not trust a plain-`ANALYZE` denominator when the predicate selects less than about 1% of the table. Treat `reltuples = 0` on a partial index as "unknown" unless the fork is large, since a 10-row predicate produced `0` on four of six runs while `VACUUM` wrote the exact `10`. And expect roughly ±2% of drift noise even at 10%, which is why a trigger of 1.02 is not usable in practice even though this fixture's band allows it.

### Two bloat shapes, and only one repays a rebuild

`REINDEX` byte reclaim is not a proxy for query gain, because two structurally different states both produce large reclaim. Both were measured with the visibility map fully set so that every index-only scan reported no heap fetches, best of five runs, `enable_seqscan`, `enable_bitmapscan` and parallelism off. The probes name the predicate column and still planned as index-only scans, because `check_index_predicates()` proves the predicate implies that qual and drops it from `indrestrictinfo` ([indxpath.c#check_index_predicates](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L3505-L3533)).

**Shape A: deleted-page bloat.** Churn that empties whole leaf pages. `_bt_pagedel()` unlinks the emptied page from the tree, and `btvacuumpage()` records it in the index free space map once its `btpo.xact` is old enough ([nbtpage.c#_bt_pagedel](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1284-L1305), [nbtree.c#btvacuumpage-deleted](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1167-L1183)). The fork keeps the pages, scans never visit them, and leaf density does not move:

| State | Bytes | Index pages | Leaf pages | Deleted pages | `avg_leaf_density` | Full index-only scan |
|---|---|---|---|---|---|---|
| after 5 churn rounds inside the predicate | 3,162,112 | 386 | 274 | 110 | 89.83 | 276 blocks / 9.094 ms |
| after `REINDEX` (28.50% reclaim) | 2,260,992 | 276 | 274 | 0 | 89.83 | 276 blocks / 9.238 ms |

**Shape B: low-density bloat.** Scattered deletion inside the predicate with no reinsertion. No page empties, so nothing is deleted or reclaimed; the surviving entries just sit thinly:

| State | Bytes | Index pages | Leaf pages | Deleted pages | `avg_leaf_density` | Full index-only scan |
|---|---|---|---|---|---|---|
| half the qualifying rows deleted, scattered | 2,260,992 | 276 | 274 | 0 | 45.06 | 276 blocks / 4.522 ms |
| after `REINDEX` (49.64% reclaim) | 1,138,688 | 139 | 137 | 0 | 89.83 | 139 blocks / 4.969 ms |

Shape A gave back 28.50% of its bytes and changed nothing a query could see. Shape B halved the blocks an index-only scan reads, 276 to 139 on the full scan and 140 to 71 on a half-range scan, and still bought no wall-clock time on a fully cached server: 4.522 ms against 4.969 ms on the full scan and 2.764 ms against 2.426 ms on the range scan, both best of five. A point lookup read 2 blocks in both states, and the heap-fetching variant moved 6,312 blocks to 6,243 with times of 11.549 ms and 11.565 ms.

The operational consequence: drift cannot tell shape A from shape B, but `pgstatindex()` can, in one call. Low `avg_leaf_density` means a rebuild will cut scan blocks. High density with many `deleted_pages` means the rebuild returns disk space that the index itself would have reused — plus the planner effect below.

### Bloat the planner charges for even when no page is read

`get_relation_info()` sets `IndexOptInfo.pages` from the current physical size, and for a partial index it also *estimates* `tuples` with `estimate_rel_size()` instead of pinning it to the table's row count ([plancat.c#partial-index-size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L990-L1026)). `genericcostestimate()` turns those two numbers into pages to be read and charges `random_page_cost` for them ([selfuncs.c#genericcostestimate-pages](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5755-L5780), [selfuncs.c#btcostestimate-descent](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6097-L6102)), with `random_page_cost` at its default 4.0 ([guc.c#random_page_cost](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3218-L3227), [cost.h#DEFAULT_RANDOM_PAGE_COST](../../../../raw/postgres-12/src/include/optimizer/cost.h#L25)).

Estimated total cost for the identical query on identical data:

| Fixture | Index pages | Estimated cost | Delta | Runtime blocks |
|---|---|---|---|---|
| shape A, bloated | 386 | 3300.55 | | 276 |
| shape A, rebuilt | 276 | 2887.22 | -413.33 | 276 |
| shape B, bloated | 276 | 1988.82 | | 276 |
| shape B, rebuilt | 139 | 1397.72 | -591.10 | 139 |

Shape A is the one that matters: the planner priced 110 pages that were unlinked from the tree and that the scan provably never touched, and the rebuilt index was 12.5% cheaper to plan while being no faster to run. A bloated partial index can therefore lose a plan to a sequential scan on the strength of pages that cost nothing to skip. This is the strongest argument for acting on shape A bloat despite its zero runtime gain.

### Collateral damage: a partial index's predicate column blocks HOT for the whole table

The most expensive partial-index effect measured here does not show up in the partial index at all. `RelationGetIndexAttrBitmap()` folds every attribute named in an index predicate into the all-index bitmap ([relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4782-L4787), [relcache.c#pull_varattnos-predicate](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4956-L4960)), and `heap_update()` uses that bitmap as its HOT gate: a HOT update is possible only when no member changed value ([heapam.c#hot_attrs](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2957-L2966), [heapam.c#use_hot_update](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600)). A predicate column is HOT-blocking for every row in the table, including rows the predicate never selects.

Two identical 1,000,000-row tables, heap `fillfactor = 40`, identical statements (`UPDATE ... SET st = 'closed' WHERE st = 'done'`, then back), touching only rows that never satisfy either predicate:

| Table | Partial indexes present | Updates | `n_tup_hot_upd` | Full index bytes | Full index drift | Full index reclaim |
|---|---|---|---|---|---|---|
| `p_pred_col` | 2 | 1,800,000 | **0** | 22,487,040 -> 89,939,968 | 3.9996 | 75.00% |
| `p_pred_ctl` | 0 | 1,800,000 | **1,800,000** | 22,487,040 -> 22,487,040 | 1.0000 | 0.00% |

The two partial indexes themselves reported drift `1.0000` and 0.00% reclaim, so the index that quadrupled its neighbour is the one index the metric calls healthy. HOT counts come from `pgstat_count_heap_update()` ([heapam.c#pgstat_count_heap_update](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3746), [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)). The control for the control: 2,000,000 payload updates on a column in no index and no predicate produced 2,000,000 HOT updates and left every index byte-identical.

Practical reading: when a partial index's drift is flat but a sibling index on the same table is drifting, check whether the partial index's predicate column is being written. `RelationGetIndexPredicate()` and `pg_get_expr(indpred, indrelid)` give the predicate for that check ([relcache.c#RelationGetIndexPredicate](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4712-L4779)).

### Reading reltuples without losing digits

`pg_class.reltuples` is `float4` ([pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)), and the obvious cast in monitoring SQL silently rounds it. `float4_numeric()` formats the value with `%.*g` at `FLT_DIG`, which is 6 significant digits ([numeric.c#float4_numeric](../../../../raw/postgres-12/src/backend/utils/adt/numeric.c#L3444-L3472)). Measured on a GIN index whose build counted 20,000,020 entries:

| Expression | Result |
|---|---|
| `reltuples::float8` | 20000020 |
| `reltuples::numeric` | 20000000 |
| `reltuples::text` | 2.000002e+07 |

Two separate limits, then. Cast through `float8` before `numeric` to keep every digit the catalog holds, as the shipped SQL below does. And remember that the storage type itself rounds above 2^24: `16777219::float4::float8` is `16777220`. For a bloat ratio the second limit is harmless — 7 significant digits of a row count cannot move a 4-decimal drift — but the first one moves the last digits of the stored baseline, which is enough to make a recapture look like drift.

### Tested capture and detection SQL

Both statements below were executed against the pinned build. Timeouts are session-scoped: `statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so they need neither reload nor restart ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)). The refresh is `VACUUM (ANALYZE)`, which cannot run inside a transaction block ([vacuum.c#PreventInTransactionBlock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L235-L249)).

```sql
SET /* wiki_index_bpr_timeout */ statement_timeout = '15min';
SET /* wiki_index_bpr_lock_timeout */ lock_timeout = '5s';

VACUUM (ANALYZE) /* wiki_index_bpr_refresh */ app.orders;
```

Capture. This reads only; review the generated `comment_sql` values before running them. It refuses GIN and BRIN by access method, refuses an index with no usable row estimate, and refuses to overwrite a comment it did not write.

```sql
WITH /* wiki_capture_index_bpr */ measured AS MATERIALIZED
(
    SELECT ins.nspname                         AS index_schema,
           ic.relname                          AS index_name,
           am.amname                           AS amname,
           i.indrelid                          AS table_oid,
           (i.indpred IS NOT NULL)             AS is_partial,
           pg_relation_size(ic.oid)            AS index_bytes,
           ic.reltuples::float8::numeric       AS index_rows,
           tc.reltuples::float8::numeric       AS table_rows,
           obj_description(ic.oid, 'pg_class') AS existing_comment
    FROM pg_index AS i
    JOIN pg_class AS ic ON ic.oid = i.indexrelid
    JOIN pg_namespace AS ins ON ins.oid = ic.relnamespace
    JOIN pg_am AS am ON am.oid = ic.relam
    JOIN pg_class AS tc ON tc.oid = i.indrelid
    WHERE ic.relkind = 'i'
      AND i.indislive AND i.indisready AND i.indisvalid
      AND ins.nspname IN ('app')
),
judged AS
(
    SELECT m.*,
           CASE
               WHEN m.amname IN ('gin', 'brin')
                   THEN 'unsupported access method: its reltuples is not one row per indexed row'
               WHEN m.index_rows <= 0
                   THEN 'no usable index row estimate'
               WHEN m.existing_comment IS NOT NULL
                    AND m.existing_comment NOT LIKE 'wiki_bpr_v3;%'
                   THEN 'foreign comment; refusing to overwrite'
           END AS skip_reason
    FROM measured AS m
)
SELECT /* wiki_capture_index_bpr */
       index_schema, index_name, amname, is_partial, index_bytes,
       trunc(index_rows)::bigint AS index_rows,
       trunc(table_rows)::bigint AS table_rows,
       round(index_bytes / nullif(index_rows, 0), 6) AS bpr,
       skip_reason,
       CASE
           WHEN skip_reason IS NOT NULL THEN NULL
           ELSE format(
                    'COMMENT /* wiki_capture_index_bpr */ ON INDEX %I.%I IS %L;',
                    index_schema, index_name,
                    format('wiki_bpr_v3;relid=%s;am=%s;pred=%s;bytes=%s;rows=%s;bpr=%s',
                           table_oid, amname, is_partial, index_bytes,
                           trunc(index_rows)::bigint,
                           round(index_bytes / index_rows, 6)))
       END AS comment_sql
FROM judged
ORDER BY index_schema, index_name;
```

Detection. The stored `am=` and `pred=` fields must still match the index, so a comment that survived a definition change is rejected rather than trusted. `nullif(index_rows, 0)` is not optional: without it an empty partition's child index raises `ERROR: division by zero` before the guard in the `CASE` can run.

```sql
WITH /* wiki_detect_index_bpr */ settings (size_floor_bytes) AS
(
    VALUES (8 * 1024 * 1024)
),
threshold (amname, drift_trigger, policy) AS
(
    VALUES ('btree',  1.30::numeric, 'rebuild candidate; confirm with pgstatindex'),
           ('hash',   1.30, 'rebuild candidate'),
           ('bloom',  1.30, 'rebuild candidate'),
           ('gist',   1.30, 'rebuild candidate; in-domain churn hides reclaimable space'),
           ('spgist', 1.30, 'rebuild candidate; in-domain churn hides reclaimable space'),
           ('gin',    NULL, 'do not use this signal; reltuples counts entries after a build and rows after VACUUM'),
           ('brin',   NULL, 'do not use this signal; reltuples counts summarized ranges')
),
measured AS MATERIALIZED
(
    SELECT ins.nspname                         AS index_schema,
           ic.relname                          AS index_name,
           am.amname                           AS amname,
           i.indrelid                          AS table_oid,
           (i.indpred IS NOT NULL)             AS is_partial,
           pg_get_expr(i.indpred, i.indrelid)  AS predicate,
           pg_relation_size(ic.oid)            AS index_bytes,
           ic.reltuples::float8::numeric       AS index_rows,
           tc.reltuples::float8::numeric       AS table_rows,
           obj_description(ic.oid, 'pg_class') AS stored_comment
    FROM pg_index AS i
    JOIN pg_class AS ic ON ic.oid = i.indexrelid
    JOIN pg_namespace AS ins ON ins.oid = ic.relnamespace
    JOIN pg_am AS am ON am.oid = ic.relam
    JOIN pg_class AS tc ON tc.oid = i.indrelid
    WHERE ic.relkind = 'i'
      AND i.indislive AND i.indisready AND i.indisvalid
      AND ins.nspname IN ('app')
),
parsed AS
(
    SELECT m.*,
           CASE
               WHEN m.stored_comment ~ ('^wiki_bpr_v3;relid=[0-9]+;am=[a-z]+;pred=[tf];'
                                        || 'bytes=[0-9]+;rows=[0-9]+;bpr=[0-9]+[.][0-9]+$')
                    AND split_part(split_part(m.stored_comment, 'relid=', 2), ';', 1)::oid
                        = m.table_oid
                    AND split_part(split_part(m.stored_comment, 'am=', 2), ';', 1)
                        = m.amname
                    AND split_part(split_part(m.stored_comment, 'pred=', 2), ';', 1)
                        = CASE WHEN m.is_partial THEN 't' ELSE 'f' END
               THEN split_part(m.stored_comment, ';bpr=', 2)::numeric
           END AS base_bpr
    FROM measured AS m
)
SELECT /* wiki_detect_index_bpr */
       p.index_schema, p.index_name, p.amname, p.is_partial,
       pg_size_pretty(p.index_bytes)  AS size,
       trunc(p.index_rows)::bigint    AS index_rows,
       trunc(p.table_rows)::bigint    AS table_rows,
       p.base_bpr,
       round(p.index_bytes / nullif(p.index_rows, 0), 6) AS cur_bpr,
       round((p.index_bytes / nullif(p.index_rows, 0)) / p.base_bpr, 4) AS drift,
       round(((p.index_bytes / nullif(p.index_rows, 0)) / p.base_bpr - 1) * 100, 1)
           AS pct_increase,
       t.drift_trigger,
       CASE
           WHEN t.drift_trigger IS NULL          THEN t.policy
           WHEN p.index_rows <= 0
                AND p.index_bytes >= s.size_floor_bytes
               THEN 'empty index; a rebuild reclaims every page'
           WHEN p.index_rows <= 0                THEN 'no usable index row estimate'
           WHEN p.base_bpr IS NULL               THEN 'no usable baseline'
           WHEN p.index_bytes < s.size_floor_bytes THEN 'below size floor'
           WHEN (p.index_bytes / nullif(p.index_rows, 0)) / p.base_bpr
                >= t.drift_trigger               THEN t.policy
           ELSE 'below threshold'
       END AS decision
FROM parsed AS p
CROSS JOIN settings AS s
LEFT JOIN threshold AS t ON t.amname = p.amname
ORDER BY p.index_schema, p.index_name;
```

### The runbook, executed on the pin

Five steps, run end to end on a 1,000,000-row `app.orders` carrying a full B-tree, a partial B-tree on `WHERE st = 'open'`, a GIN index, a BRIN index, an `indisvalid = false` index and a partitioned parent index. `size_floor_bytes` was lowered to `1024 * 1024` for the run so the 2.2 MB fixture index was not suppressed.

1. `VACUUM (ANALYZE) app.orders` — refresh the denominator.
2. Run the capture query, review the generated statements, then execute them.
3. Run the detection query to confirm a quiet baseline.
4. Let the workload run: delete half the qualifying rows scattered across the key range, and append 1,000,000 rows that the predicate does not select. Refresh again.
5. Act on the decision, then recapture.

| Step | `orders_k` (full) | `orders_k_open` (partial) | `orders_tags_gin` | `orders_k_brin` | `part_1_k_idx` |
|---|---|---|---|---|---|
| capture after build | bpr 22.487040, 1,000,000 rows | bpr 22.609920, 100,000 rows | skipped, unsupported AM | skipped, unsupported AM | skipped, no usable index row estimate |
| detect after capture | 1.0000, below threshold | 1.0000, below threshold | refused by AM | refused by AM | no usable index row estimate |
| detect after the workload | 1.0249, below threshold | **2.0000, rebuild candidate** | refused | refused | unchanged |
| the same cells under the table denominator | 23.047087 against 22.487040 | **1.159496 against 22.609920, drift 0.0513** | — | — | — |
| detect after `REINDEX INDEX CONCURRENTLY` | 1.0249 | 1.0072, below threshold | refused | refused | unchanged |
| detect after recapture | 1.0000 | 1.0000 | refused | refused | unchanged |

The table-denominator row is the point of the exercise: it would have reported a 95% *decrease* for an index that had just lost half its rows and held 49.64% reclaimable space. The BRIN index illustrates its own section from the same run: `cur_bpr` read 183.402985 at capture, 91.360595 after the workload's `VACUUM (ANALYZE)`, and 0.012603 after a later plain `ANALYZE`, with the fork fixed at 24,576 bytes throughout.

The empty-index rule was exercised separately: a partial index drained by updating every qualifying row out of its predicate reported `reltuples = 0` with a 2,260,992-byte fork, and the detection query returned `empty index; a rebuild reclaims every page`. `pgstatindex` confirmed 1 leaf page, 273 deleted pages and `avg_leaf_density` 0.05.

`REINDEX INDEX CONCURRENTLY` behaved correctly for both halves of the metric: the replacement index carried the exact build count of 50,000 rows, because `index_concurrently_build()` runs the ordinary `index_build()` path ([index.c#index_concurrently_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L1390-L1439)) and `index_concurrently_swap()` never touches `relpages` or `reltuples` ([index.c#index_concurrently_swap](../../../../raw/postgres-12/src/backend/catalog/index.c#L1440-L1505)); and the stored comment moved to the new relation ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)). Comment durability was re-checked across five more operations in the same run — `REINDEX INDEX`, `ALTER INDEX ... RENAME`, `ALTER INDEX ... SET (fillfactor = 80)`, `VACUUM` and `ANALYZE` — and all preserved the baseline. That matches the code path: `COMMENT` resolves the object under `ShareUpdateExclusiveLock`, checks ownership, then writes one `pg_description` row keyed by `(objoid, classoid, objsubid)` ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225), [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57)), and a plain `REINDEX` keeps the index's OID and only swaps its relfilenode ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531)).

### Operating rules, failure modes, and cheaper fixes

- **Divide by the index's own `reltuples`.** For ordinary indexes it is the same number the table denominator gave; for partial indexes it is the only correct one.
- **Refresh with `VACUUM (ANALYZE)`, and know when it lies.** B-tree and hash can skip the index entirely, leaving a sampled value in place. For indexes you screen, set the B-tree reloption `vacuum_cleanup_index_scale_factor = 0`, or accept a few percent of denominator noise.
- **Never screen GIN or BRIN with this metric.** Their `reltuples` counts entries and ranges. Screen GIN by its pending list and BRIN by its unsummarized ranges instead.
- **Require the predicate to select at least about 1% of the table** before trusting a partial index's plain-`ANALYZE` denominator.
- **Treat `reltuples = 0` as two different findings.** With a large fork it is an empty index whose pages are all reclaimable. With a small fork it is an unknown count.
- **Confirm with `pgstatindex()` before rebuilding.** Low `avg_leaf_density` predicts fewer scan blocks. High density with many `deleted_pages` predicts disk space and a lower planner cost, not a faster scan. contrib `pgstattuple` provides it, and at the extension's default version 1.5 every reader function is revoked from `PUBLIC` and granted to `pg_stat_scan_tables` ([pgstattuple.control:3](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L3), [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31), [pgstattuple--1.4--1.5.sql#privileges](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)).
- **Expect the in-domain-churn blind spot.** When deletes and inserts share a key domain, the fork stops growing and drift stays at 1.00 while a rebuild still reclaims a quarter to half the index. Nothing in this metric can see that; a periodic `pgstatindex` sweep can.
- **Before rebuilding, try the cheaper action.** For GIN with `fastupdate = on`, run `gin_clean_pending_list()`; it requires index ownership and rejects non-GIN relations, other sessions' temporary indexes, and recovery ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074)). For BRIN, run `VACUUM` or `brin_summarize_new_values()`. For B-tree, a page deleted by one VACUUM only becomes recyclable once its `btpo.xact` precedes `RecentGlobalXmin`, so a second VACUUM can return space the first could not ([nbtree.c#btvacuumpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1149-L1183)).
- **One comment per object.** `CreateComments()` maintains a single row per `(objoid, classoid, objsubid)`, so the baseline and human prose cannot coexist. If your indexes already carry comments, use a side table.
- **Recapture policy.** Recapture only after a successful, measured rebuild or an approved definition change. Never recapture to silence unexplained drift.
- **Rebuild cost and locking.** Plain `REINDEX` locks out writes on the table for the duration; `REINDEX INDEX CONCURRENTLY` trades that for a session-level `SHARE UPDATE EXCLUSIVE` lock plus extra passes, and a failure leaves an invalid `_ccnew`, or `_ccold` when the old definition could not be dropped ([reindex.sgml#locking](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L155-L165), [reindex.sgml#concurrently](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L300-L315), [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)).
- **Scope.** Run per database with an explicit schema allowlist; never comment on or rebuild system catalogs from this loop.

### Caller, callee, data structures, and build boundaries

- **Numerator path.** `pg_relation_size(regclass)` (SQL) -> `pg_relation_size(regclass, text)` -> `calculate_relation_size()` -> `smgr` path names and `stat()` per segment ([dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308)).
- **Denominator path, build.** `DefineIndex()`/`ReindexIndex()` -> `index_build()` -> the access method's `ambuild` -> `IndexBuildResult` -> `index_update_stats()` -> `heap_inplace_update()` on the index's `pg_class` row ([index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987), [index.c#index_update_stats-reltuples](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2786)).
- **Denominator path, VACUUM.** `ExecVacuum()` -> `lazy_scan_heap()` -> `lazy_cleanup_index()` -> `index_vacuum_cleanup()` -> the access method's `amvacuumcleanup` -> `IndexBulkDeleteResult` -> `vac_update_relstats()`, gated on `estimated_count` ([vacuumlazy.c#lazy_cleanup_index](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1798-L1815), [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1196)).
- **Denominator path, ANALYZE.** `analyze_rel()` -> `do_analyze_rel()` -> `compute_index_stats()` for partial and expression indexes -> `vac_update_relstats()` once per index, skipped entirely when running inside VACUUM ([analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629), [analyze.c#compute_index_stats-predicate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L771-L777)).
- **Key structures.** `Form_pg_class` (`relpages`, `reltuples` as `float4`, `relam`, `relkind`), `Form_pg_index` (`indislive`, `indisready`, `indisvalid`, `indpred`), `IndexBuildResult` (`heap_tuples`, `index_tuples`), `IndexBulkDeleteResult` (`num_index_tuples`, `estimated_count`, `pages_deleted`, `pages_free`), `IndexVacuumInfo` (`num_heap_tuples`, `estimated_count`), `IndexAmRoutine`, `GinStatsData`, `BrinBuildState.bs_numtuples`, `HashMetaPage.hashm_ntuples`.
- **Generated artifacts.** The proposal adds no catalog or header: it reads `pg_class`, `pg_index`, `pg_am`, and `pg_description`, whose `_d.h` headers and BKI data are produced by `genbki.pl` during the build ([catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90)).
- **Extension boundary.** contrib `bloom` supplies an ordinary `IndexAmRoutine` from outside core ([blutils.c#blhandler](../../../../raw/postgres-12/contrib/bloom/blutils.c#L100-L148)), and its build and cleanup both count one tuple per indexed row, which is why it behaves like a core row-counting method here. A third-party access method has no obligation to do the same, so the eligibility list in the detection query is an allowlist, not a denylist.

### Test coverage in the pinned tree

There is no upstream test for a bloat proxy of any kind; the relevant coverage is of the pieces this proposal leans on.

- Index comments surviving both rebuild forms are tested directly ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)).
- The GIN pending list is exercised with `fastupdate = on`, an explicit `gin_pending_list_limit` reloption, `gin_clean_pending_list()`, VACUUM-driven flushing, and a later switch to `fastupdate = off` ([gin.sql#pending-list](../../../../raw/postgres-12/src/test/regress/sql/gin.sql#L7-L35)).
- BRIN summarization and desummarization have dedicated coverage ([brin.sql#summarize](../../../../raw/postgres-12/src/test/regress/sql/brin.sql#L404-L416)).
- Partial B-tree indexes have build coverage on three predicate shapes and planner coverage for predicate implication, quals dropped from the plan, the update-target recheck exception, the not-applicable case, and the bitmap-scan recheck ([create_index.sql#btree-partial](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L62-L72), [select.sql#partial-index-planning](../../../../raw/postgres-12/src/test/regress/sql/select.sql#L190-L224)). Concurrent builds of partial indexes are covered too ([create_index.sql#concurrent-partial](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L479-L481)).
- The one `reltuples` assertion in the tree is about a *table*: the `vacuum-reltuples` isolation spec checks that VACUUM leaves the previous value in place when page pins force it to skip pages, "since that interferes badly with planning" ([vacuum-reltuples.spec](../../../../raw/postgres-12/src/test/isolation/specs/vacuum-reltuples.spec#L1-L48)).
- Explicit absence: nothing in the tree measures `pg_relation_size` against `reltuples`, calibrates a rebuild threshold, or asserts an access method's reclaimable fraction. No test asserts any *index's* own `pg_class.reltuples`, from any writer, so the unit differences between `ginbuild()`, `ginvacuumcleanup()` and `ANALYZE` are unguarded; the only other `reltuples` writes in the regression suite are hand-forged `UPDATE pg_class` statements in a join test ([join_hash.sql#forged-reltuples](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L68-L84)). Every threshold on this page comes from the fixtures described above, not from upstream tests.

## Context Reviewed

- Pinned checkout `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`), built and installed out of tree under `.wiki-runtime/tmp/`, with an isolated cluster on port 55432, `autovacuum = off`, and contrib `bloom`, `pgstattuple` and `pageinspect` installed.
- The three writers of an index's `pg_class.reltuples`: `index_update_stats()` from `index_build()`, `do_analyze_rel()`, and `lazy_cleanup_index()`, including the `estimated_count` gate and the `ANALYZE`-inside-VACUUM skip.
- Per-access-method `ambuild` results and `amvacuumcleanup` counts for B-tree, hash, GiST, SP-GiST, GIN, BRIN and contrib `bloom`, including GIN's entry counting and heap-count substitution, BRIN's range counting and `include_partial = false`, and the B-tree and hash cases that return NULL and skip the update.
- `_bt_vacuum_needs_cleanup()`, `_bt_initmetapage()`, and the `vacuum_cleanup_index_scale_factor` GUC and B-tree reloption.
- Relation-size functions; `float4` storage of `reltuples` and the `float4_numeric()` 6-digit cast.
- Per-access-method space behavior: free-space-map recording, the disabled SP-GiST truncation, GIN pending-list insertion and cleanup, BRIN summarization, hash splitpoint allocation, `gistchoose()`'s random tie-break.
- `IndexAmRoutine` callback surface and the absence of any reclaimable-space callback.
- Partial-index filtering at build, at insert, and in the `ANALYZE` sample; `get_relation_info()`'s partial-index size estimate; `genericcostestimate()`'s page charge; `check_index_predicates()`.
- The HOT attribute bitmap, `heap_update()`'s HOT gate, and `pgstat_count_heap_update()`.
- `COMMENT ON INDEX` locking, ownership, and `pg_description` storage; `index_concurrently_build()` and `index_concurrently_swap()`.
- Measurements: a 96-cell matrix (12 workloads x 8 indexes) with seven capture phases each; a 13-cell partial-B-tree matrix with `pgstatindex` at three phases; GIN keys-per-row and pending-list fixtures with probe timings and `pageinspect` metapage reads; a BRIN writer-divergence fixture; a `VACUUM (VERBOSE, ANALYZE)` index-skip sequence; a four-selectivity `ANALYZE` noise sweep; two bloat-shape fixtures with planner costs and four probe shapes each; a HOT collateral-damage pair; and the full capture/detect/rebuild/recapture runbook including the empty-index rule and six comment-durability operations.
- Regression coverage for comments, GIN pending lists, BRIN summarization, partial-index builds and planning; contrib `pgstattuple` control file and 1.4 SQL script.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| The numerator is main-fork allocated bytes, measured under `AccessShareLock` | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308) |
| `reltuples` is an approximate `float4` count, and an index has its own | [pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63) |
| A build writes the access method's own `index_tuples` into the index's `reltuples` | [index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987), [index.c#index_update_stats-reltuples](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2786) |
| Plain `ANALYZE` writes `ceil(tupleFract * totalrows)` and skips indexes inside VACUUM | [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629) |
| A non-partial index keeps `tupleFract = 1.0`, so the swap is a no-op for it | [analyze.c#skip-plain-index](../../../../raw/postgres-12/src/backend/commands/analyze.c#L730-L732), [analyze.c#tupleFract-init](../../../../raw/postgres-12/src/backend/commands/analyze.c#L437-L438) |
| VACUUM writes the index count only when the access method reports it as exact | [vacuumlazy.c#lazy_cleanup_index](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1798-L1815), [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-12/src/include/access/genam.h#L55-L81) |
| GIN counts extracted entries at build and substitutes the heap count at VACUUM | [gininsert.c#indtuples](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L260-L272), [ginvacuum.c#ginvacuumcleanup-heap-count](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L723-L731) |
| BRIN counts range summary tuples at build and at cleanup, excluding the trailing partial range | [brin.c#brinbuild-idxtuples](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L726-L742), [brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L790-L815), [brin.c#brinsummarize-counts](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1299-L1312) |
| B-tree and hash can skip the cleanup scan and leave the previous value in place | [nbtree.c#btvacuumcleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L927), [nbtree.c#_bt_vacuum_needs_cleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L787-L846), [hash.c#hashvacuumcleanup](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L640-L656) |
| B-tree, GiST and SP-GiST clamp their count to the heap's | [nbtree.c#btvacuumcleanup-clamp](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L929-L941), [gistvacuum.c#gistvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L100-L145), [spgvacuum.c#spgvacuumcleanup](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L931-L969) |
| No index access method shrinks its own fork; pages go to the FSM instead | [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L56), [nbtree.c#recyclable-page](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1183), [spgvacuum.c#disabled-truncation](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886) |
| No generic callback reports reclaimable index space | [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L233) |
| GIN defaults to `fastupdate = on`, so every scan reads the pending list first | [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-12/src/include/access/gin_private.h#L23-L39), [ginget.c#gingetbitmap-order](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1866-L1884) |
| Cleaning the pending list frees pages to the FSM without shrinking the fork | [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667), [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074) |
| BRIN ignores deletes, leaves out-of-range inserts unsummarized, and returns every page of an unsummarized range | [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784), [brin.c#brininsert-break](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L212-L218), [brin.c#bringetbitmap](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L355-L366) |
| A GiST baseline is not byte-reproducible | [gistutil.c#gistchoose-random](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L502-L537) |
| Hash allocates bucket pages a splitpoint at a time and counts the hole | [hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L108-L160), [hashpage.c#splitpoint-batch](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L779-L800) |
| A partial index's contents are filtered at build, at insert, and in the `ANALYZE` sample | [heapam_handler.c#partial-index-discard](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1607-L1615), [execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L334-L353), [analyze.c#compute_index_stats-predicate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L771-L777) |
| An emptied B-tree leaf is unlinked and recorded free, so it is invisible to scans but not to `pg_relation_size` | [nbtpage.c#_bt_pagedel](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1284-L1305), [nbtree.c#btvacuumpage-deleted](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1167-L1183) |
| The planner estimates a partial index's row count and charges `random_page_cost` per estimated page | [plancat.c#partial-index-size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [plancat.c#estimate_rel_size-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L990-L1026), [selfuncs.c#genericcostestimate-pages](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5755-L5780) |
| A partial index's predicate columns are HOT-blocking for every row of the table | [relcache.c#pull_varattnos-predicate](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4956-L4960), [heapam.c#hot_attrs](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2957-L2966), [heapam.c#use_hot_update](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600) |
| The `ANALYZE` sample is 300 rows per unit of `attstattarget`, which limits a selective predicate's fraction | [analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700), [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994) |
| `reltuples::numeric` rounds to six significant digits | [numeric.c#float4_numeric](../../../../raw/postgres-12/src/backend/utils/adt/numeric.c#L3444-L3472) |
| A concurrent rebuild leaves the exact build count and moves the comment | [index.c#index_concurrently_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L1390-L1439), [index.c#index_concurrently_swap](../../../../raw/postgres-12/src/backend/catalog/index.c#L1440-L1505), [index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656) |
| One index comment is one `pg_description` row, written under `ShareUpdateExclusiveLock` with an ownership check | [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225) |
| VACUUM cannot run in a transaction block; ANALYZE can | [vacuum.c#PreventInTransactionBlock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L235-L249) |
| The timeouts and the GIN and cleanup-scale settings in the runbook are session-scoped | [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397), [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [guc.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3423-L3431) |

## Open Questions

- The per-access-method bands rest on 12 observations each from a single fixture shape (200,000 rows, one key column per access method, `int[]` GIN keys, `point` GiST/SP-GiST keys). How wide are the bands for other operator classes, especially `tsvector` and `jsonb` GIN, `box`/`range` GiST, and text SP-GiST?
- The four false negatives are all `w_churn2`, in-domain churn. No cheap catalog-only signal separates "the fork stopped growing because space is being reused well" from "the fork stopped growing but half of it is reusable". Only `pgstatindex` distinguished them here, and it is a full index scan.
- GIN's entry-versus-row unit change is only visible when keys per row exceeds 1. Whether a partial GIN index compounds the two problems, and what `ginvacuumcleanup()`'s heap-count substitution then stores, was not measured.
- Ground truth is a single `REINDEX` size delta. A production rule needs a cost model that weighs reclaimed bytes and query gain against rebuild WAL, I/O, and lock time; none of those were measured.
- Every timing figure came from a fully cached 512 MB `shared_buffers` server. The block-count reductions are cache-independent, but the wall-clock gains on an I/O-bound instance are unmeasured and are likely larger. Two rebuilt indexes were marginally *slower* than their bloated originals in best-of-five timings, which this fixture cannot separate from noise.
- The planner-cost deltas track `IndexOptInfo.pages` but did not equal `pages x random_page_cost` exactly (-413.33 for 110 pages, -591.10 for 137). The residual was not attributed to `index_pages_fetched()`'s non-linear model versus the changed tuple estimate, and no plan actually flipped, so the selectivity at which a bloated partial index loses to a sequential scan was not located.
- Whether a partial index's `deleted_pages` reach a steady state under indefinite queue churn was not measured. The queue fixture stopped after five rounds, so how much of its 99.88% reclaim later rounds would have reused is unknown.
- The 8 MB size floor is arbitrary. It was lowered to 1 MB for the runbook so a 2.2 MB fixture index would not be suppressed; no principled floor was derived from a real index-size distribution.
- `vacuum_cleanup_index_scale_factor = 0` forces an exact B-tree denominator at the cost of a full index scan on every VACUUM. The break-even between that cost and the denominator noise it removes was not measured, and hash has no equivalent reloption.
- Third-party access methods are out of scope: they can register any `IndexAmRoutine` and are under no obligation to make `index_tuples` mean one row.
- Repeated-observation policy (how many consecutive maintenance windows must agree before a rebuild) was not tested; every measurement here is a single observation per cell.

## Source References

- [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)
- [dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467)
- [pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)
- [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192)
- [pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L40-L43)
- [pg_index.h#indpred](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L56-L57)
- [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57)
- [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L233)
- [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-12/src/include/access/genam.h#L55-L81)
- [cost.h#DEFAULT_RANDOM_PAGE_COST](../../../../raw/postgres-12/src/include/optimizer/cost.h#L25)
- [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130)
- [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)
- [index.c#index_concurrently_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L1390-L1439)
- [index.c#index_concurrently_swap](../../../../raw/postgres-12/src/backend/catalog/index.c#L1440-L1505)
- [index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)
- [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680)
- [index.c#index_update_stats-reltuples](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2786)
- [index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987)
- [index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531)
- [analyze.c#tupleFract-init](../../../../raw/postgres-12/src/backend/commands/analyze.c#L437-L438)
- [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)
- [analyze.c#skip-plain-index](../../../../raw/postgres-12/src/backend/commands/analyze.c#L730-L732)
- [analyze.c#compute_index_stats-predicate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L771-L777)
- [analyze.c#tupleFract-estimate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L817-L822)
- [analyze.c:1700](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1700)
- [sampling.c#BlockSampler_Init](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L51)
- [vacuum.c#PreventInTransactionBlock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L235-L249)
- [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1196)
- [vacuumlazy.c#lazy_cleanup_index](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1798-L1815)
- [heapam.c#hot_attrs](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2957-L2966)
- [heapam.c#use_hot_update](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600)
- [heapam.c#pgstat_count_heap_update](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3746)
- [heapam_handler.c#reltuples-count](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1395-L1400)
- [heapam_handler.c#partial-index-discard](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1607-L1615)
- [execIndexing.c#ExecInsertIndexTuples-predicate](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L334-L353)
- [relcache.c#RelationGetIndexPredicate](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4712-L4779)
- [relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4782-L4787)
- [relcache.c#pull_varattnos-predicate](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4956-L4960)
- [plancat.c#partial-index-size](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407)
- [plancat.c#estimate_rel_size-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L990-L1026)
- [indxpath.c#check_index_predicates](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L3505-L3533)
- [selfuncs.c#genericcostestimate-pages](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5755-L5780)
- [selfuncs.c#btcostestimate-descent](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6097-L6102)
- [numeric.c#float4_numeric](../../../../raw/postgres-12/src/backend/utils/adt/numeric.c#L3444-L3472)
- [reloptions.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L423-L431)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)
- [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L56)
- [heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3164-L3199)
- [nbtsort.c#btbuild-result](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L351-L355)
- [nbtree.c#_bt_vacuum_needs_cleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L787-L846)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L927)
- [nbtree.c#btvacuumcleanup-clamp](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L929-L941)
- [nbtree.c#btvacuumscan-counts](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L967-L973)
- [nbtree.c#IndexFreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1085-L1095)
- [nbtree.c#btvacuumpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1149-L1183)
- [nbtree.c#recyclable-page](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1183)
- [nbtree.c#btvacuumpage-deleted](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1167-L1183)
- [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L46-L65)
- [nbtpage.c#_bt_pagedel](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1284-L1305)
- [gistbuild.c#gistbuild-result](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L228-L236)
- [gistvacuum.c#gistvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L100-L145)
- [gistvacuum.c#delete-empty-pages](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L115-L129)
- [gistvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L303-L312)
- [gistutil.c#gistchoose-random](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L502-L537)
- [spginsert.c#spgbuild-result](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L145-L149)
- [spgvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L655-L665)
- [spgvacuum.c#disabled-truncation](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)
- [spgvacuum.c#spgvacuumcleanup](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L931-L969)
- [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-12/src/include/access/gin_private.h#L23-L39)
- [gininsert.c#indtuples](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L260-L272)
- [gininsert.c#ginbuild-result](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L419-L427)
- [ginfast.c#needCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L462)
- [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667)
- [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074)
- [ginget.c#scanPendingInsert](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1750-L1760)
- [ginget.c#gingetbitmap-order](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1866-L1884)
- [ginutil.c#gin-reloptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L604-L613)
- [ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L630-L655)
- [ginvacuum.c#recyclable](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L786)
- [ginvacuum.c#ginvacuumcleanup-heap-count](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L723-L731)
- [ginvacuum.c#ginUpdateStats](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L784)
- [brin.c#brininsert-unsummarized](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L134-L142)
- [brin.c#brininsert-break](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L212-L218)
- [brin.c#bringetbitmap](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L355-L366)
- [brin.c#brinbuild-idxtuples](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L726-L742)
- [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)
- [brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L786-L800)
- [brin.c#brin_summarize_new_values](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L850-L860)
- [brin.c#brinsummarize-counts](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1299-L1312)
- [brin.c#FreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1490-L1500)
- [brin_pageops.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/brin/brin_pageops.c#L455-L466)
- [hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L108-L160)
- [hash.c#hashbuild-result](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L179-L188)
- [hash.c#hashbulkdelete-count](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L574-L632)
- [hash.c#hashvacuumcleanup](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L640-L656)
- [hashpage.c#splitpoint-batch](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L779-L800)
- [hashovfl.c#reinit-freed-overflow](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L593-L611)
- [blinsert.c#blbuild-result](../../../../raw/postgres-12/contrib/bloom/blinsert.c#L155-L159)
- [blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L168-L216)
- [blvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L195-L217)
- [blutils.c#blhandler](../../../../raw/postgres-12/contrib/bloom/blutils.c#L100-L148)
- [pgstattuple.control:3](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L3)
- [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31)
- [pgstattuple--1.4--1.5.sql#privileges](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)
- [guc.c#default_statistics_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1986-L1994)
- [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)
- [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)
- [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)
- [guc.c#random_page_cost](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3218-L3227)
- [guc.c#vacuum_cleanup_index_scale_factor](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3423-L3431)
- [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90)
- [indices.sgml#indexes-partial](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L757-L772)
- [reindex.sgml#locking](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L155-L165)
- [reindex.sgml#concurrently](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L300-L315)
- [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)
- [create_index.sql#btree-partial](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L62-L72)
- [create_index.sql#concurrent-partial](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L479-L481)
- [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854)
- [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)
- [select.sql#partial-index-planning](../../../../raw/postgres-12/src/test/regress/sql/select.sql#L190-L224)
- [join_hash.sql#forged-reltuples](../../../../raw/postgres-12/src/test/regress/sql/join_hash.sql#L68-L84)
- [vacuum-reltuples.spec](../../../../raw/postgres-12/src/test/isolation/specs/vacuum-reltuples.spec#L1-L48)
- [gin.sql#pending-list](../../../../raw/postgres-12/src/test/regress/sql/gin.sql#L7-L35)
- [brin.sql#summarize](../../../../raw/postgres-12/src/test/regress/sql/brin.sql#L404-L416)

## Navigation

- [v12/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 12 (unverified)](comment-stored-index-heap-ratio-bloat.md)
- [Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in PostgreSQL 12? (unverified)](comment-stored-table-dml-counters-gin-reindex.md)
- [Physical Index Statistics, Tuple Counts, and Bytes per Tuple in PostgreSQL 12 (unverified)](physical-index-statistics-tuple-counts-and-bytes.md)
- [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](btree-index-bloat-core-sql-only.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
