---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Proposing and Testing pgstattuple_approx for Table Heap Bloat in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What the function computes](#what-the-function-computes)
  - [Why the skipped-page numbers are trustworthy in v12](#why-the-skipped-page-numbers-are-trustworthy-in-v12)
  - [Page arithmetic behind approx_tuple_len](#page-arithmetic-behind-approx_tuple_len)
  - [The proposed metric](#the-proposed-metric)
  - [Step 1 catalog screen](#step-1-catalog-screen)
  - [Step 2 approx measurement](#step-2-approx-measurement)
  - [Step 3 confirmation before a rewrite](#step-3-confirmation-before-a-rewrite)
  - [Test setup](#test-setup)
  - [Cost measured on a 1 GB table](#cost-measured-on-a-1-gb-table)
  - [Accuracy against a full rewrite](#accuracy-against-a-full-rewrite)
  - [False positives fillfactor and unusable page tails](#false-positives-fillfactor-and-unusable-page-tails)
  - [False negatives dropped columns and retained line pointers](#false-negatives-dropped-columns-and-retained-line-pointers)
  - [approx versus exact pgstattuple](#approx-versus-exact-pgstattuple)
  - [approx versus a catalog-only estimate](#approx-versus-a-catalog-only-estimate)
  - [The reltuples dependency](#the-reltuples-dependency)
  - [Snapshots concurrency and privileges](#snapshots-concurrency-and-privileges)
  - [Accepted relation kinds and the TOAST blind spot](#accepted-relation-kinds-and-the-toast-blind-spot)
  - [Settings and apply scope](#settings-and-apply-scope)
  - [Test coverage in the pinned tree](#test-coverage-in-the-pinned-tree)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Propose and test the use of `pgstattuple_approx` to measure table heap bloat.

## Answer

### Verdict

Use `pgstattuple_approx` as the **measurement** stage of a heap-bloat procedure, not as the whole procedure and not as its raw output.

Three findings from the pinned 12.2 tree and from measurements on an isolated server built from it:

1. **The cost argument holds.** On a 1.08 GB table whose pages are all marked all-visible, `pgstattuple_approx` returned in **6.9 ms** reading **42 pages** from disk, while `pgstattuple` took **484.8 ms** and read **137,932** pages. That is a 70x time reduction and a 3,284x physical-read reduction on the same relation.
2. **The raw output is not a bloat number.** `approx_free_percent + dead_tuple_percent` overstated reclaimable space by **31.25 points** on a `fillfactor = 70` table with zero bloat, and by **30.08 points** on a table of 2,832-byte stored rows with zero bloat. A fillfactor correction fixes the first case (error falls to +1.79); nothing fixes the second from this function's output alone.
3. **Once corrected, the estimate is good but biased low.** Against `VACUUM FULL` ground truth over nine fixtures, the corrected estimate landed between **−3.11 and +1.79 points** on the eight ordinary-row-shape fixtures, and the sign of the bias is explained by source: PostgreSQL 12 never shrinks a heap page's line-pointer array, and the free space map rounds free space **down** to a 32-byte category.

`pgstattuple_approx` is also blind to two real forms of table bloat: dropped-column bytes still stored in live tuples (measured: **85.40%** reclaimable, function reported **1.18%**) and TOAST-relation bloat (measured: the function reported 46.15% free on a 1 MB main fork while the table's 156 MB TOAST relation was 59.52% free, and it **refuses** to read that relation at all).

### What the function computes

`statapprox_heap()` walks every block of the main fork once. For each block it takes one of two paths ([pgstatapprox.c#statapprox_heap](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L64-L214)):

| Path | Condition | tuple_len contribution | free_space contribution | dead counters |
|---|---|---|---|---|
| Skip | `VM_ALL_VISIBLE(rel, blkno, &vmbuffer)` | `BLCKSZ - GetRecordedFreeSpace(rel, blkno)` | the free space map value | untouched |
| Scan | otherwise | sum of `t_len` for live tuples | `PageGetHeapFreeSpace(page)` | exact, from the page |

The skip test reads the visibility map through the `VM_ALL_VISIBLE` macro ([visibilitymap.h#VM_ALL_VISIBLE](../../../../raw/postgres-12/src/include/access/visibilitymap.h#L31-L35)), and the free-space value for a skipped page comes from the free space map, not the page ([pgstatapprox.c#VM-skip](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L90-L100)).

On scanned pages the function classifies each line pointer with `HeapTupleSatisfiesVacuum()`, counting `HEAPTUPLE_LIVE` and `HEAPTUPLE_DELETE_IN_PROGRESS` as live and `HEAPTUPLE_DEAD`, `HEAPTUPLE_RECENTLY_DEAD`, and `HEAPTUPLE_INSERT_IN_PROGRESS` as dead ([pgstatapprox.c#tuple-classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L157-L179), [heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1163-L1300)). The horizon is `GetOldestXmin(rel, PROCARRAY_FLAGS_VACUUM)` ([pgstatapprox.c:74](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L74), [procarray.c#GetOldestXmin](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1307), [procarray.h#PROCARRAY_FLAGS_VACUUM](../../../../raw/postgres-12/src/include/storage/procarray.h#L52)).

Live *count* is not measured. It is extrapolated with the same routine VACUUM uses for `pg_class.reltuples` ([pgstatapprox.c#vac_estimate_reltuples-call](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L187-L196), [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1058-L1113)). `table_len` is exact: `nblocks * BLCKSZ` ([pgstatapprox.c:185](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L185)).

The documented contract matches this reading: exact dead-tuple statistics, approximated live count, live length, and free space ([pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L515-L539)).

### Why the skipped-page numbers are trustworthy in v12

The estimate for a skipped page is only as good as its free space map entry. In this pinned tree three facts combine to make that entry a VACUUM-written value rather than a guess:

1. **Only VACUUM sets visibility-map bits.** `visibilitymap_set()` has exactly five call sites in the backend: four in `vacuumlazy.c` and one in the WAL redo routine ([vacuumlazy.c#set-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1288-L1315), [vacuumlazy.c:961](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L961), [vacuumlazy.c:1369](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1369), [vacuumlazy.c:1674](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1674), [heapam.c#heap_xlog_visible](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7937)). There is no `COPY FREEZE` shortcut in v12.
2. **The same VACUUM writes the page's free space map entry**, either directly after the first pass or from the second heap pass ([vacuumlazy.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1380-L1388)).
3. **Every heap write that changes space usage clears the visibility-map bit**, so no row has been added to or removed from a skipped page since that VACUUM ([heapam.c#heap_insert-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1914-L1923), [heapam.c#heap_delete-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2704-L2710), [heapam.c#heap_update-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3669-L3682), [heapam.c#heap_multi_insert-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2208-L2215)). Tuple locking is the exception that proves the rule: it clears only the all-frozen bit and adds no bytes ([heapam.c#heap_lock_tuple-clears-frozen](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L4582-L4586)).

A page with dead line pointers can never be skipped, because `lazy_scan_heap()` clears `all_visible` when it meets an `LP_DEAD` item ([vacuumlazy.c#LP_DEAD-clears-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1019-L1030)). Measured consequence: after `VACUUM (INDEX_CLEANUP FALSE)`, `relallvisible` was 0 of 1,725 pages and `scanned_percent` was 100; a following plain `VACUUM` moved `relallvisible` to 1,725 and `scanned_percent` to 0.

The residual inaccuracy is the free space map's own resolution. `GetRecordedFreeSpace()` returns a category lower bound, and the category step is `BLCKSZ / 256` = 32 bytes ([freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L225-L247), [freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L404-L416), [freespace.c#FSM_CAT_STEP](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L35-L65)). If no free space map fork exists yet, the function returns 0, which makes the page look entirely full ([freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L544-L607)).

### Page arithmetic behind approx_tuple_len

`approx_tuple_len` is **not** the sum of tuple bytes. On a skipped page it is everything that is not free space map free space, which includes the 24-byte page header, 4 bytes per line pointer (used or not), per-tuple `MAXALIGN` padding, and the bytes the free space map rounded away ([pgstatapprox.c:97](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L97), [bufpage.c#PageGetHeapFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L650-L715), [htup_details.h#SizeofHeapTupleHeader](../../../../raw/postgres-12/src/include/access/htup_details.h#L184)).

A 35-page fixture of 133-byte tuples, inspected with same-checkout `pageinspect` and `pg_freespacemap`, reconciled exactly:

| Page state | line pointers | `pd_lower` | `pd_upper` | `PageGetHeapFreeSpace` | free space map | approx charges as tuple_len |
|---|---|---|---|---|---|---|
| full | 58 | 256 | 304 | 44 | 32 | 8,160 |
| emptied then vacuumed | 58 (all `LP_UNUSED`) | 256 | 8,192 | 7,932 | 7,904 | 288 |

Whole-relation check on the same table: predicted `sum(8192 - pg_freespace(blk))` = 123,968 = reported `approx_tuple_len`; predicted `sum(pg_freespace(blk))` = 162,752 = reported `approx_free_space`; exact `pgstattuple` reported `tuple_len` 111,720 and `free_space` 163,500.

The emptied-page row is the important one. `PageRepairFragmentation()` resets only `pd_upper` when a page becomes empty; the line-pointer array stays ([bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L472-L570)). So in v12 a page that once held rows keeps charging `24 + 4 x retained_line_pointers + free-space-map rounding` as occupied space no matter how many rows survive. Two measured instances: 288 bytes per fully emptied page of 133-byte rows, which retains 58 pointers, and 672 bytes per page of 48-byte rows that retains 157 pointers while holding 16 live rows. That is the source of the systematic low bias in the results below, and it grows as rows get narrower.

The three observed free-space-map shortfalls on that fixture were 12, 20, and 28 bytes, all consistent with rounding down to a multiple of 32.

### The proposed metric

Do not report `approx_free_percent` as bloat. Report predicted reclaimable bytes, correcting for fillfactor, which a rewrite honours ([rewriteheap.c#raw_heap_insert](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L684-L693)):

```text
occupied_bytes    = table_len - approx_free_space - dead_tuple_len
target_bytes      = occupied_bytes * 100 / fillfactor
reclaimable_bytes = max(table_len - target_bytes, 0)
reclaimable_pct   = 100 * reclaimable_bytes / table_len
```

`dead_tuple_len` is included because it is exact for every scanned page and zero for skipped pages by construction. It is *removable* space only if no old snapshot pins those row versions; see [Snapshots concurrency and privileges](#snapshots-concurrency-and-privileges).

### Step 1 catalog screen

Run this first. It reads no heap pages, so it is safe to run against every table on the instance. Both settings are `PGC_USERSET`, so `SET` applies for the session or transaction, with no reload or restart ([guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396)).

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

SELECT /* wiki_bloat_screen */
       c.oid::regclass                          AS rel,
       pg_size_pretty(pg_relation_size(c.oid))  AS main_size,
       c.relpages,
       c.relallvisible,
       CASE WHEN c.relpages > 0
            THEN round(100.0 * c.relallvisible / c.relpages, 1) END AS pct_skippable,
       s.n_live_tup,
       s.n_dead_tup,
       s.last_vacuum,
       s.last_autovacuum
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_stat_all_tables s ON s.relid = c.oid
WHERE c.relkind IN ('r', 'm')
  AND c.relpersistence = 'p'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
  AND pg_relation_size(c.oid) > 8 * 1024 * 1024
ORDER BY pg_relation_size(c.oid) DESC
LIMIT 10;

RESET statement_timeout;
RESET lock_timeout;
```

`relallvisible` is written by VACUUM and by ANALYZE from a visibility-map count, so `pct_skippable` predicts how much of the table step 2 will skip ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1192-L1195), [vacuumlazy.c#visibilitymap_count](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L349), [analyze.c#relallvisible](../../../../raw/postgres-12/src/backend/commands/analyze.c#L588-L605), [pg_class.h#relallvisible](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L60-L66)). `n_live_tup` and `n_dead_tup` come from the statistics collector, not the catalog ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)); in the workflow run below one just-analyzed table still reported `n_live_tup = 0` because the collector had not caught up. Treat those two columns as lagging hints and the `pg_class` columns as the reliable ones.

### Step 2 approx measurement

```sql
SET statement_timeout = '5min';
SET lock_timeout = '5s';

WITH cand AS (
    SELECT c.oid,
           coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'fillfactor'), 100) AS fillfactor
    FROM pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE c.relkind IN ('r', 'm')
      AND c.relpersistence = 'p'
      AND c.relam = 2
      AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
      AND pg_relation_size(c.oid) > 8 * 1024 * 1024
    ORDER BY pg_relation_size(c.oid) DESC
    LIMIT 20
)
SELECT /* wiki_bloat_measure_batch */
       cand.oid::regclass AS rel,
       a.scanned_percent,
       pg_size_pretty(a.table_len) AS size,
       round(a.dead_tuple_percent::numeric, 2)  AS dead_pct,
       round(a.approx_free_percent::numeric, 2) AS free_pct,
       cand.fillfactor,
       pg_size_pretty(greatest(a.table_len
              - (a.table_len - a.approx_free_space - a.dead_tuple_len)
                * 100.0 / cand.fillfactor, 0)::bigint) AS reclaimable,
       round(100.0 * greatest(a.table_len
              - (a.table_len - a.approx_free_space - a.dead_tuple_len)
                * 100.0 / cand.fillfactor, 0) / nullif(a.table_len, 0), 2) AS reclaimable_pct
FROM cand, LATERAL pgstattuple_approx(cand.oid) a
ORDER BY 8 DESC NULLS LAST;

RESET statement_timeout;
RESET lock_timeout;
```

`relam = 2` is `HEAP_TABLE_AM_OID`; the function rejects anything else ([pgstatapprox.c#relam-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L292-L294)). `statement_timeout` is effective because the block loop calls `CHECK_FOR_INTERRUPTS()` on every page ([pgstatapprox.c:88](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L88)). The function takes only `AccessShareLock` and releases it at the end of the call ([pgstatapprox.c:268](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L268), [pgstatapprox.c:298](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L298)).

Output from the isolated 12.2 server, on four freshly vacuumed and analyzed fixtures:

| rel | scanned_percent | size | dead_pct | free_pct | fillfactor | reclaimable | reclaimable_pct |
|---|---|---|---|---|---|---|---|
| `d_bloat` | 0 | 67 MB | 0.00 | 58.13 | 100 | 39 MB | 58.13 |
| `d_wide` | 0 | 78 MB | 0.00 | 30.08 | 100 | 23 MB | 30.08 |
| `d_ff70` | 0 | 98 MB | 0.00 | 31.25 | 70 | 1786 kB | 1.79 |
| `d_ok` | 0 | 67 MB | 0.00 | 0.39 | 100 | 272 kB | 0.39 |

### Step 3 confirmation before a rewrite

The same four tables were then rewritten with `VACUUM FULL`:

| rel | before | after | actually reclaimed | predicted with fillfactor correction | predicted raw | error, points |
|---|---|---|---|---|---|---|
| `d_bloat` | 67 MB | 27 MB | 40 MB | 39 MB | 39 MB | −1.87 |
| `d_ff70` | 98 MB | 98 MB | 0 bytes | 1786 kB | 31 MB | +1.79 corrected, +31.25 raw |
| `d_ok` | 67 MB | 67 MB | 0 bytes | 272 kB | 272 kB | +0.39 |
| `d_wide` | 78 MB | 78 MB | 0 bytes | 23 MB | 23 MB | +30.08 |

`d_wide` is the case the function cannot get right on its own, and it is why step 3 exists. Before rewriting a table, check whether its rows are wide enough that the leftover space per page cannot hold another row. `pageinspect` on `d_wide` shows 2 line pointers per page, `lp_len` 2,832, `pd_lower` 32, `pd_upper` 2,528, so 2,492 bytes are free and a third 2,832-byte row does not fit. Compare `approx_free_space / relpages` against the widest plausible row; if one more row does not fit, the free space is not reclaimable.

### Test setup

PostgreSQL 12.2 was built out of tree from `raw/postgres-12` at the pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42` and installed under `.wiki-runtime/`, with `pgstattuple`, `pageinspect`, `pg_freespacemap`, and `pg_visibility` from the same checkout. The server ran isolated on its own port and socket directory with `autovacuum = off`, `shared_buffers = 256MB`, and 8 kB `block_size`, so every VACUUM in the results was explicit. Ground truth for "bloat" is defined as bytes the main fork loses to `VACUUM FULL`: `pg_relation_size(rel, 'main')` before minus after.

The base row shape was `(id int, v int, pad text)` with a 100-character `pad`, giving `t_len` 133, stored length 136 after `MAXALIGN`, and 58 rows per page.

### Cost measured on a 1 GB table

An 8,000,000-row table, 137,932 pages, 1.08 GB, all pages all-visible. Buffer counts are from `EXPLAIN (ANALYZE, BUFFERS)`; the server had just been restarted for the first pair, and the operating system page cache stayed warm throughout, so the read times are a lower bound on the benefit.

| Call | Execution time | shared hit | shared read |
|---|---|---|---|
| `pgstattuple_approx`, cold shared buffers | 6.876 ms | 137,902 | 42 |
| `pgstattuple`, cold shared buffers | 484.833 ms | 137,932 | 137,932 |
| `pgstattuple_approx`, warm | 6.737 ms | 137,937 | 0 |
| `pgstattuple`, warm | 438.713 ms | 137,964 | 137,900 |

The 42 physical reads are the whole point. Rebuilt at the same size, the relation's free space map fork is 36 pages and its visibility map fork is 5 pages, so the cold call read the two auxiliary forks plus a handful of catalog pages instead of 137,932 heap pages. The high *hit* count for `pgstattuple_approx` is real work, not an artefact: `GetRecordedFreeSpace()` pins and releases a free space map buffer once per heap block, so the function is still O(nblocks) in buffer lookups even when it reads almost nothing. Measured, that is roughly 50 ns per skipped page, or about 7 ms per gigabyte.

`pgstattuple` touches each page twice, once through the scan and once for the free-space pass, which is why its hit and read counts are both near `nblocks` ([pgstattuple.c#pgstat_heap](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L365-L396)).

Two more cost points:

- **1% of pages dirty** (1,403 of 137,932 not all-visible): 9.483 ms, `scanned_percent` 1, `dead_tuple_count` 1,379 exact, `approx_tuple_count` 7,999,945 against a true 8,000,000, a 0.0007% error.
- **No visibility map at all**, immediately after `VACUUM FULL`: `pgstattuple_approx` 216.722 ms with 137,900 reads, `pgstattuple` 470.808 ms. Even in its worst case the function is 2.2x faster, because it reads each page once. Note that a freshly rewritten table has no all-visible pages until the next VACUUM, so a monitoring job that runs right after maintenance pays full price.

Both functions request a `BAS_BULKREAD` strategy, a 32-buffer ring, so neither one evicts the working set ([pgstatapprox.c:75](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L75), [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L542-L571)). The `shared hit=32` in the worst-case run is exactly that ring.

### Accuracy against a full rewrite

Nine fixtures, each 500,000 rows unless noted, measured and then rewritten. `approx_raw` is `approx_free_percent + dead_tuple_percent`; `approx_ff` is the proposed metric; `exact_ff` is the same formula fed by `pgstattuple`; `pg_class` is a catalog-only estimate described further below.

| fixture | shape | ground truth | approx_raw | err | approx_ff | err | exact_ff | err | pg_class | err |
|---|---|---|---|---|---|---|---|---|---|---|
| `m_fresh` | no bloat | 0.00 | 0.39 | +0.39 | 0.39 | +0.39 | 0.54 | +0.54 | 0.00 | 0.00 |
| `m_upd50` | 50% non-HOT updates, vacuumed | 33.34 | 32.43 | −0.91 | 32.43 | −0.91 | 32.64 | −0.70 | 33.34 | 0.00 |
| `m_del50` | random 50% deleted, vacuumed | 50.03 | 48.53 | −1.50 | 48.53 | −1.50 | 48.72 | −1.31 | 50.03 | 0.00 |
| `m_del90h` | first 90% deleted, vacuumed | 89.99 | 86.88 | −3.11 | 86.88 | −3.11 | 87.20 | −2.79 | 89.99 | 0.00 |
| `m_dead` | 50% deleted, not vacuumed | 49.99 | 47.62 | −2.37 | 47.62 | −2.37 | 47.62 | −2.37 | 49.99 | 0.00 |
| `m_hot` | 3 rounds of HOT updates, fillfactor 90 | 70.23 | 70.69 | +0.46 | 67.43 | −2.80 | 67.71 | −2.52 | 70.23 | 0.00 |
| `m_ff70` | fillfactor 70, no bloat | 0.00 | 31.25 | **+31.25** | 1.79 | +1.79 | 1.86 | +1.86 | 0.00 | 0.00 |
| `m_ff70b` | fillfactor 70, 50% deleted | 50.00 | 64.45 | **+14.45** | 49.22 | −0.78 | 49.29 | −0.71 | 50.00 | 0.00 |
| `m_wide` | 2,832-byte stored rows, no bloat | 0.00 | 30.08 | **+30.08** | 30.08 | **+30.08** | 30.42 | +30.42 | 0.00 | 0.00 |

Reading the table:

- The fillfactor correction is not optional. It removes a 31.25-point false positive and a 14.45-point overstatement, at the cost of about 2 points of extra under-reporting on `m_hot`.
- Excluding `m_wide`, the corrected estimate spans −3.11 to +1.79 points.
- The corrected estimate is biased low wherever pages were emptied, exactly as the page arithmetic predicts. In block terms on `m_del90h`: ground truth 7,758 blocks, estimate 7,489, shortfall 269 blocks, against 7,758 emptied pages each charged 288 phantom bytes = 273 blocks.
- `m_wide` is not repaired by any correction available from this function's output.

### False positives fillfactor and unusable page tails

Two shapes make free space look reclaimable when it is not.

**Fillfactor.** A `fillfactor = 70` table with zero bloat reported `approx_free_percent` 31.25. `VACUUM FULL` reclaimed nothing, because the rewrite reserves the same fraction ([rewriteheap.c#raw_heap_insert](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L684-L693)). Correcting by the table's own `fillfactor` reloption reduced the error to +1.79.

**Unusable page tails.** With 2,832-byte stored rows only two fit per page; 30.08% of the relation is a permanent remainder. Both `pgstattuple_approx` and `pgstattuple` report it as free space, and both are 30 points wrong about reclaimability. A related, milder case: bimodal row widths, 90% at 44 bytes and 10% at 3,004 bytes, produced an 8.59-point false positive with zero true bloat.

### False negatives dropped columns and retained line pointers

**Dropped columns.** `ALTER TABLE ... DROP COLUMN` leaves the old bytes in existing tuples, but a rewrite reconstructs every tuple against the new descriptor and nulls out dropped attributes ([heapam_handler.c#reform_and_rewrite_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2523-L2548)). Measured on a 200,000-row table after dropping a 200-character column: `VACUUM FULL` went from 6,061 to 885 blocks, **85.40%** reclaimed, while the corrected approx estimate said **1.18%**. The function measures free space, and there was none: the space was inside live tuples.

**Retained line pointers.** Quantified above; it costs 2 to 3 points on delete-heavy tables and more as rows get narrower. The 383-page fixture of 48-byte rows with 90% of rows deleted was rebuilt and inspected page by page: no page was fully emptied, but every page kept its 157 line pointers and `pd_lower` 652 after VACUUM, so 267 pages holding 16 live rows report 1,440 occupied bytes against 768 bytes of live tuples, and 115 pages holding 15 rows report 1,376 against 720. That 672-and-656-byte-per-page surcharge is the whole of the measured 7.12-point shortfall between the 89.82% ground truth and the 82.69% estimate.

**Recently dead rows.** See the snapshot test below: the dead space is reported but is not reclaimable while an old snapshot exists.

### approx versus exact pgstattuple

Run on the identical bloated state of all nine fixtures, before any rewrite:

| fixture | `approx_tuple_percent` | `tuple_percent` | delta | `approx_free_percent` | `free_percent` | delta |
|---|---|---|---|---|---|---|
| `m_fresh` | 99.61 | 94.16 | +5.45 | 0.39 | 0.54 | −0.15 |
| `m_upd50` | 67.57 | 62.77 | +4.80 | 32.43 | 32.64 | −0.21 |
| `m_del50` | 51.47 | 47.04 | +4.43 | 48.53 | 48.72 | −0.19 |
| `m_del90h` | 13.12 | 9.42 | +3.70 | 86.88 | 87.20 | −0.32 |
| `m_dead` | 47.08 | 47.08 | 0.00 | 0.54 | 0.54 | 0.00 |
| `m_hot` | 29.31 | 25.13 | +4.18 | 70.69 | 70.94 | −0.25 |
| `m_ff70` | 68.75 | 64.94 | +3.81 | 31.25 | 31.30 | −0.05 |
| `m_ff70b` | 35.55 | 32.47 | +3.08 | 64.45 | 64.50 | −0.05 |
| `m_wide` | 69.92 | 69.14 | +0.78 | 30.08 | 30.42 | −0.34 |

Three conclusions:

1. **`approx_tuple_percent` is always inflated when pages are skipped**, here by 0.78 to 5.45 points, because it absorbs page headers, line pointers, and alignment padding. Never compare it against `pgstattuple.tuple_percent` and never use it as a density measure. `m_dead` is the control: nothing was all-visible, `scanned_percent` was 100, and the two functions agree to the byte.
2. **`approx_free_percent` is close and always slightly low**, by 0.05 to 0.34 points, the free space map's rounding-down plus the extra `ItemIdData` that `PageGetHeapFreeSpace()` subtracts ([bufpage.c#PageGetFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)).
3. **Live counts matched exactly** on eight of nine fixtures; `m_hot` differed by 18 rows out of 500,000, that is 0.004%.

The two functions also disagree by design on in-progress work. `pgstattuple` scans with `SnapshotAny` and classifies with `SnapshotDirty` ([pgstattuple.c#pgstat_heap](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L336-L352)), and `HeapTupleSatisfiesDirty()` treats another transaction's uncommitted insert as visible ([heapam_visibility.c#in-insertion-by-other](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L834-L853)). `pgstattuple_approx` follows VACUUM and calls it dead. Measured with 20,000 uncommitted inserts open in a second session:

| Function | live | dead | dead_percent |
|---|---|---|---|
| `pgstattuple_approx` | 99,950 | 20,000 | 15.69 |
| `pgstattuple` | 120,000 | 0 | 0.00 |

A bulk load in flight therefore shows up as a 15.69-point phantom "dead tuple" reading on a table with no dead tuples at all.

### approx versus a catalog-only estimate

The catalog-only estimate used here needs no page reads:

```text
est_row_bytes  = 8 * ceil((24 + sum((1 - null_frac) * avg_width)) / 8)
rows_per_page  = floor(((8192 - 24) * fillfactor / 100) / (est_row_bytes + 4))
est_pages      = ceil(reltuples / rows_per_page)
bloat_pct      = 100 * (actual_pages - est_pages) / actual_pages
```

The constants are from source: 24-byte page header and 4-byte line pointer ([bufpage.c#PageGetFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)), `MAXALIGN(SizeofHeapTupleHeader)` = 24 ([htup_details.h#SizeofHeapTupleHeader](../../../../raw/postgres-12/src/include/access/htup_details.h#L184), [htup_details.h#MaxHeapTuplesPerPage](../../../../raw/postgres-12/src/include/access/htup_details.h#L563-L576)).

On the nine well-behaved fixtures above it was **exact on all nine**, error 0.00 everywhere, for zero I/O. That is the honest headline: on a table with fixed-width rows, no NULLs, no dropped columns, and fresh statistics, the catalogs already answer the question and `pgstattuple_approx` adds nothing but confirmation.

Its accuracy is entirely borrowed from `reltuples` and `avg_width`, though. Seven adversarial fixtures, same ground-truth method, with the catalog estimate taken against the *real* current size:

| fixture | shape | ground truth | approx_ff | err | catalog | err |
|---|---|---|---|---|---|---|
| `n_align` | bool/bigint interleaved, alignment padding | 0.00 | 0.41 | +0.41 | 21.32 | **+21.32** |
| `n_noanalyze` | never analyzed, no `pg_stats` rows | 0.00 | 0.41 | +0.41 | −5698.78 | **−5698.78** |
| `n_stale2` | analyzed at 100k rows, grown to 500k | 0.00 | 0.51 | +0.51 | 79.99 | **+79.99** |
| `n_skew` | bimodal row widths | 0.00 | 8.59 | +8.59 | 4.76 | +4.76 |
| `n_stale` | grown then vacuumed, not analyzed | 0.00 | 0.39 | +0.39 | 0.01 | +0.01 |
| `n_drop` | dropped 200-char column | 85.40 | 1.18 | **−84.22** | 85.40 | 0.00 |
| `n_toastptr` | out-of-line TOAST pointers, 90% deleted | 89.82 | 82.69 | −7.12 | 89.82 | 0.00 |

Notes on the failures:

- `n_align` is pure alignment. `pg_stats.avg_width` sums to 27 bytes for `(bool, bigint, bool, bigint, bool, bigint)`, but `pageinspect` reports `lp_len` 72 because each `bool` is followed by 7 bytes of padding, and only 107 rows fit per page. The estimate predicted 2,942 pages for a perfectly packed 3,739-page table, while `pgstattuple_approx` reported 0.41% free.
- `n_noanalyze` has no `pg_stats` rows at all, and the `greatest(floor(...), 1)` guard in the formula above silently produced −5698.78% because SQL `GREATEST` ignores NULLs and the clamp fired. A production version of a catalog estimator must test for missing statistics explicitly rather than clamping.
- `n_stale2` shows the difference between the two denominators. Against the stale `relpages` the estimate reports 0.00%, because stale row counts and stale page counts cancel. Against the real 8,621-block size it reports 79.99% bloat on a table with none.
- `n_stale` is a negative result worth recording: a plain `VACUUM` refreshes `reltuples` and `relpages` ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1195)), so a vacuumed table does not have stale row counts even without `ANALYZE`.
- `n_toastptr` confirms a source reading: ANALYZE records the *stored* width of a varlena, `VARSIZE_ANY` of the on-disk pointer, not the logical value ([analyze.c#compute_scalar_stats-width](../../../../raw/postgres-12/src/backend/commands/analyze.c#L2225-L2246)). A column of out-of-line values therefore contributes about 18 bytes to `avg_width`, and the catalog estimate handles TOAST pointers correctly.

The division of labour that follows from these two tables:

| Situation | Use |
|---|---|
| Fixed-width rows, fresh `ANALYZE`, no dropped columns | catalogs; `pgstattuple_approx` only to confirm a candidate |
| Missing or stale statistics, alignment-heavy or variable rows | `pgstattuple_approx` |
| Dropped columns | catalogs, or a rewrite test on a copy; `pgstattuple_approx` cannot see it |
| Wide rows with unusable page tails | neither, without a row-width check |
| Final go/no-go before a rewrite window | `pgstattuple` on the candidate, then the rewrite |

### The reltuples dependency

`approx_tuple_count` is an estimate keyed to `pg_class`, and when every page is skipped it is not an estimate at all: `vac_estimate_reltuples()` returns the stored `reltuples` unchanged when `scanned_pages` is 0 ([vacuum.c#scanned_pages-zero](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1087-L1094)). Measured on a 1,725-page, 100,000-row table, all all-visible, with `pg_class.reltuples` forged to 7:

| state | reltuples | scanned_percent | `approx_tuple_count` | `pgstattuple.tuple_count` |
|---|---|---|---|---|
| honest | 100,000 | 0 | 100,000 | 100,000 |
| forged to 7 | 7 | 0 | **7** | 100,000 |
| forged to 7, one page dirtied | 7 | 0 | 73 | 100,000 |

The third row is the blend at [vacuum.c#density-blend](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1103-L1112): 2 pages were not all-visible and carried 66 live tuples, and `floor(7 / 1725 * 1723 + 66 + 0.5)` = 73. The bloat metric proposed here deliberately uses no tuple count, so it is unaffected; anything that consumes `approx_tuple_count` inherits the quality of `reltuples`.

`scanned_percent` is also less informative than it looks. It is computed as an integer division into a `uint64` field ([pgstatapprox.c#scanned_percent](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L198-L207)), so it truncates: with 1 page of 1,725 scanned it reported **0**, not 0.058, and the documentation's own example shows the integer `2` ([pgstattuple.sgml#example](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L498-L511)). A monitoring job cannot distinguish "skipped everything" from "scanned up to 1% of the table" using this column alone; compare `relallvisible` against `relpages` instead.

### Snapshots concurrency and privileges

**Dead is not the same as reclaimable.** With a repeatable-read snapshot held open in a second session, 50,000 of 100,000 rows deleted and vacuumed:

| step | result |
|---|---|
| `pgstattuple_approx` | `dead_tuple_percent` 47.06, `approx_free_percent` 0.59, signal 47.65 |
| `pgstattuple` | identical, 47.06 and 0.59 |
| `VACUUM FULL` with the snapshot open | 1,725 blocks before, 1,725 after, nothing reclaimed |
| `VACUUM FULL` after `COMMIT` | 863 blocks, 50% reclaimed |

Both functions correctly reported dead rows; neither can tell you whether they are removable. Pair the measurement with the transaction horizon before scheduling a rewrite.

**Locking and horizon impact.** `pgstattuple_approx` holds `AccessShareLock` for the duration and runs inside an ordinary snapshot, so a long call holds back the xmin horizon just as any long query does. That is a second, indirect argument for the approximate function: 6.9 ms instead of 485 ms per gigabyte is 70x less time spent pinning the horizon on a monitoring pass.

**Privileges.** Version 1.5 revokes `EXECUTE` from `PUBLIC` and grants it to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql#approx-grants](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119), [pgstattuple.control:1](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5)). The C function keeps a superuser check only in the pre-1.5 entry point ([pgstatapprox.c#superuser-gate](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L216-L234), [pgstatapprox.c#v1_5-entry](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L236-L249)). Measured `proacl` on the fresh install was `{postgres=X/postgres,pg_stat_scan_tables=X/postgres}`, matching the documented policy ([pgstattuple.sgml#privileges](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L15-L24)).

There is no table-level permission check in the function. A test role that was denied `SELECT` on a table still received full page statistics for it after being granted `pg_stat_scan_tables`:

| role state | `pgstattuple_approx('e_priv')` | `SELECT count(*) FROM e_priv` |
|---|---|---|
| plain login role | `ERROR: permission denied for function pgstattuple_approx` | `ERROR: permission denied for table e_priv` |
| after `GRANT pg_stat_scan_tables` | succeeded, 1,000 rows, 8.93% free | `ERROR: permission denied for table e_priv` |

Grant the monitoring role membership in `pg_stat_scan_tables` knowingly: it exposes sizes, row counts, and dead-row counts for every table in the database, though not row contents.

### Accepted relation kinds and the TOAST blind spot

`pgstattuple_approx` accepts only `RELKIND_RELATION` and `RELKIND_MATVIEW`, and only the heap access method ([pgstatapprox.c#relkind-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L294)). Measured on the pinned build:

| target | result |
|---|---|
| ordinary table, materialized view | works |
| empty table, 0 pages | all output columns 0 |
| partitioned table, view, index, sequence | `ERROR: "..." is not a table or materialized view` |
| TOAST relation | `ERROR: "pg_toast_16554" is not a table or materialized view` |
| another session's temporary table | `ERROR: cannot access temporary tables of other sessions` |
| sequence, through exact `pgstattuple` | `ERROR: only heap AM is supported` |

The last row is a v12 quirk worth knowing if you fall back to the exact function: `pgstat_relation()` lists `RELKIND_SEQUENCE` among the accepted kinds, but sequences have `relam = 0` in this version and `pgstat_heap()` rejects them ([pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L255-L306), [pgstattuple.c#relam-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L331-L334)).

The TOAST refusal matters for the original question. A 20,000-row table with a 6,400-byte incompressible column stored 1,048,576 bytes in its main fork and 163,840,000 bytes in its TOAST relation. After deleting half the rows and vacuuming, both forks were still the same size and the freed space sat inside them:

| relation | size | free_percent |
|---|---|---|
| main fork, via `pgstattuple_approx` | 1 MB | 46.15 |
| TOAST relation, via `pgstattuple` | 156 MB | 59.52 |

`pgstattuple_approx` reported 46.15% free on a 1 MB fork while a 156 MB relation that was 59.52% free sat behind a pointer column it will not follow. Any bloat procedure built on this function must add a TOAST pass using the exact function, which does accept `RELKIND_TOASTVALUE`:

```sql
SET statement_timeout = '5min';
SET lock_timeout = '5s';

SELECT /* wiki_bloat_toast */
       c.oid::regclass                          AS parent,
       t.relname                                AS toast_rel,
       pg_size_pretty(pg_relation_size(t.oid))  AS toast_size,
       round(s.free_percent::numeric, 2)        AS toast_free_pct,
       round(s.dead_tuple_percent::numeric, 2)  AS toast_dead_pct
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
CROSS JOIN LATERAL pgstattuple(t.oid) s
WHERE c.oid = 'public.my_table'::regclass;

RESET statement_timeout;
RESET lock_timeout;
```

That query is a full scan of the TOAST relation. Run it only on candidates the screen has already flagged.

### Settings and apply scope

| Setting | Role here | Context | Apply scope |
|---|---|---|---|
| `statement_timeout` | bounds a call that has to scan many non-all-visible pages | `PGC_USERSET` | session or transaction, no reload |
| `lock_timeout` | bounds waiting for `AccessShareLock` behind DDL | `PGC_USERSET` | session or transaction, no reload |
| `autovacuum` and its thresholds | decide how much of the table is skippable at all | not exercised here | see the VACUUM pages |

Both timeouts are defined at [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396). There is no cost-delay control for this function: it is not VACUUM and never sleeps. `vacuum_cost_delay` has no effect on it.

Note also what the *table* setting does. `fillfactor` is the only reloption the proposed metric reads, and it must be read per table rather than assumed, because the rewrite honours it.

### Test coverage in the pinned tree

The regression suite exercises `pgstattuple_approx` four times, and never for accuracy ([pgstattuple.sql#unsupported-relations](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L65-L101)):

| Call | Purpose |
|---|---|
| `pgstattuple_approx('test_partitioned')` | expected error |
| `pgstattuple_approx('test_view')` | expected error |
| `pgstattuple_approx('test_foreign_table')` | expected error |
| `pgstattuple_approx('test_partition')` | empty partition, all-zero output ([pgstattuple.out#empty-partition](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L205-L213)) |

There is no upstream test in this checkout that compares `pgstattuple_approx` against `pgstattuple`, exercises the visibility-map skip path, or covers the free space map estimate. The module builds one shared library from three source files and registers one regression script ([Makefile#pgstattuple](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13)).

## Context Reviewed

- `contrib/pgstattuple`: `pgstatapprox.c`, `pgstattuple.c`, the 1.4 and 1.4-to-1.5 SQL scripts, the control file, the Makefile, the regression script, and its expected output.
- Free space map: `GetRecordedFreeSpace()`, `fsm_readbuf()`, the category functions, and the `FSM_CAT_STEP` and `MaxFSMRequestSize` definitions.
- Visibility map: the `VM_ALL_VISIBLE` macro and every `visibilitymap_set()` and `visibilitymap_clear()` call site in the backend.
- VACUUM: `lazy_scan_heap()` page classification, all-visible decision, free-space recording, `INDEX_CLEANUP` handling, `vac_estimate_reltuples()`, and `vac_update_relstats()`.
- Heap page layout: `PageGetFreeSpace()`, `PageGetHeapFreeSpace()`, `PageRepairFragmentation()`, and the `htup_details.h` size macros.
- Rewrite path: `raw_heap_insert()` fillfactor handling and `reform_and_rewrite_tuple()`.
- Statistics: `analyze.c` width accounting and `relallvisible` maintenance, `pg_class` columns, and the `pg_stat_all_tables` view definition.
- Visibility rules: `HeapTupleSatisfiesVacuum()` and `HeapTupleSatisfiesDirty()`.
- Documentation: the `pgstattuple` chapter, including the privileges note and the `pgstattuple_approx` column table.
- Packaged extensions used as instruments from the same checkout: `pageinspect`, `pg_freespacemap`, `pg_visibility`.

## Evidence Map

| Claim | Evidence |
|---|---|
| Skipped pages take free space from the free space map and charge the rest as tuple length | [pgstatapprox.c#VM-skip](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L90-L100) |
| Scanned pages produce exact dead statistics | [pgstatapprox.c#tuple-classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L157-L179) |
| Live count comes from `vac_estimate_reltuples()` | [pgstatapprox.c#vac_estimate_reltuples-call](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L187-L196), [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1058-L1113) |
| All-visible pages cannot have been written since the VACUUM that recorded their free space | [heapam.c#heap_insert-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1914-L1923), [vacuumlazy.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1380-L1388) |
| Free space map values round down to 32-byte categories | [freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L404-L416), [freespace.c#FSM_CAT_STEP](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L35-L65) |
| Emptied heap pages keep their line-pointer array in v12 | [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L472-L570) |
| Pages with dead line pointers are never all-visible | [vacuumlazy.c#LP_DEAD-clears-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1019-L1030) |
| A rewrite honours `fillfactor` | [rewriteheap.c#raw_heap_insert](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L684-L693) |
| A rewrite removes dropped-column bytes | [heapam_handler.c#reform_and_rewrite_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2523-L2548) |
| Exact `pgstattuple` counts another session's uncommitted insert as live | [pgstattuple.c#pgstat_heap](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L336-L352), [heapam_visibility.c#in-insertion-by-other](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L834-L853) |
| `scanned_percent` truncates to an integer | [pgstatapprox.c#scanned_percent](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L198-L207), [pgstattuple.sgml#example](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L498-L511) |
| Only tables and materialized views on the heap AM are accepted | [pgstatapprox.c#relkind-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L294) |
| Execution is granted to `pg_stat_scan_tables`, not `PUBLIC` | [pgstattuple--1.4--1.5.sql#approx-grants](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119) |
| `ANALYZE` records the stored width of a varlena, not the logical width | [analyze.c#compute_scalar_stats-width](../../../../raw/postgres-12/src/backend/commands/analyze.c#L2225-L2246) |
| Both timeout settings apply at session or transaction scope | [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396) |
| Upstream tests cover only error cases and one empty partition | [pgstattuple.sql#unsupported-relations](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L65-L101), [pgstattuple.out#empty-partition](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L205-L213) |

## Open Questions

- **`scanned_percent` arithmetic on very large tables.** The expression is `100 * scanned / nblocks` where `scanned` is a `BlockNumber`, so the multiplication is performed in 32-bit unsigned arithmetic before the result is stored in the `uint64` field ([pgstatapprox.c#output_type](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52), [pgstatapprox.c#scanned_percent](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L198-L207)). By that reading it wraps once `scanned` reaches 42,949,673 blocks, about 328 GiB scanned. This was not tested: the fixtures here top out at 1.08 GB.
- **Which condition left 4 pages non-all-visible after a bulk load.** The first `VACUUM` of the 137,932-page table left 4 ordinary, full pages not all-visible and reported "Skipped 0 pages due to buffer pins"; a second `VACUUM` cleared them. The pin-skip path is therefore ruled out ([vacuumlazy.c#pin-skip](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L863-L878)), leaving the `HeapTupleHeaderXminCommitted` hint-bit check as the likely cause ([vacuumlazy.c#xmin-committed](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1097-L1130)), but this was not proven directly.
- **Standby behaviour.** The redo path maintains the standby's free space map for cleanup and visibility records ([heapam.c#heap_xlog_clean-fsm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7786-L7799), [heapam.c#heap_xlog_visible-fsm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7889-L7897)), which suggests running the measurement on a replica is viable. No standby was built for this page, so the accuracy of the estimate on a replica is untested.
- **Threshold calibration.** The results support "the corrected estimate is within about 3 points on ordinary row shapes", but no alerting threshold was calibrated against a real workload. The wide-row false positive means any threshold must be paired with a row-width check.
- **Concurrent DML during the call.** All accuracy fixtures were measured on a quiescent table. The behaviour of the estimate while heavy DML runs concurrently, where visibility-map bits are being cleared during the walk, was not measured.
- **`pgstattuple_approx` versus the free space map after a crash.** Free space map updates are not fully WAL-logged ([freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L544-L607)), so a crash could leave skipped-page estimates stale. Not tested.

## Source References

- [pgstatapprox.c#output_type](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52)
- [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L64-L214)
- [pgstatapprox.c#VM-skip](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L90-L100)
- [pgstatapprox.c#new-and-empty-pages](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L109-L125)
- [pgstatapprox.c#tuple-classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L157-L179)
- [pgstatapprox.c#vac_estimate_reltuples-call](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L187-L196)
- [pgstatapprox.c#scanned_percent](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L198-L207)
- [pgstatapprox.c#superuser-gate](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L216-L234)
- [pgstatapprox.c#v1_5-entry](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L236-L249)
- [pgstatapprox.c#relkind-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L294)
- [pgstatapprox.c#result-columns](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L300-L311)
- [pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L255-L306)
- [pgstattuple.c#pgstat_heap](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L318-L404)
- [pgstattuple.c#relam-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L331-L334)
- [pgstattuple.c#build_pgstattuple_type](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L89-L151)
- [pgstattuple--1.4.sql#approx-signature](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L83-L95)
- [pgstattuple--1.4--1.5.sql#approx-grants](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119)
- [pgstattuple.control:1](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [Makefile#pgstattuple](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13)
- [pgstattuple.sql#unsupported-relations](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L65-L101)
- [pgstattuple.out#empty-partition](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L205-L213)
- [pgstattuple.sgml#privileges](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L15-L24)
- [pgstattuple.sgml#example](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L498-L511)
- [pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L515-L539)
- [pgstattuple.sgml#approx-columns](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L541-L613)
- [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1058-L1113)
- [vacuum.c#scanned_pages-zero](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1087-L1094)
- [vacuum.c#density-blend](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1103-L1112)
- [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1116-L1195)
- [vacuumlazy.c#pin-skip](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L863-L878)
- [vacuumlazy.c#new-pages](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L885-L928)
- [vacuumlazy.c#empty-pages](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L930-L970)
- [vacuumlazy.c#LP_DEAD-clears-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1019-L1030)
- [vacuumlazy.c#xmin-committed](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1097-L1130)
- [vacuumlazy.c#index-cleanup-disabled](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1238-L1265)
- [vacuumlazy.c#set-all-visible](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1288-L1315)
- [vacuumlazy.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1380-L1388)
- [vacuumlazy.c#new_live_tuples](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1400-L1404)
- [vacuumlazy.c#visibilitymap_count](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L349)
- [freespace.c#FSM_CAT_STEP](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L35-L65)
- [freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L225-L247)
- [freespace.c#fsm_space_avail_to_cat](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L379-L402)
- [freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L404-L416)
- [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L544-L607)
- [visibilitymap.h#VM_ALL_VISIBLE](../../../../raw/postgres-12/src/include/access/visibilitymap.h#L31-L35)
- [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L472-L570)
- [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)
- [bufpage.c#PageGetHeapFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L650-L715)
- [htup_details.h#SizeofHeapTupleHeader](../../../../raw/postgres-12/src/include/access/htup_details.h#L184)
- [htup_details.h#MaxHeapTupleSize](../../../../raw/postgres-12/src/include/access/htup_details.h#L550-L560)
- [htup_details.h#MaxHeapTuplesPerPage](../../../../raw/postgres-12/src/include/access/htup_details.h#L563-L576)
- [heapam.c#heap_insert-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1914-L1923)
- [heapam.c#heap_multi_insert-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2208-L2215)
- [heapam.c#heap_delete-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2704-L2710)
- [heapam.c#heap_update-clears-vm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3669-L3682)
- [heapam.c#heap_lock_tuple-clears-frozen](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L4582-L4586)
- [heapam.c#heap_xlog_clean-fsm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7786-L7799)
- [heapam.c#heap_xlog_visible-fsm](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7889-L7897)
- [heapam.c#heap_xlog_visible](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7937)
- [heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1163-L1300)
- [heapam_visibility.c#in-insertion-by-other](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L834-L853)
- [heapam_handler.c#reform_and_rewrite_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2523-L2548)
- [rewriteheap.c#raw_heap_insert](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L684-L693)
- [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L542-L571)
- [procarray.c#GetOldestXmin](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1307)
- [procarray.h#PROCARRAY_FLAGS_VACUUM](../../../../raw/postgres-12/src/include/storage/procarray.h#L52)
- [analyze.c#relallvisible](../../../../raw/postgres-12/src/backend/commands/analyze.c#L588-L605)
- [analyze.c#compute_scalar_stats-width](../../../../raw/postgres-12/src/backend/commands/analyze.c#L2225-L2246)
- [pg_class.h#relallvisible](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L60-L66)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)
- [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396)

## Navigation

- [v12/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Detecting Index Bloat with COMMENT-Stored Bytes per Tuple in PostgreSQL 12 (unverified)](../indexing/comment-stored-bytes-per-tuple-bloat.md)
- [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](../indexing/btree-index-bloat-core-sql-only.md)
- [PostgreSQL 12 Database Health Checklist (unverified)](../server-administration/database-health-checklist.md)
