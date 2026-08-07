---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Detecting Index Bloat with COMMENT-Stored Bytes per Tuple in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict and metric](#verdict-and-metric)
  - [Why ANALYZE must precede both measurements](#why-analyze-must-precede-both-measurements)
  - [Capture the baseline](#capture-the-baseline)
  - [Detect drift](#detect-drift)
  - [Interpretation by access method](#interpretation-by-access-method)
  - [Reindex and recapture](#reindex-and-recapture)
  - [Exact-pin validation](#exact-pin-validation)
  - [Operational and failure boundaries](#operational-and-failure-boundaries)
  - [Caller callee data and build boundaries](#caller-callee-data-and-build-boundaries)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Propose a way to detect bloat in all index types by using the `COMMENT` command to store the index’s bytes-per-tuple ratio when the index is created. Suppose the stored initial ratio is 1.0. A maintenance process later recalculates it as 1.4—an increase of approximately 40%—and therefore decides to reindex.

## Answer

### Verdict and metric

Use the proposal as an **allocation-drift screen**, not as an access-method-independent bloat detector. The catalog query can cover every physical index, including extension and third-party index access methods, but PostgreSQL 12 exposes no generic callback that says how much of an index is rebuild-reclaimable. `IndexAmRoutine` exposes build, insert, vacuum, cost, option, and scan callbacks, but no bloat or free-space callback ([amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L159-L233)).

Define bytes per tuple as bytes per **estimated indexed heap row**, not bytes per internal index tuple:

```text
baseline_bpt = baseline_index_main_fork_bytes / baseline_estimated_indexed_rows
current_bpt  = current_index_main_fork_bytes  / current_estimated_indexed_rows
drift        = current_bpt / baseline_bpt
increase_pct = (drift - 1) * 100
```

The stored baseline must be the actual `baseline_bpt`, such as `23.3472`. The value `1.0` in the question is the normalized starting drift, because `baseline_bpt / baseline_bpt = 1.0`. Storing only the literal value `1.0` discards the scale needed to calculate later drift. A later drift of `1.4` is a 40% increase, but it should produce `investigate`, not an unconditional `REINDEX`.

`pg_relation_size(index_oid)` measures the main fork. Its one-argument SQL definition explicitly selects `main`, and its implementation opens the relation with `AccessShareLock` and stats every relation segment ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L335)). Main-fork bytes exclude the free-space map and other auxiliary forks by design.

### Why ANALYZE must precede both measurements

Do not capture the denominator immediately after `CREATE INDEX`. The common build layer calls the access method's `ambuild` function and stores the returned `IndexBuildResult.index_tuples` in the index's `pg_class.reltuples` value ([genam.h#IndexBuildResult](../../../../raw/postgres-12/src/include/access/genam.h#L27-L34), [index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2824-L2904), [index.c#index-build-statistics](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986)). That returned value does not mean the same thing for every access method:

- GIN adds the number of keys extracted from each indexed value to its build count, so one heap row can add many `index_tuples` ([gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L245-L270), [gininsert.c#ginbuild-result](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L420-L427)).
- BRIN returns the number of summary tuples, normally one per completed heap page range, rather than the number of indexed heap rows ([brin.c#brinbuildCallback](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L592-L630), [brin.c#brinbuild-result](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L713-L742)).

A subsequent **plain table-level `ANALYZE`** supplies one consistent denominator. It initializes an ordinary index's fraction to `1.0`, estimates the qualifying fraction of a partial index from the sampled predicate, and writes `ceil(tupleFract * totalrows)` to each index's `reltuples` ([analyze.c#index-initialization](../../../../raw/postgres-12/src/backend/commands/analyze.c#L420-L439), [analyze.c#partial-index-fraction](../../../../raw/postgres-12/src/backend/commands/analyze.c#L714-L822), [analyze.c#index-relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L628)). Use plain `ANALYZE`, not `VACUUM (ANALYZE)`, for this normalization: that update is deliberately skipped when analysis is part of `VACUUM` ([analyze.c#index-relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L628)).

The denominator remains an estimate. Large-table `ANALYZE` samples random blocks and rows, then extrapolates live-row density to unscanned blocks, and `pg_class` itself labels `reltuples` as not always current ([analyze.c#acquire_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L977-L1007), [analyze.c#totalrows-estimate](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1111-L1121), [pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66)). Selective partial indexes are therefore especially vulnerable to sampling noise.

The normal sequence is:

1. Create the index.
2. Run plain `ANALYZE` on its table.
3. Measure and store the actual baseline BPT in the index comment.
4. Before each comparison, run plain `ANALYZE` again.
5. Treat `drift >= 1.4` as a candidate, then apply an access-method-specific confirmation.
6. Reindex only a confirmed candidate; after success, analyze and capture a new baseline.

### Capture the baseline

Reserve a versioned comment format. This example preserves the raw measurements for auditability:

```text
wiki_index_bpt_v1;bytes=466944;tuples=20000;bpt=23.347200000
```

An ordinary PostgreSQL object has one comment. Writing another comment replaces the existing `pg_description` row, while a null or empty comment removes it ([comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225)). Therefore, do not overwrite a human comment. If indexes already use comments, use a dedicated metadata table instead.

Set bounded session timeouts, refresh each approved table, then generate `COMMENT` statements. Replace `app` and `app.orders` with an allowlisted application schema and table. Review and execute the generated `comment_sql` values; the read query itself does not change comments.

```sql
SET /* wiki_index_bpt_timeout */ statement_timeout = '15min';
SET /* wiki_index_bpt_lock_timeout */ lock_timeout = '5s';

ANALYZE /* wiki_refresh_index_bpt_denominators */ app.orders;

WITH /* wiki_capture_index_bpt */ measured AS MATERIALIZED
(
    SELECT n.nspname AS index_schema,
           c.relname AS index_name,
           am.amname,
           c.reltuples::numeric AS estimated_indexed_rows,
           pg_relation_size(c.oid) AS index_bytes,
           obj_description(c.oid, 'pg_class') AS existing_comment
    FROM pg_class AS c
    JOIN pg_namespace AS n ON n.oid = c.relnamespace
    JOIN pg_am AS am ON am.oid = c.relam
    JOIN pg_index AS i ON i.indexrelid = c.oid
    WHERE c.relkind = 'i'
      AND i.indislive
      AND i.indisready
      AND i.indisvalid
      AND n.nspname IN ('app')
)
SELECT /* wiki_capture_index_bpt */ index_schema,
       index_name,
       amname,
       index_bytes,
       estimated_indexed_rows,
       round(index_bytes::numeric / estimated_indexed_rows, 6)
           AS measured_bpt,
       existing_comment,
       CASE
           WHEN existing_comment IS NULL THEN
               format(
                   'COMMENT /* wiki_capture_index_bpt */ ON INDEX %I.%I IS %L;',
                   index_schema,
                   index_name,
                   format(
                       'wiki_index_bpt_v1;bytes=%s;tuples=%s;bpt=%s',
                       index_bytes,
                       estimated_indexed_rows,
                       round(index_bytes::numeric / estimated_indexed_rows, 9)
                   )
               )
       END AS comment_sql
FROM measured
WHERE estimated_indexed_rows > 0
ORDER BY index_schema, index_name;
```

The query selects `relkind = 'i'`, the physical secondary-index kind, and excludes partitioned index roots (`relkind = 'I'`), which have no index storage of their own ([pg_class.h#relation-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L182-L192)). It also excludes indexes that are not live, ready, and valid, using the three state fields defined by `pg_index` ([pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L44)). Empty indexes and indexes without a positive analyzed estimate remain uncaptured.

`statement_timeout` and `lock_timeout` are both `PGC_USERSET` in PostgreSQL 12. They can be set for a session or transaction and need neither reload nor restart ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)). The maintenance role must own the target index for `COMMENT`; `CommentObject` takes `ShareUpdateExclusiveLock`, resolves the object, and checks ownership before writing the comment ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L34-L127)).

### Detect drift

Run plain `ANALYZE` on the approved tables immediately before this read-only query. It parses only the reserved comment format, keeps the numeric cast inside a guarded `CASE`, measures each index once, and reports the threshold as `investigate`.

```sql
WITH /* wiki_detect_index_bpt_drift */ measured AS MATERIALIZED
(
    SELECT n.nspname AS index_schema,
           c.relname AS index_name,
           tn.nspname AS table_schema,
           t.relname AS table_name,
           am.amname,
           c.reltuples::numeric AS estimated_indexed_rows,
           pg_relation_size(c.oid) AS index_bytes,
           obj_description(c.oid, 'pg_class') AS stored_comment
    FROM pg_class AS c
    JOIN pg_namespace AS n ON n.oid = c.relnamespace
    JOIN pg_am AS am ON am.oid = c.relam
    JOIN pg_index AS i ON i.indexrelid = c.oid
    JOIN pg_class AS t ON t.oid = i.indrelid
    JOIN pg_namespace AS tn ON tn.oid = t.relnamespace
    WHERE c.relkind = 'i'
      AND i.indislive
      AND i.indisready
      AND i.indisvalid
      AND n.nspname IN ('app')
),
parsed AS
(
    SELECT measured.*,
           CASE
               WHEN stored_comment ~
                    '^wiki_index_bpt_v1;bytes=[0-9]+;tuples=[0-9]+([.][0-9]+)?;bpt=[0-9]+([.][0-9]+)?$'
               THEN split_part(stored_comment, ';bpt=', 2)::numeric
           END AS baseline_bpt
    FROM measured
),
scored AS
(
    SELECT parsed.*,
           index_bytes::numeric / estimated_indexed_rows AS current_bpt
    FROM parsed
    WHERE estimated_indexed_rows > 0
      AND baseline_bpt > 0
)
SELECT /* wiki_detect_index_bpt_drift */ index_schema,
       index_name,
       table_schema,
       table_name,
       amname,
       index_bytes,
       estimated_indexed_rows,
       round(baseline_bpt, 6) AS baseline_bpt,
       round(current_bpt, 6) AS current_bpt,
       round(current_bpt / baseline_bpt, 4) AS normalized_ratio,
       round((current_bpt / baseline_bpt - 1) * 100, 1)
           AS percent_increase,
       round(greatest(
           index_bytes::numeric - baseline_bpt * estimated_indexed_rows,
           0
       ))::bigint AS bytes_above_baseline_trend,
       CASE
           WHEN current_bpt / baseline_bpt >= 1.4
               THEN 'investigate'
           ELSE 'below candidate threshold'
       END AS decision
FROM scored
ORDER BY normalized_ratio DESC, index_schema, index_name;
```

`bytes_above_baseline_trend` is the allocation above `baseline_bpt * current_estimated_rows`. It is useful for ignoring tiny indexes, but it is **not** proven reclaimable space. A production rule should require both a calibrated drift threshold and a minimum byte value.

Audit the capture output as well as the detector output. A physical index is intentionally absent from the detector when its comment is missing, belongs to a human, is malformed, its estimate is nonpositive, or its state flags are unsuitable. `obj_description(oid, 'pg_class')` reads the object-level row with `objsubid = 0` from `pg_description` ([pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314), [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57)).

### Interpretation by access method

The query is access-method-neutral; the result is not. PostgreSQL 12 registers six core index access methods in `pg_am`, while an extension can register another handler and type ([pg_am.dat#index-access-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L35), [pg_am.h#pg_am](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L20-L58)). Contrib Bloom is one such additional index AM.

| Access method | What can change healthy bytes per estimated row | Safe use of `1.4` |
|---|---|---|
| B-tree | Actual index-tuple widths, fillfactor, internal pages, duplicate representation, and split history affect size; a fresh build packs pages against the configured fillfactor ([nbtsort.c#_bt_buildadd](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L837-L900), [nbtree.h#fillfactors](../../../../raw/postgres-12/src/include/access/nbtree.h#L158-L171)). | A useful candidate rule only after calibrating for stable definition, key-width distribution, and reloptions. |
| Hash | Initial buckets are rounded to a splitpoint, and expansion can allocate a batch of buckets at once ([hashpage.c#initial-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L493-L525), [hashpage.c#splitpoint-allocation](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L781-L852)). | Do not use a universal threshold; healthy step changes can cross it. |
| GiST | Layout follows the operator class's `picksplit` result and the indexed value distribution ([gistsplit.c#gistSplit](../../../../raw/postgres-12/src/backend/access/gist/gistsplit.c#L396-L448)). | Candidate only after per-operator-class and workload calibration. |
| SP-GiST | Operator-class partitioning and split/concurrency paths can leave redirect and placeholder tuples ([spgdoinsert.c#redirect-and-placeholder](../../../../raw/postgres-12/src/backend/access/spgist/spgdoinsert.c#L1120-L1172)). | Candidate only after per-operator-class and workload calibration. |
| GIN | One row can produce many extracted entries; posting representation and the fast-update pending list can dominate bytes ([gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L245-L270), [ginfast.c#ginInsertCleanup](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L765-L817)). | Never an automatic bloat verdict. Changes in array/document key cardinality can create extreme false positives. |
| BRIN | The index summarizes heap page ranges, not rows; range transitions form summary tuples ([brin.c#brinbuildCallback](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L592-L630), [brin_revmap.c#revmap-layout](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L1-L44)). | Never an automatic bloat verdict. Row-count changes and heap-page-range changes are different dimensions. |
| Bloom | Each non-null indexed row produces one configurable fixed-length signature; signature length and per-column bit settings change the baseline ([blutils.c#bloom-options](../../../../raw/postgres-12/contrib/bloom/blutils.c#L55-L96), [blinsert.c#blbuild](../../../../raw/postgres-12/contrib/bloom/blinsert.c#L90-L157)). | Often calibratable when definition and options remain fixed, but still confirm benefit. |
| Third-party AM | Core knows only the handler and generic callback table; storage semantics belong to the extension ([pg_am.h#pg_am](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L29-L40), [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L159-L233)). | Unknown until that AM is separately calibrated. |

This distinction is why `1.4` cannot mean “40% bloat” across all index types. It means “current main-fork bytes per estimated indexed row are 40% above this index's stored baseline.”

### Reindex and recapture

Use the candidate list as the start of a decision process:

1. Require a fresh successful plain `ANALYZE`, valid metadata, and a material `bytes_above_baseline_trend`.
2. Check whether the index definition, reloptions, indexed value distribution, partial predicate selectivity, and access-method-specific state still resemble the baseline regime.
3. Require access-method-specific evidence or a history of measured rebuild yield for this index family.
4. Reindex only after those checks. Record pre-rebuild bytes and compare them with post-rebuild bytes.
5. If the rebuild succeeded and the new index is the intended healthy reference, run plain `ANALYZE` and replace the old baseline. Do not recapture unexplained drift before rebuilding, because that normalizes the signal away.

For a production table where writes must continue, PostgreSQL 12 supports `REINDEX INDEX CONCURRENTLY`. It performs two table scans and waits for relevant transactions, so it takes more work and usually longer than standard `REINDEX`; standard `REINDEX` blocks writers ([reindex.sgml#concurrent-rebuild](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L270-L297)). It cannot run in a transaction block and cannot concurrently rebuild an exclusion-constraint index ([reindex.sgml#concurrent-restrictions](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L392-L413)). Use workload-specific timeouts; these are example session values:

```sql
SET /* wiki_reindex_index_bpt_timeout */ statement_timeout = '2h';
SET /* wiki_reindex_index_bpt_lock_timeout */ lock_timeout = '5s';

REINDEX /* wiki_reindex_confirmed_bpt_candidate */
    INDEX CONCURRENTLY app.orders_customer_id_idx;

ANALYZE /* wiki_refresh_reindexed_bpt_denominator */ app.orders;

RESET /* wiki_reindex_index_bpt_lock_timeout */ lock_timeout;
RESET /* wiki_reindex_index_bpt_timeout */ statement_timeout;
```

The index comment survives both ordinary and concurrent reindexing in the upstream regression test, and the concurrent swap explicitly moves the old index's `pg_description` row to the new index OID ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117), [index.c#move-index-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)). That survival is useful, but it means the successful rebuild still carries the **old** baseline until the maintenance process deliberately replaces it.

Generate that replacement only for the confirmed index and only while its existing comment belongs to this scheme:

```sql
WITH /* wiki_recapture_index_bpt */ measured AS MATERIALIZED
(
    SELECT n.nspname AS index_schema,
           c.relname AS index_name,
           c.reltuples::numeric AS estimated_indexed_rows,
           pg_relation_size(c.oid) AS index_bytes,
           obj_description(c.oid, 'pg_class') AS existing_comment
    FROM pg_class AS c
    JOIN pg_namespace AS n ON n.oid = c.relnamespace
    WHERE c.oid = 'app.orders_customer_id_idx'::regclass
)
SELECT /* wiki_recapture_index_bpt */ format(
           'COMMENT /* wiki_recapture_index_bpt */ ON INDEX %I.%I IS %L;',
           index_schema,
           index_name,
           format(
               'wiki_index_bpt_v1;bytes=%s;tuples=%s;bpt=%s',
               index_bytes,
               estimated_indexed_rows,
               round(index_bytes::numeric / estimated_indexed_rows, 9)
           )
       ) AS comment_sql
FROM measured
WHERE estimated_indexed_rows > 0
  AND existing_comment LIKE 'wiki_index_bpt_v1;%';
```

Review and execute the generated `COMMENT` statement as the final maintenance step. If concurrent reindexing fails, do not update the baseline. PostgreSQL can leave an invalid `_ccnew` or `_ccold` index that consumes storage and write overhead; the documented recovery is to drop the invalid leftover and retry ([reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L389)).

### Exact-pin validation

An isolated server built from the pinned PostgreSQL 12.2 checkout, with contrib Bloom installed and autovacuum disabled, exercised B-tree, hash, GiST, SP-GiST, GIN, BRIN, and Bloom. The final capture, detector, guarded parser, and recapture queries executed successfully.

Two tests show why the threshold is only a screen:

| Workload | Result before rebuild | Result of rebuild |
|---|---|---|
| Delete 80% of 20,000 rows in scattered positions, then plain `ANALYZE`; leave the BRIN file at 24,576 bytes | BRIN drift became `5.0000` because the estimated rows fell fivefold | `REINDEX` reclaimed `0.0%`; the healthy BRIN stayed 24,576 bytes |
| Keep a 20,000-row GIN baseline with one repeated array key per row; add 20,000 rows with 20 distinct keys per row, then plain `ANALYZE` | GIN drift became `137.2000` | `REINDEX` produced the same 22,478,848-byte file and reclaimed `0.0%` |

The BRIN result follows its range-summary structure, while the GIN result follows per-value key extraction rather than one-entry-per-row storage ([brin.c#brinbuildCallback](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L592-L630), [gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L245-L270)). Both are false positives for rebuild-reclaimable bloat, despite ratios far above `1.4`.

The same scattered-delete fixture showed that B-tree, hash, GiST, SP-GiST, GIN, and Bloom did shrink after rebuilding, so the signal can still identify useful candidates. The experiment does not establish a production threshold for any workload or third-party AM.

### Operational and failure boundaries

- **Shared namespace:** one object comment cannot simultaneously hold independent human prose and this metadata. `CreateComments` updates the one matching `(objoid, classoid, objsubid)` row ([comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225)).
- **Dump and restore:** normal `pg_dump` emits an index comment, but the `--no-comments` path suppresses comment output; a restore through that option loses the baseline ([pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463), [pg_dump.c#dumpComment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9491-L9502)).
- **Statistics freshness:** `reltuples` is approximate and can be stale. A failed or timed-out `ANALYZE` invalidates the comparison window; log the failure and skip the affected table ([pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66)).
- **Sampling noise:** the partial-index denominator is derived by applying its predicate to the sample. Small qualifying fractions need wider alert bands or repeated observations ([analyze.c#partial-index-fraction](../../../../raw/postgres-12/src/backend/commands/analyze.c#L714-L822)).
- **Empty and new indexes:** a nonpositive estimate has no defined BPT. Keep it visible in an audit report, but do not divide or auto-capture.
- **Index state:** invalid, not-ready, and not-live indexes are separate operational problems. Do not fold them into this bloat decision ([pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L40-L44)).
- **Coverage:** run the process once per database and use explicit schema allowlists. Do not automatically comment on or rebuild system indexes.
- **Cost:** `pg_relation_size` opens and stats every selected index. Schedule broad scans, and keep timeouts and concurrency bounded ([dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L335)).
- **Baseline mutation:** recapture only after a successful, measured rebuild or an explicitly approved definition/reloption change. A failed rebuild must leave the old baseline untouched.

### Caller callee data and build boundaries

- **Build path:** the common index layer calls `rd_indam->ambuild`, receives `IndexBuildResult`, then updates heap and index catalog statistics ([index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2824-L2904), [index.c#index-build-statistics](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986)). Individual AM callbacks define the immediate post-build meaning of `index_tuples`.
- **Analyze path:** table analysis opens the table's indexes, evaluates partial predicates on sample rows, and writes a row-count estimate to each index for plain `ANALYZE` ([analyze.c#index-initialization](../../../../raw/postgres-12/src/backend/commands/analyze.c#L420-L439), [analyze.c#index-relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L628)).
- **Size path:** the one-argument SQL wrapper selects the main fork; the C implementation resolves the relation, holds `AccessShareLock`, and sums file segments ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L335)).
- **Comment path:** `COMMENT ON INDEX` resolves and locks the object, checks ownership, then inserts, replaces, or deletes a `pg_description` tuple ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L34-L127), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225)).
- **Catalog and generated headers:** `pg_class`, `pg_index`, `pg_am`, and `pg_description` are catalog-header inputs; the build derives their `_d.h` headers and BKI artifacts through `genbki.pl` ([catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L58), [catalog/Makefile#generated-catalogs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L90)). This proposal changes no server header or catalog layout; it uses existing catalog rows through SQL.
- **Extension boundary:** Bloom supplies the normal `IndexAmRoutine` callbacks from contrib, and a third-party AM can do the same. Core cannot infer either one's reclaimable space without AM-specific knowledge ([blutils.c#blhandler](../../../../raw/postgres-12/contrib/bloom/blutils.c#L100-L148), [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L159-L233)).
- **Tests:** upstream regression coverage proves comment survival across ordinary and concurrent table reindexing, but it contains no generic bytes-per-tuple bloat policy or cross-AM threshold test ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)).

## Context Reviewed

- Pinned checkout `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- Common index build and catalog-statistics update paths.
- Plain `ANALYZE`, partial-index predicate sampling, and row-count estimation.
- `pg_relation_size`, `pg_class`, `pg_index`, `pg_am`, `pg_description`, and generated catalog artifacts.
- Core B-tree, hash, GiST, SP-GiST, GIN, and BRIN implementations plus contrib Bloom.
- `COMMENT`, ordinary `REINDEX`, `REINDEX CONCURRENTLY`, comment preservation, dump/restore, locks, ownership, timeout contexts, and failure leftovers.
- Exact-pin SQL validation across all seven in-tree or packaged index access methods.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Build-time `index_tuples` is supplied by each AM | [index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2824-L2904), [genam.h#IndexBuildResult](../../../../raw/postgres-12/src/include/access/genam.h#L27-L34) |
| GIN and BRIN make immediate post-build `reltuples` incomparable | [gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L245-L270), [brin.c#brinbuild-result](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L713-L742) |
| Plain `ANALYZE` writes an estimated indexed-row denominator | [analyze.c#partial-index-fraction](../../../../raw/postgres-12/src/backend/commands/analyze.c#L714-L822), [analyze.c#index-relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L628) |
| `pg_relation_size(regclass)` measures main-fork segments | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L335) |
| One index comment occupies one `pg_description` row | [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225), [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57) |
| No generic bloat callback exists | [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L159-L233) |
| Concurrent reindex preserves the comment and has failure leftovers | [index.c#move-index-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656), [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L389) |
| Timeout changes are session or transaction scoped | [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396) |

## Open Questions

- Which drift and minimum-byte thresholds predict worthwhile rebuilds for each production index family? The isolated tests disprove a universal `1.4` rule but do not calibrate a particular workload.
- How much repeated-sample noise do highly selective partial indexes show at the production statistics target?
- Which third-party access methods have a stable bytes-per-indexed-row relationship, and which expose better AM-specific diagnostics?
- Should an installation that already uses comments move this metadata to a dedicated table keyed by database and index identity?
- What minimum measured post-rebuild saving should cause the process to retain, tighten, or disable this signal for an index family?

## Source References

- [genam.h#IndexBuildResult](../../../../raw/postgres-12/src/include/access/genam.h#L27-L34)
- [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L159-L233)
- [index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2824-L2904)
- [index.c#index-build-statistics](../../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986)
- [index.c#move-index-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)
- [analyze.c#index-relstats-update](../../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L628)
- [analyze.c#partial-index-fraction](../../../../raw/postgres-12/src/backend/commands/analyze.c#L714-L822)
- [analyze.c#acquire_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L977-L1007)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L335)
- [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L34-L127)
- [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L132-L225)
- [pg_class.h#reltuples](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66)
- [pg_index.h#index-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L44)
- [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L40-L57)
- [pg_am.dat#index-access-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L35)
- [gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L245-L270)
- [brin.c#brinbuild-result](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L713-L742)
- [hashpage.c#initial-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L493-L525)
- [gistsplit.c#gistSplit](../../../../raw/postgres-12/src/backend/access/gist/gistsplit.c#L396-L448)
- [spgdoinsert.c#redirect-and-placeholder](../../../../raw/postgres-12/src/backend/access/spgist/spgdoinsert.c#L1120-L1172)
- [blutils.c#bloom-options](../../../../raw/postgres-12/contrib/bloom/blutils.c#L55-L96)
- [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)
- [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)
- [reindex.sgml#concurrent-rebuild](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L270-L297)
- [reindex.sgml#concurrent-failure](../../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L389)
- [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854)
- [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2117)
- [catalog/Makefile#generated-catalogs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L90)

## Navigation

- [v12/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [Detecting Bloat with a COMMENT-Stored Index/Heap Ratio in PostgreSQL 12 (unverified)](comment-stored-index-heap-ratio-bloat.md)
- [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](btree-index-bloat-core-sql-only.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
