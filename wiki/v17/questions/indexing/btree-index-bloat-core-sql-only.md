---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: not yet
---

# Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [How the test was run](#how-the-test-was-run)
  - [The fifteen fixtures on 17.10](#the-fifteen-fixtures-on-1710)
  - [Method A against a rebuild](#method-a-against-a-rebuild)
  - [Deduplication is what breaks the model](#deduplication-is-what-breaks-the-model)
  - [The duplication-ratio sweep](#the-duplication-ratio-sweep)
  - [NULL runs are deduplicated too](#null-runs-are-deduplicated-too)
  - [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17)
  - [Method A-prime still fixes variable key width](#method-a-prime-still-fixes-variable-key-width)
  - [Method B: leaf counts still exact, density formula not](#method-b-leaf-counts-still-exact-density-formula-not)
  - [Method C: unchanged answer, different write path](#method-c-unchanged-answer-different-write-path)
  - [Method D: new message, no row count, new blind spot](#method-d-new-message-no-row-count-new-blind-spot)
  - [The v14 unknown reltuples sentinel](#the-v14-unknown-reltuples-sentinel)
  - [Partial indexes and the statistics state](#partial-indexes-and-the-statistics-state)
  - [The 54-cell matrix](#the-54-cell-matrix)
  - [Accuracy on 17 against the v12 page's reported accuracy](#accuracy-on-17-against-the-v12-pages-reported-accuracy)
  - [What to change before running the v12 SQL on 17](#what-to-change-before-running-the-v12-sql-on-17)
  - [Cost of each method on this server](#cost-of-each-method-on-this-server)
  - [Settings and apply scopes](#settings-and-apply-scopes)
  - [What no core-SQL method can measure on v17](#what-no-core-sql-method-can-measure-on-v17)
  - [Follow-up: the same sweep on a 12.2 server and a 17.10 server](#follow-up-the-same-sweep-on-a-122-server-and-a-1710-server)
  - [Follow-up: the INCLUDE-column false positive on v17](#follow-up-the-include-column-false-positive-on-v17)
  - [Follow-up: the v12 hazard the reltuples guard does not cover](#follow-up-the-v12-hazard-the-reltuples-guard-does-not-cover)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17: test the SQL from the PostgreSQL 12 question "Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)" on PostgreSQL 17 and compare whether it measures bloat with the same accuracy as in version 12.

Follow-up: does the deduplication-aware sweep for v17 work for indexes from v12 and v17?

> Prompt note: the follow-up is filed as an approved grammar-corrected restatement
> of the original wording ("does A deduplication-aware sweep for v17 works for
> indexes from v12 and v7?"), per the repository's prompt-hygiene rule. The asker
> confirmed that "v7" meant v17, and that "indexes from v12" means indexes on a
> live PostgreSQL 12 server: does this statement parse, run, and stay correct when
> it is pointed at a 12 server. Indexes carried onto a 17 server by `pg_upgrade`
> are out of scope; see [Open Questions](#open-questions).

## Answer

### Verdict

Every statement in the v12 page runs unchanged on 17.10, and on indexes with distinct keys it is **exactly as accurate**: Method A hit the rebuilt block count to the block on 9 of 14 fixtures and on all 24 non-partial matrix cells outside the duplicate-key type, and Method B reproduced `pgstatindex.leaf_pages` exactly on all 12 eligible fixtures and all 36 eligible matrix cells — the same outcome the v12 page reports for v12.

One v13 feature breaks it. B-tree **deduplication** merges equal keys into posting-list tuples, both when a page is about to split and when a fresh index is built ([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1160)). The v12 page-fill model charges one full index tuple per row, so wherever duplicates exist it overestimates the rebuilt size, and the error scales with duplication:

| Symptom | v17 measurement |
|---|---|
| Method A on an all-duplicate 1,000,000-row index | model 2745 blocks, rebuild 849 blocks — **+223.3%** |
| Method A on 5 rows per key | model 2745, rebuild 1426 — **+92.5%**, on an index with 0% reclaimable |
| Method A on 25% NULL keys | model 2745, rebuild 2271 — **+20.9%** |
| Method B density formula on the same cells | 313.58% against `pgstatindex`'s 96.15% — **+217 points** |

Three smaller v17 differences also matter: `VACUUM VERBOSE` no longer prints an index row count and can skip the index line entirely (Method D), `pg_class.reltuples` now uses `-1` for "unknown" and turns a healthy 21 MB index into a 100.0% bloat reading, and a plain `ANALYZE` after the build costs partial-index cells their exactness. The [deduplication-aware sweep below](#a-deduplication-aware-sweep-for-v17) restores the worst case from +1896 blocks to −24 blocks (2.9%) without changing a single already-exact cell.

That corrected sweep is also safe to run against a PostgreSQL 12 server, where it silently reduces to the v12 page's own Method A; the follow-up sections measure it on both a 12.2 and a 17.10 server and name the one case where it is wrong on 17.10 — see [Follow-up: the same sweep on a 12.2 server and a 17.10 server](#follow-up-the-same-sweep-on-a-122-server-and-a-1710-server).

### How the test was run

One isolated PostgreSQL 17.10 server, built out of tree from the pinned checkout under `.wiki-runtime/`, `block_size` 8192, `autovacuum = off`, `fsync = off`. `pgstattuple` was installed **only as ground truth**; no method below uses it.

Two populations, both taken from the v12 page's own descriptions:

- the **15 named fixtures** (`idx_seq` … `idx_empty`), each rebuilt from the recipe in the v12 page's fixture table;
- the **54-cell matrix**: 9 bloat types x 3 scales (200,000 / 500,000 / 1,000,000 rows) x {non-partial, partial}, two indexes per table over the same key, `flag = (id % 5 = 0)` so the partial index holds 20% of the rows, delete patterns on modulus 7 and 11.

Ground truth per index is a `CREATE INDEX CONCURRENTLY` rebuild (Method C, exact reclaimable size) plus `pgstatindex` page classes. The v12 page's SQL was executed verbatim; only the `actual_bytes > 1024 * 1024` triage filter was removed so that sub-megabyte partial indexes are scored, and `expected_blocks` was exposed so the model can be diffed against the rebuild.

The recipes are reconstructions. Where a v17 number differs from the v12 page's, the mechanism is named below and checked against v17 source; where the difference is a fixture artifact rather than a version change, it is called out as one.

### The fifteen fixtures on 17.10

`pgstatindex` ground truth, with the v12 page's reported figures beside it:

| index | v17 blocks | v17 leaf | v17 internal | v17 deleted | v17 density | v12 page blocks / density | same? |
|---|---|---|---|---|---|---|---|
| `idx_seq` | 2745 | 2733 | 11 | 0 | 90.06 | 2745 / 90.06 | identical |
| `idx_uuid` | 1543 | 1533 | 9 | 0 | 90.01 | 1543 / 90.01 | identical |
| `idx_text` | 3607 | 3572 | 34 | 0 | 89.98 | 3607 / 89.98 | identical |
| `idx_ff50` | 4971 | 4951 | 19 | 0 | 49.85 | 4971 / 49.85 | identical |
| `idx_part` | 139 | 137 | 1 | 0 | 89.83 | 139 / 89.83 | identical |
| `idx_del` | 2745 | 2733 | 11 | 0 | 9.27 | 2745 / 9.27 | identical |
| `idx_range` | 2745 | 411 | 3 | 2330 | 89.83 | 2745 / 89.83 | identical |
| `idx_stale` | 2745 | 2733 | 11 | 0 | 90.06 | 2745 / 90.06 | identical |
| `idx_empty` | 1 | 0 | 0 | 0 | `NaN` | 1 / `NaN` | identical |
| `idx_multi` | 3587 | 3572 | 14 | 0 | 89.58 | 3606 / 89.96 | fixture artifact |
| `idx_var` | 3211 | 3171 | 39 | 0 | 90.32 | 3169 / 90.43 | fixture artifact |
| `idx_rand` | 3788 | 3773 | 14 | 0 | 65.32 | 3758 / 65.81 | different seed |
| `idx_null` | 2271 | 2260 | 10 | 0 | 90.07 | 2746 / 90.09 | **v17 deduplication** |
| `idx_dup` | 396 | 392 | 3 | 0 | 96.12 | 1291 / 96.00 | **v17 deduplication** |
| `idx_churn` | 2471 | 2460 | 10 | 0 | 74.80 | 3293 / 67.63 | different update pattern |

Nine of the fifteen reproduce the v12 page's block count to the block, which is the control this comparison needs: the page-fill arithmetic itself did not move. `_bt_pagestate` still starts a leaf "full" threshold at `BLCKSZ * (100 - fillfactor) / 100` and internal levels at `BTREE_NONLEAF_FILLFACTOR` 70 ([nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)), `_bt_blnewpage` still pre-reserves the high-key line pointer ([nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629)), and a page is still finished when `PageGetFreeSpace()` drops below that threshold ([nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L809-L855)). The usable-space constant is unchanged as well ([nbtsplitloc.c:157-160](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L157-L160)).

`idx_multi`'s 14 internal pages against the v12 page's 33 is a fixture difference, not a version difference: this run's leading `int` column is unique, so `_bt_truncate` strips the trailing `text` attribute from every pivot tuple and the internal fanout roughly triples. The leaf count, 3572, is identical.

### Method A against a rebuild

Method A run verbatim, scored against the Method C rebuild:

| index | live | rebuilt | Method A | model − rebuilt | model bloat % | true bloat % |
|---|---|---|---|---|---|---|
| `idx_seq` | 2745 | 2745 | 2745 | 0 | 0.0 | 0.0 |
| `idx_uuid` | 1543 | 1543 | 1543 | 0 | 0.0 | 0.0 |
| `idx_text` | 3607 | 3607 | 3607 | 0 | 0.0 | 0.0 |
| `idx_ff50` | 4971 | 4971 | 4971 | 0 | 0.0 | 0.0 |
| `idx_empty` | 1 | 1 | 1 | 0 | 0.0 | 0.0 |
| `idx_rand` | 3788 | 2745 | 2745 | 0 | 27.5 | 27.5 |
| `idx_churn` | 2471 | 825 | 825 | 0 | 66.6 | 66.6 |
| `idx_range` | 2745 | 414 | 414 | 0 | 84.9 | 84.9 |
| `idx_del` | 2745 | 276 | 276 | 0 | 89.9 | 89.9 |
| `idx_stale` | 2745 | 276 | 276 | 0 | 89.9 | 89.9 |
| `idx_part` | 139 | 139 | 135 | −4 | 2.9 | 0.0 |
| `idx_multi` | 3587 | 3587 | 3607 | +20 | −0.6 | 0.0 |
| `idx_var` | 3211 | 3211 | 3316 | +105 | −3.3 | 0.0 |
| `idx_null` | 2271 | 2271 | 2745 | **+474** | −20.9 | 0.0 |
| `idx_dup` | 396 | 426 | 1374 | **+948** | −247.0 | −7.6 |

Nine cells exact, three inside the same error classes the v12 page documents (partial-index sampling, internal-page modelling, variable key width), and two that are new on 17. Both new failures are deduplication.

`idx_dup` also reproduces the v12 page's "a rebuild can make an index bigger" result, with different numbers: the live index sits at 96.12% density because an all-duplicate split uses `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416)), while the sorted rebuild caps each posting list at a tenth of a page, so rebuilding grew it from 396 to 426 blocks (−7.6%).

### Deduplication is what breaks the model

Two independent code paths merge equal keys into a single posting-list tuple that stores one copy of the key plus a 6-byte `ItemPointerData` per row.

**Build path.** `_bt_load` turns deduplication on for any non-unique index whose opclass is all-equal-image and whose `deduplicate_items` reloption is on, which is the default ([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1160), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1150)). Posting lists built this way are capped at `MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)`, i.e. 812 bytes at the default block size ([nbtsort.c:1292-1308](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1292-L1308)), and a TID is appended only while `MAXALIGN(basetupsize + nhtids * sizeof(ItemPointerData))` stays under that cap ([nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L484-L531)). For an 8-byte key that is 132 TIDs in an 808-byte tuple.

**Insert path.** A page about to split is deduplicated first, with a larger cap of `BTMaxItemSize(page) / 2` ([nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L75-L93)).

Because Method A charges `MAXALIGN(hoff + data) + 4` per **row**, it cannot see either. Measured on the 54-cell matrix, the `dup` rows are the only ones that move:

| cell | live | rebuilt | Method A | error |
|---|---|---|---|---|
| `m_dup_200_full` | 159 | 171 | 551 | +380 (+222.2%) |
| `m_dup_500_full` | 396 | 426 | 1374 | +948 (+222.5%) |
| `m_dup_1000_full` | 789 | 849 | 2745 | +1896 (+223.3%) |
| `m_dup_200_part` | 34 | 36 | 111 | +75 (+208.3%) |
| `m_dup_500_part` | 81 | 87 | 279 | +192 (+220.7%) |
| `m_dup_1000_part` | 159 | 171 | 557 | +386 (+225.7%) |

The v12 page reports these same six cells at 4 blocks / 0.2% and 2 blocks / 0.8%.

### The duplication-ratio sweep

Duplication is not binary, so a second exact-pin fixture set fixed the row count at 1,000,000 and varied only the rows per distinct key (`id = g / q`), each with `VACUUM (ANALYZE)` and a `CREATE INDEX CONCURRENTLY` rebuild:

| rows per key | `pg_stats.n_distinct` | live | rebuilt | true bloat | Method A | Method A error | dedup-aware model | its error |
|---|---|---|---|---|---|---|---|---|
| 2 | −0.509185 | 2475 | 2475 | 0.0% | 2745 | +270 (+10.9%) | 2745 | +270 |
| 5 | −0.208039 | 1426 | 1426 | 0.0% | 2745 | +1319 (+92.5%) | 2745 | +1319 |
| 10 | 97311 | 1157 | 1157 | 0.0% | 2745 | +1588 (+137.3%) | 1012 | −145 (−12.5%) |
| 20 | 50492 | 950 | 950 | 0.0% | 2745 | +1795 (+188.9%) | 922 | −28 (−2.9%) |
| 100 | 9991 | 839 | 839 | 0.0% | 2745 | +1906 (+227.2%) | 844 | +5 (+0.6%) |
| 1000 | 1000 | 896 | 896 | 0.0% | 2745 | +1849 (+206.4%) | 827 | −69 (−7.7%) |

Every one of these indexes is freshly built and has **zero** reclaimable space. The v12 sweep reports 10.9% to 227.2% phantom bloat on all six, and its answer never changes, because the model has no term that depends on duplication. At 5 rows per key it claims 92.5% of a 22 MB index is wasted when a rebuild reclaims nothing.

### NULL runs are deduplicated too

`idx_null` (1,000,000 `bigint`, 25% NULL) is 2271 blocks on 17.10 against 2745 predicted. NULLs are ordinary index entries and they are all equal to each other, so the 250,000 NULL rows collapse into posting lists exactly like any other duplicate run. This is worth separating from the `dup` case because `null_frac` is a directly measured statistic, so the correction below is reliable here even when `n_distinct` is not.

### A deduplication-aware sweep for v17

The fix is one extra term and one gate, both derived from source. Where the index can deduplicate, the leaf requirement becomes `key_groups * slot + (rows - key_groups) * 6` bytes instead of `rows * slot`, since each extra row after the first in a key group costs one `ItemPointerData` ([nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L484-L531)).

The gate matters as much as the term. `ANALYZE` stores `stadistinct` as a **positive absolute count** only when it estimated at most 10% of the table's rows to be distinct; above that it converts to a negative fraction ([analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612)), and the negative form is the unreliable one. This run measured `t_var` at `n_distinct = -0.87447` against a true 392,917 of 400,000 distinct values; crediting that 11% shortfall as deduplication moved the `idx_var` estimate from +105 to −271 blocks. So the sweep credits duplicate compression only when `n_distinct > 0`, and always accounts for NULLs from `null_frac`.

Three more source-level conditions gate the whole term: build-time deduplication is skipped for unique indexes (`!btspool->isunique`), requires `deduplicate_items` (default on), and requires an all-equal-image opclass, whose support function is visible in core SQL as `pg_amproc.amprocnum = 4`.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

WITH RECURSIVE
idx AS (
    SELECT /* wiki_btree_bloat_sweep_v17 */
           c.oid AS idxoid, n.nspname AS schemaname, t.relname AS tablename,
           c.relname AS indexname, t.oid AS tbloid, x.indkey, x.indisunique,
           (x.indpred IS NOT NULL)                      AS is_partial,
           coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'fillfactor'), 90)  AS fillfactor,
           coalesce((SELECT option_value::bool FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'deduplicate_items'), true) AS dedup_on,
           CASE WHEN c.reltuples = -1 THEN NULL
                WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
                ELSE least(c.reltuples::numeric,
                           coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
           END                                          AS live_rows,
           c.reltuples::numeric                         AS idx_reltuples,
           coalesce(s.n_dead_tup, 0)                    AS tbl_n_dead_tup,
           greatest(s.last_vacuum, s.last_autovacuum)   AS last_vacuum,
           greatest(s.last_analyze, s.last_autoanalyze) AS last_analyze,
           pg_relation_size(c.oid)                      AS actual_bytes,
           pg_relation_size(c.oid, 'fsm')               AS fsm_bytes,
           current_setting('block_size')::int           AS bs,
           (SELECT bool_and(EXISTS (SELECT 1 FROM pg_amproc ap
                                     WHERE ap.amprocfamily = op.opcfamily
                                       AND ap.amproclefttype = op.opcintype
                                       AND ap.amprocrighttype = op.opcintype
                                       AND ap.amprocnum = 4))
              FROM generate_subscripts(x.indclass, 1) k
              JOIN pg_opclass op ON op.oid = x.indclass[k]
             WHERE k < x.indnkeyatts)                   AS all_equalimage
      FROM pg_class c
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_am am       ON am.oid = c.relam
      LEFT JOIN pg_stat_all_tables s ON s.relid = t.oid
     WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
),
cols AS (
    SELECT i.idxoid, a.attlen, a.attalign,
           CASE WHEN a.attlen > 0 THEN a.attlen::numeric
                ELSE coalesce(se.avg_width, st.avg_width, 32)::numeric END AS width,
           coalesce(se.null_frac, st.null_frac, 0)::numeric                AS null_frac,
           coalesce(se.n_distinct, st.n_distinct, -1)::numeric             AS n_distinct
      FROM idx i
      JOIN pg_attribute a ON a.attrelid = i.idxoid AND a.attnum > 0 AND NOT a.attisdropped
      LEFT JOIN pg_stats se ON se.schemaname = i.schemaname
                           AND se.tablename = i.indexname AND se.attname = a.attname
      LEFT JOIN pg_attribute ta ON ta.attrelid = i.tbloid
                               AND ta.attnum = i.indkey[a.attnum - 1]
      LEFT JOIN pg_stats st ON st.schemaname = i.schemaname
                           AND st.tablename = i.tablename AND st.attname = ta.attname
),
tuple AS (
    SELECT i.*,
           (SELECT sum((1 - c.null_frac) *
                       CASE WHEN c.attlen < 0 AND c.width <= 127 THEN c.width
                            ELSE ceil(c.width / al.a) * al.a END)
              FROM cols c
              CROSS JOIN LATERAL (SELECT CASE c.attalign WHEN 'c' THEN 1 WHEN 's' THEN 2
                                              WHEN 'i' THEN 4 ELSE 8 END AS a) al
             WHERE c.idxoid = i.idxoid)                                    AS data_size,
           (SELECT 1 - coalesce(exp(sum(ln(greatest(1 - c.null_frac, 1e-9)))), 1)
              FROM cols c WHERE c.idxoid = i.idxoid)                       AS p_null,
           (SELECT least(round(exp(sum(ln(greatest(
                       CASE WHEN c.n_distinct > 0
                            THEN c.n_distinct
                                 + CASE WHEN c.null_frac > 0 THEN 1 ELSE 0 END
                            ELSE (1 - c.null_frac) * greatest(i.live_rows, 0)
                                 + CASE WHEN c.null_frac > 0 THEN 1 ELSE 0 END
                       END, 1))))),
                         greatest(i.live_rows, 0))
              FROM cols c WHERE c.idxoid = i.idxoid)                       AS key_groups
      FROM idx i
),
sized AS (
    SELECT t.*, ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8 + 4         AS slot
      FROM tuple t
),
fit AS (
    SELECT s.*,
           greatest(floor((s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100)) / s.slot), 1)
               AS leaf_cap,
           greatest(floor((s.bs - 48 - floor(s.bs * 30 / 100)) / s.slot), 2)
               AS int_cap,
           (s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100))          AS leaf_bytes,
           (NOT s.indisunique AND s.dedup_on AND coalesce(s.all_equalimage, false))
               AS dedup_applies,
           least(greatest(s.live_rows, 0), s.key_groups)                   AS groups_est
      FROM sized s
),
leaves AS (
    SELECT f.*,
           CASE WHEN f.dedup_applies AND f.groups_est < greatest(f.live_rows, 0)
                THEN ceil((f.groups_est * f.slot
                           + (greatest(f.live_rows, 0) - f.groups_est) * 6) / f.leaf_bytes)
                ELSE ceil(greatest(f.live_rows, 0) / f.leaf_cap)
           END AS leaf_pages
      FROM fit f
),
levels AS (
    SELECT idxoid, leaf_pages AS pages, int_cap FROM leaves
    UNION ALL
    SELECT l.idxoid, ceil(l.pages / l.int_cap), l.int_cap FROM levels l WHERE l.pages > 1
),
modelled AS (
    SELECT l.*, (SELECT sum(pages) FROM levels v WHERE v.idxoid = l.idxoid) + 1
                    AS expected_blocks
      FROM leaves l
)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(actual_bytes) AS index_size,
       CASE
         WHEN live_rows IS NULL THEN 'unmeasured: reltuples unknown'
         WHEN is_partial AND (tbl_n_dead_tup > 0 OR last_analyze IS NULL)
              THEN 'unmeasured: analyze the table first'
         ELSE 'ok'
       END                                            AS status,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         pg_size_pretty(greatest(actual_bytes - expected_blocks * bs, 0)::bigint) END AS wasted,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         round((100 * (1 - (expected_blocks * bs) / greatest(actual_bytes, 1)))::numeric, 1)
       END                                            AS bloat_pct,
       (dedup_applies AND groups_est < greatest(live_rows, 0)) AS dedup_credited,
       fsm_bytes > 0                                  AS has_freed_pages,
       tbl_n_dead_tup                                 AS dead_tuples,
       (last_vacuum IS NULL AND last_analyze IS NULL) AS never_analyzed,
       idx_reltuples::bigint                          AS idx_reltuples,
       groups_est::bigint                             AS key_groups,
       live_rows::bigint                              AS modelled_rows
  FROM modelled
 WHERE actual_bytes > 1024 * 1024
 ORDER BY greatest(actual_bytes - expected_blocks * bs, 0) DESC NULLS FIRST
 LIMIT 20;
```

Scored on the same 54 cells, against the same rebuilds:

| | Method A (v12 SQL) | dedup-aware sweep |
|---|---|---|
| exact | 30 / 54 | 30 / 54 |
| within 5 blocks | 42 | 46 |
| within 16 blocks | 42 | 47 |
| worst absolute error | 1896 blocks | 499 blocks |
| worst `dup` error | +1896 (+223.3%) | −24 (−2.9%) |

The 499-block worst case is not deduplication; it is the partial-index staleness the v12 page already documents, and it clears the same way — see [Partial indexes and the statistics state](#partial-indexes-and-the-statistics-state). On the fixture set the same two changes move `idx_dup` from +948 to −12 blocks and `idx_null` from +474 to −5, and leave all nine already-exact fixtures untouched.

The correction is deliberately conservative and it under-corrects in two named places: it stops crediting deduplication once `n_distinct` flips to the negative form (rows 1 and 2 of the ratio sweep), and it ignores the posting-list size cap, which costs about 3% on all-duplicate indexes.

### Method A-prime still fixes variable key width

`pg_stats.avg_width` is a sample mean of the stored width, so a MAXALIGN of that single average mis-prices keys whose per-row width straddles an alignment boundary. Measuring the slot per row instead:

```sql
SET statement_timeout = '60s';

SELECT /* wiki_btree_measure_slot */
       count(*)                                                 AS rows_measured,
       avg(ceil((8 + pg_column_size(k)) / 8.0) * 8 + 4)         AS avg_slot_bytes
  FROM t_var TABLESAMPLE BERNOULLI (1) REPEATABLE (42);
```

returned 57.165 bytes from a 1% sample (3,883 rows, 23.9 ms) and 58.000 from a full scan, against Method A's catalog-derived 60. Feeding 58 back into the closed form gives `leaf_cap` 126, 3175 leaves, `int_cap` 98, and 3210 total blocks against a true rebuild of 3211 — the error falls from +105 blocks (−3.3% phantom negative bloat) to −1 block (−0.03%). Same conclusion as the v12 page.

### Method B: leaf counts still exact, density formula not

The census is unchanged: `live_leaf_pages = full_scan_blocks - descent_blocks`, both probes twice in one session, second reading used. A forward scan still reads one buffer per right link and skips ignorable pages ([nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2181-L2240)), and an index-only scan still falls back to the heap whenever the visibility-map bit is unset ([nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L150-L170)), reported as `Heap Fetches` ([explain.c:1993](../../../../raw/postgres-17/src/backend/commands/explain.c#L1993)).

Results: **exact on every eligible cell** — 12 of 12 fixtures and 36 of 36 matrix cells, partial and non-partial alike. The 18 ineligible matrix cells are exactly the three never-vacuumed bloat types, and the `Heap Fetches` check caught all of them (`m_churn_unvac_1000_full` reported 1,559,855 leaf pages against a true 8,197).

The density reconstruction is where v17 diverges. The v12 formula assumes one slot per row:

| bloat type | cells | v12 density formula error, non-partial | v12 formula error, partial |
|---|---|---|---|
| `fresh`, `scatter`, `range`, `random`, `churn_vac` | 30 | −0.05 to −0.04 points | −1.09 to +0.54 points |
| `dup` | 6 | **+216.67 to +217.43 points** | **+209.85 to +220.16 points** |

`m_dup_1000_full` is the clean example: 313.58% "density" against `pgstatindex`'s 96.15%. Substituting the same posting-list term — `(key_groups * slot + (rows - key_groups) * 6) / (leaf_pages * (BLCKSZ - 40))` — brings the six `dup` cells to −4.38 to −1.30 points and costs the other 30 cells about a quarter of a point. The denominator `BLCKSZ - 40` is still what `pgstatindex` uses ([pgstatindex.c:310-316](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L310-L316), [pgstatindex.c:363-367](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367)).

### Method C: unchanged answer, different write path

Method C is exact by construction on 17.10 and stays the arbiter used above. Its restrictions are the same: `CREATE INDEX CONCURRENTLY` is rejected inside a transaction block ([utility.c:1462](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1455-L1465)), takes `ShareUpdateExclusiveLock` on the table ([indexcmds.c:678](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L672-L682)), is refused on a partitioned table ([indexcmds.c:729](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L723-L733)), leaves an invalid index behind on failure because `indisvalid` is set as the last step ([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521)), and costs two table scans ([create_index.sgml:625-635](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L625-L635)).

One v17 internal difference is worth knowing before quoting its cost: the sorted build now writes through the bulk-write facility rather than issuing its own page writes and an unconditional `smgrimmedsync` ([nbtsort.c:1149](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1152), [nbtsort.c:1376](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1370-L1377)). `smgr_bulk_finish` still WAL-logs the built pages when the relation needs WAL, but it registers the fsync with the checkpointer and only calls `smgrimmedsync` when a checkpoint started concurrently ([bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L220)).

The DDL generator still needs the same two fixes, because `pg_get_indexdef` emits reloptions but neither `CONCURRENTLY` nor the tablespace ([ruleutils.c#pg_get_indexdef](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L1158-L1175)).

### Method D: new message, no row count, new blind spot

`VACUUM VERBOSE` output was restructured. v17 prints one consolidated report per table, and its per-index line carries four page classes and **no index row count**:

```text
index scan needed: 4425 pages from table (100.00% of total) had 900000 dead item identifiers removed
index "d_work_idx": pages: 2745 in total, 0 newly deleted, 0 currently deleted, 0 reusable
```

That is the exact format string in [vacuumlazy.c:725-731](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732), fed by `IndexBulkDeleteResult`, which gained a `pages_newly_deleted` counter ([genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L75-L84), [nbtree.c#btvacuumpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1160-L1190)). Anything parsing the v12 wording ("now contains N row versions in M pages") gets nothing, and the tuple count is simply not available from Method D any more.

The v12 caveat that a no-op VACUUM prints no index line still holds — `btvacuumcleanup` returns NULL when `_bt_vacuum_needs_cleanup` says no scan is needed ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L924)) — and v17 adds a second, larger hole. When fewer than 2% of the table's pages hold dead items, VACUUM bypasses index vacuuming entirely ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1864-L1935)) and prints a distinct line with no per-index output at all. Measured on a 4,425-page table with 4,000 dead rows on 18 pages:

```text
index scan bypassed: 18 pages from table (0.41% of total) have 4000 dead item identifiers
```

Re-running the same VACUUM with `INDEX_CLEANUP ON` — which sets `consider_bypass_optimization` false ([vacuumlazy.c:392-407](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L388-L407)) — produced the index line and reported 10 newly deleted pages. On v17, Method D therefore reports only when VACUUM both had work to do **and** did not bypass it.

### The v14 unknown reltuples sentinel

`pg_class.reltuples` now defaults to `-1`, documented in the catalog header as "unknown" ([pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)), and `index_update_stats` deliberately preserves it when an index is created on an empty table ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2796-L2813)). `TRUNCATE` puts an index back into that state: measured on a 300,000-row fixture, the index's `reltuples` went 300000 → −1 across a truncate and stayed −1 after the reload.

The v12 SQL has no `-1` branch, so the sentinel flows straight into the arithmetic. On a 1,000,000-row table whose index was created before the rows and never analyzed:

| index | live blocks | rebuilt blocks | true bloat | `idx_reltuples` | v12 `modelled_rows` | v12 model | v12 `bloat_pct` |
|---|---|---|---|---|---|---|---|
| `m1_neg_idx` | 2745 | 2745 | 0.0% | −1 | −1 | 1 block | **100.0** |

A perfectly healthy 21 MB index is reported as 100% wasted. Falling back to the cumulative statistics counter is not a fix either: at the moment of the sweep `pg_stat_all_tables.n_live_tup` for that table read 2,000,000 against 1,000,000 real rows. The sweep above therefore treats `reltuples = -1` as *unmeasured* and emits NULL for `wasted` and `bloat_pct` rather than a number:

```text
 tablename |  indexname  | index_size |             status            | wasted | bloat_pct | idx_reltuples
 m1_neg    | m1_neg_idx  | 21 MB      | unmeasured: reltuples unknown |        |           |            -1
```

`ANALYZE` or `VACUUM` on the table clears it. This is one place where v17 is *better* instrumented than the v12 model assumes: `0` and "unknown" are now distinguishable, so an empty index and an unmeasured index no longer produce the same reading.

### Partial indexes and the statistics state

Partial indexes were exact for the v12 page and are exact here too, but only in the statistics state the v12 recipes produce. Which of the two writers touched the index's `reltuples` last decides it:

- a build writes the true count through `index_update_stats` ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2796-L2813));
- `ANALYZE` writes `ceil(tupleFract * totalrows)` from the heap sample ([analyze.c:637-660](../../../../raw/postgres-17/src/backend/commands/analyze.c#L637-L660));
- VACUUM writes the true count, but only when the index scan actually ran and produced a non-estimated count ([vacuumlazy.c:3078-3098](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3078-L3098), [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L924)).

Measured directly on the three `fresh` partial cells, first with build-time statistics and then after one plain `ANALYZE`:

| cell | live blocks | model with build-time reltuples | `idx_reltuples` after ANALYZE | model after ANALYZE | error |
|---|---|---|---|---|---|
| `m_fresh_200_part` | 112 | 112 (exact) | 40267 | 113 | +1 |
| `m_fresh_500_part` | 276 | 276 (exact) | 99850 | 275 | −1 |
| `m_fresh_1000_part` | 551 | 551 (exact) | 199100 | 548 | −3 |

The catastrophic partial-index case is unchanged from v12: an index whose predicate subset shrank, with no VACUUM and no ANALYZE, keeps its pre-delete `reltuples` because `pg_stat_all_tables.n_live_tup` counts the whole table. One `ANALYZE` repairs it:

| cell | live | rebuilt | model before | model after | error after | `reltuples` after |
|---|---|---|---|---|---|---|
| `m_lpdead_200_part` | 112 | 12 | 112 | 12 | 0 | 3636 |
| `m_lpdead_500_part` | 276 | 27 | 276 | 27 | 0 | 9116 |
| `m_lpdead_1000_part` | 551 | 52 | 551 | 52 | 0 | 18070 |
| `m_stale_200_part` | 112 | 12 | 112 | 12 | 0 | 3636 |
| `m_stale_500_part` | 276 | 27 | 276 | 27 | 0 | 9050 |
| `m_stale_1000_part` | 551 | 52 | 551 | 51 | −1 | 17910 |

Worst error 499 blocks before, 1 block after — the v12 page reports 510 blocks before and 1 after. The non-partial siblings on the same six tables were exact both times, because the collector's `n_live_tup` tracked the delete.

### The 54-cell matrix

Worst error per bloat type, as collected, before any repair `ANALYZE`:

| bloat type | non-partial: worst Δblocks (exact cells) | partial: worst Δblocks (exact cells) | dedup-aware, non-partial | dedup-aware, partial |
|---|---|---|---|---|
| `fresh` | 0 (3/3) | 5 (0/3) | 0 (3/3) | 5 (0/3) |
| `scatter` | 0 (3/3) | 1 (2/3) | 0 (3/3) | 1 (2/3) |
| `range` | 0 (3/3) | 1 (1/3) | 0 (3/3) | 1 (1/3) |
| `random` | 0 (3/3) | 4 (0/3) | 0 (3/3) | 4 (0/3) |
| `dup` | **1896** (0/3) | **386** (0/3) | 24 (0/3) | 3 (0/3) |
| `churn_vac` | 0 (3/3) | 3 (0/3) | 0 (3/3) | 3 (0/3) |
| `churn_unvac` | 0 (3/3) | 0 (3/3) | 0 (3/3) | 0 (3/3) |
| `lpdead` | 0 (3/3) | **499** (0/3) | 0 (3/3) | 499 (0/3) |
| `stale` | 0 (3/3) | **499** (0/3) | 0 (3/3) | 499 (0/3) |

Every non-partial cell outside `dup` is exact, which is the strongest single statement about accuracy transfer: for indexes with distinct keys, the v12 arithmetic predicts a v17 rebuild to the block at three scales and nine bloat shapes.

`churn_unvac` is worth one note. Its non-partial cells are exact at every scale, but the live index is a different size than the v12 page reports: two full-table updates left 8228 blocks here against a rebuild of 2745, and the `idx_churn` fixture reads 2471 blocks at 74.80% density against the v12 page's 3293 at 67.63%. v17 does have a mechanism the v12 checkout lacks — bottom-up index deletion deletes dead entries when a page is about to split, instead of splitting ([nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L306-L421)) — but this run did not separate it from the update pattern, so the cause stays open. The model is unaffected either way, because it predicts the *rebuilt* size and the rebuild is a fresh sorted build.

### Accuracy on 17 against the v12 page's reported accuracy

Direct comparison, v17 measured here against the figures the v12 page reports for v12:

| Claim | v12 page | this v17 run |
|---|---|---|
| Method A exact on the fixture set | 10 of 14, ±2 blocks on 3, −4.6% on 1 | 9 of 14 (+ `idx_empty`), ±4 to +105 on 3, +474 and +948 on 2 |
| Method A on the matrix | exact on 39 of 54, within 5 on 47, within 16 on 48, 6 catastrophic | exact on 30 of 54, within 5 on 42, within 16 on 42, 12 catastrophic (6 `dup`, 6 `lpdead`/`stale` partial) |
| Method B `leaf_pages` | exact on 12 of 12 fixtures, 36 of 36 eligible cells | exact on 12 of 12 fixtures, 36 of 36 eligible cells |
| Method B density | −0.03 to −0.15 points | −1.09 to +0.54 points, except `dup` at +209.85 to +220.16 points |
| Method A-prime | fixes the variable-width fixture to +0.32% | fixes it to −0.03% |
| Method C | exact by construction | exact by construction |
| Partial-index repair | worst error 510 blocks → 1 after ANALYZE | worst error 499 blocks → 1 after ANALYZE |
| Method D | exact page census, silent when VACUUM has no work | no row count at all, silent when VACUUM has no work **or** bypasses the index |

So: same accuracy on distinct-key indexes, same failure mode and same repair for stale partial indexes, and one new class of catastrophic error that did not exist in v12. The v12 page's own scoreboard falls from 39 exact cells to 30 for a single reason, and the [dedup-aware sweep](#a-deduplication-aware-sweep-for-v17) recovers all but 2.9% of it.

### What to change before running the v12 SQL on 17

1. Add the posting-list term and its `n_distinct > 0` gate, or the sweep will condemn healthy indexes with duplicate keys. This is the only change that matters for correctness.
2. Add the `reltuples = -1` branch and report those indexes as unmeasured.
3. Flag partial indexes whose table has dead tuples or has never been analyzed as unmeasured; run `ANALYZE` before believing them.
4. Keep `least(reltuples, n_live_tup)` for non-partial indexes: it is still what rescues an unvacuumed bulk delete (`idx_stale`, model 276 against a rebuild of 276).
5. If a monitoring job parses Method D, rewrite the parser for `index "name": pages: N in total, N newly deleted, N currently deleted, N reusable`, and treat `index scan bypassed` as "no data", not "no bloat".
6. Method B and Method C need no changes; only Method B's density formula does.

### Cost of each method on this server

| Method | Reads | Writes | Measured cost | Accuracy against a rebuild |
|---|---|---|---|---|
| A: catalog sweep | catalogs only | none | 27.8 ms for 79 indexes over 128,306 blocks | exact on 9/14 fixtures and 24/27 non-partial matrix cells; +223% on duplicates |
| A-prime: `pg_column_size` | one 1% sample | none | 23.9 ms over 400,000 rows | fixes the variable-width fixture to −1 block |
| B: index-only-scan census | the whole index | none | 93 ms on a 21 MB index (warm) | `leaf_pages` exact on 48 of 48 eligible cells |
| C: CIC rebuild | table, writes a new index | yes | 204 ms plus 21 MB on the same index | exact by definition |
| D: `VACUUM VERBOSE` | the whole index | yes | a VACUUM | exact page classes, when it prints them |

### Settings and apply scopes

| Setting | Context in v17 | Apply scope |
|---|---|---|
| `statement_timeout` | `PGC_USERSET` ([guc_tables.c:2611-2619](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2610-L2619)) | session/transaction |
| `lock_timeout` | `PGC_USERSET` ([guc_tables.c:2622-2630](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2621-L2630)) | session/transaction |
| `maintenance_work_mem` | `PGC_USERSET` ([guc_tables.c:2465-2473](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2464-L2473)) | session/transaction |
| `enable_seqscan`, `enable_bitmapscan`, `max_parallel_workers_per_gather` | `PGC_USERSET` | session/transaction |
| `block_size` | `PGC_INTERNAL` preset ([guc_tables.c:3268-3276](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276)) | read-only; always read it with `current_setting('block_size')` |
| `wal_level` | `PGC_POSTMASTER` ([guc_tables.c:4974-4981](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4973-L4981)) | restart; only relevant because it decides whether Method C WAL-logs every built page |

No method here requires changing a setting that needs a reload or a restart. The per-index `deduplicate_items` reloption read by the sweep is set with `ALTER INDEX ... SET (deduplicate_items = off)`, which takes `AccessExclusiveLock` and is not required to run the sweep.

### What no core-SQL method can measure on v17

The v12 page's list is unchanged, and deduplication adds one entry:

- **`leaf_fragmentation`** — it needs page headers in physical order ([pgstatindex.c:318-325](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L325)); nothing in core exposes them.
- **`LP_DEAD` space inside live leaves** — Method B counts returned rows, so it reports live density.
- **Deleted versus half-dead pages** — Methods A and B lump both into "not in the live leaf chain".
- **How much deduplication actually happened** — the number of posting-list tuples and their TID counts is per-page state; core SQL can only estimate it from `n_distinct` and `null_frac`.
- **Per-page detail of any kind** — there is still no core page reader. `pg_proc.dat` contains no `pgstatindex`, `pgstattuple`, `get_raw_page`, `bt_page_stats` or `pg_freespace` entry; all of them are contrib, none of `pgstattuple`, `pageinspect`, `pg_freespacemap` or `amcheck` sets `trusted = true` in its control file, and `read_extension_control_file` defaults `superuser` to true and `trusted` to false ([extension.c:778-790](../../../../raw/postgres-17/src/backend/commands/extension.c#L778-L790)), so a non-superuser without the trust flag gets `permission denied to create extension` ([extension.c#execute_extension_script](../../../../raw/postgres-17/src/backend/commands/extension.c#L1019-L1035)). `pgstatindex` re-checks `superuser()` in C at entry ([pgstatindex.c:145-160](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L145-L160)), as does `pageinspect` ([rawpage.c:148-154](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L148-L154)). `TABLESAMPLE` still cannot be applied to an index ([parse_clause.c:1136-1146](../../../../raw/postgres-17/src/backend/parser/parse_clause.c#L1136-L1146)).

### Follow-up: the same sweep on a 12.2 server and a 17.10 server

Yes on both, and the deduplication term is what makes it portable rather than what breaks it. Every v17-specific term in the sweep is gated on a catalog fact that a PostgreSQL 12 server cannot produce, so the same statement pointed at a 12 server turns the correction off by itself and leaves exactly the v12 page's Method A running.

The gate is the SQL form of the engine's own test. `_bt_load` deduplicates only when the build's `allequalimage` flag is set, the index is not unique, and `deduplicate_items` is on ([nbtsort.c:1147-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)), and that flag comes from `_bt_allequalimage`, which looks up a `BTEQUALIMAGE_PROC` support function per key column and returns false when any opclass lacks one ([nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563), [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)). `BTEQUALIMAGE_PROC` is support number 4 ([nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L702-L712)), which is why the sweep asks `pg_amproc` for `amprocnum = 4`.

What each v17 term needs, and what a 12.2 server offers:

| Sweep term | What it reads | 17.10 | 12.2, measured |
|---|---|---|---|
| posting-list leaf formula | a `pg_amproc` row at `amprocnum = 4` for every key column's opclass | present; `all_equalimage` is true on all 21 fixture indexes | no such row: `max(amprocnum)` over btree opfamilies is 3 and the count at 4 is 0, so `all_equalimage` is false on all 20 fixture indexes |
| `deduplicate_items` reloption | `pg_options_to_table(reloptions)` ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtutils.c#btoptions](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4561-L4576)) | exists, defaults to on | the option does not exist: `ALTER INDEX idx_dup SET (deduplicate_items = off)` fails with `ERROR: unrecognized parameter "deduplicate_items"` |
| `reltuples = -1` guard | `pg_class.reltuples` ([pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)) | reads `-1` after `TRUNCATE` and reports `unmeasured: reltuples unknown` | the sentinel never appears: an index on an empty table and an index after `TRUNCATE` both read `0` |

None of the three can be back-ported into a 12 catalog by hand either. A support-function number is bounded by the access method's `amsupport`, which for B-tree is `BTNProcs` ([nbtree.c#bthandler](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L100-L107)) — 5 in v17, and `ALTER OPERATOR FAMILY ... ADD FUNCTION n` rejects anything above it ([opclasscmds.c:840-845](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L840-L845), [opclasscmds.c#invalid-function-number](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L956-L962)); `btvalidate` accepts exactly support numbers 1 through 5 and reports every other number as invalid ([nbtvalidate.c:90-126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L90-L126)). On 12.2 the same DDL returns `ERROR: invalid function number 4, must be between 1 and 3`.

All three constructs postdate 12 in this checkout's own history: `612a1ab7672` (2020-02-26, "Add equalimage B-Tree support functions.") and `0d861bbb702` (2020-02-26, "Add deduplication to nbtree.") first ship in `REL_13_0`, and `3d351d916b2` (2020-08-30, "Redefine pg_class.reltuples to be -1 before the first VACUUM or ANALYZE.") first ships in `REL_14_0`. None is an ancestor of `REL_12_2`.

Measured with the statement unchanged on two isolated servers, each built from its own pin, carrying the same DDL and the same generated data:

| | 12.2 | 17.10 |
|---|---|---|
| statement runs | yes | yes |
| indexes swept / blocks covered, rebuild copies included | 32 / 68,108 | 34 / 48,753 |
| warm run time, same statement | 10.9 ms | 10.5 ms |
| indexes credited with deduplication | 0 | 15 |
| cells whose `expected_blocks` differs from the v12 page's Method A | 0 of 32 | 15 of 34 |
| indexes reported `unmeasured` | 0 | 1 |

Per fixture, 1,000,000 rows each, `bigint` key, `pg_relation_size` in blocks and a `CREATE INDEX CONCURRENTLY` rebuild as ground truth:

| fixture | 12.2 live | 12.2 rebuilt | 12.2 model | 12.2 reported | 17.10 live | 17.10 rebuilt | 17.10 model | 17.10 reported |
|---|---|---|---|---|---|---|---|---|
| `idx_seq`, distinct keys | 2745 | 2745 | 2745 | 0.0% | 2745 | 2745 | 2745 | 0.0% |
| `idx_dup`, one key | 2749 | 2749 | 2745 | 0.1% | 849 | 849 | 825 | 2.8% |
| `idx_null`, 25% NULL | 2746 | 2746 | 2745 | 0.0% | 2271 | 2271 | 2265 | 0.3% |
| `idx_q2`, 2 rows/key | 2749 | 2749 | 2745 | 0.1% | 2475 | 2475 | 2745 | −10.9% |
| `idx_q5`, 5 rows/key | 2748 | 2748 | 2745 | 0.1% | 1426 | 1426 | 2745 | −92.5% |
| `idx_q10`, 10 rows/key | 2749 | 2749 | 2745 | 0.1% | 1157 | 1157 | 2745 | −137.3% |
| `idx_q100`, 100 rows/key | 2749 | 2749 | 2745 | 0.1% | 839 | 839 | 844 | −0.6% |
| `idx_seq_part`, 20% partial | 551 | 551 | 551 | 0.0% | 551 | 551 | 550 | 0.2% |
| `idx_dupdel`, 10 keys, 90% deleted + VACUUM | 2749 | 278 | 276 | 90.0% | 850 | 87 | 84 | 90.1% |

Four readings matter:

- **Every fresh fixture is 0% reclaimable on both servers** — live equals rebuilt in all sixteen cells — so any non-zero reading in the "reported" columns is model error, not bloat.
- **On 12.2 the sweep misses by at most 4 blocks across these nine fixtures**, including the all-duplicate index. That is the case the deduplication term exists for, and crediting it there would have been wrong: 12.2 stores one index tuple per row, so `idx_dup` really is 2749 blocks. The gate is what keeps the answer right.
- **`idx_dupdel` is the load-bearing case**: a duplicate-key index with real, VACUUM-confirmed bloat. Both servers report about 90% and both are right — 276 modelled blocks against a 12.2 rebuild of 278 (true bloat 89.9%), 84 against a 17.10 rebuild of 87 (true bloat 89.8%) — reached by different arithmetic on each server. The uncorrected v12 model would have reported 67.5% on the 17.10 index.
- **The negative percentages on 17.10 at 2, 5 and 10 rows per key** are the `n_distinct > 0` gate declining to credit deduplication, the blind spot already documented in [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17). The same cells are accurate on 12.2 because there is nothing to credit. At ten rows per key this run landed on the negative form (`key_groups` 1,000,000, no credit) where the earlier ratio sweep on this page recorded a positive `n_distinct` of 97311; 100,001 distinct values in 1,000,000 rows sits exactly on the 10%-of-rows boundary that decides the sign ([analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612)), and the estimate is sampled.

Six further index shapes over a 200,000-row table were swept on both servers without error — multi-column, `INCLUDE`, expression, unique, `text`, and a low-cardinality `int`. Scored against the freshly built live size rather than a rebuild, the model lands within one block on every one of the twelve cells, on both servers. The `indclass` probe survives the version change because `oidvector` subscripts start at 0, so `k < x.indnkeyatts` covers every key column on both servers (measured `array_lower(indclass, 1) = 0` on 12.2 and 17.10).

### Follow-up: the INCLUDE-column false positive on v17

This comparison exposed one case where the dedup-aware sweep is wrong on 17.10, and it is not a version-portability problem: `_bt_allequalimage` returns false immediately for any index whose total attribute count differs from its key attribute count, because "INCLUDE indexes can never support deduplication" ([nbtutils.c:5144-5148](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5148)). The sweep's catalog probe only inspects key columns and never checks for included ones, so it credits deduplication the engine will refuse.

It only bites when the *included* column is also low-cardinality, because `key_groups` multiplies per-column `n_distinct` across every index attribute. Measured on 17.10 over 1,000,000 rows with 1000 distinct `a` and 5 distinct `d`:

| index | key groups | model | live | rebuilt | reported | true |
|---|---|---|---|---|---|---|
| `idx_inc_lowcard` — `(a) INCLUDE (d)` | 5000 | 842 | 3853 | 3853 | **78.1%** | 0.0% |
| `idx_inc_bothkeys` — `(a, d)` | 5000 | 842 | 897 | 897 | 6.1% | 0.0% |
| `idx_inc_keyonly` — `(a)` | 1000 | 827 | 896 | 896 | 7.7% | 0.0% |
| `idx_inc_include` — `(a) INCLUDE (b)`, `b` distinct | 1,000,000 | 3853 | 3853 | 3853 | 0.0% | 0.0% |

The fix is one conjunct. Add `(x.indnatts = x.indnkeyatts) AS no_included_cols` to the `idx` CTE and require it in the gate:

```sql
(NOT s.indisunique AND s.dedup_on AND s.no_included_cols
     AND coalesce(s.all_equalimage, false))                    AS dedup_applies
```

Re-scored on the same server, `idx_inc_lowcard` moves from 78.1% to 0.0% with a model of 3853 blocks against a rebuild of 3853 — exact — and all 33 other indexes in that database are unchanged. On 12.2 the same three indexes read 0.1% to 0.2% with errors of −4 to −6 blocks, because the gate is closed there for every index anyway.

The 6.1% and 7.7% on the two indexes that *can* deduplicate are the posting-list-cap under-correction this page already records at 2.9%. It is not a constant: it was 24 blocks at 1,000,000 rows per key and 69 blocks at 1000 rows per key.

### Follow-up: the v12 hazard the reltuples guard does not cover

The `reltuples = -1` branch is dead code on 12.2, and the v12-era failure it was added for is still live there as a *zero*. Same fixture on both servers — 300,000 rows, `TRUNCATE`, reload, no `ANALYZE`:

| server | table / index `reltuples` | 825-block healthy index reported as |
|---|---|---|
| 12.2 | `0` / `0` | **99.9% bloat** |
| 17.10 | `-1` / `-1` | `unmeasured: reltuples unknown`, no number |

`least(reltuples, n_live_tup)` cannot rescue it: the collector had the right 300,000 while `pg_class` had 0, and `least()` takes the zero. So a v12 server needs its own unmeasured rule, and the size check is what separates a stale zero from a genuinely empty index:

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

SELECT /* wiki_btree_zero_reltuples_v12 */
       n.nspname                                              AS schemaname,
       c.relname                                              AS indexname,
       c.reltuples::bigint                                    AS reltuples,
       pg_relation_size(c.oid)                                AS bytes
  FROM pg_class c
  JOIN pg_index x     ON x.indexrelid = c.oid
  JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE c.relkind = 'i' AND x.indisvalid
   AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
   AND c.reltuples = 0
   AND pg_relation_size(c.oid) > current_setting('block_size')::int
 ORDER BY pg_relation_size(c.oid) DESC;
```

Run as written on the 12.2 test database, that returns the truncated index and nothing else — one row, `idx_trunc`, `reltuples` 0, 6,758,400 bytes — because the genuinely empty index is exactly one metapage (8192 bytes) and every populated index carries a non-zero `reltuples`.

So, pointing this sweep at a 12 server needs no change for deduplication and one change for statistics:

1. Leave the posting-list term and its `pg_amproc` gate alone. They cost nothing and cannot fire.
2. Replace the `reltuples = -1` branch with the zero rule above, or keep both — `-1` on 14 and later, `0`-with-bytes on 12 and 13.
3. Keep the partial-index `unmeasured` status. It is statistics-driven and version-independent.
4. Expect the model to sit 1 to 4 blocks *under* a 12.2 rebuild on duplicate-key and NULL-heavy indexes, against 0 blocks on distinct keys.

## Context Reviewed

- nbtree build, split and deduplication: `nbtsort.c` (`_bt_blnewpage`, `_bt_pagestate`, `_bt_buildadd`, `_bt_load`, `_bt_sort_dedup_finish_pending`), `nbtdedup.c` (`_bt_dedup_pass`, `_bt_dedup_start_pending`, `_bt_dedup_save_htid`, `_bt_form_posting`, `_bt_bottomupdel_pass`), `nbtsplitloc.c` (`_bt_findsplitloc`, single-value strategy), `nbtree.h` (fillfactor constants, `BTGetDeduplicateItems`, `BTMaxItemSize`, `BTPageOpaqueData`, `P_HIKEY`), `README`.
- VACUUM and page recycling: `vacuumlazy.c` (VERBOSE report, `BYPASS_THRESHOLD_PAGES`, `lazy_vacuum`, `lazy_cleanup_all_indexes`, `lazy_cleanup_one_index`, index relstats update), `nbtree.c` (`btvacuumcleanup`, `btvacuumscan`, `btvacuumpage`), `nbtpage.c` (`_bt_pagedel`, `BTPageIsRecyclable`), `indexfsm.c`, `genam.h`.
- Catalog and statistics surfaces: `pg_class.h`, `index.c` (`index_update_stats`, `index_set_state_flags`), `vacuum.c` (`vac_update_relstats`), `analyze.c` (`do_analyze_rel`, `compute_scalar_stats`, `compute_distinct_stats`), `pg_statistic.h`, `system_views.sql` (`pg_stats`, `pg_stat_all_tables`, `pg_stat_all_indexes`), `pg_proc.dat`, `dbsize.c`, `relpath.c`, `varlena.c`, `guc_tables.c`.
- Executor and EXPLAIN: `nodeIndexonlyscan.c`, `nbtsearch.c` (`_bt_readnextpage`), `explain.c` (`show_buffer_usage`, `Heap Fetches`), `explain.sgml`.
- Rebuild path: `indexcmds.c` (`DefineIndex`), `utility.c`, `ruleutils.c` (`pg_get_indexdef`), `bulk_write.c`, `create_index.sgml`, `maintenance.sgml`.
- Contrib boundary: `pgstattuple.control`, `pgstatindex.c`, `pageinspect.control`, `rawpage.c`, `extension.c`, the 22 `trusted = true` contrib control files, `pgstattuple.sgml`.
- Exact-pin execution: one isolated 17.10 server built from the pinned checkout under `.wiki-runtime/`, carrying the 15 named fixtures, the 9 x 3 x {full, partial} matrix (54 indexes over 27 tables), a six-point duplication-ratio sweep, a `reltuples = -1` probe, and Method D bypass fixtures. Methods A, A-prime, B, C and D were executed against them, with `pgstattuple` installed solely as ground truth and Method C rebuilds as the reclaimable-size arbiter. Method B was driven through `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` so the plan's index name, `Heap Fetches` and buffer counts could be scored programmatically.
- Deduplication gate and its version dependence, for the v12/v17 follow-up: `nbtsort.c` (`_bt_load`'s `deduplicate` condition, `_bt_leafbuild` setting `inskey->allequalimage`), `nbtutils.c` (`_bt_allequalimage`, `btoptions`), `nbtree.h` (`BTORDER_PROC` through `BTNProcs`, `BTGetDeduplicateItems`), `nbtree.c` (`bthandler`'s `amsupport`), `nbtvalidate.c` (accepted support numbers), `reloptions.c` (`deduplicate_items` entry), `opclasscmds.c` (`maxProcNumber` and the `invalid function number` error), `varlena.c` (`btvarstrequalimage`), `pg_class.h` (`reltuples` `-1`), `analyze.c` (the negative-`stadistinct` rule). Commit history for the three constructs' first release tags was read from the same pinned checkout.
- Follow-up exact-pin execution, two servers: the same DDL and generated data on one isolated 12.2 server and one isolated 17.10 server, each built from its own pin under `.wiki-runtime/`, both with `autovacuum = off`, `fsync = off`, `block_size` 8192, and no contrib extension installed. Fixtures: nine 1,000,000-row indexes (distinct, all-duplicate, 25% NULL, four duplication ratios, a 20% partial sibling, and a 10-key index with 90% of rows deleted and vacuumed), six shape indexes over a 200,000-row table, three `INCLUDE`-versus-key-column indexes over a 1,000,000-row table, an empty-table index, and a `TRUNCATE`-and-reload index. The dedup-aware sweep and the v12 page's Method A were installed as views on both servers with only the 1 MB triage filter and `LIMIT` removed and `expected_blocks` exposed; `CREATE INDEX CONCURRENTLY` rebuilds were the ground truth. Catalog probes covered `pg_amproc` support numbers, `array_lower(indclass, 1)`, `ALTER INDEX ... SET (deduplicate_items = off)`, `ALTER OPERATOR FAMILY ... ADD FUNCTION 4`, and `pg_class.reltuples` after build, `TRUNCATE` and reload. Both servers were stopped afterwards, the test databases dropped, and the 17.10 data directory removed.

## Evidence Map

| Claim | Evidence |
|---|---|
| Leaf fill rule, high-key line pointer, and non-leaf fillfactor 70 are unchanged | [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629), [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L809-L855), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202) |
| Usable page space and the 8152-byte denominator | [nbtsplitloc.c:157-160](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L157-L160), [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L210-L216), [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L69) |
| Entry cost is `MAXALIGN(hoff + data) + 4`, NULLs widen `hoff` | [itup.h#IndexInfoFindDataOffset](../../../../raw/postgres-17/src/include/access/itup.h#L96-L110), [indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L65-L140), [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L263), [itemid.h#ItemIdData](../../../../raw/postgres-17/src/include/storage/itemid.h#L25-L31) |
| A sorted build deduplicates non-unique, all-equal-image indexes by default | [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1160), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1150) |
| Build-time posting lists are capped at 812 bytes; each extra row costs one 6-byte TID | [nbtsort.c:1292-1308](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1292-L1308), [nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L484-L531) |
| Insert-time deduplication uses a different, larger cap | [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L75-L93) |
| All-duplicate leaf splits use fillfactor 96 | [nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416), [nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202) |
| `ANALYZE` stores `stadistinct` as a negative fraction above 10% distinct | [analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612), [analyze.c:2544-2549](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2544-L2549), [pg_statistic.h#stadistinct](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L52-L69) |
| `reltuples = -1` means unknown, and index creation on an empty table preserves it | [pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2796-L2813) |
| Index `reltuples` is written by the build, by ANALYZE's sample, or by a non-estimated VACUUM count | [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2796-L2813), [analyze.c:637-660](../../../../raw/postgres-17/src/backend/commands/analyze.c#L637-L660), [vacuumlazy.c:3078-3098](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3078-L3098), [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1470) |
| `n_live_tup`/`n_dead_tup` come from the cumulative statistics system | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L700) |
| `pg_stats` exposes `avg_width`, `null_frac` and `n_distinct` | [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L215) |
| `pg_relation_size` is a live filesystem measurement of one fork | [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343), [relpath.c#forkNames](../../../../raw/postgres-17/src/common/relpath.c#L33-L40) |
| `pg_column_size` returns the stored datum size | [varlena.c#pg_column_size](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L5062-L5102) |
| A forward index scan reads one buffer per right link and ignores dead pages | [nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2181-L2240) |
| Index-only scans fall back to the heap when the VM bit is unset, reported as `Heap Fetches` | [nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L150-L170), [explain.c:1993](../../../../raw/postgres-17/src/backend/commands/explain.c#L1993) |
| `BUFFERS` is one per-node counter with no per-relation split | [explain.c#show_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3743-L3800), [explain.sgml#BUFFERS](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L182-L200) |
| v17 `VACUUM VERBOSE` prints four page classes per index and no row count | [vacuumlazy.c:718-732](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732), [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L75-L84) |
| Index vacuuming is bypassed below 2% of pages, and forced by `INDEX_CLEANUP ON` | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1864-L1935), [vacuumlazy.c:388-407](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L388-L407) |
| A no-op VACUUM prints no index line | [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L924) |
| Deleted pages stay in the file and are only recorded in the FSM | [nbtree.c#btvacuumpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1160-L1190), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [README:236-246](../../../../raw/postgres-17/src/backend/access/nbtree/README#L236-L246) |
| Partly-emptied pages remain allocated, the documented bloat mechanism | [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1026-L1034) |
| CIC restrictions, lock level and invalid-index leftover | [utility.c:1455-1465](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1455-L1465), [indexcmds.c:672-682](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L672-L682), [indexcmds.c:723-733](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L723-L733), [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3449-L3521), [create_index.sgml:625-635](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L625-L635) |
| v17 builds write through the bulk-write facility, syncing via the checkpointer | [nbtsort.c:1145-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1152), [nbtsort.c:1370-1377](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1370-L1377), [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L220) |
| `pg_get_indexdef` emits reloptions but not `CONCURRENTLY` or the tablespace | [ruleutils.c#pg_get_indexdef](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L1158-L1175) |
| `pgstatindex`'s density denominator and fragmentation definition | [pgstatindex.c:310-316](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L310-L316), [pgstatindex.c:363-367](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367), [pgstatindex.c:318-325](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L325) |
| The bloat tooling is contrib and superuser-gated | [extension.c:778-790](../../../../raw/postgres-17/src/backend/commands/extension.c#L778-L790), [extension.c#execute_extension_script](../../../../raw/postgres-17/src/backend/commands/extension.c#L1019-L1035), [pgstatindex.c:145-160](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L145-L160), [rawpage.c:148-154](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L148-L154) |
| `TABLESAMPLE` cannot be applied to an index | [parse_clause.c:1136-1146](../../../../raw/postgres-17/src/backend/parser/parse_clause.c#L1136-L1146) |
| GUC contexts used by these methods | [guc_tables.c:2610-2630](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2610-L2630), [guc_tables.c:2464-2473](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2464-L2473), [guc_tables.c:3267-3276](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276), [guc_tables.c:4973-4981](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4973-L4981) |
| The build-time deduplication decision is `allequalimage AND NOT isunique AND deduplicate_items` | [nbtsort.c:1147-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152), [nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563) |
| `allequalimage` is false when any key column's opclass lacks a `BTEQUALIMAGE_PROC` entry, and unconditionally false for an INCLUDE index | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183), [nbtutils.c:5144-5148](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5148) |
| Equal image is B-tree support function number 4, which is why the sweep probes `pg_amproc.amprocnum = 4` | [nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L702-L712) |
| A support-function number cannot exceed the access method's `amsupport`, so a pre-13 catalog cannot advertise number 4 | [nbtree.c#bthandler](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L100-L107), [opclasscmds.c:840-845](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L840-L845), [opclasscmds.c#invalid-function-number](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L956-L962), [nbtvalidate.c:90-126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L90-L126) |
| `deduplicate_items` is a B-tree reloption defaulting to on | [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtutils.c#btoptions](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4561-L4576), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1150) |
| Equal-image support functions, nbtree deduplication and the `-1` `reltuples` sentinel all postdate 12 | this checkout's history: `612a1ab7672` and `0d861bbb702` (2020-02-26, first in `REL_13_0`), `3d351d916b2` (2020-08-30, first in `REL_14_0`); none is an ancestor of `REL_12_2` |
| The equal-image function can exist and still return false, so a presence-only catalog probe is not exact | [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613) |

## Open Questions

- **The v12 fixtures were reconstructed, not recovered.** The v12 page records each fixture's shape and its resulting block counts but not its DDL, so `idx_multi`, `idx_var`, `idx_rand` and `idx_churn` differ from the v12 page's block counts for reasons that include fixture choice. Nine of fifteen fixtures reproduce the v12 block count exactly, which bounds but does not eliminate the risk that a difference attributed to v17 is really a difference in the recipe.
- **v12 numbers in the original comparison are quoted, not re-measured.** Every "v12 page" figure in the sections above comes from that page's own tables, so those figures are attributions rather than evidence from the v12 checkout. The 12.2 server used for the v12/v17 follow-up carried new fixtures and does not re-measure the v12 page's own numbers.
- **The 12.2 column of the follow-up is measurement plus history, not v12 source citation.** This page may cite only `raw/postgres-17/`, so every 12.2 statement above rests on exact-pin execution against a 12.2 server plus this checkout's own commit history. The v12-side source analysis — where v12's `BTNProcs` is 3, what its `btoptions` accepts, and what writes its `reltuples` — belongs on [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md) and is not filed there yet.
- **Only 12 and 17 were exercised.** Majors 13 through 16 were not run. 13 is the interesting gap: deduplication and the equal-image support function exist there, so the sweep would credit the posting-list term, while the `-1` `reltuples` sentinel does not exist until 14, so the stale-zero hazard described above applies at the same time.
- **The `pg_upgrade` reading was not measured.** A duplicate-key index built by 12 and carried onto a 17 server holds no posting lists, while the sweep predicts the size a 17 rebuild would produce. The reported "bloat" should therefore be the deduplication win a `REINDEX` would realize, which follows from the model's definition, but no `pg_upgrade` run was made to confirm it.
- **On 12.2 the model ran 1 to 6 blocks under a rebuild whenever keys repeated** — 2745 modelled against 2749 built on four single-column fixtures, 2748 on one, 2746 on the NULL fixture, and 3853 against 3859 on the two two-attribute `INCLUDE` cases — while distinct keys were exact. The cause was not isolated; no page-level tool was installed on that server, so leaf-versus-internal attribution was not possible.
- **The nondeterministic-collation false positive is source-derived only.** `btvarstrequalimage` returns false for a nondeterministic collation while the `pg_amproc` row still exists, so the sweep's presence-only probe would credit deduplication the engine refuses. This build has no ICU support (`CREATE COLLATION ... provider = icu` fails with `ICU is not supported in this build`), so the case was not measured.
- **The INCLUDE fix was scored on one server.** Adding `x.indnatts = x.indnkeyatts` to the gate corrected the one affected index and left the other 33 in that 17.10 database unchanged, but it was not re-scored against the 54-cell matrix above.
- **The dedup term ignores the posting-list cap.** Charging a flat 6 bytes per extra row underestimates an all-duplicate rebuild by about 3% (825 modelled against 849 measured at 1,000,000 rows) because it ignores both the 812-byte cap and the base tuple that each capped posting list repeats. The exact arithmetic is derivable from `_bt_dedup_save_htid` plus the `_bt_buildadd` `truncextra` rule, but it was not implemented in SQL.
- **The gate is a heuristic with a measured blind spot.** Refusing to credit deduplication when `n_distinct` is negative leaves +10.9% error at 2 rows per key and +92.5% at 5. A `SET (n_distinct = ...)` column override would close it, but that was not tested.
- **Multi-column group estimation is a product of per-column `n_distinct`.** No cell in the 54-cell matrix exercised a multi-column index with duplicates. The follow-up added one: `(a, d)` with 1000 and 5 distinct values over 1,000,000 rows produced 5000 groups and modelled 842 blocks against a rebuild of 897, so the product rule was 6.1% optimistic on the one case measured. Whether that error is the product rule or the posting-list cap was not separated.
- **`n_live_tup` read 2,000,000 for a 1,000,000-row table** after a truncate-and-reload in one session, while a separate clean run of the same sequence showed the counter resetting correctly. The discrepancy was not traced to a specific flush path; the recommendation above avoids depending on it, but the cause is unresolved.
- **Half-dead pages were never produced.** No fixture reached a non-zero `empty_pages`, so the claim that Methods A and B lump half-dead pages in with deleted ones follows from the code path, not from measurement.
- **Bottom-up index deletion was not isolated.** `idx_churn` and the `churn_*` matrix cells differ from the v12 page's sizes, and v14's bottom-up deletion is the obvious mechanism, but no fixture here separates it from the update pattern; the model's accuracy does not depend on which it is.
- **Block sizes other than 8192 were not exercised**, and `MAXALIGN` was assumed to be 8.
- **No upstream test covers these estimates.** The pinned tree has no regression test comparing a modelled index size against a built one, so every accuracy number here rests on the exact-pin fixtures described above.

## Source References

- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L641-L671)
- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L783-L940)
- [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1135-L1377)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571)
- [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)
- [nbtutils.c#btoptions](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4561-L4576)
- [nbtvalidate.c#btvalidate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L40-L287)
- [nbtree.c#bthandler](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L100-L153)
- [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)
- [opclasscmds.c#AlterOpFamily](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L816-L878)
- [opclasscmds.c#AlterOpFamilyAdd](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L880-L1028)
- [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)
- [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L58-L210)
- [nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L477-L544)
- [nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L856-L920)
- [nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L129-L430)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)
- [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1131-L1150)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L845-L924)
- [nbtree.c#btvacuumpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1160-L1190)
- [vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L618-L735)
- [vacuumlazy.c#lazy_vacuum](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1864-L1935)
- [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L70)
- [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2760-L2900)
- [analyze.c#compute_scalar_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2440-L2620)
- [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L215)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343)
- [varlena.c#pg_column_size](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L5062-L5102)
- [nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2181-L2338)
- [nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L61-L258)
- [explain.c#show_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3743-L3906)
- [indexcmds.c#DefineIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L740)
- [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L220)
- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L380)
- [extension.c#execute_extension_script](../../../../raw/postgres-17/src/backend/commands/extension.c#L993-L1060)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1012-L1048)
- [create_index.sgml#CONCURRENTLY](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L610-L700)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md)
- [Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 17 (unverified)](comment-stored-index-heap-ratio-bloat.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](create-index-concurrently.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md)
- [versions](../../../versions.md)
