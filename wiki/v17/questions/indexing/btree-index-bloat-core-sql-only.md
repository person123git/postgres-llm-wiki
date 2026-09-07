---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The current recommended statement](#the-current-recommended-statement)
  - [Reading the output](#reading-the-output)
  - [How the model uses PostgreSQL 17](#how-the-model-uses-postgresql-17)
  - [Deduplication eligibility](#deduplication-eligibility)
  - [Row counts and exclusions](#row-counts-and-exclusions)
  - [Operational and verification limits](#operational-and-verification-limits)
  - [Plan review](#plan-review)
  - [Revised implementation order](#revised-implementation-order)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
  - [Input diagnostics and missing statistics](#input-diagnostics-and-missing-statistics)
  - [Partial-index widths and zero counts](#partial-index-widths-and-zero-counts)
  - [Statistics publication and test ordering](#statistics-publication-and-test-ordering)
  - [Posting-tuple tails](#posting-tuple-tails)
  - [Page geometry and tuple representation](#page-geometry-and-tuple-representation)
  - [Statistics selection and joint distributions](#statistics-selection-and-joint-distributions)
  - [Custom operator classes](#custom-operator-classes)
  - [Signed-byte output](#signed-byte-output)
  - [Reproducible accuracy and cost coverage](#reproducible-accuracy-and-cost-coverage)
  - [Alert thresholds and rebuild savings](#alert-thresholds-and-rebuild-savings)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17: test the SQL from the PostgreSQL 12 question "Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)" on PostgreSQL 17 and compare whether it measures bloat with the same accuracy as in version 12.

Follow-up prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, for the question "Testing the PostgreSQL 12
> Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)", review all the
> plans to fix the open questions.

The original read `follow agents.md, in postgresql 17 , for question:  Testing
the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified) ,
review all the plans to fix the open questions`: `agents.md` for AGENTS.md,
lowercase `postgresql`, a space before two commas, a double space after the
colon, and no sentence capitalisation or terminal period. The review is filed
under [Plan review](#plan-review) and [Revised implementation order](#revised-implementation-order).

## Answer

Use the [current recommended statement](#the-current-recommended-statement) as a
catalog-based estimate of the difference between an index's current main-fork
size and a modelled rebuild. Its baseline is the sorted B-tree build: the builder
packs leaf pages using the index's fillfactor, uses a separate target for internal
pages, and can combine duplicate keys into posting tuples. The statement models
those rules from statistics; it does not inspect the index's leaf contents.
[nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671),
[nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L784-L855),
[nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1349),
[dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371).

Treat the output as candidate information. Both percentages depend on estimated
row counts, widths and NULL fractions, and the column named `floor` is not a
guaranteed lower bound on rebuild savings. The current exclusions leave some
uncertain inputs in the report and remove other indexes entirely. The retained
limitations and proposed repairs are under [Open Questions](#open-questions).
[Current SQL](#the-current-recommended-statement),
[analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663),
[indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L142-L163).

### The current recommended statement

This is the single operational estimator on this page. Its executable expressions,
filters, output names and `wiki_btree_wasted_space_sweep_12_17` tag are retained.
The tag identifies the existing statement; version claims here are limited to the
pinned PostgreSQL 17 source.

Run the two timeout settings with the query in a dedicated session. They set
`statement_timeout` to 30 seconds and `lock_timeout` to 2 seconds. Both are
`PGC_USERSET`: they apply at session/transaction scope without a reload or restart.
The supplied `SET` commands affect the session.
[guc_tables.c#statement_timeout-and-lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631).

The catalog and function definitions used by the block are mapped in
[Evidence Map](#evidence-map). The following sections explain the model and its
limits; the proposals under Open Questions are not implemented in this SQL.

```sql
SET /* wiki_btree_wasted_space_statement_timeout */ statement_timeout = '30s';
SET /* wiki_btree_wasted_space_lock_timeout */ lock_timeout = '2s';


WITH RECURSIVE
env AS (
    -- Server constants and table-level auto-analyze defaults.
    SELECT /* wiki_btree_wasted_space_sweep_12_17 */
           current_setting('block_size')::int                         AS bs,
           current_setting('server_version_num')::int                 AS server_version_num,
           current_setting('autovacuum_analyze_threshold')::int       AS anl_threshold,
           current_setting('autovacuum_analyze_scale_factor')::float8 AS anl_scale_factor
),
idx AS (
    -- Candidate indexes, physical fork sizes, reloptions and row-count inputs.
    -- Negative reltuples stops the model; zero is accepted without validation.
    SELECT c.oid AS idxoid, n.nspname AS schemaname, t.relname AS tablename,
           c.relname AS indexname, t.oid AS tbloid,
           x.indkey, x.indclass, x.indcollation, x.indisunique, x.indnkeyatts,
           t.relrowsecurity                             AS tbl_rls,
           (x.indpred IS NOT NULL)                      AS is_partial,
           (x.indexprs IS NOT NULL)                     AS has_expressions,
           (x.indnatts = x.indnkeyatts)                 AS keys_only,
           e.bs, e.server_version_num, z.actual_bytes, z.fsm_bytes,
           o.fillfactor, o.dedup_on,
           c.reltuples::numeric                         AS idx_reltuples,
           coalesce(s.n_live_tup, 0)::numeric           AS tbl_live_tup,
           coalesce(s.n_dead_tup, 0)::numeric           AS tbl_dead_tup,
           coalesce(s.n_mod_since_analyze, 0)::numeric  AS tbl_mod_since_analyze,
           (o.anl_threshold
            + o.anl_scale_factor * greatest(t.reltuples, 0))::numeric
                                                        AS tbl_autoanalyze_threshold,
           greatest(s.last_analyze, s.last_autoanalyze) AS last_analyze,
           CASE
             WHEN c.reltuples < 0 THEN NULL
             WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
             ELSE least(c.reltuples::numeric,
                        coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
           END                                          AS live_rows
      FROM pg_class c
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_am am       ON am.oid = c.relam
      LEFT JOIN pg_stat_all_tables s ON s.relid = t.oid
      CROSS JOIN env e
      CROSS JOIN LATERAL (
            SELECT pg_relation_size(c.oid)        AS actual_bytes,
                   pg_relation_size(c.oid, 'fsm') AS fsm_bytes) z
      CROSS JOIN LATERAL (
            SELECT coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                              WHERE option_name = 'fillfactor'), 90)          AS fillfactor,
                   coalesce((SELECT option_value::bool FROM pg_options_to_table(c.reloptions)
                              WHERE option_name = 'deduplicate_items'), true) AS dedup_on,
                   coalesce((SELECT option_value::int FROM pg_options_to_table(t.reloptions)
                              WHERE option_name = 'autovacuum_analyze_threshold'
                                AND option_value::int >= 0),
                            e.anl_threshold)                                  AS anl_threshold,
                   coalesce((SELECT option_value::float8 FROM pg_options_to_table(t.reloptions)
                              WHERE option_name = 'autovacuum_analyze_scale_factor'
                                AND option_value::float8 >= 0),
                            e.anl_scale_factor)                               AS anl_scale_factor) o
     WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
),
gate AS (
    -- Recognize known internal equal-image functions for every key opclass.
    SELECT i.idxoid,
           (SELECT bool_and(EXISTS (SELECT 1 FROM pg_amproc ap
                                     JOIN pg_proc pr     ON pr.oid = ap.amproc
                                     JOIN pg_language pl ON pl.oid = pr.prolang
                                     WHERE ap.amprocfamily = op.opcfamily
                                       AND ap.amproclefttype = op.opcintype
                                       AND ap.amprocrighttype = op.opcintype
                                       AND ap.amprocnum = 4
                                       AND pl.lanname = 'internal'
                                       AND pr.prosrc IN ('btequalimage',
                                                         'btvarstrequalimage')))
              FROM generate_subscripts(i.indclass, 1) k
              JOIN pg_opclass op ON op.oid = i.indclass[k]
             WHERE k < i.indnkeyatts)                     AS all_equalimage,
           NOT EXISTS (SELECT 1 FROM generate_subscripts(i.indcollation, 1) k
                         JOIN pg_collation cl ON cl.oid = i.indcollation[k]
                        WHERE k < i.indnkeyatts
                          AND NOT cl.collisdeterministic) AS all_deterministic
      FROM idx i
),
keyatts AS (
    -- Match extended-statistics keys by the sorted table attribute numbers.
    SELECT i.idxoid,
           string_agg(i.indkey[k]::text, ', ' ORDER BY i.indkey[k]) AS ext_key,
           count(*)         AS nkeys,
           min(i.indkey[k]) AS min_attnum
      FROM idx i, generate_subscripts(i.indkey, 1) k
     WHERE k < i.indnkeyatts
     GROUP BY i.idxoid
),
extstat AS (
    -- Use the maximum visible whole-key distinct estimate when available.
    SELECT k.idxoid, max(e.nd) AS ext_ndistinct
      FROM keyatts k
      JOIN idx i           ON i.idxoid = k.idxoid
      JOIN pg_stats_ext se ON se.schemaname = i.schemaname
                          AND se.tablename = i.tablename
      CROSS JOIN LATERAL (
            SELECT ((se.n_distinct::text)::json ->> k.ext_key)::numeric AS nd) e
     WHERE k.nkeys > 1 AND k.min_attnum > 0 AND e.nd > 0
     GROUP BY k.idxoid
),
cols AS (
    -- Index-expression statistics, table-column statistics, then defaults.
    SELECT i.idxoid, a.attnum, a.attlen, a.attalign,
           CASE WHEN a.attlen > 0 THEN a.attlen::numeric
                ELSE coalesce(se.avg_width, st.avg_width, 32)::numeric END AS width,
           coalesce(se.null_frac, st.null_frac, 0)::numeric      AS null_frac,
           coalesce(se.n_distinct, st.n_distinct, 0)::numeric    AS n_distinct,
           coalesce(se.most_common_freqs, st.most_common_freqs)  AS mcf,
           (se.attname IS NULL AND st.attname IS NULL)           AS no_stats_row,
           (NOT coalesce(has_column_privilege(i.tbloid, ta.attnum, 'SELECT'),
                         has_table_privilege(i.tbloid, 'SELECT'))
            OR (i.tbl_rls AND row_security_active(i.tbloid)))    AS stats_hidden
      FROM idx i
      JOIN pg_attribute a ON a.attrelid = i.idxoid AND a.attnum > 0 AND NOT a.attisdropped
      LEFT JOIN pg_stats se ON se.schemaname = i.schemaname
                           AND se.tablename = i.indexname AND se.attname = a.attname
      LEFT JOIN pg_attribute ta ON ta.attrelid = i.tbloid
                               AND ta.attnum = i.indkey[a.attnum - 1]
      LEFT JOIN pg_stats st ON st.schemaname = i.schemaname
                           AND st.tablename = i.tablename AND st.attname = ta.attname
),
statvis AS (
    -- Aggregate missing/hidden statistics and variable-width INCLUDE attributes.
    SELECT c.idxoid,
           bool_or(c.no_stats_row)                    AS any_no_stats,
           bool_or(c.no_stats_row AND c.stats_hidden) AS any_stats_hidden,
           bool_or(c.attnum > i.indnkeyatts AND c.attlen < 0)
                                                      AS any_varlena_include
      FROM cols c
      JOIN idx i ON i.idxoid = c.idxoid
     GROUP BY c.idxoid
),
tuple AS (
    -- Approximate datum widths, joint NULL probability and distinct key groups.
    SELECT i.*,
           (SELECT sum((1 - c.null_frac) *
                       CASE WHEN c.attlen < 0 AND c.width <= 127 THEN c.width
                            ELSE ceil(c.width / al.a) * al.a END)
              FROM cols c
              CROSS JOIN LATERAL (SELECT CASE c.attalign WHEN 'c' THEN 1 WHEN 's' THEN 2
                                              WHEN 'i' THEN 4 ELSE 8 END AS a) al
             WHERE c.idxoid = i.idxoid)                          AS data_size,
           (SELECT 1 - coalesce(exp(sum(ln(greatest(1 - c.null_frac, 1e-9)))), 1)
              FROM cols c WHERE c.idxoid = i.idxoid)             AS p_null,
           CASE WHEN e.ext_ndistinct IS NOT NULL
                THEN least(e.ext_ndistinct, greatest(i.live_rows, 0))
                ELSE
           (SELECT least(round(exp(sum(ln(greatest(
                       CASE WHEN c.n_distinct > 0 THEN c.n_distinct
                            WHEN c.n_distinct < 0 AND NOT i.is_partial
                                 THEN (- c.n_distinct) * greatest(i.live_rows, 0)
                            ELSE (1 - c.null_frac) * greatest(i.live_rows, 0)
                       END
                       + CASE WHEN c.null_frac > 0 THEN 1 ELSE 0 END, 1))))),
                         greatest(i.live_rows, 0))
              FROM cols c
             WHERE c.idxoid = i.idxoid AND c.attnum <= i.indnkeyatts)
           END                                                   AS key_groups,
           (e.ext_ndistinct IS NOT NULL)                         AS ext_used,
           v.any_no_stats, v.any_stats_hidden, v.any_varlena_include
      FROM idx i
      LEFT JOIN extstat e ON e.idxoid = i.idxoid
      LEFT JOIN statvis v ON v.idxoid = i.idxoid
),
page AS (
    -- Approximate page geometry with eight-byte alignment and build posting limits.
    SELECT t.*, g.all_equalimage, g.all_deterministic,
           s.slot, f.leaf_cap, f.int_cap, f.leaf_bytes, f.dedup_applies,
           f.groups_est, f.maxposting, p.nmax
      FROM tuple t
      JOIN gate g ON g.idxoid = t.idxoid
      CROSS JOIN LATERAL (
            SELECT ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8 + 4         AS slot) s
      CROSS JOIN LATERAL (
            SELECT greatest(floor((t.bs - 48 - floor(t.bs * (100 - t.fillfactor) / 100)) / s.slot), 1)
                                                                              AS leaf_cap,
                   greatest(floor((t.bs - 48 - floor(t.bs * 30 / 100)) / s.slot), 2)
                                                                              AS int_cap,
                   (t.bs - 48 - floor(t.bs * (100 - t.fillfactor) / 100))     AS leaf_bytes,
                   (NOT t.indisunique AND t.dedup_on AND t.keys_only
                        AND coalesce(g.all_equalimage, false)
                        AND g.all_deterministic)                              AS dedup_applies,
                   least(greatest(t.live_rows, 0), greatest(t.key_groups, 1)) AS groups_est,
                   floor(floor(t.bs * 10 / 100) / 8) * 8 - 4                  AS maxposting) f
      CROSS JOIN LATERAL (
            SELECT greatest(floor(4 * floor((f.maxposting - (s.slot - 4)) / 8) / 3), 1)
                                                                              AS nmax) p
),
kstat AS (
    -- Use a NULL/MCV mixture only for an eligible single-key non-partial index.
    SELECT p.idxoid,
           CASE WHEN p.is_partial THEN 0 ELSE c.null_frac END AS null_frac,
           CASE WHEN p.is_partial THEN '{}'::real[]
                ELSE coalesce(c.mcf, '{}'::real[]) END        AS mcf
      FROM page p
      JOIN cols c ON c.idxoid = p.idxoid AND c.attnum = 1
     WHERE p.indnkeyatts = 1 AND p.dedup_applies AND p.live_rows > 0
),
gclass AS (
    -- NULL run, each most-common value, remaining values, or one multicolumn class.
    SELECT p.idxoid, greatest(p.live_rows, 0) * k.null_frac AS class_rows,
           1::numeric AS class_groups
      FROM page p JOIN kstat k ON k.idxoid = p.idxoid
     WHERE k.null_frac > 0
    UNION ALL
    SELECT p.idxoid, greatest(p.live_rows, 0) * f, 1::numeric
      FROM page p JOIN kstat k ON k.idxoid = p.idxoid
      CROSS JOIN LATERAL unnest(k.mcf) f
    UNION ALL
    SELECT p.idxoid,
           greatest(greatest(p.live_rows, 0)
                    * (1 - k.null_frac
                         - coalesce((SELECT sum(f) FROM unnest(k.mcf) f), 0)), 0),
           greatest(p.groups_est
                    - CASE WHEN k.null_frac > 0 THEN 1 ELSE 0 END
                    - coalesce(array_length(k.mcf, 1), 0), 1)
      FROM page p JOIN kstat k ON k.idxoid = p.idxoid
    UNION ALL
    SELECT p.idxoid, greatest(p.live_rows, 0), p.groups_est
      FROM page p
     WHERE p.indnkeyatts > 1 AND p.dedup_applies AND p.live_rows > 0
),
classfit AS (
    -- Cap the estimated TIDs per posting tuple within each class.
    SELECT g.idxoid, g.class_rows, g.class_groups,
           least(g.class_rows / greatest(g.class_groups, 1), p.nmax) AS tids,
           p.slot, p.leaf_bytes
      FROM gclass g
      JOIN page p ON p.idxoid = g.idxoid
     WHERE g.class_rows > 0
),
classpages AS (
    -- Round up posting tuples per group; tail tuples still use the full size.
    SELECT c.idxoid,
           sum(least(c.class_groups
                     * ceil(c.class_rows / c.class_groups / greatest(c.tids, 1)),
                     c.class_rows)
               / greatest(floor((c.leaf_bytes
                                 + CASE WHEN c.tids > 1 THEN c.tids * 6 ELSE 0 END)
                                / CASE WHEN c.tids > 1
                                       THEN ceil(((c.slot - 4) + c.tids * 6) / 8) * 8 + 4
                                       ELSE c.slot END), 1)) AS leaf_frac,
           max(c.tids) AS max_tids
      FROM classfit c
     GROUP BY c.idxoid
),
leaves AS (
    -- Leaf-page models with and without deduplication credit.
    SELECT p.*, coalesce(cp.max_tids, 1) AS tids,
           CASE WHEN p.dedup_applies AND cp.leaf_frac IS NOT NULL
                THEN greatest(ceil(cp.leaf_frac), 1)
                ELSE ceil(greatest(p.live_rows, 0) / p.leaf_cap)
           END                                                AS leaf_pages,
           ceil(greatest(p.live_rows, 0) / p.leaf_cap)         AS leaf_pages_floor
      FROM page p
      LEFT JOIN classpages cp ON cp.idxoid = p.idxoid
),
levels AS (
    -- Add internal levels recursively for both models.
    SELECT idxoid, 'dedup'::text AS variant, leaf_pages AS pages, int_cap FROM leaves
    UNION ALL
    SELECT idxoid, 'floor'::text, leaf_pages_floor, int_cap FROM leaves
    UNION ALL
    SELECT l.idxoid, l.variant, ceil(l.pages / l.int_cap), l.int_cap
      FROM levels l WHERE l.pages > 1
),
modelled AS (
    -- Add the metapage and derive statistics flags and all five exclusion terms.
    SELECT l.*, b.expected_blocks, b.floor_blocks,
           w.stats_row_missing, w.dedup_credited, w.stats_stale,
           ((l.is_partial AND (w.stats_row_missing
                            OR w.dedup_credited
                            OR w.stats_stale
                            OR l.any_varlena_include))
            OR (NOT l.is_partial AND l.has_expressions
                AND w.stats_row_missing))                       AS suppress_row
      FROM leaves l
      CROSS JOIN LATERAL (
            SELECT (SELECT sum(v.pages) FROM levels v
                     WHERE v.idxoid = l.idxoid AND v.variant = 'dedup') + 1 AS expected_blocks,
                   (SELECT sum(v.pages) FROM levels v
                     WHERE v.idxoid = l.idxoid AND v.variant = 'floor') + 1 AS floor_blocks) b
      CROSS JOIN LATERAL (
            SELECT (l.any_no_stats AND NOT l.any_stats_hidden
                    AND l.last_analyze IS NOT NULL)                        AS stats_row_missing,
                   (l.dedup_applies AND l.tids > 1)                        AS dedup_credited,
                   (l.tbl_mod_since_analyze > l.tbl_autoanalyze_threshold) AS stats_stale) w
)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(actual_bytes)                     AS index_size,
       CASE
         WHEN idx_reltuples < 0 THEN 'unmeasured: reltuples unknown'
         ELSE 'ok'
       END                                              AS status,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         round((100 * (1 - (expected_blocks * bs) / greatest(actual_bytes, 1)))::numeric, 1)
       END                                              AS wasted_space_pct,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         round((100 * (1 - (floor_blocks * bs) / greatest(actual_bytes, 1)))::numeric, 1)
       END                                              AS wasted_space_pct_floor,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         pg_size_pretty((actual_bytes - expected_blocks * bs)::bigint) END AS wasted_space,
       array_to_string(array_remove(ARRAY[
         CASE WHEN last_analyze IS NULL THEN 'never analyzed' END,
         CASE WHEN any_stats_hidden
              THEN 'statistics not visible to this role' END,
         CASE WHEN stats_row_missing
              THEN 'no statistics row for an index column' END,
         CASE WHEN NOT is_partial
                   AND greatest(tbl_live_tup, idx_reltuples)
                       > 1.1 * greatest(least(tbl_live_tup, idx_reltuples), 1)
              THEN 'row-count sources disagree: analyze first' END,
         CASE WHEN is_partial AND (tbl_dead_tup > 0 OR last_analyze IS NULL)
              THEN 'partial: predicate subset may be stale' END,
         CASE WHEN is_partial AND stats_stale
              THEN 'partial: table changed since the last ANALYZE' END,
         CASE WHEN is_partial AND dedup_credited
              THEN 'partial: duplicates from table statistics' END,
         CASE WHEN dedup_credited THEN 'deduplication credited' END,
         CASE WHEN ext_used THEN 'key groups from extended statistics' END
       ], NULL), '; ')                                  AS caveats,
       key_groups::bigint                               AS key_groups,
       round(tids::numeric, 1)                          AS tids_per_tuple,
       live_rows::bigint                                AS modelled_rows,
       idx_reltuples::bigint                            AS idx_reltuples,
       fsm_bytes > 0                                    AS fsm_written_since_build,
       server_version_num
  FROM modelled
 WHERE actual_bytes > 1024 * 1024 AND NOT suppress_row
 ORDER BY (actual_bytes - floor_blocks * bs) DESC NULLS FIRST
 LIMIT 20;
```

### Reading the output

The following definitions come from the statement's final projection and filters.
`index_size` and `wasted_space` use the core `pg_size_pretty` function, whose result
is text. The size input is the main fork returned by `pg_relation_size`; the FSM
fork is read separately.
[Current SQL](#the-current-recommended-statement),
[pg_proc.dat#size-functions](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7507),
[dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371).

| Output | Meaning in this statement |
|---|---|
| `status` | `unmeasured: reltuples unknown` when the index's `reltuples` is negative; otherwise `ok`. `ok` is not a freshness or accuracy certification. |
| `wasted_space_pct` | Signed percentage difference from `expected_blocks`, the model that can credit deduplication. |
| `wasted_space_pct_floor` | Signed percentage difference from `floor_blocks`, the model without deduplication credit. Row-count and width uncertainty still apply. |
| `wasted_space` | Formatted signed `actual_bytes - expected_blocks * bs`; a negative value means the model predicts a larger rebuild. |
| `caveats` | Visible qualifications assembled by the query. A filtered-out index has no output row or explanation. |
| `key_groups`, `tids_per_tuple` | Modelled distinct groups and posting-list occupancy. A TID is a heap tuple identifier. |
| `modelled_rows`, `idx_reltuples` | The row estimate used by the model and the index's catalog estimate. Read a zero as an input requiring validation. |
| `fsm_written_since_build` | The test `fsm_bytes > 0`; it does not count free pages or establish that any page remains reusable. |
| `server_version_num` | The server's version number, reported without selecting a different formula. |

For triage, read `wasted_space_pct_floor` with `status` and `caveats`. Do not
promote a reading with `never analyzed`, `row-count sources disagree: analyze first`
or `statistics not visible to this role` into a rebuild decision. A wide gap between
the percentages identifies dependence on the duplication estimate. These checks
do not validate default widths, partial-index populations or a zero row count.
[Current SQL](#the-current-recommended-statement),
[analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975),
[system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275).

The `statistics not visible to this role` test has a gap for expression
attributes. `pg_stats` keeps a row only when `has_column_privilege` passes on the
relation that owns the statistics row, and for an expression attribute that
relation is the index. The statement tests the table's privileges instead. A role
that can read the table but lacks the index owner's rights therefore receives
`no statistics row for an index column`, and a non-partial expression index is
suppressed without explanation. See [Plan review](#plan-review), finding 1.
[Current SQL](#the-current-recommended-statement),
[system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275),
[acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569),
[aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412).

The report keeps indexes larger than `1024 * 1024` bytes, excludes `suppress_row`,
orders by the signed floor-model byte difference with NULLs first, and returns at
most 20 rows. An absent index can therefore be below the size cutoff, suppressed,
or outside the top 20. Do not interpret absence as a clean bill of health.
[Current SQL](#the-current-recommended-statement).

The index free space map (FSM) stores whether pages are free or used.
`GetFreeIndexPage` marks a returned page used through `RecordUsedIndexPage`.
The statement measures the fork's length rather than those entries, so its FSM
boolean is not a current free-page census.
[indexfsm.c#index-FSM](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L65).

### How the model uses PostgreSQL 17

The query's common table expressions (CTEs) divide the calculation into the
following stages. This table describes the SQL above; the source column identifies
the engine input or build rule each stage approximates.

| Stages | Current calculation | Source boundary |
|---|---|---|
| `env`, `idx` | Read server constants, valid physical B-tree indexes, reloptions, main/FSM sizes and row-count inputs. | [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L26-L62), [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66), [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371). |
| `gate` | Recognize the two built-in equal-image implementations and reject nondeterministic collations. | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183). |
| `keyatts`, `extstat` | Look for a whole-key distinct-count entry; use the maximum matching visible estimate. | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309), [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385). |
| `cols`, `statvis`, `tuple` | Prefer index-expression statistics, otherwise table-column statistics, then defaults; estimate tuple width and key groups. | [analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478), [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211), [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262). |
| `page` | Approximate tuple slots, leaf/internal capacity and the build's posting-list size cap. | [nbtsort.c#page-initialization](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L671), [nbtsort.c#posting-size-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1308). |
| `kstat`, `gclass`, `classfit`, `classpages` | Split eligible single-key rows into NULL, most-common-value and remaining classes; use one class for multicolumn keys; round each group's posting-tuple count up. | [nbtsort.c#group-boundaries](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1310-L1349), [nbtdedup.c#posting-size-check](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L500-L517). |
| `leaves`, `levels`, `modelled` | Estimate leaf pages, recursively add internal levels and a metapage, then derive suppression and caveat conditions. | [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571), [nbtsort.c#BTPageState](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L229-L252). |

The engine build path is `index_build` → the access method's `ambuild` callback →
`btbuild` → `_bt_leafbuild` → `_bt_load` → `_bt_buildadd`. `BTWriteState` holds the
index and allocation state; each `BTPageState` holds the working page and a link to
its parent level. The estimator approximates this path without invoking a build.
[index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3048-L3053),
[nbtree.c#ambuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L128-L129),
[nbtsort.c#btbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L290-L328),
[nbtsort.c#build-state](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L229-L252),
[nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571),
[nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1349),
[nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1026-L1050).

Tuple geometry is an approximation. `IndexTupleData` contains the TID and size/flag
word; a posting tuple adds an array of `ItemPointerData`. `PageHeaderData` includes
the line-pointer array, and `BTPageOpaqueData` holds B-tree sibling/level metadata.
Actual tuple formation omits NULL values, aligns attributes in order and can
compress variable-width values. The SQL hard-codes eight-byte alignment and uses
average widths, so those inputs do not reconstruct every physical tuple.
[itup.h#IndexTupleData](../../../../raw/postgres-17/src/include/access/itup.h#L35-L60),
[itemptr.h#ItemPointerData](../../../../raw/postgres-17/src/include/storage/itemptr.h#L25-L47),
[nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L864-L910),
[bufpage.h#PageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L155-L171),
[nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71),
[indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L94-L163),
[heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262),
[Current SQL](#the-current-recommended-statement).

### Deduplication eligibility

Deduplication stores one key image with several heap TIDs. For a fresh build,
`_bt_leafbuild` recomputes equal-image eligibility. `_bt_allequalimage` refuses
INCLUDE indexes, then retrieves and calls each key's support function. The sorted
build also requires a non-unique index and `deduplicate_items` enabled.
[nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L557-L570),
[nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183),
[nbtsort.c#deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1135-L1152).

The SQL deliberately recognizes `LANGUAGE internal` functions whose `prosrc` is
`btequalimage` or `btvarstrequalimage`. The first returns true; the second checks
collation suitability. `prosrc` is also how the function manager resolves an
internal-function alias. This policy does not execute custom support functions,
so their eligibility can remain unknown even when the engine would deduplicate.
[Current SQL](#the-current-recommended-statement),
[datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L424-L438),
[varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615),
[fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240).

### Row counts and exclusions

The SQL treats negative index `reltuples` as unknown and takes zero at face value.
For a partial index, whose predicate selects a subset of table rows, it uses the
index's `reltuples`. For a non-partial index it takes the lesser of that estimate
and the table's nonzero `n_live_tup`, falling back to `reltuples` when the table
counter is zero. Neither input is an exact current count.
[Current SQL](#the-current-recommended-statement),
[pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66),
[analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663),
[system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703).

ANALYZE is not the only writer of an index's `reltuples`. A VACUUM that removed
tuples counts the live heap TIDs on each leaf page, clamps the total to the heap
count when that count is exact, and writes it through the same relation-statistics
path. A cleanup-only scan counts index tuples instead, marks the result as an
estimate, and the write is skipped. The statement reads only the ANALYZE
timestamps, so it cannot tell which writer produced the value it uses.
[nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1347-L1362),
[nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L870-L920),
[vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3099),
[Current SQL](#the-current-recommended-statement).

`ANALYZE` gives index attributes their own column statistics only for expressions.
`IndexInfo` identifies expression attributes and the predicate;
`compute_index_stats` evaluates that predicate over its sample. Plain keys and
INCLUDE columns therefore fall back to table statistics in this query, even when
the index covers only a subset.
[analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478),
[analyze.c#index-statistics-write](../../../../raw/postgres-17/src/backend/commands/analyze.c#L588-L602),
[analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L845-L884),
[execnodes.h#IndexInfo](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L183-L193),
[Current SQL](#the-current-recommended-statement).

Five conditions set `suppress_row`. The table spells out the current expressions,
including their qualifications; it does not promise that a later ANALYZE clears
each condition.
[Current SQL](#the-current-recommended-statement).

| Index scope | Suppression condition |
|---|---|
| Partial | At least one attribute has no returned statistics row, none is classified as hidden, and a table ANALYZE timestamp exists. |
| Partial | The model credits deduplication and computes more than one TID per tuple. |
| Partial | The table's modification counter exceeds the model's auto-analyze threshold. |
| Partial | At least one INCLUDE attribute has negative `attlen`, identifying variable width. |
| Non-partial expression index | The same missing-statistics condition as the first row. |

The staleness threshold uses the auto-analyze formula, with per-table reloptions
overriding the GUC defaults: threshold plus scale factor times nonnegative table
`reltuples`. It is a table-wide scheduling threshold, not a test of an individual
predicate subset's freshness.
[autovacuum.c#analyze-reloptions](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017),
[autovacuum.c#analyze-threshold](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3095),
[pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-17/src/include/pgstat.h#L399-L429),
[Current SQL](#the-current-recommended-statement).

Every reported `modelled_rows = 0` needs a separate population check before its
near-total-waste estimate is trusted. ANALYZE estimates partial-index membership
from its sample, so a zero does not prove an empty subset. A zero
`n_mod_since_analyze` does not prove freshness either: ANALYZE resets it, and
cumulative-statistics publication has its own timing. The current SQL contains no
zero-validation guard.
[analyze.c#sample-membership](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975),
[pgstat_relation.c#analyze-counter-reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L337),
[pgstat.c#publication-timing](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665),
[Current SQL](#the-current-recommended-statement).

### Operational and verification limits

`pg_relation_size` takes and releases `AccessShareLock` on each relation. It returns
NULL if a relation has disappeared, and the file-size loop can raise a file-access
error. These calls inspect file lengths; they do not provide a single physical
snapshot of all indexes and statistics in the report.
[dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371).

No contrib extension is needed by the recommended statement: its inputs are core
catalogs, views and functions. The custom-operator-class boundary remains the
whitelist described above. The build-dependent constants still matter: catalog
headers include generated `_d.h` files, `genbki.pl` builds catalog outputs from
the source definitions, and `Gen_fmgrtab.pl` uses `pg_proc.dat` to generate function
lookup support. The SQL's literal layout assumptions need separate validation
on a different build configuration.
[Current SQL](#the-current-recommended-statement),
[pg_class.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L22),
[pg_index.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L21-L22),
[catalog/Makefile#genbki](../../../../raw/postgres-17/src/include/catalog/Makefile#L132-L143),
[utils/Makefile#Gen_fmgrtab](../../../../raw/postgres-17/src/backend/utils/Makefile#L47-L53).

This cleanup does not rerun the historical fixture databases or establish fresh
cross-version accuracy figures. Agent verification remains `not yet`. The pinned
regression files provide adjacent engine coverage: deduplication in
`btree_index.sql`, and statistics-setting restrictions for plain, expression and
INCLUDE attributes in `index_including.sql`. They do not constitute an accuracy
test of this wiki's estimator.
[btree_index.sql#deduplication-tests](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L186-L213),
[index_including.sql#statistics-tests](../../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L150-L158).

### Plan review

**Keep all ten plans. Three of them describe defects that are already
deterministic in the current statement, and those repairs come first.** This
review checks each plan under [Open Questions](#open-questions) against the
pinned PostgreSQL 17 source at `786db8dcf168bd9df8f55047337525ac19118b1c`. It
adds no execution result: no server was built or started, the recommended SQL is
unchanged, and the historical measurements are not re-verified.

| Plan | Verdict | Principal finding |
|---|---|---|
| 1. Input diagnostics | Keep, extend | Expression-attribute statistics are visible only to superusers, the index owner and the owner's role members; the statement tests the table's privileges, so other roles get `no statistics row` rather than `not visible`. |
| 2. Partial-index probes | Keep, simplify | The zero check can be an `EXISTS` probe; for gated indexes a `GROUP BY` on the key reproduces the builder's grouping; VACUUM is a second writer of index `reltuples`. |
| 3. Publication barrier | Keep, strengthen | A forced flush completes before the writer's `ReadyForQuery` outside a transaction block, so the barrier is ordered; the counter artifact has a source mechanism. |
| 4. Posting tails | Keep | The two-size formula follows the build; posting capacity varies per group for variable-width keys. |
| 5. Page geometry | Narrow | The leaf closed form reproduces the builder's soft limit for uniform tuples up to 896 bytes; residual errors are pivots, size variance and posting overhead. |
| 6. Statistics selection | Upgrade to defect | An index on an inheritance parent joins two `pg_stats` rows per attribute and double-counts the width, NULL and distinct inputs. |
| 7. Operator classes | Keep, bound | All 29 built-in B-tree support-function-4 records use the two recognized functions; pattern operator classes reject nondeterministic collations at index creation; the metapage flag is a harness oracle. |
| 8. Signed bytes | Keep, simplify | `pg_size_pretty(numeric)` exists, and the existing `bigint` casts can raise `bigint out of range`. |
| 9. Reproducibility | Keep, constrain | Builds must be out of tree; the SQL block hash baseline moved with the 2026-09-07 comment edit. |
| 10. Thresholds | Keep, refine | Insert-grown density depends on the split path: fillfactor, 50:50, or 96 percent. Calibrate per insertion pattern. |

1. **Input diagnostics: test the index relation's privilege for expression
   attributes.** `pg_stats` keeps a row only when `has_column_privilege` passes
   for the relation that owns the statistics row, and for an expression attribute
   that relation is the index. The check tries the column ACL first, and columns
   have no default privileges; it then falls back to the relation ACL. An index's
   `relacl` is empty because `GRANT` refuses indexes, so the fallback builds the
   default table ACL, which grants the owner and nobody else. A role passes only
   as a superuser, as the owner, or through membership in the owner. The
   statement's `stats_hidden` evaluates the table's privileges, so any other role
   is classified `stats_row_missing`, gets `no statistics row for an index
   column`, and loses non-partial expression indexes to `suppress_row`. The
   diagnostic must evaluate `has_column_privilege(index, attnum, 'SELECT')` for
   expression attributes. It should also read `attstattarget`, which is nullable
   on this pin: NULL means the default target and zero makes ANALYZE skip the
   attribute, which separates a disabled column from an unanalyzed one.
   [system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275),
   [acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569),
   [acl.c#acldefault](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L813-L821),
   [aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412),
   [acl.c#aclmask](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L1388-L1445),
   [aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863),
   [pg_attribute.h#attstattarget](../../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L168-L176),
   [analyze.c#examine_attribute](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1019-L1030),
   [Current SQL](#the-current-recommended-statement).

2. **Partial-index probes: three simplifications and one missing input.** The
   population check for a suspected empty subset can be
   `EXISTS (SELECT 1 FROM ONLY t WHERE <predicate>)`, with the predicate text from
   `pg_get_expr`: the executor stops at the first row, and the planner can use the
   partial index itself when the query clauses imply its predicate. For an index
   that passes the deduplication gate, the build groups adjacent sorted tuples by
   binary image equality in `_bt_keep_natts_fast`, treating two NULLs as equal, so
   a `GROUP BY` over the key expressions with the index collation counts the
   builder's groups; leave the group probe undefined for an unrecognized operator
   class. The probe needs no separate client: `DO` is core grammar and `initdb`
   installs PL/pgSQL, so a generated statement can run through `EXECUTE`. The
   missing input is the second writer of `reltuples` described under
   [Row counts and exclusions](#row-counts-and-exclusions): a count-validation
   policy must compare the index value against the later of the table's VACUUM and
   ANALYZE timestamps and record which writer it assumes.
   [pg_proc.dat#pg_get_expr](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8219-L8220),
   [ruleutils.c#pg_get_expr_ext](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L2648-L2662),
   [nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296),
   [indxpath.c#check_index_predicates](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3244-L3350),
   [nbtsort.c#_bt_load-grouping](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1314-L1316),
   [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905),
   [datum.c:271](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L271),
   [gram.y#DoStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L9037),
   [initdb.c#load_plpgsql](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1974-L1977),
   [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L870-L920),
   [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3099),
   [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703).

3. **Publication barrier: the flush is ordered, and the artifact has a
   mechanism.** `pg_stat_force_next_flush()` sets a process-local flag. When a
   backend finishes a statement outside a transaction block, it calls
   `pgstat_report_stat(false)` before sending `ReadyForQuery`; that call absorbs
   the flag, treats the report as forced, skips the one-second minimum interval,
   and waits for locks instead of giving up. After the writer's forced-flush
   statement returns in autocommit mode, its committed counts are therefore in
   shared memory, and the harness needs no polling deadline for that step. The
   barrier is also necessary. A transaction's inserts, updates and deletes become
   pending deltas at commit; an unforced flush is deferred when the backend
   flushed less than one second earlier; ANALYZE writes live and dead counts as
   absolute values and resets the change counter directly in shared memory; and
   the deferred flush later adds the pending deltas on top. A fixture that runs
   DML, ANALYZE and the estimator in quick succession can therefore report
   `n_live_tup` above the table count and a nonzero `n_mod_since_analyze` after
   ANALYZE, depending only on flush timing. The observer needs
   `pg_stat_clear_snapshot()` only inside a transaction block, because the
   snapshot is discarded at transaction end. `stats_fetch_consistency` is
   `PGC_USERSET`: session or transaction scope, no reload or restart.
   [pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708),
   [postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705),
   [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600),
   [pgstat.c#flush-intervals](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122),
   [pgstat.c#publication-timing](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665),
   [pgstat_relation.c#AtEOXact_PgStat_Relations](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L560-L574),
   [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337),
   [pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860),
   [pgstat.c#pgstat_clear_snapshot](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L786-L800),
   [pgstatfuncs.c#snapshot-and-flush-functions](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c#L1680-L1695),
   [guc_tables.c#stats_fetch_consistency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974),
   [stats.sql#forced-flush](../../../../raw/postgres-17/src/test/regress/sql/stats.sql#L101-L102).

4. **Posting tails: the formula is confirmed; posting capacity is per group.**
   The build caps a posting tuple at `MAXALIGN_DOWN(bs * 10 / 100) - sizeof(ItemIdData)`,
   refuses another TID when `MAXALIGN(basetupsize + (nhtids + 1) * sizeof(ItemPointerData))`
   would exceed that cap, writes a one-item group as the base tuple, and sizes a
   longer list as `MAXALIGN(keysize + nhtids * sizeof(ItemPointerData))`. The
   plan's `S(1) = K + I`, `S(n) = align_up(K + nT, A) + I` and `q * S(m) + S(t)`
   follow those lines. Two additions are needed. The capacity `m` depends on the
   base tuple size `K`, so for variable-width keys it differs per group; the
   acceptance set must include groups whose `K` straddles a capacity boundary,
   not only fixed-width keys. And the builder subtracts the last tuple's posting
   overhead from its soft-limit test through `btps_lastextra`, which the statement
   approximates by adding `tids * 6` to the usable bytes; the tail pricing must
   pass the tail's own overhead to that term, not the full tuple's.
   [nbtsort.c#maxpostingsize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1304-L1305),
   [nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513),
   [nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1026-L1050),
   [nbtdedup.c#_bt_form_posting-size](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L879-L884),
   [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854),
   [nbtsort.c#BTPageState](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L229-L252),
   [Current SQL](#the-current-recommended-statement).

5. **Page geometry: the leaf closed form is exact for uniform tuples, so the
   reference implementation belongs in the harness.** A new page pre-allocates
   the high-key line pointer, and `PageGetFreeSpace` subtracts one more line
   pointer; with the 24-byte page header and the 16-byte opaque area that is the
   statement's 48-byte deduction. The builder closes a page when free space plus
   the last posting overhead drops below `bs * (100 - fillfactor) / 100`, then
   moves the last item to the next page and turns its old slot into the high key.
   For tuples of one aligned size `s`, the items left on each closed page are
   `floor((bs - 48 - btps_full) / (s + 4))`, which is the statement's `leaf_cap`.
   Applying the fit test to every aligned size shows that this identity holds up
   to 896-byte tuples at the default block size and alignment; larger tuples can
   lose one item per page to the hard-fit reservation of
   `MAXALIGN(sizeof(ItemPointerData))`. Internal levels differ: the first item on
   an internal page is an 8-byte minus-infinity tuple, pivots carry no posting
   list and may be truncated or gain a heap TID, and the rightmost page of each
   level is never closed by the soft limit. The plan should aim its reference
   implementation at size variance, oversized tuples, pivot sizing and posting
   overhead, and stop treating leaf packing of uniform tuples as unknown.
   [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629),
   [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L907-L923),
   [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L214),
   [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71),
   [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671),
   [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1147),
   [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854),
   [nbtsort.c#page-boundary](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L874-L935),
   [nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735),
   [Current SQL](#the-current-recommended-statement).

6. **Statistics selection: the missing `inherited` filter is a defect, not a
   preference.** `analyze_rel` runs the non-inherited pass and then, when the
   table has children, the inherited pass; `update_attstats` stores each pass
   under its own `stainherit` value, which `pg_stats` exposes as `inherited`. The
   statement joins `pg_stats` by schema, table and attribute name only, so an
   index on an inheritance parent receives two rows per attribute. The `cols`
   aggregates then double `data_size`, apply each NULL fraction twice in `p_null`,
   and multiply each key's distinct count twice into `key_groups`. Partitioned
   parents are unaffected: their indexes are not `relkind = 'i'`, and they receive
   only the inherited pass. The repair is `inherited = false` on both `pg_stats`
   joins; `pg_stats_ext` carries the same column, and the `max()` in `extstat`
   must exclude inherited objects as well. The `pg_ndistinct` key format the
   statement parses, attribute numbers joined by a comma and a space, matches the
   output function.
   [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259),
   [analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647),
   [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211),
   [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309),
   [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385),
   [Current SQL](#the-current-recommended-statement).

7. **Operator classes: the whitelist covers every built-in family, and the
   harness has an oracle.** `pg_amproc.dat` defines 29 B-tree support-function-4
   records: 26 name `btequalimage` and three, for `text`, `name` and `bpchar` in
   `text_ops` and `bpchar_ops`, name `btvarstrequalimage`. No built-in family uses
   another function, so the unknown state can arise only from functions created
   outside the catalog. The gate's separate determinism test is stricter than
   the engine only for a `btequalimage` family whose column has a collation,
   which in core means the `text_pattern_ops`, `varchar_pattern_ops` and
   `bpchar_pattern_ops` classes; index creation rejects a nondeterministic
   collation for exactly those classes, so no built-in case is under-credited.
   `btvarstrequalimage` accepts the default collation unconditionally, and the
   default locale is always deterministic on this pin. The build stores
   `_bt_allequalimage`'s result in the metapage, and contrib `pageinspect`'s
   `bt_metap` reads that field, so a disposable harness cluster can score
   `dedup_applies` against `allequalimage AND NOT indisunique AND deduplicate_items`
   per fixture. The production statement stays core-only and cannot read it.
   [pg_amproc.dat#bpchar_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L31-L33),
   [pg_amproc.dat#text_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L205-L212),
   [index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849),
   [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615),
   [pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575),
   [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L557-L570),
   [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84),
   [nbtsort.c:1126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1126),
   [nbtsort.c#deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152),
   [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921),
   [Current SQL](#the-current-recommended-statement).

8. **Signed bytes: prefer `numeric`, and remove the existing overflow path.**
   `pg_size_pretty` has a `numeric` overload, so the byte delta can be formatted
   without a `bigint` cast; the `bigint` version formats a negative input from its
   absolute value and keeps the sign. The statement's current casts of
   `wasted_space`, `key_groups`, `modelled_rows` and `idx_reltuples` to `bigint`
   are unguarded: a `numeric` outside the range raises `bigint out of range` and
   aborts the whole report, and `reltuples` is a `float4` that can hold such a
   value. The proposed additive field should be `numeric`, or share one range
   guard with those casts. The ordering expression keeps the floor-model bytes,
   as the plan requires.
   [pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507),
   [dbsize.c#pg_size_pretty-sign](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L569-L600),
   [numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549),
   [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66),
   [Current SQL](#the-current-recommended-statement).

9. **Reproducibility: build out of tree, re-baseline the hash, and allow contrib
   in the harness.** The checkout is read-only evidence, so a reference server
   must be configured from a separate build directory; the pinned installation
   documentation describes a VPATH build for `configure` and a mandatory build
   directory for `meson setup`. The fenced SQL block on this page changed on
   2026-09-07 when stage comments replaced change numbers, so the harness must
   hash the block it extracts rather than the earlier `cb6fb5ce…` value; the
   current block's SHA-256 is
   `bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357`. Installing
   `pageinspect` in a disposable cluster is compatible with the core-only contract
   as long as the production statement stays unchanged, and finding 7 needs it.
   Each fixture must also record which writer last set each index's `reltuples`
   (finding 2) and whether the barrier (finding 3) completed, or the run scores
   an input the estimator never saw.
   [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432),
   [installation.sgml#meson-setup](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L2012-L2025),
   [catalog/Makefile#genbki](../../../../raw/postgres-17/src/include/catalog/Makefile#L132-L143),
   [Current SQL](#the-current-recommended-statement).

10. **Thresholds: calibrate per insertion pattern, because the split path sets
    the steady-state density.** A rightmost leaf split applies the leaf
    fillfactor and a rightmost internal split applies 70 percent; other leaf
    splits use 50:50 unless the split-after-new-item heuristic applies; and a
    leaf page full of one value, on the rightmost side of that value's pages,
    uses the single-value strategy at 96 percent. An append-only index therefore
    settles near the sorted build's density, a random-insert index below it, and
    a duplicate-heavy index above the leaf fillfactor. A single percentage
    threshold mixes those classes. The sweep should label each fixture with its
    insertion pattern and report the four outcome counts per class, and the
    prospective observation should establish a production index's class before a
    rebuild is scheduled.
    [nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335),
    [nbtsplitloc.c#single-value-strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416),
    [nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033),
    [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L199-L202).

### Revised implementation order

1. Repair the deterministic defects in one reviewed change: the `inherited = false`
   filters (finding 6), the index-relation privilege test for expression
   attributes (finding 1), and `numeric` output in place of the unguarded `bigint`
   casts (finding 8). Score them with catalog-only fixtures: an inheritance parent
   with its own index, a non-owner role reading an expression index, and an index
   whose `reltuples` exceeds the `bigint` range.
2. Build the harness out of tree and adopt the ordered barrier (findings 3 and 9),
   recording the last `reltuples` writer per index.
3. Add the diagnostic projection and the probe generator (findings 1 and 2), with
   the `EXISTS` zero check and the `GROUP BY` group probe for gated indexes.
4. Change the arithmetic one step at a time: tail pricing with per-group capacity
   (finding 4), then pivot and size-variance modelling only where the harness
   shows a residual after the uniform-size identity holds (finding 5).
5. Add the three-state operator-class output, scored against `bt_metap` in the
   harness (finding 7).
6. Run the threshold sweep by insertion pattern (finding 10) before promoting any
   revised statement.

## Context Reviewed

- PostgreSQL 17 pin `786db8dcf168bd9df8f55047337525ac19118b1c`; the source checkout is read-only.
- The current statement's catalog inputs, CTE dependencies, formulas, output projection, exclusions and timeout settings.
- The sorted-build caller/callee path, page and tuple structures, expression-statistics selection, partial-index sample counts, statistics visibility/publication, and the generated catalog/function boundary cited above.
- The adjacent regression files listed above. No database server was built or started for this editorial cleanup.
- Plan review on 2026-09-07, same pin: privilege resolution behind the statistics views (`acl.c`, `aclchk.c`), VACUUM's index tuple counting and relation-statistics update (`nbtree.c`, `vacuumlazy.c`), cumulative-statistics flush ordering in the backend main loop and the ANALYZE writer (`postgres.c`, `pgstat.c`, `pgstat_relation.c`), the sorted build's page-transition and posting-size arithmetic (`nbtsort.c`, `nbtdedup.c`, `bufpage.c`), the inheritance passes of ANALYZE, the built-in support-function-4 inventory and the pattern-operator-class collation check (`pg_amproc.dat`, `index.c`), the metapage flag and its contrib reader, size formatting and numeric range errors, split strategies, `EXISTS` evaluation and partial-index predicate implication, `DO` and PL/pgSQL availability, and the VPATH and Meson build documentation. No server was built or started; the leaf-capacity identity in finding 5 is arithmetic applied to the cited fit test, not a measurement.

## Evidence Map

| Claim group | Primary evidence |
|---|---|
| Candidate and count inputs | [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L26-L62), [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66), [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703). |
| Sizes, relation locks and missing-relation behavior | [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371). |
| Sorted-build geometry and deduplication | [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L784-L855), [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1349), [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183). |
| Width, NULL and expression inputs | [indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L94-L163), [analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478), [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211). |
| Joint statistics and access filtering | [system_views.sql#statistics-views](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L309), [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385). |
| Partial counts, table threshold and publication limits | [analyze.c#sample-membership](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975), [autovacuum.c#analyze-threshold](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3095), [pgstat.c#publication-timing](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665). |
| Core-SQL contract and model-specific choices | [The current recommended statement](#the-current-recommended-statement). These expressions are the wiki's model, not a PostgreSQL engine guarantee. |
| Statistics visibility and privilege defaults (plan review 1) | [acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569), [acl.c#acldefault](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L813-L821), [aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412), [aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863). |
| Index `reltuples` writers and probe paths (plan review 2) | [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L870-L920), [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3099), [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905), [nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296). |
| Flush ordering and the counter artifact (plan review 3) | [postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705), [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600), [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337), [pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860). |
| Build page transitions and posting sizes (plan review 4 and 5) | [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629), [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854), [nbtsort.c#page-boundary](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L874-L935), [nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513), [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L907-L923). |
| Inheritance statistics rows (plan review 6) | [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259), [analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647). |
| Built-in equal-image inventory and the metapage flag (plan review 7) | [pg_amproc.dat#text_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L205-L212), [index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84), [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921). |
| Output formatting, range errors and split strategies (plan review 8 and 10) | [dbsize.c#pg_size_pretty-sign](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L569-L600), [numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549), [nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033). |

## Open Questions

These are unresolved limitations and proposed work on the current statement.
They are not alternate operational statements or completed fixes.

### Input diagnostics and missing statistics

The current query hides suppressed rows, uses default widths, and does not suppress
every plain-column case with missing statistics. An attribute with statistics target
zero is skipped by ANALYZE. A missing `pg_stats` result can also reflect access
filtering; it cannot prove that a hidden statistics row exists. A diagnostic result
should expose each attribute's input source and each exclusion reason before the
size and top-20 filters. Its effect on recovered and lost detections is unmeasured.
[Current SQL](#the-current-recommended-statement),
[analyze.c#examine_attribute](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1019-L1030),
[system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275).

Plan review, finding 1: the diagnostic must evaluate `has_column_privilege` on
the index relation for expression attributes, because `pg_stats` filters those
rows by the index's owner-only default ACL, and it should read the nullable
`attstattarget` to separate a disabled column from an unanalyzed one.
[Plan review](#plan-review),
[acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569),
[aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412),
[pg_attribute.h#attstattarget](../../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L168-L176).

### Partial-index widths and zero counts

Width provenance and count freshness need independent repairs. Removing a
deduplication condition cannot validate the widths and row counts used by the floor
model. Plain keys in a mixed expression index still need their own provenance;
variable-width INCLUDE exclusions can hide real savings; and a partial subset can
change below the table-wide threshold. Proposed subset probes must validate the
predicate population, widths, NULL pattern and expression results. An empty sample
must remain inconclusive. Neither the probes nor a zero-validation policy is
implemented in the current statement.
[Current SQL](#the-current-recommended-statement),
[analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478),
[analyze.c#sample-membership](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975),
[autovacuum.c#analyze-threshold](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3095).

Plan review, finding 2: the zero check can be an `EXISTS` probe that the planner
may satisfy from the partial index; for indexes that pass the gate, a `GROUP BY`
over the key reproduces the builder's binary-equality grouping; the probe can run
from a `DO` block; and the count policy must account for VACUUM writing
`reltuples` from live heap TIDs.
[Plan review](#plan-review),
[nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296),
[nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905),
[nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1347-L1362).

### Statistics publication and test ordering

The model mixes catalog estimates with cumulative table counters. Publication
timing and ANALYZE's counter reset can affect those inputs independently. A
reproducible fixture must assert writer/observer ordering and the observed
maintenance/count state before scoring. The historical counter artifacts have not
been reproduced in this cleanup, and a zero counter remains insufficient evidence
of a current subset count.
[pgstat.c#publication-timing](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665),
[pgstat_relation.c#analyze-counter-reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L337),
[Current SQL](#the-current-recommended-statement).

Plan review, finding 3: a `pg_stat_force_next_flush()` issued by the writer
outside a transaction block completes before that statement's `ReadyForQuery`,
which gives the fixture an ordered barrier; the counter artifact follows from
ANALYZE's absolute write and a later additive flush of pending deltas.
[Plan review](#plan-review),
[postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705),
[pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600),
[pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860).

### Posting-tuple tails

`classpages` rounds each estimated key group up to whole posting tuples but prices
the last partial tuple like a full one. The engine caps the accumulated posting
list and flushes it at group boundaries, so a proposed repair must price full
tuples and the tail separately while conserving the estimated row count. Skew and
uncertain group counts remain separate input errors. Tests around every posting
capacity boundary are still required.
[Current SQL](#the-current-recommended-statement),
[nbtdedup.c#posting-size-check](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L500-L517),
[nbtsort.c#group-boundaries](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1310-L1349).

Plan review, finding 4: the two-size formula follows the build's cap, refusal
and sizing rules; add groups whose base tuple size changes the posting capacity,
and pass the tail's own overhead to the page-fill term.
[Plan review](#plan-review),
[nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513),
[nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854).

### Page geometry and tuple representation

The model uses one averaged slot size for leaf and internal capacity. The builder
has distinct hard-fit and fillfactor conditions, and leaf boundary processing can
change pivot size. Exact leaf packing, internal fanout, NULL/alignment interactions,
compression, other block sizes and non-eight-byte alignment remain validation
gaps. A proposed page-transition model must separate these errors from posting-tail
and statistics errors.
[Current SQL](#the-current-recommended-statement),
[nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L809-L855),
[indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L94-L163),
[heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262).

Plan review, finding 5: the leaf closed form reproduces the builder's soft limit
for uniform tuples up to 896 bytes at the default block size and alignment. The
remaining leaf-level gaps are size variance and tuples above that size; the
internal-level gaps are pivot sizing and the minus-infinity first item.
[Plan review](#plan-review),
[nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854),
[nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735).

### Statistics selection and joint distributions

The current joins do not select `inherited = false`, and `extstat` takes the maximum
matching distinct estimate. The views expose inheritance state and distinct,
dependency and most-common-value data separately. A proposed selection rule must
identify the physical relation and complete key, retain attribute-level provenance,
and test correlated, independent, skewed and expression-key populations. A joint
distinct count alone does not supply the duplicate-group frequency distribution.
[Current SQL](#the-current-recommended-statement),
[system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211),
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309).

Plan review, finding 6: without `inherited = false`, an index on an inheritance
parent receives two `pg_stats` rows per attribute, and the width, NULL and
distinct inputs are double-counted. This is a defect in the current statement,
not a policy choice.
[Plan review](#plan-review),
[analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259),
[analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647).

### Custom operator classes

The two-function policy can decline deduplication credit that a custom support
function would allow. The engine calls that function; the query does not. A
proposed diagnostic should distinguish recognized eligibility, known ineligibility
and unknown semantics. Any policy change still needs complete-estimate tests,
including internal aliases, mixed keys and custom functions returning true or false.
[Current SQL](#the-current-recommended-statement),
[nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183),
[fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240).

Plan review, finding 7: all 29 built-in B-tree support-function-4 records use
the two recognized functions, pattern operator classes reject nondeterministic
collations at index creation, and the metapage flag read by `pageinspect` is the
harness oracle. The unknown state applies only to functions created outside the
catalog.
[Plan review](#plan-review),
[index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849),
[btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921).

### Signed-byte output

The current signed byte result is formatted text. A proposed additive numeric
`wasted_space_bytes` field needs overflow, NULL, negative-value and consumer tests.
It must not silently replace the existing ordering expression, which uses the
floor model rather than the point estimate. No such field is implemented here.
[Current SQL](#the-current-recommended-statement),
[pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507).

Plan review, finding 8: `pg_size_pretty(numeric)` removes the need for a
`bigint` cast, and the statement's existing casts can raise `bigint out of range`.
[Plan review](#plan-review),
[pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507),
[numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549).

### Reproducible accuracy and cost coverage

The current statement needs a durable, consolidated fixture suite, with the exact
statement extracted from this page. Assert each fixture's row counts, predicate
membership, deletion fraction, widths, NULL pattern and duplicate groups before
scoring it. Preserve results under `.wiki-runtime/` and generators under `scripts/`
or `tests/`. Recheck partial/expression/INCLUDE exclusions, true savings hidden by
them, zero counts, custom opclasses, geometry boundaries and concurrent changes.
Cross-version claims require independent evidence and runs for each claimed
version. The historical measurements and performance comparisons are not a new
verification of the current text on other majors or platforms.

Plan review, finding 9: the reference server must be built out of tree, the
fenced-block hash baseline is now
`bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357`, and contrib
may be installed in disposable clusters as an oracle.
[Plan review](#plan-review),
[installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432),
[installation.sgml#meson-setup](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L2012-L2025).

### Alert thresholds and rebuild savings

No universal alert threshold has been established. A sorted build and an index
grown through insertion have different page-filling paths, so a difference from
the model need not identify removable churn. Ordinary leaf splits can use 50:50
packing, while rightmost splits use fillfactor. Proposed calibration must score
true savings, false positives, missed savings and withheld rows separately, using
both percentage and absolute bytes, then test whether savings persist under the
target workload. The column named `floor` needs its own calibration.
[nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671),
[nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335),
[Current SQL](#the-current-recommended-statement).

Plan review, finding 10: calibrate per insertion pattern, since rightmost,
50:50 and single-value splits settle at different densities.
[Plan review](#plan-review),
[nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033),
[nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L199-L202).

## Source References

- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671)
- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L784-L855)
- [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1349)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371)
- [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)
- [indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L142-L163)
- [guc_tables.c#statement_timeout-and-lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631)
- [pg_proc.dat#size-functions](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7507)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371)
- [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975)
- [system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275)
- [indexfsm.c#index-FSM](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L65)
- [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L26-L62)
- [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66)
- [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)
- [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309)
- [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385)
- [analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478)
- [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211)
- [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262)
- [nbtsort.c#page-initialization](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L671)
- [nbtsort.c#posting-size-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1308)
- [nbtsort.c#group-boundaries](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1310-L1349)
- [nbtdedup.c#posting-size-check](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L500-L517)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571)
- [nbtsort.c#BTPageState](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L229-L252)
- [index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3048-L3053)
- [nbtree.c#ambuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L128-L129)
- [nbtsort.c#btbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L290-L328)
- [nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1026-L1050)
- [itup.h#IndexTupleData](../../../../raw/postgres-17/src/include/access/itup.h#L35-L60)
- [itemptr.h#ItemPointerData](../../../../raw/postgres-17/src/include/storage/itemptr.h#L25-L47)
- [nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L864-L910)
- [bufpage.h#PageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L155-L171)
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71)
- [indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L94-L163)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L557-L570)
- [nbtsort.c#deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1135-L1152)
- [datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L424-L438)
- [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615)
- [fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703)
- [analyze.c#index-statistics-write](../../../../raw/postgres-17/src/backend/commands/analyze.c#L588-L602)
- [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L845-L884)
- [execnodes.h#IndexInfo](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L183-L193)
- [autovacuum.c#analyze-reloptions](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017)
- [autovacuum.c#analyze-threshold](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3095)
- [pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-17/src/include/pgstat.h#L399-L429)
- [pgstat_relation.c#analyze-counter-reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L337)
- [pgstat.c#publication-timing](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665)
- [pg_class.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L22)
- [pg_index.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L21-L22)
- [catalog/Makefile#genbki](../../../../raw/postgres-17/src/include/catalog/Makefile#L132-L143)
- [utils/Makefile#Gen_fmgrtab](../../../../raw/postgres-17/src/backend/utils/Makefile#L47-L53)
- [btree_index.sql#deduplication-tests](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L186-L213)
- [index_including.sql#statistics-tests](../../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L150-L158)
- [system_views.sql#statistics-views](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L309)
- [analyze.c#examine_attribute](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1019-L1030)
- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L809-L855)
- [pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507)
- [nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335)
- [acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569)
- [aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412)
- [nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1347-L1362)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L870-L920)
- [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3099)
- [acl.c#acldefault](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L813-L821)
- [acl.c#aclmask](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L1388-L1445)
- [aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863)
- [pg_attribute.h#attstattarget](../../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L168-L176)
- [pg_proc.dat#pg_get_expr](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8219-L8220)
- [ruleutils.c#pg_get_expr_ext](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L2648-L2662)
- [nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296)
- [indxpath.c#check_index_predicates](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3244-L3350)
- [nbtsort.c#_bt_load-grouping](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1314-L1316)
- [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905)
- [datum.c:271](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L271)
- [gram.y#DoStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L9037)
- [initdb.c#load_plpgsql](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1974-L1977)
- [pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708)
- [postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705)
- [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600)
- [pgstat.c#flush-intervals](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122)
- [pgstat_relation.c#AtEOXact_PgStat_Relations](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L560-L574)
- [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337)
- [pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860)
- [pgstat.c#pgstat_clear_snapshot](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L786-L800)
- [pgstatfuncs.c#snapshot-and-flush-functions](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c#L1680-L1695)
- [guc_tables.c#stats_fetch_consistency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974)
- [stats.sql#forced-flush](../../../../raw/postgres-17/src/test/regress/sql/stats.sql#L101-L102)
- [nbtsort.c#maxpostingsize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1304-L1305)
- [nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513)
- [nbtdedup.c#_bt_form_posting-size](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L879-L884)
- [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854)
- [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629)
- [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L907-L923)
- [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L214)
- [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1147)
- [nbtsort.c#page-boundary](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L874-L935)
- [nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735)
- [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259)
- [analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647)
- [pg_amproc.dat#bpchar_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L31-L33)
- [pg_amproc.dat#text_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L205-L212)
- [index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849)
- [pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575)
- [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84)
- [nbtsort.c:1126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1126)
- [nbtsort.c#deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)
- [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921)
- [dbsize.c#pg_size_pretty-sign](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L569-L600)
- [numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549)
- [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432)
- [installation.sgml#meson-setup](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L2012-L2025)
- [nbtsplitloc.c#single-value-strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416)
- [nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033)
- [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L199-L202)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [versions](../../../versions.md)
