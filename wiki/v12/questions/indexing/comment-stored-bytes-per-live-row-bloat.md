---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Calibrating a COMMENT-Stored Bytes-per-Live-Row REINDEX Threshold in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What the ratio measures, input by input](#what-the-ratio-measures-input-by-input)
  - [Why index bytes are a high-water mark](#why-index-bytes-are-a-high-water-mark)
  - [Calibration fixture and workloads](#calibration-fixture-and-workloads)
  - [The measured drift-versus-reclaim matrix](#the-measured-drift-versus-reclaim-matrix)
  - [Calibrated thresholds per index type](#calibrated-thresholds-per-index-type)
  - [What a rebuild actually bought: space and query time](#what-a-rebuild-actually-bought-space-and-query-time)
  - [GIN: the pending list and keys per row](#gin-the-pending-list-and-keys-per-row)
  - [BRIN and SP-GiST: no threshold separates them](#brin-and-sp-gist-no-threshold-separates-them)
  - [Partial indexes: the denominator is wrong by construction](#partial-indexes-the-denominator-is-wrong-by-construction)
  - [Which live-row estimate to divide by](#which-live-row-estimate-to-divide-by)
  - [Baseline reproducibility and the noise floor](#baseline-reproducibility-and-the-noise-floor)
  - [Tested capture and detection SQL](#tested-capture-and-detection-sql)
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

## Answer

### Verdict

The scheme works, but only as a **per-access-method screen with a per-access-method threshold**, and only after you exclude two access methods outright. On an isolated PostgreSQL 12.2 server built from the pin, over 12 workloads x 8 indexes with `REINDEX` as ground truth:

| Access method | Calibrated trigger | Measured separating band (benign max / harmful min) | Verdict |
|---|---|---|---|
| B-tree | +30% (`drift >= 1.30`) | 1.0000 / 1.4991 | usable; wide clean band |
| GiST | +30% | 1.0000 / 1.4374 | usable, but the baseline itself is only reproducible to 1.3% |
| hash | +30% | 1.1398 / 1.4002 | usable; healthy growth already reaches 1.14 |
| GIN, `fastupdate = off` | +40% | 1.2918 / 1.4898 | usable, but the band is narrow |
| GIN, `fastupdate = on` (v12 default) | +40% **after** cleaning the pending list | 1.0000 / 1.5273 | unusable while the pending list is unflushed |
| contrib `bloom` | +60% | 1.2513 / 1.9975 | usable |
| SP-GiST | none | 2.2041 / 2.0000 | **not separable**: bands overlap |
| BRIN | none | 2.0000 / no harmful cell | **do not use**: the signal is inverted |

Scored as a classifier over the first 11 workloads (88 cells; the twelfth workload was added later and appears in the bands above and in the matrix below), a single cross-access-method threshold of `+30%` to `+40%` gave 43 true positives, 5 false positives, 0 false negatives and 40 true negatives (94.3% accuracy), with harmful defined as "`REINDEX` reclaims at least 25% of the index". All five false positives were BRIN or SP-GiST cells. Restricted to the other five access methods (66 cells over six indexes), `drift >= 1.30` and `drift >= 1.40` were both exactly right: 37 true positives, 0 false positives, 0 false negatives, 29 true negatives.

Three results matter more than the threshold numbers:

1. **The ratio is dominated by its denominator.** In all three delete workloads the drift was exactly `2.0000` for every one of the eight indexes, because index bytes never shrank and the live-row count halved, while the actually reclaimable fraction ranged from 0.00% (BRIN) to 54.19% (GiST). The metric cannot tell those cells apart.
2. **A high ratio often means a cheaper fix is due, not a rebuild.** A GIN pending list drove drift to `4.1231` and made a probe query 12.0x slower, and `gin_clean_pending_list()` restored the query cost while *raising* the ratio to `4.6182`. An unsummarized BRIN range set made a probe read 49,961 blocks instead of 1,411 while the ratio read `0.1000`, a 90% *decrease*.
3. **Reclaimed space is not query improvement.** A B-tree at 45.13% leaf density gave back 49.96% of its bytes on rebuild, which bought exactly 50% fewer index blocks on index-only scans but only 6% to 10% less time, 0% on a point lookup, and 4% to 6% on a query that also fetched heap rows.

Store the baseline, alert on it, and require a second, access-method-specific confirmation before rebuilding. Never wire the ratio straight to `REINDEX`.

### What the ratio measures, input by input

```text
bpr      = pg_relation_size(index_oid) / <table live-row estimate>
drift    = bpr_now / bpr_baseline
increase = (drift - 1) * 100
```

**Numerator.** The one-argument `pg_relation_size(regclass)` is a SQL-language wrapper that passes `'main'`, and the C implementation opens the relation under `AccessShareLock` and `stat()`s every segment of that one fork ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)). It therefore measures *allocated* bytes, including pages that are empty, deleted, or recorded as free in the index free space map. It excludes the index FSM fork, which `pg_table_size` and `pg_total_relation_size` would include ([dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467)).

**Denominator.** `pg_class.reltuples` is documented in the catalog header as "# of tuples (not always up-to-date)" ([pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)). Four paths write it for a table:

- `CREATE INDEX` and `REINDEX` write the *exact* heap count they just scanned, non-transactionally, together with `relpages` ([index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2656-L2680), [index.c#index_update_stats-reltuples](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2795), [index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987)).
- Plain `ANALYZE` writes a sample-derived total ([analyze.c#relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L587-L605)).
- `VACUUM` writes `vac_estimate_reltuples()`, which uses the count it saw for scanned pages and the previous `reltuples / relpages` density for pages it skipped ([vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1058-L1113), [vacuumlazy.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L353-L370)).
- All of them land in `vac_update_relstats()`, which overwrites the `pg_class` row in place ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1196)).

`pg_class.reltuples` counts only live tuples by definition, which is why it is the right shape for this metric ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1151-L1153)).

**Ratio semantics.** The value you must store is the measured baseline, for example `22.511616`, not the literal `1.0`. The `1.0` in the proposal is the normalized starting drift, `bpr_baseline / bpr_baseline`. Storing `1.0` throws away the scale that later comparisons need.

The core reason the ratio is only a screen: PostgreSQL 12 exposes no access-method callback that reports reclaimable space. `IndexAmRoutine` carries build, insert, bulk-delete, cleanup, cost, options, validate, and scan callbacks and nothing about bloat or free space ([amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L233)). Any generic screen is therefore inferring waste from allocation.

### Why index bytes are a high-water mark

No core index access method shrinks its own main fork in PostgreSQL 12. VACUUM returns reusable index pages to the index free space map instead, through `RecordFreeIndexPage()`, which records the page as fully free ([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L56)):

| Access method | What VACUUM does with emptied pages |
|---|---|
| B-tree | records recyclable pages in the FSM and vacuums the FSM if any were found ([nbtree.c#recyclable-page](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1183), [nbtree.c#IndexFreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1085-L1095)) |
| GiST | unlinks empty leaf pages from the tree in a second stage, then records them ([gistvacuum.c#delete-empty-pages](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L115-L129), [gistvacuum.c#gistdeletepage](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L586-L672), [gistvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L303-L312)) |
| SP-GiST | records new or empty pages ([spgvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L655-L665)) |
| GIN | records recyclable posting-tree pages and freed pending-list pages ([ginvacuum.c#recyclable](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L786), [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667)) |
| contrib `bloom` | records new or deleted pages ([blvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L195-L217)) |
| hash | no `RecordFreeIndexPage()` call exists under `src/backend/access/hash`; a freed overflow page is reinitialized as `LH_UNUSED_PAGE` and tracked in the hash index's own overflow bitmap ([hashovfl.c#reinit-freed-overflow](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L593-L611)) |
| BRIN | records free space on its own regular pages during insertion and vacuums the FSM at cleanup ([brin.c#FreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1490-L1500), [brin_pageops.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/brin/brin_pageops.c#L455-L466)) |

SP-GiST contains the only index-truncation code, and it is compiled out: the block is wrapped in `#ifdef NOT_USED` with the comment "disabled because it's unsafe due to possible concurrent inserts ... Note that btree doesn't do this either" ([spgvacuum.c#disabled-truncation](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)). For indexes, the only `RelationTruncate()` caller left is `TRUNCATE TABLE`, which resets every index to zero blocks ([heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3164-L3199)).

Consequence for the metric: bytes per live row can only go up or stay flat as long as the row count is stable, no matter how much of the index is reusable free space. "More space per live row" is literally true and still does not mean "wasted space a rebuild can recover".

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

Sequence per workload: build, `ANALYZE`, capture baseline, run the workload, `VACUUM` where noted, `ANALYZE`, capture current, probe queries, `REINDEX TABLE`, capture post-rebuild size, probe again. Ground truth is `reclaim_pct = (bytes_before - bytes_after) / bytes_before`.

| Workload | Definition | Observed table counters |
|---|---|---|
| `w_noop` | nothing after the build | — |
| `w_grow_seq` | append 200,000 rows, ascending keys | 400,000 ins |
| `w_grow_10x` | append 1,800,000 rows, ascending keys | 2,000,000 ins |
| `w_grow_rand` | append 200,000 rows, keys drawn from the existing domain | 400,000 ins |
| `w_hot_ff40` | heap `fillfactor = 40`, two full-table updates of the non-indexed column | 400,000 upd, **400,000 HOT** |
| `w_hot_upd` | default fillfactor, two full-table updates of the non-indexed column | 400,000 upd, **14 HOT** |
| `w_idx_upd` | two full-table updates that change every indexed column | 400,000 upd, 0 HOT |
| `w_del_range` | delete the 100,000 lowest ids, then `VACUUM` | 100,000 del |
| `w_del_scatter` | delete 100,000 scattered ids, then `VACUUM` | 100,000 del |
| `w_del_novac` | the same scattered delete, no `VACUUM` | 100,000 del, 100,000 dead |
| `w_churn` | 4 rounds of insert 50,000 / delete 50,000, `VACUUM` per round, keys advance | 400,000 ins, 200,000 del |
| `w_churn2` | the same churn with keys kept inside the original domain | 400,000 ins, 200,000 del |

`w_hot_ff40` versus `w_hot_upd` is the one comparison that isolates HOT. Identical statements against a heap with and without free space produced 400,000 HOT updates and 14 HOT updates respectively, read from `pg_stat_all_tables.n_tup_hot_upd` ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)). "Update-heavy" on its own says nothing about index growth.

### The measured drift-versus-reclaim matrix

Each cell is `drift / reclaim_pct`. Negative reclaim means the rebuild produced a **larger** index.

| Workload | btree | hash | gist | spgist | gin (fastupdate on) | gin (off) | brin | bloom |
|---|---|---|---|---|---|---|---|---|
| `w_noop` | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.65 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 |
| `w_grow_seq` | 0.9973 / 0.00 | 1.0894 / 8.65 | 0.9397 / -5.71 | 1.7125 / -0.05 | 1.8082 / 58.35 | 1.0000 / 24.69 | 0.5000 / 0.00 | 0.9987 / 0.13 |
| `w_grow_10x` | 0.9956 / -0.02 | 1.1398 / 12.54 | 0.8975 / -11.26 | 2.2041 / 0.01 | 1.5273 / 32.95 | 1.2918 / 20.73 | 0.1000 / -33.33 | 0.9959 / 0.03 |
| `w_grow_rand` | 0.9991 / 0.09 | 1.0882 / 5.53 | 0.8511 / -11.76 | 1.2301 / 1.50 | 1.6735 / 60.00 | 0.7204 / 7.08 | 0.5000 / 0.00 | 0.9987 / 0.13 |
| `w_hot_ff40` | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / -0.52 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.0000 / 0.00 |
| `w_hot_upd` | 3.9873 / 74.92 | 3.0973 / 67.71 | 2.6737 / 62.88 | 3.4689 / 71.17 | 6.1429 / 83.72 | 4.0408 / 75.25 | 1.0000 / 0.00 | 2.9924 / 66.58 |
| `w_idx_upd` | 2.9909 / 66.57 | 3.1667 / 68.61 | 2.6659 / 62.54 | 3.4610 / 71.11 | 6.1347 / 83.70 | 4.0408 / 75.25 | 1.0000 / 0.00 | 2.9924 / 66.58 |
| `w_del_range` | 2.0000 / 49.91 | 2.0000 / 37.47 | 2.0000 / 49.84 | 2.0000 / 46.80 | 2.0000 / 44.08 | 2.0000 / 44.08 | 2.0000 / 0.00 | 2.0000 / 49.75 |
| `w_del_scatter` | 2.0000 / 49.91 | 2.0000 / 37.47 | 2.0000 / 54.19 | 2.0000 / 48.12 | 2.0000 / 40.41 | 2.0000 / 40.41 | 2.0000 / 0.00 | 2.0000 / 49.75 |
| `w_del_novac` | 2.0000 / 49.91 | 2.0000 / 37.47 | 2.0000 / 53.47 | 2.0000 / 48.12 | 2.0000 / 40.41 | 2.0000 / 40.41 | 2.0000 / 0.00 | 2.0000 / 49.75 |
| `w_churn` | 1.4991 / 33.29 | 1.4002 / 29.19 | 1.4374 / 30.72 | 3.4058 / 67.11 | 2.5020 / 60.03 | 1.0000 / 0.00 | 1.0000 / 0.00 | 1.2513 / 20.08 |
| `w_churn2` | 2.0000 / 49.91 | 2.1764 / 53.55 | 1.7085 / 51.52 | 2.4689 / 60.67 | 3.3837 / 88.18 | 1.4898 / 73.15 | 1.0000 / 0.00 | 1.9975 / 49.94 |

What the matrix shows, cell by cell:

- **The three delete rows are identical at `2.0000` across all eight indexes** while reclaim spans 0.00% to 54.19%. Bytes were unchanged (nothing shrinks a fork) and the live-row estimate halved, so the drift is arithmetically `1 / 0.5` regardless of the access method. `w_del_novac` proves the same drift arrives whether or not VACUUM has run, because `ANALYZE` alone moves the denominator.
- **`w_hot_ff40` is the perfect true negative.** 400,000 HOT updates left every index byte-identical, drift `1.0000`, reclaim 0.00%, except GiST at -0.52%.
- **Both update rows are strong true positives** for every access method except BRIN, whose bytes never move because `brinbulkdelete()` is a no-op and updates fall inside already-summarized ranges.
- **Growth rows expose the false-positive risk per access method.** SP-GiST reached `2.2041` on healthy 10x growth with 0.01% reclaimable. Hash reached `1.1398` with 12.54%. BRIN fell to `0.1000` while the rebuild made it *larger*.
- **GIN with `fastupdate = off` produced the matrix's near-miss**: `w_grow_seq` drift exactly `1.0000` with 24.69% reclaimable, one point below the harmful line, because doubling the rows with the same key distribution doubles the bytes exactly while incremental posting-list packing still leaves a quarter of the index recoverable. Compare `w_churn` (drift `1.0000`, 0.00% reclaimed, a true negative) with `w_churn2` (drift `1.4898`, 73.15% reclaimable).

### Calibrated thresholds per index type

Scoring the 88 cells against candidate thresholds, with harmful defined as `reclaim_pct >= 25`:

| Threshold | TP | FP | FN | TN | Accuracy |
|---|---|---|---|---|---|
| 1.05 | 43 | 11 | 0 | 34 | 87.5% |
| 1.10 | 43 | 9 | 0 | 36 | 89.8% |
| 1.20 | 43 | 8 | 0 | 37 | 90.9% |
| **1.30** | **43** | **5** | **0** | **40** | **94.3%** |
| **1.40** | **43** | **5** | **0** | **40** | **94.3%** |
| 1.50 | 40 | 5 | 3 | 40 | 90.9% |
| 1.75 | 38 | 4 | 5 | 41 | 89.8% |
| 2.00 | 37 | 4 | 6 | 41 | 88.6% |
| 2.50 | 16 | 0 | 27 | 45 | 69.3% |

The five false positives at `1.30` are the three BRIN delete cells at `2.0000` plus SP-GiST `w_grow_seq` at `1.7125` and SP-GiST `w_grow_10x` at `2.2041`. Dropping BRIN and SP-GiST leaves 66 cells on which `1.30` and `1.40` are both exact: 37 true positives, 0 false positives, 0 false negatives, 29 true negatives.

Per access method, the useful number is the **separating band** between the highest benign drift and the lowest harmful drift, because the threshold must sit inside it:

| Access method | Highest benign drift (workload) | Lowest harmful drift (workload) | Recommended trigger |
|---|---|---|---|
| btree | 1.0000 (`w_noop`) | 1.4991 (`w_churn`) | 1.30 |
| gist | 1.0000 (`w_noop`) | 1.4374 (`w_churn`) | 1.30 |
| hash | 1.1398 (`w_grow_10x`, 12.54% reclaimable) | 1.4002 (`w_churn`) | 1.30 |
| gin `fastupdate = off` | 1.2918 (`w_grow_10x`, 20.73%) | 1.4898 (`w_churn2`) | 1.40 |
| gin `fastupdate = on` | 1.0000 (`w_churn`, 0%) | 1.5273 (`w_grow_10x`) | 1.40, pending list flushed first |
| bloom | 1.2513 (`w_churn`, 20.08%) | 1.9975 (`w_churn2`) | 1.60 |
| spgist | 2.2041 (`w_grow_10x`, 0.01%) | 2.0000 (`w_del_*`) | none: band inverted |
| brin | 2.0000 (`w_del_*`, 0%) | no harmful cell in 12 workloads | none |

Two honest limits on these numbers. First, each band rests on 11 or 12 workload observations per access method, so the endpoints are the extremes of this fixture, not a distribution. Second, the "harmful" line is a policy choice: at `reclaim_pct >= 50` the bloom and GIN bands move, and adding an absolute floor changes the labels of small indexes. With a 1 MB reclaim floor the GIN delete cells (864 kB reclaimable out of 1,960 kB) flip from harmful to benign and GIN's band inverts, which is why the floor belongs in the policy, not in the ground truth.

### What a rebuild actually bought: space and query time

Space and query time are close to uncorrelated. The clean B-tree measurement uses one 500,000-row table, one index, five delete/insert churn rounds with `VACUUM` per round, and probes taken twice with the visibility map fully set so that every index-only scan reports `Heap Fetches: 0`. All probes take the best of five runs, with `enable_seqscan`, `enable_bitmapscan`, and parallelism off for the index-only shapes:

| Measure | Bloated | After `REINDEX` | Change |
|---|---|---|---|
| `pgstatindex.index_size` | 22,511,616 | 11,264,000 | -49.96% |
| `pgstatindex.leaf_pages` | 2,736 | 1,367 | -50.04% |
| `pgstatindex.avg_leaf_density` | 45.13 | 90.07 | +99.6% |
| `pgstatindex.leaf_fragmentation` | 49.93 | 0 | — |
| bytes per live row | 45.0232 | 22.5280 | drift was `2.0000` |
| index-only range scan, 99,762 rows | 551 blocks, 9.863 ms | 276 blocks, 8.891 ms | -49.9% blocks, -9.9% time |
| index-only full scan, 500,000 rows | 2,739 blocks, 47.574 ms | 1,370 blocks, 44.453 ms | -50.0% blocks, -6.6% time |
| point lookup | 3 blocks, 0.001 ms | 3 blocks, 0.001 ms | none |
| bitmap scan plus heap payload | 6,233 blocks, 19.819 ms | 5,958 blocks, 18.648 ms | -4.4% blocks, -5.9% time |

The block counts track the size exactly; the times do not, because every page was already in `shared_buffers` and the remaining cost is per-tuple CPU work. On the eight-index `w_churn2` fixture the same pattern holds for the other access methods, using total blocks (`shared hit + shared read`) so that cache state cannot distort the comparison:

| Access method | reclaim | probe blocks before / after | probe time before / after |
|---|---|---|---|
| bloom | 49.94% | 787 / 394 (-49.9%) | 0.563 / 0.502 ms (-10.8%) |
| btree | 49.91% | 5,683 / 5,546 (-2.4%) | 12.023 / 12.066 ms (+0.4%) |
| gist | 51.52% | 5,713 / 5,233 (-8.4%) | 2.782 / 3.153 ms (+13.3%) |
| spgist | 60.67% | 5,486 / 5,285 (-3.7%) | 2.484 / 2.536 ms (+2.1%) |
| hash | 53.55% | 1 / 1 | 0.001 / 0.001 ms |
| brin | 0.00% | 6,845 / 5,437 (-20.6%) | 9.863 / 9.607 ms (-2.6%) |

contrib `bloom` is the one access method whose query cost is proportional to its size, because `blgetbitmap()` reads the whole index for every scan, using a `BAS_BULKREAD` strategy ([blscan.c#blgetbitmap](../../../../raw/postgres-12/contrib/bloom/blscan.c#L119-L138)). Its tuples are also fixed width, a signature of `DEFAULT_BLOOM_LENGTH` bits by default ([bloom.h#DEFAULT_BLOOM_LENGTH](../../../../raw/postgres-12/contrib/bloom/bloom.h#L88-L92), [blutils.c#bloomLength](../../../../raw/postgres-12/contrib/bloom/blutils.c#L88-L96)), so its healthy bytes per live row is nearly constant: 16.1382 at baseline and 16.0724 after 10x growth.

One measurement trap is worth recording because an earlier run of this experiment attributed it to the rebuild. After heavy churn, the *first* `VACUUM` does not leave the heap all-visible: `lazy_scan_heap()` sets the visibility map only for pages it found fully visible during the first pass ([vacuumlazy.c#set-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1287-L1315)), and the second pass that removes the dead line pointers sets the bit only if `PD_ALL_VISIBLE` is already on the page ([vacuumlazy.c#lazy_vacuum_page-vm](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1655-L1676)). Measured on the same fixture with `TIMING off`: after churn plus one `VACUUM`, an index-only scan did 99,760 heap fetches and read 100,300 blocks in 52.210 ms; after a second `VACUUM` set all 11,364 pages all-visible, the identical query on the identical, still-bloated index did 0 heap fetches, 551 blocks, and 6.700 ms. That is 182x the blocks and 7.8x the time, all of it visibility map and none of it index bloat. Any before-and-after rebuild comparison must hold the visibility map constant or it measures the wrong thing.

### GIN: the pending list and keys per row

GIN is the access method the question calls out, and it is the one where the proxy fails in the most expensive direction: it fires hardest when the correct fix is not a rebuild.

In PostgreSQL 12 `fastupdate` defaults to **on** ([gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-12/src/include/access/gin_private.h#L23-L39)), so ordinary inserts land in a pending list and are only merged into the entry tree when the list outgrows `gin_pending_list_limit`, which is `PGC_USERSET` with a 4 MB default and can be overridden per index by the reloption of the same name ([ginfast.c#needCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L462), [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [ginutil.c#gin-reloptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L604-L613)). Every GIN scan must read the whole pending list before the main index, because a pending entry can belong anywhere in the tree ([ginget.c#scanPendingInsert](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1750-L1760), [ginget.c#gingetbitmap-order](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1866-L1884)).

Measured on a 200,000-row table with two GIN indexes on identical `int[]` columns, then 200,000 more rows inserted with `gin_pending_list_limit = '1GB'` so nothing auto-flushed:

| Phase | `fastupdate = on` bytes | bpr | drift | probe blocks | probe time |
|---|---|---|---|---|---|
| baseline (200,000 rows) | 1,662,976 | 8.3149 | 1.0000 | — | — |
| 200,000 rows in the pending list | 13,713,408 | 34.2835 | **4.1231** | 2,274 | 7.605 ms |
| after `gin_clean_pending_list()` | 15,360,000 | 38.4000 | **4.6182** | 803 | 0.634 ms |
| after `REINDEX` | 2,695,168 | 6.7379 | 0.8103 | 803 | 0.467 ms |

The `pageinspect` metapage view confirmed 1,471 pending pages holding 200,000 pending tuples before the cleanup and 0 after. Three consequences:

1. The pending list alone produced drift `4.1231`, far past any calibrated trigger, on an index whose entry tree was healthy.
2. `gin_clean_pending_list()` fixed the query cost, 12.0x faster and 2.83x fewer blocks, and made the metric **worse**, from `4.1231` to `4.6182`, because merging the pending entries extends the fork while the freed pending pages only go back to the FSM ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074), [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667)). A drift-triggered rule that recaptures the baseline after the cheap fix would raise its own baseline by 4.6x.
3. Only after the cleanup does the residual `REINDEX` gain, 0.634 ms to 0.467 ms, become visible, and it is small next to the 7.605 ms the pending list cost.

The same run gives the `fastupdate = off` comparison: 1,662,976 to 3,309,568 bytes for the identical inserts, drift `0.9951`, and a rebuild that still reclaimed 18.56%. Flat drift, real reclaimable space.

The second GIN-specific factor is keys per row. GIN stores one entry per extracted key, not one per row. Appending 200,000 rows with 20 keys each to a 200,000-row table whose rows had one key each moved bytes per live row from 3.4816 to 21.9955, drift `6.3176`, and the rebuild produced a **16.85% larger** index (8,798,208 to 10,280,960 bytes). That is the worst cell measured: a maximal alert with negative payoff.

Finally, do not read the GIN metapage counters as current. `ginGetStats()` documents that only `nPendingPages` and `ginVersion` are up to date and the rest are "as of the last VACUUM" ([ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L630-L655), [ginvacuum.c#ginUpdateStats](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L784)). The measured metapage still reported `n_total_pages = 203` for a 1,875-page index.

### BRIN and SP-GiST: no threshold separates them

**BRIN** must be excluded. Its size is a function of heap page ranges, not rows, so healthy growth *lowers* bytes per live row: `0.5000` at 2x rows and `0.1000` at 10x. Deletes raise it to exactly `2.0000` with 0.00% reclaimable, because `brinbulkdelete()` deliberately does nothing ([brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)). In 12 workloads BRIN never had a harmful cell, so every trigger at or below `2.0000` is a pure false positive.

Worse, the signal is inverted where it matters. Rows appended past the last summarized range are not summarized, because `brininsert()` returns without doing anything when the revmap has no entry for the range ([brin.c#brininsert-unsummarized](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L134-L142), [brin.c#brininsert-break](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L212-L218)), and a scan must return **all** pages of an unsummarized range regardless of the scan keys ([brin.c#bringetbitmap](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L355-L366)). Measured on `w_grow_10x`: the probe read 49,961 blocks in 150.088 ms before the rebuild and 1,411 blocks in 6.144 ms after, 35.4x the blocks and 24.4x the time, while bytes per live row read `0.1000`, a 90% decrease, and the rebuild changed the size by -8,192 bytes. The correct action is `VACUUM` or `brin_summarize_new_values()`, since `brinvacuumcleanup()` exists precisely to "summarize ranges that are currently unsummarized" ([brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L786-L800)). Stale summaries left by deletes are the mirror image: after `w_churn` the pre-rebuild probe read 1,511 blocks against 2 afterwards, at drift `1.0000` and 0.00% reclaim.

**SP-GiST** must not be automated on this signal. Its benign maximum, `2.2041` on 10x growth with 0.01% reclaimable, sits *above* its harmful minimum, `2.0000` in the delete workloads with 46.80% to 48.12% reclaimable. No threshold can separate the two sets, and the best scoring threshold, 1.75, still carries a false positive. Report SP-GiST drift for human inspection only.

### Partial indexes: the denominator is wrong by construction

Dividing by the *table's* live-row count mixes two populations whenever the index has a `WHERE` clause: the index only holds rows that satisfy the predicate, but the denominator counts every live row. Plain `ANALYZE` computes a per-index fraction for exactly this reason and stores `ceil(tupleFract * totalrows)` in the *index's* own `reltuples` ([analyze.c#tupleFract](../../../../raw/postgres-12/src/backend/commands/analyze.c#L430-L438), [analyze.c#tupleFract-estimate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L817-L822), [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)). The proposed metric throws that fraction away.

Two measured fixtures, each with a partial B-tree index over 10% of the rows:

- **Detected anyway.** Five delete/reinsert cycles inside the predicate plus 2x growth outside it: bpr 2.3347 to 6.7584, drift `2.8948`, reclaim 82.73%. The bloat outran the denominator.
- **Missed completely.** Two cycles inside the predicate plus 8x growth outside it: bpr 2.3347 to 0.8499, drift `0.3640`, a 63.6% *decrease*, while `REINDEX` reclaimed 65.66% (1,359,872 to 466,944 bytes). No threshold above 1.0 can fire on a falling ratio.

The first fixture also shows how table growth silently rescales the baseline. After the rebuild the index was byte-identical to its baseline, 466,944 bytes, yet its bytes per live row read 1.1674 against a stored 2.3347, because the table had doubled. A freshly rebuilt index therefore reports drift `0.5000` until the baseline is recaptured. For partial indexes, divide by the index's own `reltuples` instead, or accept that the number is not comparable over time.

### Which live-row estimate to divide by

Two catalog sources answer "estimated live rows", and they disagree by design. `pg_stat_all_tables.n_live_tup` comes from `pg_stat_get_live_tuples()`, which returns the collector's `n_live_tuples` counter ([pgstatfuncs.c#pg_stat_get_live_tuples](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L151-L164)). That counter accumulates per-transaction deltas and is clamped at zero ([pgstat.c#n_live_tuples-delta](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L5974-L5994)), and it is *overwritten*, not adjusted, by every VACUUM and ANALYZE report ([pgstat.c#pgstat_recv_vacuum](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6206-L6225), [pgstat.c#pgstat_recv_analyze](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6240-L6260), [pgstat.c#pgstat_report_analyze](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1428-L1482)).

Measured on one 200,000-row table:

| State | `reltuples` | `n_live_tup` | bpr from `reltuples` | bpr from `n_live_tup` |
|---|---|---|---|---|
| after `CREATE INDEX`, never analyzed | 200,000 | 200,000 | 22.5690 | 22.5690 |
| after deleting half, no `VACUUM`, no `ANALYZE` | 200,000 | 100,000 | 22.5690 | 45.1379 |
| after `pg_stat_reset()` | 200,000 | **0** | 22.5690 | division by zero |

Use `pg_class.reltuples`, for three reasons the table above makes concrete: it survives a statistics reset, which discards every table entry wholesale ([pgstat.c#pgstat_recv_resetcounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6086-L6122)); it is exact immediately after the build with no `ANALYZE` at all, because `index_update_stats()` writes the count the build scanned; and it moves only when a maintenance command moves it, so drift reflects the index rather than DML traffic. The cost of that choice is a stale denominator: the second row above is a real 2x under-report of bytes per live row that persists until the next `ANALYZE` or `VACUUM`. Run plain `ANALYZE` before both the capture and the comparison, not `VACUUM (ANALYZE)`, since the index-side `reltuples` update is deliberately skipped when analysis runs as part of VACUUM ([analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)) and VACUUM's own index update happens only when the access method reports an exact count ([vacuumlazy.c#index-relstats](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1800-L1816)).

One more trap: statistics snapshots are per transaction. Six `ANALYZE` runs inside a single `DO` block moved `pg_class.reltuples` each time while `n_live_tup` stayed frozen at the first value, 1,199,367, until the transaction ended.

### Baseline reproducibility and the noise floor

The threshold has to clear the noise in its own inputs. Both are small, but one is not zero.

**Denominator noise.** On a 1,200,000-row table of variable-width rows occupying 76,097 pages, well past the 30,000-row sample that `default_statistics_target = 100` requests, six consecutive identical `ANALYZE` runs returned 1,199,367 / 1,199,438 / 1,199,728 / 1,200,090 / 1,200,671 / 1,201,447 rows: a 2,080-row spread, 0.173% of the true 1,200,000, with per-run error from -0.0528% to +0.1206%. On the 200,000-row fixtures every `ANALYZE` was exact, because a table smaller than the sample is scanned completely and `vac_estimate_reltuples()` returns the count as-is ([vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1083-L1085)); the only sampled fixture in the matrix, `w_grow_10x`, reported 2,000,035 rows against 2,000,000 actual, an error of 0.00175%. Sampling noise is not what limits this metric.

**Numerator noise.** Across ten identical builds of the same fixture, seven of the eight indexes were byte-identical every time. GiST produced nine distinct sizes spanning 17,350,656 to 17,580,032 bytes, a 1.322% spread, because `gistchoose()` breaks equal-penalty ties with `random()` ([gistutil.c#gistchoose-random](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L502-L537)). A GiST baseline is therefore only reproducible to about 1.3%, and a GiST rebuild can legitimately land larger than the index it replaced: -5.71% to -11.76% "reclaim" in the three growth workloads.

### Tested capture and detection SQL

Both statements below were executed against the pinned build. Timeouts are session-scoped: `statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so they need neither reload nor restart ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)).

```sql
SET /* wiki_index_bpr_timeout */ statement_timeout = '15min';
SET /* wiki_index_bpr_lock_timeout */ lock_timeout = '5s';

ANALYZE /* wiki_index_bpr_refresh */ app.orders;
```

Capture. This reads only; review the generated `comment_sql` values before running them. It refuses to overwrite a comment it did not write, and skips any index whose table has no positive live-row estimate.

```sql
WITH /* wiki_capture_index_bpr */ measured AS MATERIALIZED
(
    SELECT ins.nspname                        AS index_schema,
           ic.relname                         AS index_name,
           am.amname                          AS amname,
           i.indrelid                         AS table_oid,
           pg_relation_size(ic.oid)           AS index_bytes,
           tc.reltuples::numeric              AS table_rows,
           obj_description(ic.oid, 'pg_class') AS existing_comment
    FROM pg_index AS i
    JOIN pg_class AS ic ON ic.oid = i.indexrelid
    JOIN pg_namespace AS ins ON ins.oid = ic.relnamespace
    JOIN pg_am AS am ON am.oid = ic.relam
    JOIN pg_class AS tc ON tc.oid = i.indrelid
    WHERE ic.relkind = 'i'
      AND i.indislive AND i.indisready AND i.indisvalid
      AND ins.nspname IN ('app')
)
SELECT /* wiki_capture_index_bpr */
       index_schema, index_name, amname, index_bytes,
       trunc(table_rows)::bigint AS table_rows,
       round(index_bytes / nullif(table_rows, 0), 6) AS bpr,
       existing_comment,
       CASE
           WHEN table_rows <= 0 THEN NULL
           WHEN existing_comment IS NOT NULL
                AND existing_comment NOT LIKE 'wiki_bpr_v1;%' THEN NULL
           ELSE format(
                    'COMMENT /* wiki_capture_index_bpr */ ON INDEX %I.%I IS %L;',
                    index_schema, index_name,
                    format('wiki_bpr_v1;relid=%s;am=%s;bytes=%s;rows=%s;bpr=%s',
                           table_oid, amname, index_bytes,
                           trunc(table_rows)::bigint,
                           round(index_bytes / table_rows, 6)))
       END AS comment_sql
FROM measured
ORDER BY index_schema, index_name;
```

Detection. The per-access-method thresholds are the calibrated values from this fixture; re-derive them for your own workload before trusting them. `nullif(table_rows, 0)` is not optional: without it an empty partition's child index raises `ERROR: division by zero` before the guard in the `CASE` can run, which is how this query failed on its first execution here.

```sql
WITH /* wiki_detect_index_bpr */ threshold (amname, drift_trigger, policy) AS
(
    VALUES ('btree',  1.30, 'rebuild candidate'),
           ('gist',   1.30, 'rebuild candidate'),
           ('hash',   1.30, 'rebuild candidate'),
           ('gin',    1.40, 'clean pending list, re-measure, then rebuild candidate'),
           ('bloom',  1.60, 'rebuild candidate'),
           ('spgist', 1.75, 'inspect only; not separable in calibration'),
           ('brin',   NULL, 'do not use this signal; VACUUM or summarize instead')
),
measured AS MATERIALIZED
(
    SELECT ins.nspname                        AS index_schema,
           ic.relname                         AS index_name,
           am.amname                          AS amname,
           i.indrelid                         AS table_oid,
           pg_relation_size(ic.oid)           AS index_bytes,
           tc.reltuples::numeric              AS table_rows,
           obj_description(ic.oid, 'pg_class') AS stored_comment,
           coalesce((SELECT option_value
                       FROM pg_options_to_table(ic.reloptions)
                      WHERE option_name = 'fastupdate'), 'on (default)') AS fastupdate
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
               WHEN m.stored_comment ~ ('^wiki_bpr_v1;relid=[0-9]+;am=[a-z]+;bytes=[0-9]+;'
                                        || 'rows=[0-9]+;bpr=[0-9]+[.][0-9]+$')
                    AND split_part(split_part(m.stored_comment, 'relid=', 2), ';', 1)::oid
                        = m.table_oid
               THEN split_part(m.stored_comment, ';bpr=', 2)::numeric
           END AS base_bpr
    FROM measured AS m
)
SELECT /* wiki_detect_index_bpr */
       p.index_schema, p.index_name, p.amname, p.fastupdate,
       pg_size_pretty(p.index_bytes) AS size,
       trunc(p.table_rows)::bigint   AS table_rows,
       p.base_bpr,
       round(p.index_bytes / nullif(p.table_rows, 0), 6) AS cur_bpr,
       round((p.index_bytes / nullif(p.table_rows, 0)) / p.base_bpr, 4) AS drift,
       round(((p.index_bytes / nullif(p.table_rows, 0)) / p.base_bpr - 1) * 100, 1)
           AS pct_increase,
       t.drift_trigger,
       CASE
           WHEN p.table_rows <= 0                THEN 'no live-row estimate'
           WHEN p.base_bpr IS NULL               THEN 'no usable baseline'
           WHEN t.drift_trigger IS NULL          THEN t.policy
           WHEN p.index_bytes < 8 * 1024 * 1024  THEN 'below size floor'
           WHEN (p.index_bytes / nullif(p.table_rows, 0)) / p.base_bpr
                >= t.drift_trigger               THEN t.policy
           ELSE 'below threshold'
       END AS decision
FROM parsed AS p
LEFT JOIN threshold AS t ON t.amname = p.amname
ORDER BY p.index_schema, p.index_name;
```

The full loop was exercised end to end on the pin, on a 500,000-row table with a B-tree and a GIN index plus a partitioned parent index and a deliberately invalid index:

| Step | `i_btree` | `i_gin` | `part_1_k_idx` |
|---|---|---|---|
| capture after build | bpr 22.511616 | bpr 1.130496 | skipped, no live-row estimate |
| detect after capture | 1.0000, below threshold | 1.0000, below size floor | no live-row estimate |
| detect after 5 churn rounds | **2.0000, rebuild candidate** | 9.4058, below size floor | no live-row estimate |
| detect after `REINDEX INDEX CONCURRENTLY` | 1.0007, below threshold | 1.0000, below size floor | no live-row estimate |
| detect after recapture | 1.0000, below threshold | 1.0000, below size floor | no live-row estimate |

The partitioned parent index and the `indisvalid = false` index were both filtered out by the `relkind = 'i'` and state-flag predicates ([pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L40-L43), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192)). The 8 MB floor also shows its cost: it suppressed the GIN index at drift `9.4058`, which a rebuild shrank from 5,316,608 to 565,248 bytes, 89.37% reclaimable. A floor buys quiet at the price of small-index waste; set it from the size distribution you actually have.

Comment durability was measured across six operations, all of which preserved the stored baseline: `REINDEX INDEX`, `REINDEX INDEX CONCURRENTLY`, `ALTER INDEX ... RENAME`, `ALTER INDEX ... SET (fillfactor = 80)`, `VACUUM`, and `ANALYZE`. That matches the code path: `COMMENT` resolves the object under `ShareUpdateExclusiveLock`, checks ownership, then writes one `pg_description` row keyed by `(objoid, classoid, objsubid)`, replacing any existing row and deleting it for a NULL or empty comment ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225), [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57)). A plain `REINDEX` keeps the index's OID and only swaps its relfilenode, so the comment row is never touched ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531)), and the concurrent path explicitly moves the comment to the replacement index ([index.c#index_concurrently_swap](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)).

### Operating rules, failure modes, and cheaper fixes

- **One comment per object.** `CreateComments()` maintains a single row per `(objoid, classoid, objsubid)`, so the baseline and human prose cannot coexist. If your indexes already carry comments, use a side table instead.
- **Ordering.** Run plain `ANALYZE` immediately before both capture and detection. `VACUUM` cannot run inside a transaction block while `ANALYZE` can ([vacuum.c#PreventInTransactionBlock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L235-L249)), so a maintenance script that wraps its work in a transaction can refresh the denominator but not vacuum.
- **Before rebuilding, try the cheaper action.** For GIN with `fastupdate = on`, run `gin_clean_pending_list()`; it requires index ownership, rejects non-GIN relations, other sessions' temporary indexes, and recovery ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074)). For BRIN, run `VACUUM` or `brin_summarize_new_values()` ([brin.c#brin_summarize_new_values](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L850-L860)). For B-tree, a page deleted by one VACUUM only becomes recyclable once its `btpo.xact` precedes `RecentGlobalXmin`, so a second VACUUM can return space that the first one could not ([nbtree.c#btvacuumpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1149-L1183)).
- **Confirm with a physical measurement.** contrib `pgstattuple` gives `pgstatindex()` for B-tree, with `avg_leaf_density` and `leaf_fragmentation`, and generic `pgstattuple()` for the others. Both live outside core; at the extension's default version 1.5 every reader function is revoked from `PUBLIC` and granted to the `pg_stat_scan_tables` role ([pgstattuple.control:3](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L3), [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31), [pgstattuple--1.4--1.5.sql#privileges](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)).
- **Rebuild cost and locking.** Plain `REINDEX` locks out writes on the table for the duration; `REINDEX INDEX CONCURRENTLY` trades that for a session-level `SHARE UPDATE EXCLUSIVE` lock plus extra passes, and a failure leaves an invalid `_ccnew`, or `_ccold` when the old definition could not be dropped ([reindex.sgml#locking](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L155-L165), [reindex.sgml#concurrently](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L300-L315), [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)).
- **Recapture policy.** Recapture only after a successful, measured rebuild, or after an approved definition or reloption change. Never recapture to silence unexplained drift: the GIN pending-list case shows that a "fix" can raise the ratio, so a blind recapture would bake a 4.6x inflated baseline in.
- **Do not divide when the denominator is unusable.** Zero or negative `reltuples`, a never-analyzed empty partition, and a reset statistics collector all produce no defined ratio. Report them; do not coerce them to a number.
- **Scope.** Run per database with an explicit schema allowlist; never comment on or rebuild system catalogs from this loop.

### Caller, callee, data structures, and build boundaries

- **Numerator path.** `pg_relation_size(regclass)` (SQL) -> `pg_relation_size(regclass, text)` -> `calculate_relation_size()` -> `smgr` path names and `stat()` per segment ([dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308)).
- **Denominator path.** `ExecVacuum()` -> `analyze_rel()` -> `do_analyze_rel()` -> `vac_update_relstats()` for the table and, outside VACUUM, once per index; `lazy_scan_heap()` -> `vac_estimate_reltuples()` -> `vac_update_relstats()`; `index_build()` -> `index_update_stats()` for both heap and index ([analyze.c#relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L587-L629), [vacuumlazy.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L353-L370), [index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987)).
- **Statistics path.** `pgstat_report_analyze()` and `pgstat_report_vacuum()` send `PgStat_MsgAnalyze` / `PgStat_MsgVacuum`; the collector's `pgstat_recv_analyze()` / `pgstat_recv_vacuum()` overwrite `n_live_tuples` and `n_dead_tuples` in `PgStat_StatTabEntry` ([pgstat.c#pgstat_report_vacuum](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1403-L1426), [pgstat.c#pgstat_recv_analyze](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6240-L6260)).
- **Key structures.** `Form_pg_class` (`relpages`, `reltuples`, `relallvisible`, `relam`, `relkind`), `Form_pg_index` (`indislive`, `indisready`, `indisvalid`, `indpred`), `PgStat_StatTabEntry`, `IndexAmRoutine`, `IndexBulkDeleteResult`, `GinMetaPageData` / `GinStatsData`, `BrinMetaPageData`, `HashMetaPage` (`hashm_maxbucket`, `hashm_spares`, `hashm_ovflpoint`), `BloomOptions.bloomLength`.
- **Access-method size mechanics worth knowing before setting a threshold.** hash sizes its initial buckets from `estimate_rel_size()` at build time and rounds up to a splitpoint ([hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L108-L160), [hashpage.c#initial-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525)), then allocates a whole splitpoint batch at once and extends the logical EOF by writing one zero page at the end of the range, leaving a filesystem hole that `pg_relation_size` still counts ([hashpage.c#splitpoint-batch](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L779-L800), [hashpage.c#_hash_alloc_buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L965-L985)). B-tree pages become recyclable only once their deletion XID precedes `RecentGlobalXmin`, so a rebuild-worthy index can look worse for one more VACUUM cycle ([nbtree.c#btvacuumpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1149-L1183)). BRIN's revmap grows with the heap and its cleanup reports `lastRevmapPage - 1` pages ([brin.c#brinGetStats](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1093-L1108)).
- **Generated artifacts.** The proposal adds no catalog or header: it reads `pg_class`, `pg_index`, `pg_am`, and `pg_description`, whose `_d.h` headers and BKI data are produced by `genbki.pl` during the build ([catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90)).
- **Extension boundary.** contrib `bloom` supplies an ordinary `IndexAmRoutine` from outside core ([blutils.c#blhandler](../../../../raw/postgres-12/contrib/bloom/blutils.c#L100-L148)); a third-party access method can do the same, and core has no way to infer its reclaimable space. `pageinspect` and `pgstattuple` supply the physical confirmation and are likewise contrib.

### Test coverage in the pinned tree

There is no upstream test for a bloat proxy of any kind; the relevant coverage is of the pieces this proposal leans on.

- Index comments surviving both rebuild forms are tested directly ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)).
- The GIN pending list is exercised with `fastupdate = on`, an explicit `gin_pending_list_limit` reloption, `gin_clean_pending_list()`, VACUUM-driven flushing, and a later switch to `fastupdate = off` ([gin.sql#pending-list](../../../../raw/postgres-12/src/test/regress/sql/gin.sql#L7-L35)).
- BRIN summarization and desummarization have dedicated coverage ([brin.sql#summarize](../../../../raw/postgres-12/src/test/regress/sql/brin.sql#L404-L416)).
- Explicit absence: nothing in the tree measures `pg_relation_size` against `reltuples`, calibrates a rebuild threshold, or asserts an access method's reclaimable fraction. Every threshold in this page comes from the fixture described above, not from upstream tests.

## Context Reviewed

- Pinned checkout `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`), built and installed out of tree under `.wiki-runtime/tmp/`, with an isolated cluster on port 55432, `autovacuum = off`.
- Relation-size functions, `pg_class` statistics writers (build, ANALYZE, VACUUM), and the ANALYZE partial-index fraction.
- Cumulative statistics: `pg_stat_all_tables`, live/dead tuple counters, VACUUM/ANALYZE overwrite messages, statistics reset, per-transaction snapshot behavior.
- `COMMENT ON INDEX` locking, ownership, and `pg_description` storage; comment survival across rebuilds and index DDL.
- Per-access-method space behavior for B-tree, hash, GiST, SP-GiST, GIN, BRIN, and contrib `bloom`, including free-space-map recording, the disabled SP-GiST truncation, GIN pending-list insertion and cleanup, BRIN summarization, and hash splitpoint allocation.
- `IndexAmRoutine` callback surface and the absence of any reclaimable-space callback.
- Visibility-map interaction with index-only-scan measurement.
- 88-cell calibration matrix (11 workloads x 8 indexes) plus 12 additional fixtures: churn with a stable key domain, GIN pending list, GIN keys per row, two partial-index shapes, denominator noise, denominator-source divergence, comment durability, and the end-to-end runbook.
- Regression coverage for comments, GIN pending lists, and BRIN summarization; contrib `pgstattuple` control file and 1.4 SQL script.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| The numerator is main-fork allocated bytes, measured under `AccessShareLock` | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308) |
| `reltuples` is an explicitly approximate live-row count with four writers | [pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63), [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1196), [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2656-L2680) |
| No index access method shrinks its own fork; pages go to the FSM instead | [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L56), [nbtree.c#recyclable-page](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1183), [spgvacuum.c#disabled-truncation](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886) |
| No generic callback reports reclaimable index space | [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L233) |
| GIN defaults to `fastupdate = on`, so inserts accumulate in a pending list every scan must read | [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-12/src/include/access/gin_private.h#L23-L39), [ginget.c#gingetbitmap-order](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1866-L1884) |
| Cleaning the pending list frees pages to the FSM without shrinking the fork | [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667), [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074) |
| BRIN ignores deletes and leaves out-of-range inserts unsummarized, and unsummarized ranges return every page | [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784), [brin.c#brininsert-break](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L212-L218), [brin.c#bringetbitmap](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L355-L366) |
| A GiST baseline is not byte-reproducible | [gistutil.c#gistchoose-random](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L502-L537) |
| Hash allocates bucket pages a splitpoint at a time and counts the hole | [hashpage.c#splitpoint-batch](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L779-L800), [hashpage.c#_hash_alloc_buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L965-L985) |
| contrib `bloom` reads the whole index per scan with fixed-width tuples | [blscan.c#blgetbitmap](../../../../raw/postgres-12/contrib/bloom/blscan.c#L119-L138), [bloom.h#DEFAULT_BLOOM_LENGTH](../../../../raw/postgres-12/contrib/bloom/bloom.h#L88-L92) |
| Partial-index row counts are a separately estimated fraction, not the table count | [analyze.c#tupleFract-estimate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L817-L822), [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629) |
| `n_live_tup` is collector state that VACUUM/ANALYZE overwrite and a reset discards | [pgstatfuncs.c#pg_stat_get_live_tuples](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L151-L164), [pgstat.c#pgstat_recv_analyze](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6240-L6260), [pgstat.c#pgstat_recv_resetcounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6086-L6122) |
| One index comment is one `pg_description` row, written under `ShareUpdateExclusiveLock` with an ownership check | [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225) |
| Rebuilds preserve the comment | [index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531), [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854) |
| A first VACUUM after churn need not restore all-visible, so index-only-scan comparisons must control for it | [vacuumlazy.c#set-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1287-L1315), [vacuumlazy.c#lazy_vacuum_page-vm](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1655-L1676) |
| VACUUM cannot run in a transaction block; ANALYZE can | [vacuum.c#PreventInTransactionBlock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L235-L249) |
| The timeouts in the runbook are session-scoped | [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397) |
| `gin_pending_list_limit` is session-scoped and per-index overridable | [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [ginutil.c#gin-reloptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L604-L613) |

## Open Questions

- The per-access-method bands rest on 11 or 12 observations each from a single fixture shape (200,000 rows, one key column per access method, `int[]` GIN keys, `point` GiST/SP-GiST keys). How wide are the bands for other operator classes, especially `tsvector` and `jsonb` GIN, `box`/`range` GiST, and text SP-GiST?
- Ground truth here is a single `REINDEX` size delta. A production rule needs a cost model that weighs reclaimed bytes and query gain against rebuild WAL, I/O, and lock time; none of those were measured.
- Every timing figure came from a fully cached 512 MB `shared_buffers` server. The block-count reductions are cache-independent, but the wall-clock gains from a rebuild on an I/O-bound instance are unmeasured and are likely larger.
- The `w_hot_upd` cell had 14 HOT updates out of 400,000 rather than 0. Which rows those were, and whether page-level pruning explains the residual, was not traced.
- SP-GiST bytes per live row rose to `2.2041` on healthy 10x growth with 0.01% reclaimable. The exact structural cause (inner-tuple and prefix growth with tree depth) was not traced through `spgdoinsert.c`.
- Why a fresh GIN build was 16.85% *larger* than the incrementally grown index in the 20-keys-per-row fixture was not traced to the build accumulator versus incremental posting-list packing.
- The 8 MB size floor suppressed a 89.37%-reclaimable GIN index. No principled floor was derived; it should come from a real index-size distribution.
- Third-party access methods are entirely out of scope: they can register any `IndexAmRoutine` and have no obligation to make allocation track live rows.
- Repeated-observation policy (how many consecutive maintenance windows must agree before a rebuild) was not tested; every measurement here is a single observation per cell.

## Source References

- [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)
- [dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467)
- [pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)
- [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192)
- [pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L40-L43)
- [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57)
- [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L233)
- [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130)
- [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)
- [index.c#index_concurrently_swap](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)
- [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2656-L2680)
- [index.c#index_update_stats-reltuples](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2795)
- [index.c#index_build-stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2987)
- [index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531)
- [analyze.c#tupleFract](../../../../raw/postgres-12/src/backend/commands/analyze.c#L430-L438)
- [analyze.c#relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L587-L605)
- [analyze.c#index-relstats](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)
- [analyze.c#tupleFract-estimate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L817-L822)
- [vacuum.c#PreventInTransactionBlock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L235-L249)
- [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1058-L1113)
- [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1196)
- [vacuumlazy.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L353-L370)
- [vacuumlazy.c#set-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1287-L1315)
- [vacuumlazy.c#lazy_vacuum_page-vm](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1655-L1676)
- [vacuumlazy.c#index-relstats](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1800-L1816)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)
- [pgstatfuncs.c#pg_stat_get_live_tuples](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L151-L164)
- [pgstat.c#pgstat_report_vacuum](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1403-L1426)
- [pgstat.c#pgstat_report_analyze](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1428-L1482)
- [pgstat.c#n_live_tuples-delta](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L5974-L5994)
- [pgstat.c#pgstat_recv_resetcounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6086-L6122)
- [pgstat.c#pgstat_recv_vacuum](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6206-L6225)
- [pgstat.c#pgstat_recv_analyze](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6240-L6260)
- [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L56)
- [heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3164-L3199)
- [nbtree.c#IndexFreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1085-L1095)
- [nbtree.c#btvacuumpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1149-L1183)
- [nbtree.c#recyclable-page](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1183)
- [gistvacuum.c#delete-empty-pages](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L115-L129)
- [gistvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L303-L312)
- [gistvacuum.c#gistdeletepage](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L586-L672)
- [gistutil.c#gistchoose-random](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L502-L537)
- [spgvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L655-L665)
- [spgvacuum.c#disabled-truncation](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)
- [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-12/src/include/access/gin_private.h#L23-L39)
- [ginfast.c#needCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L462)
- [ginfast.c#shiftList-free](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L660-L667)
- [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1074)
- [ginget.c#scanPendingInsert](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1750-L1760)
- [ginget.c#gingetbitmap-order](../../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1866-L1884)
- [ginutil.c#gin-reloptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L604-L613)
- [ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L630-L655)
- [ginvacuum.c#recyclable](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L786)
- [ginvacuum.c#ginUpdateStats](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L784)
- [brin.c#brininsert-unsummarized](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L134-L142)
- [brin.c#brininsert-break](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L212-L218)
- [brin.c#bringetbitmap](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L355-L366)
- [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)
- [brin.c#brinvacuumcleanup](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L786-L800)
- [brin.c#brin_summarize_new_values](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L850-L860)
- [brin.c#brinGetStats](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1093-L1108)
- [brin.c#FreeSpaceMapVacuum](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L1490-L1500)
- [brin_pageops.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/brin/brin_pageops.c#L455-L466)
- [hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L108-L160)
- [hashpage.c#initial-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525)
- [hashpage.c#splitpoint-batch](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L779-L800)
- [hashpage.c#_hash_alloc_buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L965-L985)
- [hashovfl.c#reinit-freed-overflow](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L593-L611)
- [blvacuum.c#RecordFreeIndexPage](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L195-L217)
- [blscan.c#blgetbitmap](../../../../raw/postgres-12/contrib/bloom/blscan.c#L119-L138)
- [blutils.c#bloomLength](../../../../raw/postgres-12/contrib/bloom/blutils.c#L88-L96)
- [blutils.c#blhandler](../../../../raw/postgres-12/contrib/bloom/blutils.c#L100-L148)
- [bloom.h#DEFAULT_BLOOM_LENGTH](../../../../raw/postgres-12/contrib/bloom/bloom.h#L88-L92)
- [pgstattuple.control:3](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L3)
- [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31)
- [pgstattuple--1.4--1.5.sql#privileges](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)
- [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)
- [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)
- [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)
- [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90)
- [reindex.sgml#locking](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L155-L165)
- [reindex.sgml#concurrently](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L300-L315)
- [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)
- [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854)
- [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)
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
