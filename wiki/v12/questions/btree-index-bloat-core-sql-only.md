---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The proposal in one paragraph](#the-proposal-in-one-paragraph)
  - [What core SQL can and cannot see](#what-core-sql-can-and-cannot-see)
  - [What bloat is in a v12 B-tree](#what-bloat-is-in-a-v12-b-tree)
  - [The page-fill model, derived from source](#the-page-fill-model-derived-from-source)
  - [Method A: catalog-only bloat sweep](#method-a-catalog-only-bloat-sweep)
  - [Method A-prime: fix the key width with pg_column_size](#method-a-prime-fix-the-key-width-with-pg_column_size)
  - [Method B: index-only-scan page census](#method-b-index-only-scan-page-census)
  - [Method C: CREATE INDEX CONCURRENTLY rebuild probe](#method-c-create-index-concurrently-rebuild-probe)
  - [Method D: VACUUM VERBOSE page classes](#method-d-vacuum-verbose-page-classes)
  - [Two free triage signals](#two-free-triage-signals)
  - [Which method to use](#which-method-to-use)
  - [Exact-pin measurements](#exact-pin-measurements)
  - [Measured failure modes](#measured-failure-modes)
  - [Settings and apply scopes](#settings-and-apply-scopes)
  - [What no core-SQL method can measure](#what-no-core-sql-method-can-measure)
  - [Follow-up: the comparison fixture matrix](#follow-up-the-comparison-fixture-matrix)
  - [Follow-up: which pgstatindex columns core SQL reproduces](#follow-up-which-pgstatindex-columns-core-sql-reproduces)
  - [Follow-up: error by bloat type, partial and non-partial](#follow-up-error-by-bloat-type-partial-and-non-partial)
  - [Follow-up: an avg_leaf_density predictor head to head](#follow-up-an-avg_leaf_density-predictor-head-to-head)
  - [Follow-up: the partial-index failure and its fix](#follow-up-the-partial-index-failure-and-its-fix)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Propose a SQL-only method, using no contrib extensions, to measure B-tree index bloat in PostgreSQL 12.

Follow-up:

Add sections comparing the SQL bloat results to pgstatindex results, and measure what the error is with different types of index bloat and with partial and non-partial indexes.

## Answer

### The proposal in one paragraph

Compute the size the index *would* have if it were rebuilt right now, and subtract it from the size it has. Core SQL gives you the current size exactly, from the filesystem, through [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308). It does not give you the rebuilt size, but v12's B-tree build rule is fully deterministic, so the rebuilt size is *computable* from catalog data alone: `nbtsort.c` starts a new leaf page as soon as free space drops below `BLCKSZ * (100 - fillfactor) / 100` ([nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L729), [nbtsort.c#_bt_buildadd](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L856-L899)). On an exact-pin 12.2 server, that computed size matched an actual `CREATE INDEX CONCURRENTLY` rebuild **to the block on 10 of 14 fixtures** and to within 2 blocks on 3 more. Three progressively more expensive core-only methods refine or verify it: a `pg_column_size` key-width measurement, an index-only-scan buffer census that reproduces `pgstatindex`'s `avg_leaf_density` to within 0.14 points with no contrib installed, and a sibling-index rebuild that is exact by construction.

### What core SQL can and cannot see

Core v12 has **no** SQL-callable function that reads an index page, the free space map, or per-page fill. `pgstatindex`, `pgstattuple`, `get_raw_page`, `bt_metap`, `bt_page_stats`, `pg_freespace`, and `bt_index_check` all live under `contrib/`, and `pg_proc.dat` contains no equivalent. Installing them is not merely a `CREATE EXTENSION` away either: none of the four relevant control files sets `superuser = false`, and the parser defaults that field to `true` ([extension.c#read_extension_control_file](../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625)), so a non-superuser gets `permission denied to create extension` ([extension.c#execute_extension_script](../../../raw/postgres-12/src/backend/commands/extension.c#L798-L817)). `pageinspect` additionally hard-checks `superuser()` in C at every entry point ([rawpage.c#get_raw_page](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106), [pageinspect.sgml](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14)).

What core does expose:

| Surface | What it gives | Live or estimated |
|---|---|---|
| `pg_relation_size(idx)` | main-fork bytes, summed by `stat(2)` over segment files ([dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)) | live, exact |
| `pg_relation_size(idx, 'fsm')` | FSM fork bytes; fork names are `main`/`fsm`/`vm`/`init` ([relpath.c#forkNames](../../../raw/postgres-12/src/common/relpath.c#L26-L38)) | live, exact |
| `pg_class.relpages` / `reltuples` | block and tuple counts as of the last VACUUM/ANALYZE/build ([pg_class.h#relpages](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)) | stale by design |
| `pg_class.reloptions` | the index's `fillfactor`, via `pg_options_to_table` ([pg_proc.dat#pg_options_to_table](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3603-L3608)) | exact |
| `pg_stats.avg_width` / `null_frac` | sampled stored key width and null fraction ([system_views.sql#pg_stats](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L194-L197)) | estimated |
| `pg_stat_all_tables.n_live_tup` / `n_dead_tup` | collector row counts ([system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L566-L572)) | estimated |
| `pg_column_size(col)` | the stored size of an individual datum ([varlena.c#pg_column_size](../../../raw/postgres-12/src/backend/utils/adt/varlena.c#L5044-L5090)) | exact per row |
| `EXPLAIN (ANALYZE, BUFFERS)` | blocks touched per plan node ([explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2907)) | live, exact |
| `VACUUM VERBOSE` | index page count, deleted pages, reusable pages ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827)) | live, exact |

`pg_stat_all_indexes` has no size, page-count or tuple-count column at all — only `idx_scan`, `idx_tup_read`, `idx_tup_fetch` ([system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672)). And `TABLESAMPLE` cannot sample an index: the parser rejects anything that is not a table, matview, or partitioned table ([parse_clause.c#transformRangeTableSample](../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1162-L1169)).

### What bloat is in a v12 B-tree

Two distinct things, and core SQL sees them differently.

1. **Underfilled live leaf pages.** The documentation states the mechanism directly: "if all but a few index keys on a page have been deleted, the page remains allocated" ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874)).
2. **Deleted pages still inside the file.** VACUUM unlinks an entirely-empty page and records it in the FSM ([nbtree.c#btvacuumpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1188)), but the nbtree README is explicit that this "doesn't actually change its state on disk" — the page is handed back out at the next split ([README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L327-L334)). `src/backend/access/nbtree/` contains no call to `RelationTruncate` or `smgrtruncate` and never sets `pages_removed` ([genam.h#IndexBulkDeleteResult](../../../raw/postgres-12/src/include/access/genam.h#L62-L81)), so the file never shrinks without a rebuild.

Both show up as "the file is bigger than the data needs", which is exactly what the model below measures. A leaf-density metric alone sees only the first; the `idx_range` fixture below has a perfectly healthy 89.83% leaf density while 2330 of its 2745 blocks are deleted pages.

### The page-fill model, derived from source

A B-tree page reserves a 24-byte header and 16 bytes of special space, leaving 8152 usable bytes for line pointers plus tuple bodies. That is nbtree's own arithmetic ([nbtsplitloc.c#leftspace](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L155-L158)), built from [bufpage.h#SizeOfPageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L213-L216) and [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68).

One index entry occupies `MAXALIGN(hoff + data) + 4`:

- `hoff` is 8, or 16 when that tuple has any NULL ([itup.h#IndexInfoFindDataOffset](../../../raw/postgres-12/src/include/access/itup.h#L76-L90));
- `data` is `heap_compute_data_size()`, which skips NULL columns entirely and charges no alignment padding to a short varlena ([heaptuple.c#heap_compute_data_size](../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L127-L164));
- the whole thing is MAXALIGN'd once more in [indextuple.c#index_form_tuple](../../../raw/postgres-12/src/backend/access/common/indextuple.c#L121-L135);
- the trailing 4 is `sizeof(ItemIdData)`, the line pointer ([itemid.h#ItemIdData](../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30)).

A sorted build reserves one line pointer for the future high key up front ([nbtsort.c#_bt_blnewpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L621-L646)), then adds entries while `PageGetFreeSpace()` — which already subtracts one line pointer ([bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L580-L597)) — stays at or above `btps_full`. With `k` entries present, free space is therefore `BLCKSZ - 24 - 16 - 4 - 4 - k*slot`, i.e. `8144 - k*slot` at the default block size. Since `btps_full = BLCKSZ * (100 - fillfactor) / 100`, the number of data entries a finished leaf page keeps is:

```text
tuples_per_leaf = floor( (BLCKSZ - 48 - floor(BLCKSZ * (100 - fillfactor) / 100)) / slot )
```

For a `bigint` key at fillfactor 90 that is `floor((8192 - 48 - 819) / 20) = 366`. Internal levels use the fixed `BTREE_NONLEAF_FILLFACTOR` of 70 ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171), [nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L729)), and block 0 is always the metapage ([nbtree.h#BTREE_METAPAGE](../../../raw/postgres-12/src/include/access/nbtree.h#L131-L135)). Expected size is then `1 + ceil(rows / tuples_per_leaf)` plus the internal levels stacked at fanout `floor((BLCKSZ - 48 - floor(BLCKSZ * 30 / 100)) / slot)`.

Measured against the pinned build: predicted 2733 leaf + 11 internal + 1 meta = 2745 blocks for 1,000,000 `bigint` rows; the real index is 2745 blocks with 2733 leaf and 11 internal pages.

### Method A: catalog-only bloat sweep

One statement, no index I/O, no table access. It ran in 26.7 ms over the whole 14-index test database.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

WITH RECURSIVE
idx AS (
    SELECT /* wiki_btree_bloat_sweep */
           c.oid AS idxoid, n.nspname AS schemaname, t.relname AS tablename,
           c.relname AS indexname, t.oid AS tbloid, x.indkey,
           coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'fillfactor'), 90) AS fillfactor,
           CASE WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
                ELSE least(c.reltuples::numeric,
                           coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
           END                                          AS live_rows,
           c.reltuples::numeric                         AS idx_reltuples,
           coalesce(s.n_dead_tup, 0)                    AS tbl_n_dead_tup,
           greatest(s.last_vacuum, s.last_autovacuum)   AS last_vacuum,
           greatest(s.last_analyze, s.last_autoanalyze) AS last_analyze,
           pg_relation_size(c.oid)                      AS actual_bytes,
           pg_relation_size(c.oid, 'fsm')               AS fsm_bytes,
           current_setting('block_size')::int           AS bs
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
           coalesce(se.null_frac, st.null_frac, 0)::numeric                AS null_frac
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
              FROM cols c WHERE c.idxoid = i.idxoid)                       AS p_null
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
               AS int_cap
      FROM sized s
),
levels AS (
    SELECT idxoid, ceil(greatest(live_rows, 0) / leaf_cap) AS pages, int_cap FROM fit
    UNION ALL
    SELECT l.idxoid, ceil(l.pages / l.int_cap), l.int_cap FROM levels l WHERE l.pages > 1
),
modelled AS (
    SELECT f.*, (SELECT sum(pages) FROM levels l WHERE l.idxoid = f.idxoid) + 1
                    AS expected_blocks
      FROM fit f
)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(actual_bytes) AS index_size,
       pg_size_pretty(greatest(actual_bytes - expected_blocks * bs, 0)::bigint) AS wasted,
       round((100 * (1 - (expected_blocks * bs) / greatest(actual_bytes, 1)))::numeric, 1)
           AS bloat_pct,
       fsm_bytes > 0                                  AS has_freed_pages,
       tbl_n_dead_tup                                 AS dead_tuples,
       (last_vacuum IS NULL AND last_analyze IS NULL) AS never_analyzed,
       idx_reltuples::bigint                          AS idx_reltuples,
       live_rows::bigint                              AS modelled_rows
  FROM modelled
 WHERE actual_bytes > 1024 * 1024
 ORDER BY greatest(actual_bytes - expected_blocks * bs, 0) DESC;
```

Design notes that matter:

- **`live_rows` prefers the collector over `pg_class`.** ANALYZE derives an index's `reltuples` from the heap sample (`ceil(tupleFract * totalrows)`, [analyze.c#do_analyze_rel](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)); VACUUM overwrites it with the true count ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1803-L1815)). Neither runs after a bare `DELETE`, so `reltuples` alone reports 0.0% bloat on an index that is 90% reclaimable; see the measured failure below. `pg_stat_all_tables.n_live_tup` tracks the delete immediately, so `least()` of the two recovers the right answer. Partial indexes keep `reltuples`, because `n_live_tup` counts the whole table.
- **`x.indisvalid` filter.** An invalid leftover from a failed concurrent build is not a rebuild candidate; it is a cleanup candidate.
- **`fillfactor` is read per index**, so a `fillfactor=50` index is not reported as 44% bloated.

### Method A-prime: fix the key width with pg_column_size

`pg_stats.avg_width` is a sample mean of the *stored* width ([pg_statistic.h#stawidth](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L39-L49), [analyze.c#compute_scalar_stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L2223-L2231)). Because Method A MAXALIGNs that single average, a key whose per-row width straddles alignment boundaries is mispriced. `pg_column_size` fixes it by aligning each row separately:

```sql
SET statement_timeout = '60s';

SELECT /* wiki_btree_measure_slot */
       count(*)                                                 AS rows_measured,
       avg(ceil((8 + pg_column_size(k)) / 8.0) * 8 + 4)         AS avg_slot_bytes
  FROM t_var TABLESAMPLE BERNOULLI (1) REPEATABLE (42);
```

On the variable-width fixture this returned 56.893 bytes from a 1% sample and 57.000 from a full scan, against Method A's catalog-derived 60. Feeding either value back into the same closed form moved the estimate from −4.64% to +0.32% error. Add one `pg_column_size` term per key column, in `indkey` order, for a multi-column index; NULLs are handled by `hoff`, not by the column term.

### Method B: index-only-scan page census

This measures the live leaf chain directly, with no contrib and no writes, by counting the blocks a full index-only scan touches. Two probes in the same session:

```sql
SET statement_timeout = '5min';
SET enable_seqscan = off;
SET enable_bitmapscan = off;
SET max_parallel_workers_per_gather = 0;

-- (1) full walk of the leaf chain; run twice, use the second reading
EXPLAIN (ANALYZE, BUFFERS, TIMING off, COSTS off) SELECT /* wiki_btree_census_full */ count(*) FROM t_seq;
EXPLAIN (ANALYZE, BUFFERS, TIMING off, COSTS off) SELECT /* wiki_btree_census_full */ count(*) FROM t_seq;
-- (2) descent-only calibration: a key that matches nothing; same session, run twice
EXPLAIN (ANALYZE, BUFFERS, TIMING off, COSTS off) SELECT /* wiki_btree_census_descent */ id FROM t_seq WHERE id = -1;
EXPLAIN (ANALYZE, BUFFERS, TIMING off, COSTS off) SELECT /* wiki_btree_census_descent */ id FROM t_seq WHERE id = -1;
```

Both probes must run in the same session and the second reading of each must be used: on `idx_seq` the calibration query read 6 blocks in a cold backend and 3 once the metapage was cached in `rd_amcache` ([nbtpage.c#_bt_getrootheight](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L585-L615)).

`live_leaf_pages = full_scan_blocks - descent_blocks`. The full scan reads the metapage, one page per internal level, and every live leaf exactly once, because `_bt_readnextpage` follows one right link per `_bt_getbuf` call ([nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1737-L1786)); the calibration probe reads one page per internal level plus one leaf, so the difference is the leaf count. Deleted and half-dead pages are skipped by `P_IGNORE` and are correctly absent from the total. On 12 fixtures with `Heap Fetches: 0`, this reproduced `pgstatindex`'s `leaf_pages` **exactly, every time**, and `total_blocks - leaf_est` matched `internal + deleted + half-dead + metapage` exactly as well.

Two preconditions, both enforced by reading the plan:

- **`Heap Fetches` must be 0.** An index-only scan falls back to the heap for any TID whose page is not all-visible ([nodeIndexonlyscan.c#IndexOnlyNext](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170)), and `BUFFERS` has no per-relation breakdown ([instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33)), so heap blocks silently inflate the count. The measured damage: `idx_churn` reported 8452 leaf pages against a true 3279.
- **The calibration key must match (nearly) nothing.** A probe key with 5000 duplicates read 45 blocks instead of 3 and cost 41 pages of accuracy.

Reconstructed density, which is the core-SQL equivalent of `pgstatindex.avg_leaf_density`:

```text
avg_leaf_density ≈ (rows / live_leaf_pages + 1) * slot_bytes / (BLCKSZ - 40) * 100
```

The `+1` accounts for the high key that every non-rightmost leaf carries ([nbtree.h#P_HIKEY](../../../raw/postgres-12/src/include/access/nbtree.h#L198-L219)), and `BLCKSZ - 40` is the same denominator `pgstatindex` uses ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L308)). Measured error was −0.03 to −0.14 points on 11 of 12 fixtures.

### Method C: CREATE INDEX CONCURRENTLY rebuild probe

Exact by construction: build a sibling with the same definition, read its size, drop it.

```sql
-- Generate the DDL. pg_get_indexdef emits reloptions but omits CONCURRENTLY and the
-- tablespace, so both must be added.
SELECT /* wiki_btree_probe_ddl */
       regexp_replace(pg_get_indexdef(c.oid),
                      '^CREATE (UNIQUE )?INDEX ' || quote_ident(c.relname) || ' ON ',
                      'CREATE \1INDEX CONCURRENTLY wiki_bloat_probe ON ')
       || coalesce(' TABLESPACE ' || quote_ident(ts.spcname), '') AS ddl
  FROM pg_class c
  LEFT JOIN pg_tablespace ts ON ts.oid = nullif(c.reltablespace, 0)
 WHERE c.oid = 'public.idx_seq'::regclass;
```

Then, as three separate top-level statements:

```sql
SET maintenance_work_mem = '256MB';
SET lock_timeout = '5s';
CREATE INDEX CONCURRENTLY wiki_bloat_probe ON public.t_seq USING btree (id);
SELECT /* wiki_btree_probe_result */
       pg_size_pretty(pg_relation_size('public.idx_seq'))          AS live,
       pg_size_pretty(pg_relation_size('wiki_bloat_probe'))        AS fresh,
       pg_size_pretty(pg_relation_size('public.idx_seq')
                      - pg_relation_size('wiki_bloat_probe'))      AS reclaimable;
DROP INDEX CONCURRENTLY wiki_bloat_probe;
```

Constraints and costs, all from source:

- **Three separate statements.** `CREATE INDEX CONCURRENTLY` is rejected inside a transaction block, a subtransaction, or a function ([utility.c#ProcessUtilitySlow](../../../raw/postgres-12/src/backend/tcop/utility.c#L1301-L1310), [xact.c#PreventInTransactionBlock](../../../raw/postgres-12/src/backend/access/transam/xact.c#L3329-L3359)). `DROP INDEX CONCURRENTLY` has the same restriction plus no-CASCADE and one-object rules ([tablecmds.c#RemoveRelations](../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1235-L1253)).
- **Lock level is `ShareUpdateExclusiveLock`** on the table, so DML keeps running ([indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L548-L564)).
- **Not usable on a partitioned table** ([indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L604-L622)), and silently downgraded to a non-concurrent build on temp tables ([indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499)).
- **Failure leaves an invalid index** that still costs write overhead, because `indisvalid` is set non-transactionally as the last step ([index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3314-L3330), [create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596)). Always follow up with a check for a leftover `wiki_bloat_probe`.
- **It costs a full build**: two table scans ([create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L558)), a full-page WAL image per index page whenever `wal_level >= replica` ([nbtsort.c#_bt_blwritepage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L576-L580)), an `smgrimmedsync` of the whole fork ([nbtsort.c#_bt_load](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1288-L1307)), and double disk footprint while it runs. On the 39 MB fixture the probe took 258 ms against 26.7 ms for the whole Method A sweep.
- **The baseline is a fillfactor build, not a maximally packed index** ([nbtsort.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L17-L24)). That is the right baseline for "what would `REINDEX` give back", which is the question a DBA is actually asking.

### Method D: VACUUM VERBOSE page classes

`VACUUM VERBOSE` prints an exact per-index page census with no contrib at all:

```text
INFO:  index "idx_range" now contains 150000 row versions in 2745 pages
DETAIL:  50000 index row versions were removed.
2329 index pages have been deleted, 2192 are currently reusable.
```

The message text and every field come from [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827), fed by `btvacuumscan`'s `num_pages` / `pages_free` ([nbtree.c#btvacuumscan](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1090-L1098)) and `btvacuumpage`'s `pages_deleted` ([nbtree.c#btvacuumpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1188)). The split between "deleted" and "currently reusable" is the recycle horizon in [nbtpage.c#_bt_page_recyclable](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963).

Caveat, observed on the pin: a VACUUM that finds nothing to do prints **no** index line, because `btvacuumcleanup` skips the scan entirely ([nbtree.c#btvacuumcleanup](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L913-L919)). Method D therefore reports only when VACUUM had work to do.

### Two free triage signals

- **`pg_relation_size(idx, 'fsm') > 0`** means VACUUM has recorded at least one recyclable page for that index, since the FSM fork is created by the first `RecordFreeIndexPage` ([nbtree.c#btvacuumpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1188), [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55)). Across the 14 fixtures this was non-zero for exactly the one index that had deleted pages (24576 bytes for `idx_range`, 0 for the other 13). It is a binary flag, not a quantity: its size does not scale with the number of free pages.
- **`pg_stat_all_indexes.idx_scan`** ranks candidates: an index nobody scans should be dropped, not rebuilt ([system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672)).

### Which method to use

| Method | Reads | Writes | Cost on a 39 MB index | Accuracy vs an actual rebuild |
|---|---|---|---|---|
| A: catalog sweep | catalogs only | none | 26.7 ms for 14 indexes | exact on 10/14, ±2 blocks on 3, −4.6% on 1 |
| A′: `pg_column_size` | one table scan or sample | none | 1% sample | fixes the one A failure to +0.32% |
| B: IOS census | the whole index | none | 65 ms | leaf-page count exact on 12/12 |
| C: CIC rebuild | table + writes a new index | yes | 258 ms + 39 MB | exact by definition |
| D: VACUUM VERBOSE | the whole index | yes | a VACUUM | exact, but only when VACUUM works |

Run A continuously; confirm the top offenders with B or C before spending a `REINDEX`.

### Exact-pin measurements

All numbers come from one isolated PostgreSQL 12.2 server built from the pinned checkout, `block_size` 8192, `autovacuum = off`. `pgstattuple` was installed **only to provide ground truth**; nothing in Methods A–D uses it.

Fixtures, with `pgstatindex` ground truth:

| index | how it was made | blocks | leaf | internal | deleted | density |
|---|---|---|---|---|---|---|
| `idx_seq` | 1,000,000 sequential `bigint`, built after load | 2745 | 2733 | 11 | 0 | 90.06 |
| `idx_uuid` | 400,000 `uuid` | 1543 | 1533 | 9 | 0 | 90.01 |
| `idx_text` | 500,000 `md5()` `text` | 3607 | 3572 | 34 | 0 | 89.98 |
| `idx_multi` | 500,000 `(int, text)` | 3606 | 3572 | 33 | 0 | 89.96 |
| `idx_var` | 400,000 `text`, 2-81 chars | 3169 | 3126 | 42 | 0 | 90.43 |
| `idx_ff50` | 1,000,000 sequential, `fillfactor=50` | 4971 | 4951 | 19 | 0 | 49.85 |
| `idx_null` | 1,000,000 `bigint`, 25% NULL | 2746 | 2733 | 12 | 0 | 90.09 |
| `idx_part` | partial index over 50,000 of 1,000,000 rows | 139 | 137 | 1 | 0 | 89.83 |
| `idx_dup` | 500,000 identical keys inserted into an existing index | 1291 | 1283 | 7 | 0 | 96.00 |
| `idx_rand` | 1,000,000 random keys inserted into an existing index | 3758 | 3745 | 12 | 0 | 65.81 |
| `idx_del` | 1,000,000 then 90% scattered `DELETE`, VACUUM | 2745 | 2733 | 11 | 0 | 9.27 |
| `idx_range` | 1,000,000 then a contiguous 85% `DELETE`, VACUUM | 2745 | 411 | 3 | 2330 | 89.83 |
| `idx_churn` | 300,000 rows updated twice, never vacuumed | 3293 | 3279 | 13 | 0 | 67.63 |
| `idx_stale` | 1,000,000 then 90% `DELETE`, no VACUUM, no ANALYZE | 2745 | 2733 | 11 | 0 | 90.06 |
| `idx_empty` | empty table | 1 | 0 | 0 | 0 | `NaN` |

`idx_empty` is a metapage-only index; the Method A sweep's `actual_bytes > 1024 * 1024` filter excludes it, and it is left out of the accuracy tables below. Its modelled size is 1 block, which is correct.

Method A against the Method C rebuild, in blocks:

| index | live | rebuilt (exact) | Method A model | model − rebuilt | model bloat % | true bloat % |
|---|---|---|---|---|---|---|
| `idx_seq` | 2745 | 2745 | 2745 | 0 | 0.0 | 0.0 |
| `idx_uuid` | 1543 | 1543 | 1543 | 0 | 0.0 | 0.0 |
| `idx_text` | 3607 | 3607 | 3607 | 0 | 0.0 | 0.0 |
| `idx_ff50` | 4971 | 4971 | 4971 | 0 | 0.0 | 0.0 |
| `idx_part` | 139 | 139 | 139 | 0 | 0.0 | 0.0 |
| `idx_multi` | 3606 | 3606 | 3607 | +1 | 0.0 | 0.0 |
| `idx_null` | 2746 | 2746 | 2745 | −1 | 0.0 | 0.0 |
| `idx_rand` | 3758 | 2745 | 2745 | 0 | 27.0 | 27.0 |
| `idx_churn` | 3293 | 825 | 825 | 0 | 74.9 | 74.9 |
| `idx_range` | 2745 | 414 | 414 | 0 | 84.9 | 84.9 |
| `idx_del` | 2745 | 276 | 276 | 0 | 89.9 | 89.9 |
| `idx_stale` | 2745 | 276 | 276 | 0 | 89.9 | 89.9 |
| `idx_dup` | 1291 | 1376 | 1374 | −2 | −6.4 | −6.6 |
| `idx_var` | 3169 | 3169 | 3316 | +147 | −4.6 | 0.0 |

Method B against `pgstatindex`:

| index | full-scan blocks | descent blocks | leaf estimate | true leaf | error | density estimate | `pgstatindex` |
|---|---|---|---|---|---|---|---|
| `idx_seq` | 2736 | 3 | 2733 | 2733 | 0 | 90.01 | 90.06 |
| `idx_del` | 2736 | 3 | 2733 | 2733 | 0 | 9.22 | 9.27 |
| `idx_range` | 414 | 3 | 411 | 411 | 0 | 89.78 | 89.83 |
| `idx_rand` | 3748 | 3 | 3745 | 3745 | 0 | 65.76 | 65.81 |
| `idx_text` | 3575 | 3 | 3572 | 3572 | 0 | 89.93 | 89.98 |
| `idx_multi` | 3575 | 3 | 3572 | 3572 | 0 | 89.93 | 89.96 |
| `idx_uuid` | 1536 | 3 | 1533 | 1533 | 0 | 89.96 | 90.01 |
| `idx_dup` | 1286 | 3 | 1283 | 1283 | 0 | 95.86 | 96.00 |
| `idx_ff50` | 4954 | 3 | 4951 | 4951 | 0 | 49.80 | 49.85 |
| `idx_null` | 2736 | 3 | 2733 | 2733 | 0 | 90.01 | 90.09 |
| `idx_part` | 139 | 2 | 137 | 137 | 0 | 89.78 | 89.83 |
| `idx_var` | 3129 | 3 | 3126 | 3126 | 0 | 94.92 | 90.43 |

### Measured failure modes

- **Stale relstats hide real bloat.** After deleting 900,000 of 1,000,000 rows with no VACUUM and no ANALYZE, `pg_class.reltuples` still read `1e+06` for both table and index, and a `reltuples`-only model reported **0.0% bloat** against a true reclaimable 2745 → 276 blocks. `pg_stat_all_tables` knew: `n_live_tup = 100000`, `n_dead_tup = 900000`. Using `least(reltuples, n_live_tup)` produced exactly 276 blocks, matching the rebuild.
- **A rebuild can make an index bigger.** `idx_dup` (500,000 identical keys) sits at 96.00% density because an all-duplicate split uses `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L412-L422)), while a fresh sorted build packs to fillfactor 90. Rebuilding grew it from 1291 to 1376 blocks. Method A predicted this correctly as −6.4%: **negative bloat is a real reading and means "do not rebuild".**
- **Random-insert indexes report ~27% bloat forever.** `idx_rand` reached 65.81% density with zero deletions, because non-rightmost leaf splits aim for an even 50/50 division rather than fillfactor ([nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L275-L331)). The 27.0% is genuinely reclaimable — the rebuild returned 3758 → 2745 blocks — but the index will drift straight back. This is a workload property, not a maintenance failure.
- **Highly variable key widths mislead Method A by ~5%.** `idx_var` was reported at −4.6% against a true 0.0%, because a single MAXALIGN of the average width is not the average of per-row MAXALIGNs. Method A′ reduced this to +0.32%.
- **Leaf density alone misses deleted pages entirely.** `idx_range` shows a healthy 89.83% `avg_leaf_density` while 2330 of 2745 blocks are deleted; Method A reported 84.9% bloat and the rebuild confirmed 414 blocks.
- **An index-only scan with heap fetches destroys Method B.** `idx_churn` (never vacuumed, 300,000 heap fetches) yielded 8452 estimated leaf pages against a true 3279.

### Settings and apply scopes

| Setting | Context in v12 | Apply scope |
|---|---|---|
| `statement_timeout` | `PGC_USERSET` ([guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)) | session/transaction |
| `lock_timeout` | `PGC_USERSET` ([guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2393)) | session/transaction |
| `maintenance_work_mem` | `PGC_USERSET` ([guc.c#maintenance_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2252)) | session/transaction |
| `enable_seqscan`, `enable_bitmapscan`, `max_parallel_workers_per_gather` | `PGC_USERSET` | session/transaction |
| `block_size` | `PGC_INTERNAL` preset ([guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2884)) | read-only; always read it with `current_setting('block_size')` rather than hard-coding 8192 |
| `wal_level` | `PGC_POSTMASTER` ([guc.c#wal_level](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4409-L4417)) | restart; only relevant because it decides whether Method C WAL-logs every built page |

No method here requires changing a setting that needs a reload or a restart.

### What no core-SQL method can measure

- **`leaf_fragmentation`.** It counts leaves whose right link points to a lower block number ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L308)), which requires reading page headers in physical order. Nothing in core exposes that.
- **`LP_DEAD` space inside live leaves.** Entries killed by index scans still occupy their slots until the next split or VACUUM; Method B counts only returned rows, so it reports *live* density, which is why its numbers sit slightly below `pgstatindex`'s physical density.
- **The split between half-dead and deleted pages.** Methods A and B lump both into "not in the live leaf chain"; only `VACUUM VERBOSE` separates deleted from currently-reusable.
- **Per-page detail of any kind.** There is no core page reader, so no core method can say *which* pages are empty.

### Follow-up: the comparison fixture matrix

The tables above use one fixture per shape. To answer "what is the error, per bloat type, for partial and non-partial indexes", a second exact-pin run built a full matrix: **9 bloat types × 3 scales × {non-partial, partial} = 54 indexes over 27 tables**, on a freshly built 12.2 server from the same pin.

Each table carries two indexes over the same key, so both see the identical churn:

```sql
CREATE TABLE m_<type>_<scale> (id bigint, flag boolean, pad text);
CREATE INDEX m_<type>_<scale>_full ON m_<type>_<scale> (id);
CREATE INDEX m_<type>_<scale>_part ON m_<type>_<scale> (id) WHERE flag;
```

`flag` is `id % 5 = 0`, so the partial index holds 20% of the rows. Every delete pattern uses modulus 7 or 11, which are coprime with 5, so the partial index loses the same *proportion* of its entries as the non-partial one rather than all or none of them. Scales are 200,000 / 500,000 / 1,000,000 rows, which is the repeat dimension: each reported error is the worst of three independent sizes.

| type | recipe | resulting shape |
|---|---|---|
| `fresh` | build after load, VACUUM | control, no bloat |
| `scatter` | delete 6 of every 7 rows, VACUUM | underfilled leaves, no page emptied |
| `range` | delete the first 85% of the key range, VACUUM twice | mostly deleted pages, healthy density |
| `random` | insert into an already-existing index in random key order | 50/50 split fill |
| `dup` | insert N identical keys into an existing index | 96% single-value fill |
| `churn_vac` | two full-table `UPDATE`s, then VACUUM | 4x size, low density |
| `churn_unvac` | two full-table `UPDATE`s, no VACUUM | dead entries still indexed |
| `lpdead` | delete 10 of every 11 rows, no VACUUM, then index scans that set `LP_DEAD` | killed-but-present entries |
| `stale` | delete 10 of every 11 rows, no VACUUM, no ANALYZE | stale catalogs |

Ground truth per index is the Method C rebuild (exact reclaimable size) plus `pgstatindex` (exact page classes). Method A was executed by running the filed sweep query's own CTE chain, with only the `actual_bytes > 1024 * 1024` triage filter removed so that sub-megabyte partial indexes are included.

### Follow-up: which pgstatindex columns core SQL reproduces

Restricting to the columns that bear on bloat:

| `pgstatindex` column | core-SQL equivalent | measured agreement |
|---|---|---|
| `index_size` | `pg_relation_size(idx)` | identical on all 54 indexes |
| `leaf_pages` | Method B: `full_scan_blocks - descent_blocks` | **exact on all 36 eligible cells**; unavailable on the 18 cells with heap fetches |
| `avg_leaf_density` | `(rows / leaf_est + 1) * slot / (BLCKSZ - 40)` | −0.04 to −0.05 points on 30 cells, −0.14 to −0.15 on the 6 `dup` cells |
| `deleted_pages` | `blocks - leaf_est - (modelled internal + 1)` | within 1 page on all 6 `range` cells (2330 against 2330 at the top scale); a +1 to +28 residual elsewhere, equal to the internal-page modelling error |
| `empty_pages` (half-dead) | none | not separable from deleted pages |
| `internal_pages`, `tree_level`, `root_block_no` | none | no core equivalent |
| `leaf_fragmentation` | none | needs physical page order; no core equivalent |

The non-leaf total is recovered exactly: `blocks - leaf_est` equalled `internal_pages + deleted_pages + empty_pages + 1` in **all 36 eligible cells**, including the `range` cells carrying 2330 deleted pages.

### Follow-up: error by bloat type, partial and non-partial

Method A's modelled block count against the Method C rebuild, worst of the three scales:

| bloat type | non-partial: worst Δblocks / Δpoints | exact cells | partial: worst Δblocks / Δpoints | exact cells |
|---|---|---|---|---|
| `fresh` | 0 / 0.0 | 3/3 | 0 / 0.0 | 3/3 |
| `scatter` | 0 / 0.0 | 3/3 | 0 / 0.0 | 3/3 |
| `range` | 0 / 0.0 | 3/3 | 0 / 0.0 | 3/3 |
| `random` | 0 / 0.0 | 3/3 | 0 / 0.0 | 3/3 |
| `dup` | 4 / 0.2 | 0/3 | 2 / 0.8 | 1/3 |
| `churn_vac` | 5 / 0.0 | 2/3 | 0 / 0.0 | 3/3 |
| `churn_unvac` | 2 / 0.0 | 2/3 | 16 / 0.7 | 1/3 |
| `lpdead` | 0 / 0.0 | 3/3 | **510 / 92.6** | 0/3 |
| `stale` | 0 / 0.0 | 3/3 | **510 / 92.6** | 0/3 |

Across all 54 cells the model was exact on 39, within 5 blocks on 47, within 16 blocks on 48, and wrong by two orders of magnitude on the remaining 6 — all of which are partial indexes on the two fixtures that were never vacuumed *and* never analysed after the delete.

Method B against `pgstatindex`, same matrix:

| bloat type | eligible cells | `leaf_pages` error | density error (points) |
|---|---|---|---|
| `fresh`, `scatter`, `range`, `random`, `churn_vac` | 30 of 30 | 0 in every cell | −0.04 to −0.05 |
| `dup` | 6 of 6 | 0 in every cell | −0.14 to −0.15 |
| `churn_unvac`, `lpdead`, `stale` | 0 of 18 | precondition failed | precondition failed |

Partial-ness does not affect Method B at all: the partial cells were exact wherever the non-partial ones were. The `dup` cells carry the larger density error because a pivot tuple over an all-duplicate key must absorb a heap TID ([nbtutils.c#_bt_truncate](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L2082-L2093)), so the single high key the formula adds is 24 bytes rather than the 20-byte `slot` it assumes.

The 18 ineligible cells are exactly the three fixtures whose tables were never vacuumed, so the visibility map is unset and the index-only scan reaches the heap. The census overstated leaf pages by 2 to 10 times there (21,239 against 10,929 at worst) and the `Heap Fetches` check caught every one.

### Follow-up: an avg_leaf_density predictor head to head

The obvious way to use `pgstatindex` for this job — and what the companion [bloat heuristic page](index-bloat-heuristic.md) does — is to compare leaf density with the configured fillfactor. As a size predictor that is:

```text
pgstatindex_expected_blocks = ceil(leaf_pages * avg_leaf_density / fillfactor) + internal_pages + 1
```

Scored against the same rebuilds, worst error per bloat type as a percentage of the rebuilt size:

| bloat type | kind | `avg_leaf_density` predictor | core-SQL model |
|---|---|---|---|
| `fresh` | non-partial / partial | 0.1% / 0.0% | 0.0% / 0.0% |
| `scatter` | non-partial / partial | 4.5% / 3.7% | 0.0% / 0.0% |
| `range` | non-partial / partial | 2.4% / 2.4% | 0.0% / 0.0% |
| `random` | non-partial / partial | 0.4% / 0.4% | 0.0% / 0.0% |
| `dup` | non-partial / partial | 0.2% / 0.7% | 0.2% / 0.7% |
| `churn_vac` | non-partial / partial | 2.2% / 2.7% | 0.2% / 0.0% |
| `churn_unvac` | non-partial / partial | 200.7% / 200.0% | 0.1% / 2.9% |
| `lpdead` | non-partial / partial | 994.4% / 959.6% | 0.0% / 980.8% |
| `stale` | non-partial / partial | 994.4% / 959.6% | 0.0% / 980.8% |

The core-SQL model wins or ties in 16 of the 18 cells, and the two it loses are the partial-index `lpdead`/`stale` cells where both predictors are useless.

The reason is structural, not incidental: `pgstatindex` measures *physical* occupancy, and an entry belonging to a dead heap row occupies its slot exactly like a live one. `pgstatindex_impl` derives density from `PageGetFreeSpace()` ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L308)), which counts `LP_DEAD` items and not-yet-vacuumed entries as used. So on the `lpdead` fixture at the top scale `pgstatindex` reports `avg_leaf_density = 90.06` on a 2745-block index — indistinguishable from the healthy control — while a rebuild returns 251 blocks, reclaiming 90.9% of the file. A DBA reading only `avg_leaf_density` would rank that index as the healthiest in the database.

This is a like-for-like comparison of the two as *rebuild-size predictors*, not a claim that `pgstatindex` is inaccurate. It reports physical occupancy correctly; physical occupancy just is not reclaimable space. The fix for a `pgstatindex` user is the same live row count the core-SQL model already uses, which is why having the extension installed does not, by itself, improve the estimate.

What `pgstatindex` still gives that no core method matches: the exact deleted-versus-half-dead split, `leaf_fragmentation`, `tree_level`, and a leaf count that needs no visibility-map cooperation and no live row count at all.

### Follow-up: the partial-index failure and its fix

The single large error mode in the whole matrix has one cause. Method A's `live_rows` falls back to `pg_class.reltuples` for a partial index, because `pg_stat_all_tables.n_live_tup` counts the whole table and not the predicate subset:

```sql
CASE WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
     ELSE least(c.reltuples::numeric,
                coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
END AS live_rows
```

`reltuples` for an index is refreshed only by VACUUM, which writes the true count ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1803-L1815)), or by a standalone ANALYZE, which writes `ceil(tupleFract * totalrows)` from the heap sample ([analyze.c#do_analyze_rel](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)). A bare `DELETE` triggers neither, so on the `stale` and `lpdead` fixtures the partial index still claimed its pre-delete row count and the model reported 0.0% to −2.0% bloat against a true 89.3% to 90.6%. The non-partial index on the very same table was exact, because the collector's `n_live_tup` had already tracked the delete.

The failure announces itself in the sweep's own output — `dead_tuples` read 909,091 and `modelled_rows` equalled the pre-delete `idx_reltuples` — and one plain `ANALYZE` repairs it. Re-running Method A on all six failing cells after `ANALYZE`:

| cell | live blocks | rebuilt | model before ANALYZE | model after | Δ after |
|---|---|---|---|---|---|
| `stale` partial 200k | 112 | 12 | 112 | 12 | 0 |
| `stale` partial 500k | 276 | 27 | 275 | 28 | +1 |
| `stale` partial 1000k | 551 | 52 | 562 | 53 | +1 |
| `lpdead` partial 200k | 112 | 12 | 110 | 12 | 0 |
| `lpdead` partial 500k | 276 | 27 | 275 | 27 | 0 |
| `lpdead` partial 1000k | 551 | 52 | 562 | 53 | +1 |

The worst error drops from 510 blocks to 1. The residual ±1 block is ANALYZE's sampling: it recorded 18,337 index rows against a true 18,181 for the largest partial index, because `tupleFract * totalrows` is an estimate rather than a count.

Practical rule: **treat a partial index whose table shows dead tuples or no recent analyze as unmeasured, not as unbloated.** Run `ANALYZE` first, or fall back to the Method C rebuild probe, which needs no statistics at all.

## Context Reviewed

- nbtree build and split: `nbtsort.c` (`_bt_pagestate`, `_bt_blnewpage`, `_bt_buildadd`, `_bt_load`), `nbtsplitloc.c` (`_bt_findsplitloc`, `_bt_deltasortsplits`, `_bt_afternewitemoff`), `nbtinsert.c`, `nbtree.h` fillfactor and page-flag constants, `README`.
- Page and tuple layout: `bufpage.h`, `bufpage.c` (`PageInit`, `PageGetFreeSpace`, `PageAddItemExtended`), `itemid.h`, `itup.h`, `indextuple.c` (`index_form_tuple`, `index_truncate_tuple`), `heaptuple.c` (`heap_compute_data_size`), `c.h` MAXALIGN.
- VACUUM and page recycling: `nbtree.c` (`btbulkdelete`, `btvacuumcleanup`, `btvacuumscan`, `btvacuumpage`), `nbtpage.c` (`_bt_pagedel`, `_bt_page_recyclable`, `_bt_getbuf`), `indexfsm.c`, `genam.h` (`IndexBulkDeleteResult`), `vacuumlazy.c` (`lazy_vacuum_index`, `lazy_cleanup_index`), `vacuum.c` (`vac_update_relstats`).
- Core SQL surfaces: `pg_proc.dat`, `dbsize.c`, `relpath.c`/`relpath.h`, `varlena.c` (`pg_column_size`), `tuptoaster.c` (`toast_datum_size`), `system_views.sql`, `pg_class.h`, `pg_statistic.h`, `analyze.c`, `index.c` (`index_update_stats`), `pgstatfuncs.c`, `pgstat.c`, `parse_clause.c` (TABLESAMPLE), `guc.c`.
- Executor and EXPLAIN: `nodeIndexonlyscan.c`, `nbtsearch.c` (`_bt_steppage`, `_bt_readnextpage`, `_bt_endpoint`), `instrument.h`/`instrument.c`, `explain.c`, `explain.sgml`.
- Rebuild path: `indexcmds.c` (`DefineIndex`), `index.c` (`index_set_state_flags`), `tablecmds.c` (`RemoveRelations`, `RenameRelation`), `utility.c`, `xact.c`, `ruleutils.c` (`pg_get_indexdef`), `create_index.sgml`, `reindex.sgml`, `maintenance.sgml`.
- Contrib boundary: `pgstattuple.control`, `pgstattuple--1.4--1.5.sql`, `pgstatindex.c`, `pageinspect.control`, `rawpage.c`, `btreefuncs.c`, `pg_freespacemap.control`, `amcheck.control`, `extension.c`, `pgstattuple.sgml`, `pageinspect.sgml`, `contrib.sgml`.
- Exact-pin execution: one isolated 12.2 server built from the pinned checkout under `.wiki-runtime/`, 15 fixtures, Methods A–D executed against each, with `pgstattuple`/`pageinspect` installed solely as ground truth. Test objects were dropped and the server was stopped afterwards.
- Follow-up exact-pin execution: a second isolated 12.2 server from the same pin, carrying a 9-bloat-type × 3-scale × partial/non-partial matrix (54 indexes over 27 tables). Per index it collected `pgstatindex` ground truth, the filed Method A model run from the page's own CTE chain, the Method B census with its `Heap Fetches` precondition, and a Method C rebuild, then re-ran Method A on the six failing partial cells after a plain `ANALYZE`. Test objects were dropped and the server was stopped afterwards.

## Evidence Map

| Claim | Evidence |
|---|---|
| Core has no page/FSM reader; the tools are contrib and superuser-gated | [extension.c#read_extension_control_file](../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625), [extension.c#execute_extension_script](../../../raw/postgres-12/src/backend/commands/extension.c#L798-L817), [pageinspect.sgml](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14), [pgstattuple--1.4--1.5.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92) |
| `pg_relation_size` is a live filesystem measurement of one fork | [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336), [relpath.c#forkNames](../../../raw/postgres-12/src/common/relpath.c#L26-L38) |
| 8152 usable bytes per B-tree page | [nbtsplitloc.c#leftspace](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L155-L158), [bufpage.h#SizeOfPageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L213-L216), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68) |
| Entry cost is `MAXALIGN(hoff + data) + 4`; NULLs add 8 to `hoff` and drop their data | [itup.h#IndexInfoFindDataOffset](../../../raw/postgres-12/src/include/access/itup.h#L76-L90), [indextuple.c#index_form_tuple](../../../raw/postgres-12/src/backend/access/common/indextuple.c#L121-L135), [heaptuple.c#heap_compute_data_size](../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L127-L164), [itemid.h#ItemIdData](../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30) |
| Build fills a leaf until free space drops below `BLCKSZ*(100-ff)/100`; one line pointer is pre-reserved | [nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L729), [nbtsort.c#_bt_blnewpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L621-L646), [nbtsort.c#_bt_buildadd](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L856-L899), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L580-L597), [rel.h#RelationGetTargetPageFreeSpace](../../../raw/postgres-12/src/include/utils/rel.h#L304-L309) |
| Internal levels use fillfactor 70; all-duplicate leaf splits use 96 | [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171), [nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L412-L422) |
| Non-rightmost leaf splits target 50/50, so random inserts settle well below fillfactor | [nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L275-L331) |
| Deleted pages stay in the file and are only recorded in the FSM | [README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L327-L334), [nbtree.c#btvacuumpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1188), [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55) |
| Partly-emptied pages remain allocated, which is the documented bloat mechanism | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874) |
| Index `reltuples` comes from the heap sample under ANALYZE and the true count under VACUUM | [analyze.c#do_analyze_rel](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629), [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1803-L1815), [vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1181-L1196) |
| `relpages`/`reltuples` are documented planner estimates | [pg_class.h#relpages](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63), [catalogs.sgml#pg_class](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1782) |
| `n_live_tup`/`n_dead_tup` come from the collector | [system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L566-L572), [pgstatfuncs.c#pg_stat_get_live_tuples](../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L151-L180) |
| `avg_width` is a sampled, post-TOAST stored width | [pg_statistic.h#stawidth](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L39-L49), [analyze.c#compute_scalar_stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L2223-L2231) |
| `pg_column_size` returns the stored datum size | [varlena.c#pg_column_size](../../../raw/postgres-12/src/backend/utils/adt/varlena.c#L5044-L5090), [tuptoaster.c#toast_datum_size](../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L406-L458) |
| A forward index scan reads one buffer per right link | [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1737-L1786) |
| `BUFFERS` is a single per-node counter with no per-relation split, inclusive of children | [instrument.h#BufferUsage](../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33), [instrument.c#InstrStopNode](../../../raw/postgres-12/src/backend/executor/instrument.c#L74-L106), [explain.sgml#BUFFERS](../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L167-L193) |
| Index-only scans fall back to the heap when the VM bit is unset, and report `Heap Fetches` | [nodeIndexonlyscan.c#IndexOnlyNext](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170), [explain.c#ExplainNode](../../../raw/postgres-12/src/backend/commands/explain.c#L1594-L1609) |
| `pgstatindex`'s density denominator and fragmentation definition | [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L308), [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356) |
| `VACUUM VERBOSE` index message and its fields | [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827), [nbtree.c#btvacuumscan](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1090-L1098) |
| A no-op VACUUM prints no index line | [nbtree.c#btvacuumcleanup](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L913-L919) |
| CIC restrictions, lock level, invalid-index leftover, and build cost | [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L548-L564), [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L604-L622), [utility.c#ProcessUtilitySlow](../../../raw/postgres-12/src/backend/tcop/utility.c#L1301-L1310), [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3314-L3330), [create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596), [nbtsort.c#_bt_blwritepage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L576-L580) |
| `pg_get_indexdef` emits reloptions but not the tablespace or `CONCURRENTLY` | [ruleutils.c#pg_get_indexdef](../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1088-L1115), [ruleutils.c#pg_get_indexdef_worker](../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1410-L1439) |
| `TABLESAMPLE` cannot be applied to an index | [parse_clause.c#transformRangeTableSample](../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1162-L1169) |

## Open Questions

- **Why a cold-session descent probe reads more blocks than a warm one.** On `idx_seq` the calibration query read 6 blocks in a fresh backend and 3 in the same session after one prior execution, while `idx_part` read 2 in both. `_bt_getroot` caches metapage contents in `rd_amcache`, which explains at most one block. The remaining difference was not traced to a specific call site, so Method B is specified as "run both probes twice in one session and use the second reading".
- **The `+1` in the density formula assumes every counted leaf carries a high key.** The rightmost leaf does not ([nbtree.h#P_HIKEY](../../../raw/postgres-12/src/include/access/nbtree.h#L198-L219)). The error is one entry in the whole index and is inside the measured −0.03 to −0.14 point spread, but it was not isolated separately.
- **Internal-page counts are modelled with untruncated pivot tuples.** `_bt_truncate` can shrink a pivot when the leading key columns already distinguish the boundary ([nbtutils.c#_bt_truncate](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L2082-L2093)), so Method A over- or under-counts internal pages by a few. Observed effect: +1 block on `idx_multi`, −1 on `idx_null`, −2 on `idx_dup`, and roughly −10 on the corrected `idx_var` estimate.
- **`pg_column_size` and out-of-line TOAST.** It reports the external compressed payload size for an on-disk-external datum ([tuptoaster.c#toast_datum_size](../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L406-L458)) whereas `index_form_tuple` fetches and inlines that value ([indextuple.c#index_form_tuple](../../../raw/postgres-12/src/backend/access/common/indextuple.c#L66-L110)). Keys wide enough to be externalised would exceed `BTMaxItemSize` anyway, but no fixture exercised the boundary.
- **No direct upstream test covers these estimates.** The pinned tree has no regression test that compares a modelled index size against a built one, and `pgstatindex` itself has limited coverage; the accuracy claims here rest entirely on the exact-pin fixtures described above.
- **Block sizes other than 8192 were not exercised.** The formulas read `current_setting('block_size')`, but every measurement used the default 8192, and `MAXALIGN` was assumed to be 8.
- **Half-dead pages were never produced.** No fixture in either run reached a non-zero `empty_pages`, because a half-dead page requires page deletion to be interrupted between `_bt_mark_page_halfdead` and `_bt_unlink_halfdead_page` ([nbtpage.c#_bt_pagedel](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1462-L1490)). The claim that Methods A and B lump half-dead pages in with deleted ones follows from the code path, not from measurement.
- **The `avg_leaf_density` predictor formula is this page's construction.** `ceil(leaf_pages * avg_leaf_density / fillfactor) + internal_pages + 1` is the natural reading of the density-versus-fillfactor heuristic as a size predictor, but upstream defines no such formula, so the head-to-head scores it against a plausible interpretation rather than a specified one.
- **The matrix varies scale, not seed.** The three repeats per cell are 200,000 / 500,000 / 1,000,000 rows with deterministic delete patterns; only the `random` fixture draws random values, and it used a single fixed `setseed(0.42)`. Error spread across different random draws at one scale was not measured.
- **The post-`ANALYZE` residual was not chased further.** Three of the six repaired cells still differ by one block, tracked to `tupleFract * totalrows` recording 18,337 index rows against a true 18,181; whether a higher `default_statistics_target` removes it was not tested.

## Source References

- [nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L709-L734)
- [nbtsort.c#_bt_buildadd](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L836-L940)
- [nbtsplitloc.c#_bt_findsplitloc](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L126-L430)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171)
- [nbtree.c#btvacuumscan](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L956-L1099)
- [nbtree.c#btvacuumpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1188)
- [nbtpage.c#_bt_page_recyclable](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963)
- [README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L316-L334)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)
- [itup.h#IndexInfoFindDataOffset](../../../raw/postgres-12/src/include/access/itup.h#L76-L90)
- [indextuple.c#index_form_tuple](../../../raw/postgres-12/src/backend/access/common/indextuple.c#L39-L189)
- [heaptuple.c#heap_compute_data_size](../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L114-L167)
- [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)
- [varlena.c#pg_column_size](../../../raw/postgres-12/src/backend/utils/adt/varlena.c#L5044-L5090)
- [system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672)
- [analyze.c#do_analyze_rel](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)
- [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1771-L1830)
- [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1726-L1800)
- [nodeIndexonlyscan.c#IndexOnlyNext](../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L118-L184)
- [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2907)
- [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L622)
- [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3314-L3403)
- [ruleutils.c#pg_get_indexdef](../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1088-L1115)
- [extension.c#execute_extension_script](../../../raw/postgres-12/src/backend/commands/extension.c#L798-L817)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)
- [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L897)
- [create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L530-L633)
- [explain.sgml#BUFFERS](../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L167-L193)

## Navigation

- [v12/index](../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../codebase-navigation-guide.md)
- [A Heuristic to Detect B-Tree Index Bloat in PostgreSQL 12 (unverified)](index-bloat-heuristic.md) - the `pgstatindex`-based counterpart to this page.
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [How a GIN Index Becomes Bloated in PostgreSQL 12, and How to Measure It (unverified)](gin-index-bloat.md) - the GIN core-SQL-only analogue.
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](create-index-concurrently.md)
- [versions](../../versions.md)
