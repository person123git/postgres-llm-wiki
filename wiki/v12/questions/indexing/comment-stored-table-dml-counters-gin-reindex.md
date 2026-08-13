---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in PostgreSQL 12? (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What the table counters mean](#what-the-table-counters-mean)
  - [Why GIN has no row-count mapping](#why-gin-has-no-row-count-mapping)
  - [Exact-pin counterexamples](#exact-pin-counterexamples)
  - [A safer 40% inspection score](#a-safer-40-inspection-score)
  - [Capture and recapture the baseline](#capture-and-recapture-the-baseline)
  - [Detect the threshold](#detect-the-threshold)
  - [Inspect before REINDEX](#inspect-before-reindex)
  - [Lifecycle and failure boundaries](#lifecycle-and-failure-boundaries)
  - [Caller data-structure build extension and test boundaries](#caller-data-structure-build-extension-and-test-boundaries)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, can I store a GIN index’s table-level `pg_stat_user_tables` counters (`n_tup_ins`, `n_tup_upd`, and `n_tup_del`) in the index comment, use subsequent changes to estimate index churn, and trigger `REINDEX` when churn reaches 40%?

## Answer

### Verdict

You can store a baseline in the GIN index's comment, but the three table counters cannot measure that index's churn or rebuild-reclaimable space. Do **not** make 40% an automatic `REINDEX` trigger. Use it only to schedule a GIN-specific inspection.

`pg_stat_user_tables` obtains `n_tup_ins`, `n_tup_upd`, and `n_tup_del` from functions keyed by the **table OID**, then filters out system schemas. The view does not split those counters by index ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [system_views.sql#pg_stat_user_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L613-L621)). PostgreSQL 12's per-index statistics view exposes scans and tuples read or fetched, not per-index insert, update, or delete counters ([system_views.sql#pg_stat_all_indexes](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L682)).

The original proposal also lacks a denominator. This page interprets “40%” as:

```text
original_activity =
    delta_ins + delta_upd + delta_del

original_activity_pct =
    100 * original_activity / baseline_table_rows
```

That arithmetic is valid, but its meaning is only “attempted table row activity relative to a stored row estimate.” It is not “40% of this GIN index is churned.”

| Decision | Safe interpretation |
|---|---|
| Store the baseline in `COMMENT ON INDEX` | Technically valid if the comment namespace is reserved and lifecycle guards are stored with the counters. |
| Use only `n_tup_ins`, `n_tup_upd`, and `n_tup_del` | Rejected. At minimum, also store `n_tup_hot_upd`, a fixed baseline row estimate, the table OID, and the database statistics-reset timestamp. |
| Fire at 40% | Valid only as “inspect this GIN index now.” |
| Run `REINDEX` automatically at 40% | Rejected. Exact-pin tests below produce both a false positive and a false negative. |

PostgreSQL's own PostgreSQL 12 maintenance documentation says non-B-tree bloat was not well researched and recommends monitoring physical index size; it supplies no DML percentage or GIN rebuild threshold ([maintenance.sgml#routine-reindex](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L880)). The generic `REINDEX` reasons likewise describe corruption, empty or poorly used pages, changed storage parameters, and invalid concurrent-build leftovers without defining a percentage ([reindex.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L31-L75)).

### What the table counters mean

The counter names understate several important boundaries:

- `n_tup_ins`, `n_tup_upd`, `n_tup_del`, and `n_tup_hot_upd` count **attempted** heap actions even if the transaction or subtransaction aborts. Commit status instead changes the live/dead tuple deltas ([pgstat.h#PgStat_TableCounts](../../../../raw/postgres-12/src/include/pgstat.h#L77-L117), [pgstat.c#AtEOXact_PgStat](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L2087-L2171), [pgstat.c#AtEOSubXact_PgStat](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L2247-L2263)).
- `n_tup_upd` includes heap-only tuple (HOT) updates. HOT means PostgreSQL creates no separate index entry; `n_tup_hot_upd` is the counter that identifies those updates ([monitoring.sgml#pg_stat_all_tables-counters](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2752-L2775)). The heap access method returns `update_indexes = false` for a HOT tuple, and the executor calls `ExecInsertIndexTuples` only when that flag is true ([heapam_handler.c#heapam_tuple_update](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L318-L349), [nodeModifyTable.c#index-update-gate](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L1445-L1448)).
- A non-HOT update visits each ready index, but a partial-index predicate can skip that index's insertion. One table counter can therefore represent different work for a full GIN index, a partial GIN index, and another index on the same table ([execIndexing.c#ExecInsertIndexTuples](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L310-L400)).
- A heap delete does not call an index access-method delete operation. VACUUM later supplies dead heap tuple identifiers through the index bulk-delete callback ([heapam_handler.c#heapam_tuple_delete](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L304-L315), [indexam.c#index_bulk_delete](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L672-L711)).
- `n_live_tup` is an estimate, not an exact row count ([monitoring.sgml#pg_stat_all_tables-counters](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2772-L2785)).

The heap calls the statistics counter after a successful physical insert, update, or delete attempt ([heapam.c#heap-insert-counter](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2020-L2030), [heapam.c#heap-update-counter](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3740-L3747), [heapam.c#heap-delete-counter](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2821-L2835)). These counters are useful activity signals, but they are not a durable logical-change ledger and do not describe any one index.

### Why GIN has no row-count mapping

GIN is an inverted index. It stores a key with either a posting list or a posting tree of heap tuple identifiers. One indexed array or document can contain many keys ([README#GIN-structure](../../../../raw/postgres-12/src/backend/access/gin/README#L17-L31), [gin.sgml#gin-implementation](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L433-L444)). The operator class's `extractValue` callback chooses `nkeys`, and core sorts and de-duplicates that returned key array before insertion ([gin.sgml#extractValue](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L157-L175), [ginutil.c#ginExtractEntries](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L477-L599)).

The normal GIN path loops over every extracted key. With `fastupdate` enabled, it first collects pending entries; with `fastupdate` disabled, it inserts them directly into the main structure ([gininsert.c#ginHeapTupleInsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L462-L482), [gininsert.c#gininsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L484-L531)). `fastupdate` is enabled by default, and `GinOptions` separately stores its pending-list limit ([gin_private.h#GinOptions](../../../../raw/postgres-12/src/include/access/gin_private.h#L21-L39)). Thus equal table row counts can cause radically different GIN key insertions, pending-list work, page splits, and final sizes.

Deletes are asymmetric:

- GIN has no delete operation in its key, or entry, tree ([README#entry-tree-delete](../../../../raw/postgres-12/src/backend/access/gin/README#L22-L31)).
- VACUUM removes dead heap tuple identifiers from posting lists and posting trees, but never deletes entry-tree tuples or pages. Empty posting-tree pages can be deleted ([README#page-deletion](../../../../raw/postgres-12/src/backend/access/gin/README#L389-L396)).
- GIN VACUUM records recyclable pages in the free-space map and reports the current physical block count. The reviewed cleanup function has no index-file truncation step ([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L686-L794)).

Consequently, 100 deleted rows containing 1,000 short-lived distinct keys each can leave far more persistent entry-tree structure than 4,000 rows that share a few keys. Conversely, pure inserts can legitimately grow a live GIN index; rebuilding it from the same live heap does not make that necessary growth disappear. A table-row percentage sees neither distinction.

The pending list is another independent dimension. `GinMetaPageData` stores pending pages and pending heap-tuple counts separately from main-tree page statistics ([ginblock.h#GinMetaPageData](../../../../raw/postgres-12/src/include/access/ginblock.h#L54-L100)). Pending entries are live deferred work, not automatically dead bloat; searches must inspect them as well as the main index ([gin.sgml#gin-fast-update](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L470-L506)).

### Exact-pin counterexamples

An isolated server built from pin `45b88269a353ad93744772791feb6d01bc7e1e42` reported PostgreSQL 12.2. Both fixtures used a 10,000-row baseline and a GIN index with `fastupdate = off`, which removes pending-list size from the comparison. Each test recorded `pg_relation_size(index)` before activity and after `REINDEX INDEX`; the one-argument SQL function selects the main fork, and its implementation opens the relation and sums that fork's file segments ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L335)).

| Case | Table-counter result | GIN bytes before activity | GIN bytes before `REINDEX` | GIN bytes after `REINDEX` | Reclaimed |
|---|---:|---:|---:|---:|---:|
| Update an unindexed fixed-width column on 4,000 rows with heap fillfactor headroom | `delta_upd = 4000`, `delta_hot = 4000`; original score `40.00%` | 131,072 | 131,072 | 131,072 | `0.00%` |
| Give 100 rows 1,000 private keys each, delete those rows, then `VACUUM` | `delta_del = 100`; original score `1.00%` | 5,652,480 | 5,652,480 | 49,152 | `99.13%` |

The first case is a false positive for the proposed trigger. All 4,000 updates were HOT, so the executor inserted no new GIN entries and rebuilding reclaimed nothing ([heapam_handler.c#heapam_tuple_update](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L336-L349)).

The second case is a false negative. Only 1% of the table rows were deleted, but those rows supplied 100,000 distinct keys. VACUUM removed their heap tuple identifiers, while the entry-tree keys remained; a fresh GIN build started from an empty index and scanned only the surviving heap rows ([README#entry-tree-delete](../../../../raw/postgres-12/src/backend/access/gin/README#L28-L31), [gininsert.c#ginbuild](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L316-L427)).

The stored comment survived both ordinary rebuilds in the test. Upstream regression coverage also checks comment survival across ordinary and concurrent table reindexing ([create_index.sql#reindex-comment-test](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#reindex-comment-test](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)).

These two cases disprove a universal 40% automatic rule. They do not predict the rebuild yield of a particular production index.

### A safer 40% inspection score

If a cheap activity screen is still useful, subtract HOT updates and name the result honestly:

```text
delta_non_hot_upd =
    delta_upd - delta_hot

row_version_turnover =
    delta_ins + delta_non_hot_upd + delta_del

turnover_pct =
    100 * row_version_turnover / baseline_rows
```

Interpret `turnover_pct >= 40` as **“inspect GIN”**, not “40% index churn” and not “run `REINDEX`.” This correction removes the exact HOT false positive above, but it cannot correct key cardinality, aborted actions, partial predicates, legitimate insert growth, deferred delete cleanup, pending-list state, entry-tree key retention, or reusable free pages.

Store these fields together:

```text
wiki_gin_dml_v1;
table_oid=<oid>;
rows=<baseline n_live_tup>;
ins=<baseline n_tup_ins>;
upd=<baseline n_tup_upd>;
del=<baseline n_tup_del>;
hot=<baseline n_tup_hot_upd>;
reset_us=<pg_stat_database.stats_reset as epoch microseconds>
```

The table OID rejects a copied or stale comment. `rows` fixes the denominator for the whole observation window. `hot` permits the minimum correction. `reset_us` detects collector resets. `pg_stat_database` obtains that timestamp from `pg_stat_get_db_stat_reset_time` ([system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L887)).

This page's SQL deliberately supports only full, physical, live, ready, valid GIN indexes. `relkind = 'i'` selects a stored index rather than a storage-less partitioned root, and the three `pg_index` flags distinguish live, write-ready, and planner-valid state ([pg_class.h#relation-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L192), [pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L44)). Partial GIN indexes require a predicate-specific baseline and activity source that these table-wide counters cannot provide.

### Capture and recapture the baseline

Capture only after a known healthy GIN build or successful rebuild, after a fresh table `ANALYZE`, and after completed writer sessions have reported their statistics. PostgreSQL 12's collector is asynchronous, can lag activity, and caches a statistics snapshot until transaction end; `pg_stat_clear_snapshot()` discards that cached snapshot but does not force active sessions to report ([monitoring.sgml#statistics-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L259)).

Replace `app` with an allowlisted application schema. The query emits `COMMENT` statements for review; it does not execute them. PostgreSQL 12's grammar accepts a string literal or `NULL` after `IS`, so a query must generate the statement rather than place an expression directly in `COMMENT` ([gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403), [gram.y#comment-types-and-text](../../../../raw/postgres-12/src/backend/parser/gram.y#L6555-L6592)).

```sql
SET /* wiki_gin_dml_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_dml_lock_timeout */ lock_timeout = '2s';
SELECT /* wiki_gin_dml_refresh_snapshot */ pg_stat_clear_snapshot();

WITH /* wiki_capture_gin_dml_baseline */ measured AS MATERIALIZED
(
    SELECT irel.oid AS index_oid,
           ins.nspname AS index_schema,
           irel.relname AS index_name,
           trel.oid AS table_oid,
           tns.nspname AS table_schema,
           trel.relname AS table_name,
           s.n_live_tup::bigint AS baseline_rows,
           s.n_tup_ins::bigint AS baseline_ins,
           s.n_tup_upd::bigint AS baseline_upd,
           s.n_tup_del::bigint AS baseline_del,
           s.n_tup_hot_upd::bigint AS baseline_hot,
           trunc(extract(epoch FROM d.stats_reset) * 1000000)::numeric
               AS reset_us,
           obj_description(irel.oid, 'pg_class') AS existing_comment
    FROM pg_class AS irel
    JOIN pg_namespace AS ins ON ins.oid = irel.relnamespace
    JOIN pg_am AS am ON am.oid = irel.relam
    JOIN pg_index AS ix ON ix.indexrelid = irel.oid
    JOIN pg_class AS trel ON trel.oid = ix.indrelid
    JOIN pg_namespace AS tns ON tns.oid = trel.relnamespace
    JOIN pg_stat_user_tables AS s ON s.relid = trel.oid
    JOIN pg_stat_database AS d ON d.datname = current_database()
    WHERE am.amname = 'gin'
      AND irel.relkind = 'i'
      AND ix.indislive
      AND ix.indisready
      AND ix.indisvalid
      AND ix.indpred IS NULL
      AND tns.nspname IN ('app')
      AND s.n_live_tup > 0
      AND d.stats_reset IS NOT NULL
)
SELECT /* wiki_capture_gin_dml_baseline */ index_schema,
       index_name,
       table_schema,
       table_name,
       baseline_rows,
       baseline_ins,
       baseline_upd,
       baseline_del,
       baseline_hot,
       existing_comment,
       CASE
           WHEN existing_comment IS NULL
             OR existing_comment ~ '^wiki_gin_dml_v1;'
           THEN format(
               'COMMENT /* wiki_capture_gin_dml_baseline */ ON INDEX %I.%I IS %L;',
               index_schema,
               index_name,
               format(
                   'wiki_gin_dml_v1;table_oid=%s;rows=%s;ins=%s;upd=%s;del=%s;hot=%s;reset_us=%s',
                   table_oid,
                   baseline_rows,
                   baseline_ins,
                   baseline_upd,
                   baseline_del,
                   baseline_hot,
                   reset_us
               )
           )
       END AS comment_sql
FROM measured
ORDER BY index_schema, index_name;
```

Review and execute only the generated `comment_sql` values for approved indexes. A null `comment_sql` preserves an unrelated human comment. Reserve the `wiki_gin_dml_v1;` prefix; PostgreSQL stores only one comment string per object, and a new comment replaces the existing `pg_description` tuple ([comment.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L82-L100), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225)).

Run this same capture only after a successful `REINDEX` to overwrite the old machine baseline. Do **not** recapture merely because the detector fires; that would normalize unexplained activity without repairing or measuring anything.

The two timeout GUCs are `PGC_USERSET`. They can be changed for a session or transaction and need neither reload nor restart ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)).

### Detect the threshold

Run the detector outside a long-lived transaction after completed writers have reported their counts. It validates the table OID, reset timestamp, fixed positive denominator, comment grammar, and nonnegative deltas. It also reports current main-fork bytes, but does not call those bytes bloat.

```sql
SET /* wiki_gin_dml_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_dml_lock_timeout */ lock_timeout = '2s';
SELECT /* wiki_gin_dml_refresh_snapshot */ pg_stat_clear_snapshot();

WITH /* wiki_detect_gin_dml_turnover */ measured AS MATERIALIZED
(
    SELECT irel.oid AS index_oid,
           ins.nspname AS index_schema,
           irel.relname AS index_name,
           trel.oid AS table_oid,
           tns.nspname AS table_schema,
           trel.relname AS table_name,
           s.n_tup_ins::numeric AS current_ins,
           s.n_tup_upd::numeric AS current_upd,
           s.n_tup_del::numeric AS current_del,
           s.n_tup_hot_upd::numeric AS current_hot,
           trunc(extract(epoch FROM d.stats_reset) * 1000000)::numeric
               AS current_reset_us,
           pg_relation_size(irel.oid) AS index_bytes,
           obj_description(irel.oid, 'pg_class') AS stored_comment
    FROM pg_class AS irel
    JOIN pg_namespace AS ins ON ins.oid = irel.relnamespace
    JOIN pg_am AS am ON am.oid = irel.relam
    JOIN pg_index AS ix ON ix.indexrelid = irel.oid
    JOIN pg_class AS trel ON trel.oid = ix.indrelid
    JOIN pg_namespace AS tns ON tns.oid = trel.relnamespace
    JOIN pg_stat_user_tables AS s ON s.relid = trel.oid
    JOIN pg_stat_database AS d ON d.datname = current_database()
    WHERE am.amname = 'gin'
      AND irel.relkind = 'i'
      AND ix.indislive
      AND ix.indisready
      AND ix.indisvalid
      AND ix.indpred IS NULL
      AND tns.nspname IN ('app')
),
parsed AS MATERIALIZED
(
    SELECT measured.*,
           regexp_match(
               stored_comment,
               '^wiki_gin_dml_v1;table_oid=([0-9]+);rows=([0-9]+);ins=([0-9]+);upd=([0-9]+);del=([0-9]+);hot=([0-9]+);reset_us=([0-9]+)$'
           ) AS fields
    FROM measured
),
deltas AS MATERIALIZED
(
    SELECT parsed.*,
           fields[1]::numeric AS baseline_table_oid,
           fields[2]::numeric AS baseline_rows,
           current_ins - fields[3]::numeric AS delta_ins,
           current_upd - fields[4]::numeric AS delta_upd,
           current_del - fields[5]::numeric AS delta_del,
           current_hot - fields[6]::numeric AS delta_hot,
           fields[7]::numeric AS baseline_reset_us
    FROM parsed
),
scored AS MATERIALIZED
(
    SELECT deltas.*,
           delta_upd - delta_hot AS delta_non_hot_upd,
           delta_ins + (delta_upd - delta_hot) + delta_del
               AS row_version_turnover
    FROM deltas
)
SELECT /* wiki_detect_gin_dml_turnover */ index_schema,
       index_name,
       table_schema,
       table_name,
       index_bytes,
       delta_ins,
       delta_non_hot_upd,
       delta_del,
       round(100 * row_version_turnover / nullif(baseline_rows, 0), 2)
           AS turnover_pct,
       CASE
           WHEN fields IS NULL THEN 'invalid or absent baseline'
           WHEN baseline_table_oid <> table_oid::text::numeric
               THEN 'table mismatch: recapture'
           WHEN baseline_reset_us <> current_reset_us
               THEN 'statistics reset: recapture'
           WHEN baseline_rows <= 0
               THEN 'invalid denominator: recapture'
           WHEN delta_ins < 0 OR delta_upd < 0 OR delta_del < 0
             OR delta_hot < 0 OR delta_non_hot_upd < 0
               THEN 'counter regression: recapture'
           WHEN row_version_turnover / baseline_rows >= 0.40
               THEN 'inspect GIN; do not auto-reindex'
           ELSE 'below 40% inspection threshold'
       END AS decision
FROM scored
ORDER BY turnover_pct DESC NULLS LAST, index_schema, index_name;
```

This SQL executed successfully on the exact 12.2 pin. A separate production-shaped fixture captured a baseline for `app.documents_tags_gin` and parsed it successfully. After 400 table updates, only 185 were HOT, so the detector reported 215 non-HOT updates and `21.50%` rather than the uncorrected `40.00%`.

The detector's reset check is deliberately conservative. Resetting a single table's statistics changes the database-wide `stats_reset` timestamp, so all stored baselines in that database become stale ([pgstatfuncs.c#pg_stat_reset_single_table_counters](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1892-L1900), [pgstat.c#pgstat_recv_resetsinglecounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6152-L6178)).

### Inspect before REINDEX

When the score reaches 40%, inspect in this order:

1. Reject missing, malformed, reset, regressed, copied, or stale baselines. Confirm `track_counts` remained enabled for writer sessions. PostgreSQL 12 defines `track_counts` as `PGC_SUSET` with default `on`; a superuser can change it for a session or transaction without reload or restart, and a disabled session creates no table-count state ([guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1400), [pgstat.c#pgstat_initstats](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1738-L1784)).
2. Record `pg_relation_size(index)` and the pending-list state. The packaged `pgstattuple` function `pgstatginindex` returns only version, pending pages, and pending tuples; it does not return a bloat or reclaimable-space estimate ([pgstatindex.c#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L479-L573), [pgstattuple.sgml#pgstatginindex](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L298-L316)).
3. Separate pending work from persistent layout. A normal GIN pending-list cleanup moves entries into the main structure and returns pending pages to the free-space map ([ginfast.c#ginInsertCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L765-L828), [ginfast.c#pending-page-FSM](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1014-L1020)). `gin_clean_pending_list` requires the index owner, rejects recovery, the wrong access method, and another session's temporary index, and returns pages removed rather than reclaimed bytes ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1027-L1073)).
4. Check whether the indexed values now produce more or fewer keys, whether the operator class, expression, reloptions, or workload changed, and whether deletes created many transient distinct keys. GIN's operator class controls `nkeys`, so table counters cannot perform this check ([gin.sgml#extractValue](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L157-L175)).
5. Require material size, query-performance impact, and a history of useful rebuild yield for the same index family and workload. Record pre- and post-rebuild bytes. A negative result should disable or raise the local trigger; it should not be silently recaptured as a new normal.

If that inspection justifies a concurrent rebuild, use bounded session settings. These GUCs need no reload or restart:

```sql
SET /* wiki_gin_reindex_statement_timeout */ statement_timeout = '2h';
SET /* wiki_gin_reindex_lock_timeout */ lock_timeout = '5s';

REINDEX /* wiki_reindex_confirmed_gin_candidate */
    INDEX CONCURRENTLY app.documents_tags_gin;

RESET /* wiki_gin_reindex_lock_timeout */ lock_timeout;
RESET /* wiki_gin_reindex_statement_timeout */ statement_timeout;
```

`REINDEX INDEX CONCURRENTLY` performs more work and takes longer than ordinary `REINDEX`, but permits ordinary writes; it cannot run inside a transaction block and has additional restrictions and failure leftovers ([reindex.sgml#concurrent-rebuild](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L270-L297), [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L413)).

After success, record the measured yield and run the capture query again. Ordinary `REINDEX` keeps the index OID and replaces its relfilenode; concurrent reindexing creates a copy with a new index OID and explicitly moves the old index's comment row to it ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531), [index.c#index_concurrently_create_copy](../../../../raw/postgres-12/src/backend/catalog/index.c#L1234-L1388), [index.c#move-index-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)). Therefore, both paths preserve the **old** baseline. Without explicit post-success recapture, the next detector run remains above the threshold. After a concurrent rebuild, resolve the index again by qualified name rather than retaining its old OID.

### Lifecycle and failure boundaries

- **Comment ownership and locking:** `COMMENT ON INDEX` resolves the index under `ShareUpdateExclusiveLock`, checks ownership, and retains the acquired lock until transaction end ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L34-L130)). Use one metadata writer per index.
- **One visible string:** an object has one comment, and every database user can read comments. Do not overwrite prose or store secrets ([comment.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L82-L100), [comment.sgml#visibility](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L284)).
- **Catalog identity:** an index comment is a `pg_description` row keyed by the index's `pg_class` OID, the `pg_class` catalog OID, and `objsubid = 0` ([pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L8-L57)).
- **Dump and restore:** `pg_dump` emits index comments by default, while `--no-comments` suppresses them. Audit baselines after restore ([pg_dump.c#dumpComment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9468-L9502), [pg_dump.c#index-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463)).
- **Clone hazard:** `CREATE TABLE ... LIKE ... INCLUDING ALL` copies index comments. The stored table OID makes the cloned comment fail closed as `table mismatch` ([parse_utilcmd.c#CREATE_TABLE_LIKE_INDEXES](../../../../raw/postgres-12/src/backend/parser/parse_utilcmd.c#L1182-L1222)).
- **Statistics lifetime:** a clean shutdown preserves collector statistics, while crash recovery, immediate shutdown recovery, and point-in-time recovery reset them ([monitoring.sgml#statistics-persistence](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L198-L211)). The reset timestamp and negative-delta guards require recapture instead of applying the old baseline.
- **TRUNCATE:** `TRUNCATE` has dedicated heap and index storage handling, then tells the collector to apply truncate-specific counter handling ([tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1773-L1844)). Treat any counter regression or table-rewrite event as a new baseline boundary.
- **Partial and partitioned indexes:** the SQL excludes partial indexes and partitioned index roots. Run it on each physical leaf GIN index; `pg_stat_user_tables` keys its activity to the selected table OID, not to a leaf index ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [pg_class.h#relation-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L192)).
- **Definition changes:** an operator-class, expression, predicate, or reloption change invalidates the workload-to-index relationship even if all numeric guards still pass. GIN key extraction and the `fastupdate` path are definition-dependent ([ginutil.c#ginExtractEntries](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L477-L599), [ginutil.c#ginoptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L628)). Capture only after the replacement index is known healthy.
- **Failed rebuilds:** do not recapture after a failed or cancelled rebuild. Concurrent reindexing can leave an invalid `_ccnew` or `_ccold` index that needs separate cleanup ([reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L389)).

### Caller data-structure build extension and test boundaries

- **Counter path:** `pg_stat_user_tables` calls table-OID statistics functions. Heap insert, update, and delete operations increment backend-local `PgStat_TableCounts`; transaction-end code transfers attempted-action counts regardless of commit ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [pgstat.h#PgStat_TableCounts](../../../../raw/postgres-12/src/include/pgstat.h#L77-L117), [pgstat.c#AtEOXact_PgStat](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L2087-L2171)).
- **Index-insert path:** a non-HOT update reaches `ExecInsertIndexTuples`, then generic `index_insert` dispatches through `aminsert` to `gininsert`. HOT stops before that boundary ([nodeModifyTable.c#index-update-gate](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L1445-L1448), [indexam.c#index_insert](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L165-L188), [gininsert.c#gininsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L484-L531)).
- **Index-delete path:** heap delete changes no index immediately. Generic VACUUM dispatches `ambulkdelete` and `amvacuumcleanup` to GIN, which cleans pending entries, removes dead posting identifiers, records reusable pages, and refreshes metapage statistics ([indexam.c#index_bulk_delete](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L672-L711), [ginvacuum.c#ginbulkdelete](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L563-L683), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L686-L794)).
- **Access-method registration:** `ginhandler` wires `ginbuild`, `gininsert`, `ginbulkdelete`, and `ginvacuumcleanup` into `IndexAmRoutine`. `pg_am.dat` and `pg_proc.dat` register the handler ([ginutil.c#ginhandler](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L32-L80), [pg_am.dat#GIN](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29), [pg_proc.dat#ginhandler](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L889-L891)).
- **GIN data structures:** `GinOptions` controls fast update and its pending-list limit; `GinMetaPageData` stores pending and main-tree page statistics; deleted posting-tree pages are reusable only after their deletion transaction precedes `RecentGlobalXmin` ([gin_private.h#GinOptions](../../../../raw/postgres-12/src/include/access/gin_private.h#L21-L39), [ginblock.h#GinMetaPageData](../../../../raw/postgres-12/src/include/access/ginblock.h#L54-L100), [ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L124-L138)).
- **Rebuild boundary:** `REINDEX` gives the index a fresh physical file, and `ginbuild` requires an empty index, initializes fresh meta and root pages, scans the heap, and writes new GIN metapage statistics ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3531), [gininsert.c#ginbuild](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L316-L427)).
- **Generated build boundary:** the GIN objects are compiled by `access/gin/Makefile`. The catalog build consumes `pg_am.dat` and `pg_proc.dat` and generates `*_d.h` and BKI artifacts; `pg_am.h` includes the generated `pg_am_d.h` ([access/gin/Makefile#OBJS](../../../../raw/postgres-12/src/backend/access/gin/Makefile#L10-L19), [catalog/Makefile#generated-catalogs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L90), [pg_am.h#generated-header](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L17-L27)). The proposed scheme changes no server source or catalog layout.
- **Operator-class and extension boundary:** core calls an operator-class-supplied `extractValue` function and cannot infer its row-to-key cardinality. `pgstattuple` can expose pending-list counts but no rebuild-yield metric ([ginutil.c#ginExtractEntries](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L477-L599), [pgstatindex.c#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L479-L573)).
- **Upstream tests:** the core GIN regression test exercises pending cleanup, delete plus VACUUM, `fastupdate = off`, and recompression. It does not compare DML counters with GIN size, rebuild yield, or a 40% threshold ([gin.sql#GIN-regression](../../../../raw/postgres-12/src/test/regress/sql/gin.sql#L7-L36)). The separate reindex regression proves comment preservation but not baseline correctness ([create_index.sql#reindex-comment-test](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854)).

## Context Reviewed

- Pinned checkout `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- Table statistics view definitions, heap count sites, transaction and subtransaction transfer, collector snapshots, resets, persistence, `track_counts`, and truncate handling.
- Executor HOT and non-HOT update paths, partial-index predicates, generic index insertion, and deferred index deletion through VACUUM.
- GIN handler registration, operator-class key extraction, direct and fast-update insertion, pending-list cleanup, entry and posting trees, page reuse, VACUUM, and rebuild.
- `COMMENT` grammar, utility path, `pg_description` storage, ownership, locking, visibility, clone, dump/restore, ordinary reindex, and concurrent reindex.
- Catalog generation and GIN build ownership.
- Core GIN, comment-preservation, statistics, `pgstattuple`, and page-inspection test surfaces.
- Isolated exact-pin HOT false-positive, skewed-key delete false-negative, comment-preservation, capture, and detector tests.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| The three counters are table-wide attempted-action counts | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [pgstat.h#PgStat_TableCounts](../../../../raw/postgres-12/src/include/pgstat.h#L77-L117) |
| HOT updates require no separate index write | [monitoring.sgml#pg_stat_all_tables-counters](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2752-L2770), [heapam_handler.c#heapam_tuple_update](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L318-L349) |
| One indexed row can produce an operator-class-defined number of GIN keys | [gin.sgml#extractValue](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L157-L175), [ginutil.c#ginExtractEntries](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L477-L599) |
| Heap DELETE does not immediately delete index tuples | [heapam_handler.c#heapam_tuple_delete](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L304-L315), [indexam.c#index_bulk_delete](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L672-L711) |
| GIN never deletes entry-tree tuples or pages | [README#entry-tree-delete](../../../../raw/postgres-12/src/backend/access/gin/README#L28-L31), [README#page-deletion](../../../../raw/postgres-12/src/backend/access/gin/README#L389-L396) |
| GIN VACUUM records reusable pages without truncating the file | [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L686-L794) |
| Comments occupy one `pg_description` row and survive both rebuild paths | [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L8-L57), [create_index.out#reindex-comment-test](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117) |
| Concurrent reindex moves the old comment to the new index OID | [index.c#move-index-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656) |
| PostgreSQL 12 defines no 40% GIN rebuild threshold | [maintenance.sgml#routine-reindex](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L880), [reindex.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L31-L75) |

## Open Questions

- Which inspection score and minimum byte size predict useful rebuild yield for each production GIN operator class, data distribution, reloption set, and operation mix? The exact-pin tests disprove a universal 40% rebuild rule but do not select a local threshold.
- What predicate-specific denominator and activity source should replace table-wide counters for partial GIN indexes? The production query excludes them.
- PostgreSQL 12's `VACUUM` reference says “in any form” completes pending GIN insertions ([vacuum.sgml#GIN-pending-insertions](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L291-L295)). The implementation sets `useindex = false` when `INDEX_CLEANUP` is disabled and then skips index cleanup callbacks ([vacuumlazy.c#index-cleanup-choice](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L289), [vacuumlazy.c#index-cleanup-callbacks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1466-L1471)). The source therefore contradicts that documentation wording for `VACUUM (INDEX_CLEANUP false)`.
- How much collector lag and statistics loss occurs under the target workload, and should a durable metadata table record observation IDs and rebuild yield instead of putting only the latest baseline in a comment?

## Source References

- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)
- [system_views.sql#pg_stat_user_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L613-L621)
- [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L887)
- [pgstat.h#PgStat_TableCounts](../../../../raw/postgres-12/src/include/pgstat.h#L77-L117)
- [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)
- [pgstat.c#AtEOXact_PgStat](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L2087-L2171)
- [pgstat.c#pgstat_recv_resetsinglecounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6152-L6178)
- [monitoring.sgml#statistics-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L259)
- [monitoring.sgml#pg_stat_all_tables-counters](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2752-L2785)
- [heapam_handler.c#heapam_tuple_delete](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L304-L315)
- [heapam_handler.c#heapam_tuple_update](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L318-L349)
- [execIndexing.c#ExecInsertIndexTuples](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L310-L400)
- [indexam.c#index_insert](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L165-L188)
- [ginutil.c#ginhandler](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L32-L80)
- [ginutil.c#ginExtractEntries](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L477-L599)
- [gininsert.c#gininsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L462-L531)
- [gininsert.c#ginbuild](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L316-L427)
- [ginvacuum.c#ginbulkdelete](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L563-L683)
- [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L686-L794)
- [ginfast.c#ginInsertCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L765-L828)
- [README#GIN-structure](../../../../raw/postgres-12/src/backend/access/gin/README#L17-L31)
- [README#page-deletion](../../../../raw/postgres-12/src/backend/access/gin/README#L389-L396)
- [ginblock.h#GinMetaPageData](../../../../raw/postgres-12/src/include/access/ginblock.h#L54-L100)
- [gin_private.h#GinOptions](../../../../raw/postgres-12/src/include/access/gin_private.h#L21-L39)
- [gin.sgml#extractValue](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L157-L175)
- [gin.sgml#gin-fast-update](../../../../raw/postgres-12/doc/src/sgml/gin.sgml#L470-L506)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L880)
- [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L34-L130)
- [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225)
- [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L8-L57)
- [index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3432-L3531)
- [index.c#move-index-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)
- [create_index.sql#reindex-comment-test](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854)
- [gin.sql#GIN-regression](../../../../raw/postgres-12/src/test/regress/sql/gin.sql#L7-L36)
- [pgstatindex.c#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L479-L573)
- [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396)
- [catalog/Makefile#generated-catalogs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L90)

## Navigation

- [v12/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
