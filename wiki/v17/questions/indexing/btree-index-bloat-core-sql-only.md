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
  - [Widths, expression statistics and compression](#widths-expression-statistics-and-compression)
  - [Page and posting geometry](#page-and-posting-geometry)
  - [Row counts and exclusions](#row-counts-and-exclusions)
  - [Validation probes](#validation-probes)
  - [Operational and verification limits](#operational-and-verification-limits)
  - [Plan review](#plan-review)
  - [What the ten plans changed](#what-the-ten-plans-changed)
  - [Measured acceptance results](#measured-acceptance-results)
  - [Calibration by insertion pattern](#calibration-by-insertion-pattern)
  - [Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server)
  - [The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement)
  - [The collation branch, measured with ICU](#the-collation-branch-measured-with-icu)
  - [Reproducing the measurements](#reproducing-the-measurements)
  - [What remains unimplemented](#what-remains-unimplemented)
  - [Mandatory test review](#mandatory-test-review)
  - [Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement)
  - [What still needs to be tested](#what-still-needs-to-be-tested)
  - [How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
  - [Mixed key widths in one index](#mixed-key-widths-in-one-index)
  - [Alternating posting and singleton tuples](#alternating-posting-and-singleton-tuples)
  - [In-index compression of wide keys](#in-index-compression-of-wide-keys)
  - [Most-common-value class frequencies](#most-common-value-class-frequencies)
  - [Custom operator classes](#custom-operator-classes)
  - [Expression width heuristic](#expression-width-heuristic)
  - [Untested configurations](#untested-configurations)
  - [Multicolumn key groups without extended statistics](#multicolumn-key-groups-without-extended-statistics)
  - [Partial-index populations and zero counts](#partial-index-populations-and-zero-counts)
  - [Statistics publication and test ordering](#statistics-publication-and-test-ordering)
  - [Alert thresholds and rebuild savings](#alert-thresholds-and-rebuild-savings)
  - [The mandatory suite and the current statement](#the-mandatory-suite-and-the-current-statement)
  - [Scoring column for the partial-index contract](#scoring-column-for-the-partial-index-contract)
  - [Cross-version execution of the revised statement](#cross-version-execution-of-the-revised-statement)
  - [Integer-truncated widths across an alignment boundary](#integer-truncated-widths-across-an-alignment-boundary)
  - [Fixture recipes that do not reproduce](#fixture-recipes-that-do-not-reproduce)
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
under [Plan review](#plan-review).

Third prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, for the question "Testing the PostgreSQL 12
> Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)", implement the
> proposed fixes for the open questions and run all tests.

The original read `follow agents.md, in postgresql 17, on question:  Testing the
PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified) ,
implement the proposal fixes for openquestions , run all tests`: `agents.md` for
AGENTS.md, lowercase `postgresql`, `on question:` with a double space, a space
before a comma, `openquestions` for "open questions", `the proposal fixes` for
"the proposed fixes", and no sentence capitalisation or terminal period. The
asker chose to implement all ten plans, to build a server and run the core
regression suite, and to delete the sandbox after filing. The work is filed
under [What the ten plans changed](#what-the-ten-plans-changed),
[Measured acceptance results](#measured-acceptance-results) and
[Calibration by insertion pattern](#calibration-by-insertion-pattern).

Fourth prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, for the question "Testing the PostgreSQL 12
> Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)", review all the
> mandatory tests, and review what still needs to be tested and how.

The original read `follow agents.md, in postgresql 17, for question: # Testing
the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified) ,
review all mandatory tests and review what needs and how to be tested.`:
`agents.md` for AGENTS.md, lowercase `postgresql`, a stray `#` before the title,
a space before a comma, and "what needs and how to be tested" for "what still
needs to be tested and how". The asker chose to read "the mandatory tests" as
the whole numbered suite this page filed and later removed — tests 1-17, tests
18-91, fixtures 92-112, test 113 and fixtures 114-121 — together with the
2026-09-08 acceptance fixtures and the engine regression runs, and chose a
review with a runnable protocol rather than a server run. The review is filed
under [Mandatory test review](#mandatory-test-review),
[Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement),
[What still needs to be tested](#what-still-needs-to-be-tested) and
[How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement).

Fifth prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, review the question "Testing the
> PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17".

The original read `follow agents.md, in postgresql 17, review question: Testing
the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)`:
`agents.md` for AGENTS.md, lowercase `postgresql`, `review question:` without an
article, and no sentence capitalisation or terminal period. The `(unverified)`
suffix is this repository's title hint for an unverified page, not part of the
page title, so the restated prompt drops it. The asker chose a full claim-level
re-verification against the pin, a server build with measurement, and repair in
place. The results are filed under
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server),
[The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement)
and [The collation branch, measured with ICU](#the-collation-branch-measured-with-icu).

## Answer

**All ten repair plans are now implemented, and the revised statement is exact
on every fresh sorted build measured.** On an isolated 17.11 server built from
the pin, the revised statement reports `0 bytes` wasted on 10 of 10 freshly
built indexes spanning key widths from 8 to 2000 bytes, where the published
statement was wrong on 4 of those 10 (worst: `-33.9 %`); its page-geometry
closed forms reproduce the sorted build's block count exactly in 78 of 78
(key width, fillfactor) cells; and its point estimate equals the reduction that
`REINDEX INDEX` actually produced in 6 of 7 insertion-pattern fixtures.
[The current recommended statement](#the-current-recommended-statement),
[Measured acceptance results](#measured-acceptance-results),
[nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L784-L855),
[nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1349),
[dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371).

The three defects the [Plan review](#plan-review) called deterministic all
reproduced, and all three are fixed:

| Defect | Published statement | Revised statement |
|---|---|---|
| Inheritance parent, two `pg_stats` rows per attribute | `-550.8 %` on a freshly built index | `0.0 %` |
| Expression index read by a non-owner | the row disappears with no explanation | the row appears with `statistics not visible to this role` |
| An index `reltuples` past the `bigint` range | `ERROR: bigint out of range` aborts the whole report | every row prints, the value as `numeric` |

Three further defects surfaced only under measurement, and are also fixed: the
leaf capacity ignored the build's hard-fit reservation (wrong in 30 of 78 cells,
always one item too high), the internal-level fanout ignored the eight-byte
minus-infinity first item and the never-closed rightmost page of each level
(`-33.9 %` on a 2000-byte key), and the width of an index expression was taken
three bytes too wide (`-22.2 %` on a freshly built expression index).
[Measured acceptance results](#measured-acceptance-results),
[nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854),
[nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735),
[heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262).

Treat the output as candidate information all the same. The model still reads
estimated row counts, widths and NULL fractions, and four measured failure modes
remain: mixed key widths in one index (`-60.1 %`), a duplicate class that
alternates a full posting tuple with a single-TID tuple (`-10.2 %`), a wide
compressible key that the index stores compressed (`-2625.6 %`), and
most-common-value frequencies that misstate group sizes. Only the third of those
raises a caveat; the other three are silent, and are the reason this page is
still `(unverified)`. See [Open Questions](#open-questions).

**Every headline number above was re-measured on 2026-09-09 on a second,
independently built server, and every one reproduced.** That pass also scored
the deduplication-gate group against the current statement for the first time —
27 fixtures, no over-credit, `equalimage` agreeing with the metapage on all 20
`recognized` and `ineligible` rows — and measured the nondeterministic-collation
branch that no previous run could reach. It found six reporting defects, all
corrected here; the largest is that a duplicate-heavy multicolumn index with no
`ndistinct` statistics object reads `-320.0 %` to `-562.1 %`, which this page had
never stated.
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server),
[The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement),
[The collation branch, measured with ICU](#the-collation-branch-measured-with-icu).

**The partial-index half of the mandatory suite is still unscored against this
statement.** Tests 18-91 and fixtures 92-121 were last run on 2026-08-24 against
the superseded six-change text, and the 2026-09-08 rewrite was measured on new
fixtures only, so the partial-index contract this page carries remains an
expectation derived from the SQL rather than a result. The review, the expected
verdicts and a runnable protocol are under
[Mandatory test review](#mandatory-test-review).

### The current recommended statement

This is the single operational estimator on this page. It replaces the statement
filed before 2026-09-08; that text is superseded, and its measured behaviour is
kept only as the `old` column of the acceptance tables below. The tag moves from
`wiki_btree_wasted_space_sweep_12_17` to `wiki_btree_wasted_space_sweep_r2` so a
log or `pg_stat_statements` row identifies which model produced a reading.

Run the two timeout settings with the query in a dedicated session. They set
`statement_timeout` to 30 seconds and `lock_timeout` to 2 seconds. Both are
`PGC_USERSET`: they apply at session/transaction scope without a reload or restart.
The supplied `SET` commands affect the session.
[guc_tables.c#statement_timeout-and-lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631).

The catalog and function definitions used by the block are mapped in
[Evidence Map](#evidence-map). The sections after it explain the model and its
limits. Every claim about the statement's behaviour below was measured on an
isolated 17.11 server built from the pin; see
[Reproducing the measurements](#reproducing-the-measurements).

```sql
SET /* wiki_btree_wasted_space_statement_timeout */ statement_timeout = '30s';
SET /* wiki_btree_wasted_space_lock_timeout */ lock_timeout = '2s';


WITH RECURSIVE
env AS (
    -- Server constants and table-level auto-analyze defaults.
    SELECT /* wiki_btree_wasted_space_sweep_r2 */
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
           greatest(s.last_vacuum, s.last_autovacuum)   AS last_vacuum,
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
opc AS (
    -- One row per key attribute: its equal-image support function and collation.
    SELECT i.idxoid, k AS keyno,
           ap.amproc IS NOT NULL                          AS has_proc,
           (pl.lanname = 'internal'
            AND pr.prosrc = 'btequalimage')               AS proc_always,
           (pl.lanname = 'internal'
            AND pr.prosrc = 'btvarstrequalimage')         AS proc_varstr,
           coalesce(cl.collisdeterministic, true)         AS collation_ok
      FROM idx i
      CROSS JOIN LATERAL generate_subscripts(i.indclass, 1) k
      JOIN pg_opclass op ON op.oid = i.indclass[k]
      LEFT JOIN pg_amproc ap ON ap.amprocfamily = op.opcfamily
                            AND ap.amproclefttype = op.opcintype
                            AND ap.amprocrighttype = op.opcintype
                            AND ap.amprocnum = 4
      LEFT JOIN pg_proc pr     ON pr.oid = ap.amproc
      LEFT JOIN pg_language pl ON pl.oid = pr.prolang
      LEFT JOIN pg_collation cl ON cl.oid = i.indcollation[k]
     WHERE k < i.indnkeyatts
),
gate AS (
    -- Three-state equal-image verdict: recognized, ineligible, or unknown.
    SELECT i.idxoid,
           CASE
             WHEN NOT i.keys_only                       THEN 'ineligible'
             WHEN bool_or(NOT o.has_proc)               THEN 'ineligible'
             WHEN bool_or(o.proc_varstr
                          AND NOT o.collation_ok)       THEN 'ineligible'
             WHEN bool_and(o.proc_always OR o.proc_varstr) THEN 'recognized'
             ELSE 'unknown'
           END AS equalimage_state
      FROM idx i
      JOIN opc o ON o.idxoid = i.idxoid
     GROUP BY i.idxoid, i.keys_only
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
    -- Use the maximum visible non-inherited whole-key distinct estimate.
    SELECT k.idxoid, max(e.nd) AS ext_ndistinct
      FROM keyatts k
      JOIN idx i           ON i.idxoid = k.idxoid
      JOIN pg_stats_ext se ON se.schemaname = i.schemaname
                          AND se.tablename = i.tablename
                          AND se.inherited = false
      CROSS JOIN LATERAL (
            SELECT ((se.n_distinct::text)::json ->> k.ext_key)::numeric AS nd) e
     WHERE k.nkeys > 1 AND k.min_attnum > 0 AND e.nd > 0
     GROUP BY k.idxoid
),
cols AS (
    -- Index-expression statistics, table-column statistics, then defaults.
    -- Only the non-inherited ANALYZE pass is read, and an expression
    -- attribute's visibility is tested on the index, which owns its row.
    SELECT i.idxoid, a.attnum, a.attlen, a.attalign, a.attstorage,
           (i.indkey[a.attnum - 1] = 0)                          AS is_expression,
           -- index_form_tuple() compresses a varlena key wider than
           -- TOAST_INDEX_TARGET = MaxHeapTupleSize / 16 when its storage is
           -- extended or main; ANALYZE records the uncompressed width.
           (a.attlen < 0 AND a.attstorage IN ('x', 'm')
            AND w.raw_width > (i.bs - 32) / 16)                  AS compressible,
           -- ANALYZE records VARSIZE_ANY of the sampled datum. A stored column
           -- value is already short-headed, but an index expression is computed
           -- fresh with a four-byte header, and heap_compute_data_size() stores
           -- the converted short size, three bytes less, when it fits in 127.
           CASE WHEN a.attlen > 0 THEN a.attlen::numeric
                WHEN w.from_index AND w.raw_width BETWEEN 5 AND 130
                     THEN (w.raw_width - 3)::numeric
                ELSE w.raw_width::numeric END                    AS width,
           CASE WHEN a.attlen > 0    THEN false
                WHEN w.from_index    THEN w.raw_width <= 130
                ELSE w.raw_width <= 127 END                      AS short_form,
           coalesce(se.null_frac, st.null_frac, 0)::numeric      AS null_frac,
           coalesce(se.n_distinct, st.n_distinct, 0)::numeric    AS n_distinct,
           coalesce(se.most_common_freqs, st.most_common_freqs)  AS mcf,
           (se.attname IS NULL AND st.attname IS NULL)           AS no_stats_row,
           CASE WHEN i.indkey[a.attnum - 1] = 0
                THEN NOT coalesce(has_column_privilege(i.idxoid, a.attnum, 'SELECT'),
                                  false)
                ELSE (NOT coalesce(has_column_privilege(i.tbloid, ta.attnum, 'SELECT'),
                                   has_table_privilege(i.tbloid, 'SELECT'))
                      OR (i.tbl_rls AND row_security_active(i.tbloid)))
           END                                                   AS stats_hidden,
           coalesce(CASE WHEN i.indkey[a.attnum - 1] = 0 THEN a.attstattarget
                         ELSE ta.attstattarget END, -1) = 0      AS stats_disabled
      FROM idx i
      JOIN pg_attribute a ON a.attrelid = i.idxoid AND a.attnum > 0 AND NOT a.attisdropped
      LEFT JOIN pg_stats se ON se.schemaname = i.schemaname
                           AND se.tablename = i.indexname AND se.attname = a.attname
                           AND se.inherited = false
      LEFT JOIN pg_attribute ta ON ta.attrelid = i.tbloid
                               AND ta.attnum = i.indkey[a.attnum - 1]
      LEFT JOIN pg_stats st ON st.schemaname = i.schemaname
                           AND st.tablename = i.tablename AND st.attname = ta.attname
                           AND st.inherited = false
      CROSS JOIN LATERAL (
            SELECT se.attname IS NOT NULL                AS from_index,
                   coalesce(se.avg_width, st.avg_width, 32) AS raw_width) w
),
statvis AS (
    -- Classify each missing statistics row and flag variable-width INCLUDEs.
    SELECT c.idxoid,
           bool_or(c.no_stats_row AND NOT c.stats_hidden
                   AND NOT c.stats_disabled)          AS any_no_stats,
           bool_or(c.no_stats_row AND c.stats_hidden) AS any_stats_hidden,
           bool_or(c.no_stats_row
                   AND c.stats_disabled)              AS any_stats_disabled,
           bool_or(c.attnum > i.indnkeyatts AND c.attlen < 0)
                                                      AS any_varlena_include,
           bool_or(c.compressible)                    AS any_compressible
      FROM cols c
      JOIN idx i ON i.idxoid = c.idxoid
     GROUP BY c.idxoid
),
tuple AS (
    -- Approximate datum widths, joint NULL probability and distinct key groups.
    SELECT i.*,
           (SELECT sum((1 - c.null_frac) *
                       CASE WHEN c.short_form THEN c.width
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
           v.any_no_stats, v.any_stats_hidden, v.any_stats_disabled,
           v.any_varlena_include, v.any_compressible
      FROM idx i
      LEFT JOIN extstat e ON e.idxoid = i.idxoid
      LEFT JOIN statvis v ON v.idxoid = i.idxoid
),
page AS (
    -- Page geometry: soft fillfactor limit, hard-fit reservation, posting cap.
    SELECT t.*, g.equalimage_state,
           s.itupsz, s.slot, f.leaf_cap, f.int_cap, f.leaf_bytes, f.dedup_applies,
           f.groups_est, f.maxposting, f.hikey_extra, p.nmax
      FROM tuple t
      JOIN gate g ON g.idxoid = t.idxoid
      CROSS JOIN LATERAL (
            SELECT ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8              AS itupsz,
                   ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8 + 4          AS slot) s
      CROSS JOIN LATERAL (
            SELECT greatest(least(
                     floor((t.bs - 48 - floor(t.bs * (100 - t.fillfactor) / 100)) / s.slot),
                     floor((t.bs - 48 - 8 - s.itupsz) / s.slot)), 1)           AS leaf_cap,
                   greatest(least(
                     1 + floor((t.bs - 48 - floor(t.bs * 30 / 100) - 12) / s.slot),
                     1 + floor((t.bs - 48 - 12 - s.itupsz) / s.slot)), 2)      AS int_cap,
                   (t.bs - 48 - floor(t.bs * (100 - t.fillfactor) / 100))      AS leaf_bytes,
                   (t.bs - 48 - 8)                                             AS hikey_extra,
                   (NOT t.indisunique AND t.dedup_on
                        AND g.equalimage_state = 'recognized')                 AS dedup_applies,
                   least(greatest(t.live_rows, 0), greatest(t.key_groups, 1))  AS groups_est,
                   floor(floor(t.bs * 10 / 100) / 8) * 8 - 4                   AS maxposting) f
      CROSS JOIN LATERAL (
            SELECT greatest(floor((floor(f.maxposting / 8) * 8 - s.itupsz) / 6), 1)
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
    -- Split each class into whole posting tuples and one tail tuple per group.
    SELECT g.idxoid, g.class_rows, g.class_groups, p.itupsz, p.slot,
           p.leaf_bytes, p.hikey_extra, p.nmax,
           u.rows_per_group,
           floor(u.rows_per_group / p.nmax)                        AS full_per_group,
           u.rows_per_group - p.nmax * floor(u.rows_per_group / p.nmax)
                                                                   AS tail_rows
      FROM gclass g
      JOIN page p ON p.idxoid = g.idxoid
      CROSS JOIN LATERAL (
            SELECT g.class_rows / greatest(g.class_groups, 1) AS rows_per_group) u
     WHERE g.class_rows > 0
),
classsize AS (
    -- Price a full posting tuple and the tail tuple separately. The build
    -- flushes one posting tuple per capacity and one shorter tail per group.
    SELECT c.*,
           CASE WHEN c.nmax > 1 THEN ceil((c.itupsz + c.nmax * 6) / 8) * 8
                ELSE c.itupsz END                                  AS full_size,
           -- A one-TID group is a plain tuple, a longer one a posting tuple,
           -- so interpolate between the integer TID counts that bracket the
           -- class's average tail rather than rounding it.
           (1 - (c.tail_rows - floor(c.tail_rows)))
             * CASE WHEN floor(c.tail_rows) > 1
                    THEN ceil((c.itupsz + floor(c.tail_rows) * 6) / 8) * 8
                    ELSE c.itupsz END
           + (c.tail_rows - floor(c.tail_rows))
             * CASE WHEN ceil(c.tail_rows) > 1
                    THEN ceil((c.itupsz + ceil(c.tail_rows) * 6) / 8) * 8
                    ELSE c.itupsz END                              AS tail_size,
           c.full_per_group
             + CASE WHEN c.tail_rows > 0 THEN 1 ELSE 0 END         AS tuples_per_group
      FROM classfit c
),
classpages AS (
    -- Convert tuple counts to leaf pages. Pages mix full and tail tuples, so
    -- the capacity uses the class's mean tuple size and mean posting credit,
    -- while the hard-fit reservation uses its largest tuple.
    SELECT c.idxoid,
           sum(c.class_groups * c.tuples_per_group / f.cap)        AS leaf_frac,
           sum(c.class_groups * c.tuples_per_group)                AS tuples_total,
           max(CASE WHEN c.full_per_group > 0 THEN c.nmax
                    ELSE greatest(c.tail_rows, 1) END)             AS max_tids
      FROM classsize c
      CROSS JOIN LATERAL (
            SELECT (c.full_per_group * c.full_size
                    + CASE WHEN c.tail_rows > 0 THEN c.tail_size ELSE 0 END)
                     / c.tuples_per_group                          AS mean_size,
                   (c.full_per_group * (c.full_size - c.itupsz)
                    + CASE WHEN c.tail_rows > 0
                           THEN c.tail_size - c.itupsz ELSE 0 END)
                     / c.tuples_per_group                          AS mean_credit) m
      CROSS JOIN LATERAL (
            SELECT greatest(least(
                     floor((c.leaf_bytes + m.mean_credit) / (m.mean_size + 4)),
                     floor((c.hikey_extra - m.mean_size) / (m.mean_size + 4))), 1)
                                                                   AS cap) f
     GROUP BY c.idxoid
),
leaves AS (
    -- Leaf-page models with and without deduplication credit. Every level
    -- keeps one page that the fillfactor limit never closes, so it holds one
    -- item more than a closed page.
    SELECT p.*, coalesce(cp.max_tids, 1) AS tids,
           CASE WHEN p.dedup_applies AND cp.leaf_frac IS NOT NULL
                THEN greatest(d.dedup_pages, 1)
                ELSE f.floor_pages
           END                                                AS leaf_pages,
           f.floor_pages                                      AS leaf_pages_floor
      FROM page p
      LEFT JOIN classpages cp ON cp.idxoid = p.idxoid
      CROSS JOIN LATERAL (
            SELECT CASE WHEN greatest(p.live_rows, 0) <= 0 THEN 0
                        WHEN greatest(p.live_rows, 0) <= p.leaf_cap + 1 THEN 1
                        ELSE 1 + ceil((greatest(p.live_rows, 0) - p.leaf_cap - 1)
                                      / p.leaf_cap)
                   END AS floor_pages) f
      CROSS JOIN LATERAL (
            SELECT CASE WHEN cp.tuples_total <= e.cap + 1 THEN 1
                        ELSE 1 + ceil((cp.tuples_total - e.cap - 1) / e.cap)
                   END AS dedup_pages
              FROM (SELECT greatest(cp.tuples_total
                                    / greatest(cp.leaf_frac, 1e-9), 1) AS cap) e) d
),
levels AS (
    -- Add internal levels recursively for both models.
    SELECT idxoid, 'dedup'::text AS variant, leaf_pages AS pages, int_cap FROM leaves
    UNION ALL
    SELECT idxoid, 'floor'::text, leaf_pages_floor, int_cap FROM leaves
    UNION ALL
    SELECT l.idxoid, l.variant,
           CASE WHEN l.pages <= l.int_cap + 1 THEN 1
                ELSE 1 + ceil((l.pages - l.int_cap - 1) / l.int_cap)
           END, l.int_cap
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
            SELECT (l.any_no_stats AND l.last_analyze IS NOT NULL)          AS stats_row_missing,
                   (l.dedup_applies AND l.tids > 1)                         AS dedup_credited,
                   (l.tbl_mod_since_analyze > l.tbl_autoanalyze_threshold)  AS stats_stale) w
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
         pg_size_pretty((actual_bytes - expected_blocks * bs)::numeric) END AS wasted_space,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         round(actual_bytes - expected_blocks * bs)     END AS wasted_space_bytes,
       array_to_string(array_remove(ARRAY[
         CASE WHEN last_analyze IS NULL THEN 'never analyzed' END,
         CASE WHEN any_stats_hidden
              THEN 'statistics not visible to this role' END,
         CASE WHEN any_stats_disabled
              THEN 'statistics target zero on an index column' END,
         CASE WHEN stats_row_missing
              THEN 'no statistics row for an index column' END,
         CASE WHEN live_rows = 0
              THEN 'zero modelled rows: validate with a population probe' END,
         CASE WHEN any_compressible
              THEN 'wide compressible key: stored width may be over-stated' END,
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
         CASE WHEN equalimage_state = 'unknown'
              THEN 'unrecognized equal-image support function: no credit' END,
         CASE WHEN ext_used THEN 'key groups from extended statistics' END
       ], NULL), '; ')                                  AS caveats,
       equalimage_state                                 AS equalimage,
       CASE
         WHEN last_vacuum IS NULL AND last_analyze IS NULL THEN 'build'
         WHEN last_analyze IS NULL                         THEN 'vacuum'
         WHEN last_vacuum IS NULL                          THEN 'analyze'
         WHEN last_vacuum > last_analyze                   THEN 'vacuum'
         ELSE 'analyze'
       END                                              AS reltuples_writer,
       round(key_groups)                                AS key_groups,
       round(tids::numeric, 1)                          AS tids_per_tuple,
       round(live_rows)                                 AS modelled_rows,
       round(idx_reltuples)                             AS idx_reltuples,
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
fork is read separately. `wasted_space_bytes` is the same quantity as `numeric`,
so a consumer never has to parse the formatted string, and no projected value is
cast to `bigint`.
[The current recommended statement](#the-current-recommended-statement),
[pg_proc.dat#size-functions](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7507),
[pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507),
[dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371),
[dbsize.c#pg_size_pretty-sign](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L569-L600).

| Output | Meaning in this statement |
|---|---|
| `status` | `unmeasured: reltuples unknown` when the index's `reltuples` is negative; otherwise `ok`. `ok` is not a freshness or accuracy certification. |
| `wasted_space_pct` | Signed percentage difference from `expected_blocks`, the model that can credit deduplication. |
| `wasted_space_pct_floor` | Signed percentage difference from `floor_blocks`, the model without deduplication credit. Row-count and width uncertainty still apply. |
| `wasted_space` | Formatted signed `actual_bytes - expected_blocks * bs`; a negative value means the model predicts a larger rebuild. |
| `wasted_space_bytes` | The same signed difference as `numeric`, unformatted and unrounded to a unit. |
| `caveats` | Visible qualifications assembled by the query. A filtered-out index has no output row or explanation. |
| `equalimage` | `recognized`, `ineligible` or `unknown`; see [Deduplication eligibility](#deduplication-eligibility). Only `recognized` earns deduplication credit. |
| `reltuples_writer` | `analyze`, `vacuum` or `build`, from the later of the table's ANALYZE and VACUUM timestamps. It names the likely last writer of the index's `reltuples`, not a proof. |
| `key_groups`, `tids_per_tuple` | Modelled distinct groups and posting-list occupancy. A TID is a heap tuple identifier. |
| `modelled_rows`, `idx_reltuples` | The row estimate used by the model and the index's catalog estimate, both `numeric`. Read a zero as an input requiring validation. |
| `fsm_written_since_build` | The test `fsm_bytes > 0`; it does not count free pages or establish that any page remains reusable. |
| `server_version_num` | The server's version number, reported without selecting a different formula. |

The caveat strings, and what each one means:

| Caveat | Condition | Consequence |
|---|---|---|
| `never analyzed` | no ANALYZE timestamp on the table | every statistics input is a default |
| `statistics not visible to this role` | at least one attribute's `pg_stats` row is filtered by privilege or row-level security | that attribute falls back to defaults |
| `statistics target zero on an index column` | `attstattarget = 0`, so ANALYZE skips the attribute | the row is not a missing-statistics error |
| `no statistics row for an index column` | statistics are absent, visible and enabled | ANALYZE has not covered the attribute |
| `zero modelled rows: validate with a population probe` | `live_rows = 0` | run the [population probe](#validation-probes) before believing a near-total-waste reading |
| `wide compressible key: stored width may be over-stated` | a varlena key with `extended` or `main` storage whose recorded width exceeds `TOAST_INDEX_TARGET` | the index may store the key compressed; see [Open Questions](#in-index-compression-of-wide-keys) |
| `row-count sources disagree: analyze first` | the table and index row estimates differ by more than 10 % | analyze before reading the estimate |
| `partial: ...` (four strings) | partial-index freshness and duplicate-source qualifications | the row is also suppressed for the first three |
| `deduplication credited` | the model priced posting lists | compare with `wasted_space_pct_floor` |
| `unrecognized equal-image support function: no credit` | `equalimage = unknown` | the engine may deduplicate where the model does not |
| `key groups from extended statistics` | a whole-key `pg_ndistinct` entry supplied the group count | |

For triage, read `wasted_space_pct` with `status`, `equalimage` and `caveats`. Do
not promote a reading with `never analyzed`, `row-count sources disagree: analyze
first`, `statistics not visible to this role`, `zero modelled rows` or `wide
compressible key` into a rebuild decision without the corresponding check. A wide
gap between the two percentages identifies dependence on the duplication estimate.
[The current recommended statement](#the-current-recommended-statement),
[analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975),
[system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275).

The report keeps indexes larger than `1024 * 1024` bytes, excludes `suppress_row`,
orders by the signed floor-model byte difference with NULLs first, and returns at
most 20 rows. An absent index can therefore be below the size cutoff, suppressed,
or outside the top 20. Do not interpret absence as a clean bill of health.
[The current recommended statement](#the-current-recommended-statement).

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
| `env`, `idx` | Read server constants, valid physical B-tree indexes, reloptions, main/FSM sizes, row-count inputs and both maintenance timestamps. | [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L26-L62), [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66), [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371). |
| `opc`, `gate` | Resolve each key opclass's support function 4 and collation, then decide `recognized` / `ineligible` / `unknown`. | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183), [pg_amproc.dat#text_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L205-L212). |
| `keyatts`, `extstat` | Look for a whole-key distinct-count entry from the non-inherited pass; use the maximum matching visible estimate. | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309), [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385). |
| `cols`, `statvis`, `tuple` | Prefer index-expression statistics from the non-inherited pass, otherwise table-column statistics, then defaults; correct an expression's varlena header; classify hidden, disabled and compressible attributes; estimate tuple width and key groups. | [analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478), [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259), [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262). |
| `page` | Approximate tuple slots, leaf and internal capacity under both the fillfactor limit and the hard-fit reservation, and the build's posting-list size cap. | [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854), [nbtsort.c#posting-size-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1308). |
| `kstat`, `gclass`, `classfit`, `classsize`, `classpages` | Split eligible single-key rows into NULL, most-common-value and remaining classes; use one class for multicolumn keys; price whole posting tuples and one tail per group separately; convert to pages with the class's mean size and mean posting credit. | [nbtsort.c#group-boundaries](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1310-L1349), [nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513), [nbtdedup.c#_bt_form_posting-size](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L879-L884). |
| `leaves`, `levels`, `modelled` | Estimate leaf pages, recursively add internal levels with one never-closed page per level, add a metapage, then derive suppression and caveat conditions. | [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1062-L1128), [nbtsort.c#_bt_slideleft](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L682-L700), [nbtsort.c#BTPageState](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L229-L252). |

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

Tuple geometry is still an approximation. `IndexTupleData` contains the TID and
size/flag word; a posting tuple adds an array of `ItemPointerData`. `PageHeaderData`
includes the line-pointer array, and `BTPageOpaqueData` holds B-tree sibling/level
metadata. Actual tuple formation omits NULL values, aligns attributes in order and
can compress variable-width values. The SQL hard-codes eight-byte alignment and uses
average widths, so those inputs do not reconstruct every physical tuple.
[itup.h#IndexTupleData](../../../../raw/postgres-17/src/include/access/itup.h#L35-L60),
[itemptr.h#ItemPointerData](../../../../raw/postgres-17/src/include/storage/itemptr.h#L25-L47),
[nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L864-L910),
[bufpage.h#PageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L155-L171),
[nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71),
[indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L94-L163).

### Deduplication eligibility

Deduplication stores one key image with several heap TIDs. For a fresh build,
`_bt_leafbuild` recomputes equal-image eligibility. `_bt_allequalimage` refuses
INCLUDE indexes outright, then retrieves each key's support function 4 and calls
it; a missing function or a false return makes the index ineligible. The sorted
build also requires a non-unique index and `deduplicate_items` enabled.
[nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L557-L570),
[nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183),
[nbtsort.c#deduplicate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1135-L1152).

The statement reports one of three states instead of a boolean:

| `equalimage` | Condition in `gate` | Model behaviour |
|---|---|---|
| `ineligible` | the index has INCLUDE columns, or some key opclass has no support function 4, or a recognized `btvarstrequalimage` key has a nondeterministic collation | no deduplication credit; the engine agrees |
| `recognized` | every key opclass has support function 4, and each is the internal `btequalimage` or `btvarstrequalimage` | deduplication credited when the index is also non-unique with `deduplicate_items` on |
| `unknown` | every key has support function 4, but at least one is not one of those two internal functions | no credit, plus the caveat `unrecognized equal-image support function: no credit` |

The determinism test now applies only to `btvarstrequalimage` keys, because
`btequalimage` returns true unconditionally while `btvarstrequalimage` returns
`pg_locale_deterministic`. Scored against the metapage flag that the build itself
writes, the three states were right on 8 of 10 fixtures and conservative on the
two custom-opclass indexes; see
[Measured acceptance results](#measured-acceptance-results).
[datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L424-L438),
[varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615),
[pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575),
[fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240),
[nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84),
[nbtsort.c:1126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1126).

### Widths, expression statistics and compression

ANALYZE records `stawidth` as the average `VARSIZE_ANY` of the sampled datum.
A stored table column is already in short-header form, so its recorded width is
what the index tuple stores. An index expression is computed fresh and carries a
four-byte header, while `heap_compute_data_size` stores the converted short size,
three bytes less, whenever the value fits in 127 bytes. The statement therefore
subtracts three bytes from an index-expression width between 5 and 130 and keeps
the value unaligned; on a freshly built `lower(txt)` index over 23-character
values that correction moved the reading from `-22.2 %` to `0.0 %`.
[analyze.c#compute_scalar_stats-width](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2420-L2426),
[analyze.c#stawidth](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540),
[heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L234-L242),
[varatt.h#VARATT_CAN_MAKE_SHORT](../../../../raw/postgres-17/src/include/varatt.h#L257-L262),
[The current recommended statement](#the-current-recommended-statement).

`index_form_tuple` compresses a varlena key that is wider than
`TOAST_INDEX_TARGET` when the index attribute's storage is `extended` or `main`,
and the index attribute inherits the table column's storage. `TOAST_INDEX_TARGET`
is `MaxHeapTupleSize / 16`, which is 510 bytes at the default block size. No
catalog records the compressed width, so the model cannot see it; the statement
raises `wide compressible key: stored width may be over-stated` instead. Two
60,000-row tables of the same 900-character keys measured 367 blocks with
`extended` storage and 10,003 blocks with `plain`, both recording an
`avg_width` of 904, and the estimator read `-2625.6 %` against an exact `0.0 %`.
[indextuple.c#index_form_tuple-compression](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138),
[heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68),
[htup_details.h#MaxHeapTupleSize](../../../../raw/postgres-17/src/include/access/htup_details.h#L553-L563),
[pg_type.h#TYPSTORAGE_EXTENDED](../../../../raw/postgres-17/src/include/catalog/pg_type.h#L307-L310),
[index.c#ConstructTupleDescriptor-attstorage](../../../../raw/postgres-17/src/backend/catalog/index.c#L353-L360),
[Open Questions](#in-index-compression-of-wide-keys).

Both `pg_stats` joins and the `pg_stats_ext` join select `inherited = false`.
`analyze_rel` runs the non-inherited pass and then, when the table has children,
the inherited pass, and `update_attstats` stores each under its own `stainherit`
value. Without the filter an index on an inheritance parent receives two rows per
attribute and the width, NULL and distinct inputs are counted twice; the measured
cost of that was `-550.8 %` on a freshly built index whose two rows recorded
average widths of 21 and 77 bytes.
[analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259),
[analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647),
[system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211).

An expression attribute's `pg_stats` row belongs to the index, and `pg_stats`
filters rows by `has_column_privilege` on the relation that owns them. An index's
`relacl` is empty because `GRANT` refuses indexes, so the fallback builds the
default table ACL, which grants the owner alone. The statement therefore tests
`has_column_privilege(index, attnum, 'SELECT')` for expression attributes and the
table's privileges for plain keys, and it reads `attstattarget` so that a column
ANALYZE deliberately skips is reported as disabled rather than missing.
[system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275),
[acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569),
[acl.c#acldefault](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L813-L821),
[aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412),
[aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863),
[pg_attribute.h#attstattarget](../../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L168-L176),
[analyze.c#examine_attribute](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1019-L1030).

### Page and posting geometry

A new build page reserves the high-key line pointer, and `PageGetFreeSpace`
subtracts one more line pointer; with the 24-byte page header and the 16-byte
opaque area that is the statement's 48-byte deduction. `_bt_buildadd` then closes
a page under either of two conditions, and the model now applies both:

| Rule in `_bt_buildadd` | Closed-page item count for an aligned tuple size `s` |
|---|---|
| fillfactor limit: `pgspc + last_truncextra < btps_full` | `floor((bs - 48 - btps_full) / (s + 4))` |
| hard fit: `pgspc < itupsz + MAXALIGN(sizeof(ItemPointerData))` | `floor((bs - 48 - 8 - s) / (s + 4))` |

`leaf_cap` is the smaller of the two. The published statement used the fillfactor
term alone, which is one item too high in 30 of the 78 measured cells — every
`fillfactor = 100` case and four wide-key cases at `fillfactor = 90`.
[nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629),
[bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L907-L923),
[bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L214),
[nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1147),
[nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854),
[nbtsort.c#page-boundary](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L874-L935).

Internal levels differ in two ways the published model ignored. `_bt_sortaddtup`
truncates the first data item of every internal page to a bare `IndexTupleData`
of eight bytes, so a page carries that 12-byte slot plus its pivots; and the
rightmost page of each level is never closed by the fillfactor limit, because
`_bt_uppershutdown` finishes it and `_bt_slideleft` removes its unused high-key
line pointer. A page that is never closed keeps one item more than a closed one,
which makes the level recursion

```text
pages(M, cap) = 1                                     when M <= cap + 1
pages(M, cap) = 1 + ceil((M - cap - 1) / cap)         otherwise
```

Applied at the leaf level and at every internal level, that recursion reproduced
the exact `relpages` of all 78 measured builds; the previous form was exact in 53
and off by as much as 5 blocks.
[nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735),
[nbtsort.c#_bt_slideleft](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L682-L700),
[nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1062-L1128),
[nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L199-L202).

For duplicate groups the build caps a posting tuple at
`MAXALIGN_DOWN(bs * 10 / 100) - sizeof(ItemIdData)`, which is 812 bytes at the
default block size, and refuses another TID when
`MAXALIGN(basetupsize + (nhtids + 1) * sizeof(ItemPointerData))` would exceed it.
Because `basetupsize` is already aligned, the capacity is
`floor((MAXALIGN_DOWN(812) - itupsz) / 6)`, which is 132 TIDs for a four-byte
integer key. `bt_page_items` measured exactly 132 TIDs at an item length of 808
bytes on every such fixture. Each group therefore contributes
`floor(u / m)` full posting tuples and one shorter tail, and the statement prices
those two sizes separately instead of charging the tail as a full tuple.
[nbtsort.c#maxpostingsize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1304-L1305),
[nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513),
[nbtdedup.c#_bt_dedup_start_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L432-L474),
[nbtdedup.c#_bt_form_posting-size](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L879-L884),
[nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1026-L1050).

Two details of that pricing matter. A one-TID group is a plain tuple, not a
posting tuple, so the tail size interpolates between the integer TID counts that
bracket the class's average tail rather than rounding it up; without that, an
index with 380 accidental duplicate pairs among 300,000 rows read `-40.0 %`
instead of `0.0 %`. And a page mixes full and tail tuples, so the page capacity
uses the class's mean tuple size and mean posting credit; a per-size capacity
that packs tails on tail-only pages under-counted a 500-row group's index by 5 %.

### Row counts and exclusions

The SQL treats negative index `reltuples` as unknown and takes zero at face value.
For a partial index, whose predicate selects a subset of table rows, it uses the
index's `reltuples`. For a non-partial index it takes the lesser of that estimate
and the table's nonzero `n_live_tup`, falling back to `reltuples` when the table
counter is zero. Neither input is an exact current count, and a zero now raises a
caveat pointing at the [population probe](#validation-probes).
[The current recommended statement](#the-current-recommended-statement),
[pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66),
[analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663),
[system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703).

ANALYZE is not the only writer of an index's `reltuples`. A VACUUM that removed
tuples counts the live heap TIDs on each leaf page, clamps the total to the heap
count when that count is exact, and writes it through the same relation-statistics
path. A cleanup-only scan counts index tuples instead, marks the result as an
estimate, and the write is skipped. The statement now reports which maintenance
command ran last in `reltuples_writer`, derived from the later of the table's
VACUUM and ANALYZE timestamps; it is an attribution, not a proof, because a
cleanup-only VACUUM leaves a timestamp without writing the value.
[nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1347-L1362),
[nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L870-L920),
[vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3099),
[system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703).

`ANALYZE` gives index attributes their own column statistics only for expressions.
`IndexInfo` identifies expression attributes and the predicate;
`compute_index_stats` evaluates that predicate over its sample. Plain keys and
INCLUDE columns therefore fall back to table statistics in this query, even when
the index covers only a subset.
[analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478),
[analyze.c#index-statistics-write](../../../../raw/postgres-17/src/backend/commands/analyze.c#L588-L602),
[analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L845-L884),
[execnodes.h#IndexInfo](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L183-L193).

Five conditions set `suppress_row`. The missing-statistics condition is now
per attribute and excludes attributes whose statistics are hidden by privilege or
disabled by `attstattarget = 0`, so those cases are reported with a caveat instead
of vanishing.
[The current recommended statement](#the-current-recommended-statement).

| Index scope | Suppression condition |
|---|---|
| Partial | At least one attribute has no returned statistics row that is neither hidden nor disabled, and a table ANALYZE timestamp exists. |
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
[pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-17/src/include/pgstat.h#L399-L429).

### Validation probes

Two readings cannot be validated from catalogs alone: a zero row count on a
partial index, and the number of duplicate groups the build will actually form.
The generator below reads the catalogs and emits one probe statement per index
that needs one. It executes nothing itself, so the emitted text can be reviewed
before it runs; it is a companion to the estimator, not part of it.

A `population` probe is emitted for a partial index whose `reltuples` is zero or
negative. It is an `EXISTS` subquery over the predicate text from `pg_get_expr`,
so the executor stops at the first matching row, and the planner may satisfy it
from the partial index itself: on the measured empty-subset fixture it chose
`Index Only Scan using empty_open` with no filter, and on the populated fixture
it chose that index once a sequential scan was disabled.
[pg_proc.dat#pg_get_expr](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8219-L8220),
[ruleutils.c#pg_get_expr_ext](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L2648-L2662),
[nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296),
[indxpath.c#check_index_predicates](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3244-L3350).

A `groups` probe is emitted only for an index that would pass the deduplication
gate with a recognized support function. The build groups adjacent sorted tuples
by `_bt_keep_natts_fast`, which compares binary images and treats two NULLs as
equal, and `GROUP BY` over the key expressions reproduces that grouping for
exactly those opclasses. For an unrecognized or ineligible opclass the generator
emits no probe, because SQL equality and binary-image equality can disagree.
[nbtsort.c#_bt_load-grouping](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1314-L1316),
[nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905),
[datum.c:271](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L271).

```sql
-- Finding 2: generate validation probes for the rows the estimator cannot trust.
-- The generator is read-only; it emits statements for review before they run.
SET /* wiki_btree_probe_statement_timeout */ statement_timeout = '30s';
SET /* wiki_btree_probe_lock_timeout */ lock_timeout = '2s';

WITH cand AS (
    SELECT /* wiki_btree_probe_generator */
           c.oid AS idxoid, n.nspname, c.relname AS indexname,
           x.indrelid, x.indnkeyatts, c.reltuples,
           pg_get_expr(x.indpred, x.indrelid) AS pred,
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
              FROM generate_subscripts(x.indclass, 1) k
              JOIN pg_opclass op ON op.oid = x.indclass[k]
             WHERE k < x.indnkeyatts)                     AS recognized,
           NOT x.indisunique AND x.indnatts = x.indnkeyatts
             AND coalesce((SELECT option_value::bool FROM pg_options_to_table(c.reloptions)
                            WHERE option_name = 'deduplicate_items'), true) AS gateable
      FROM pg_class c
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_am am       ON am.oid = c.relam
     WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
       AND pg_relation_size(c.oid) > 1024 * 1024
)
SELECT indexname,
       CASE WHEN pred IS NOT NULL AND reltuples <= 0 THEN 'population'
            WHEN coalesce(recognized, false) AND gateable THEN 'groups'
            ELSE 'none' END AS probe_kind,
       CASE
         WHEN pred IS NOT NULL AND reltuples <= 0 THEN
           format('SELECT %L::text AS indexname, ''population''::text AS probe,'
                  ' (EXISTS (SELECT 1 FROM ONLY %s WHERE %s))::text AS result;',
                  indexname, indrelid::regclass::text, pred)
         WHEN coalesce(recognized, false) AND gateable THEN
           format('SELECT %L::text AS indexname, ''groups''::text AS probe,'
                  ' count(*)::text AS result FROM (SELECT 1 FROM ONLY %s %s GROUP BY %s) s;',
                  indexname, indrelid::regclass::text,
                  CASE WHEN pred IS NULL THEN '' ELSE 'WHERE ' || pred END,
                  (SELECT string_agg(pg_get_indexdef(idxoid, k, true), ', ' ORDER BY k)
                     FROM generate_series(1, indnkeyatts) k))
         ELSE NULL
       END AS probe_sql
  FROM cand
 ORDER BY probe_kind, indexname;
```

On the measured fixtures the generator emitted a population probe for both
partial indexes with a zero row count, a group probe for the recognized
duplicate-heavy index, and nothing for a `numeric` index. The probes then
separated a true reading from a false one: for an index whose subset really was
empty the probe returned `false`, confirming the estimator's `99.9 %`, and for an
index whose `reltuples` had been forged to zero while 300,000 rows still matched
the predicate the probe returned `true`, catching the same `99.9 %` as a lie. The
group probe counted 5,000 groups against the model's estimate of 4,997.

### Operational and verification limits

`pg_relation_size` takes and releases `AccessShareLock` on each relation. It returns
NULL if a relation has disappeared, and the file-size loop can raise a file-access
error. These calls inspect file lengths; they do not provide a single physical
snapshot of all indexes and statistics in the report.
[dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371).

No contrib extension is needed by the recommended statement or by the probe
generator: their inputs are core catalogs, views and functions. The measurement
harness is a different matter — it uses `pageinspect` and `pgstattuple` as
oracles, in a disposable cluster only. The build-dependent constants still matter:
catalog headers include generated `_d.h` files, `genbki.pl` builds catalog outputs
from the source definitions, and `Gen_fmgrtab.pl` uses `pg_proc.dat` to generate
function lookup support. Every measurement below was taken at the default
`BLCKSZ` of 8192 and `MAXIMUM_ALIGNOF` of 8; the SQL's literal layout assumptions
need separate validation on a different build configuration.
[pg_class.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L22),
[pg_index.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L21-L22),
[catalog/Makefile#genbki](../../../../raw/postgres-17/src/include/catalog/Makefile#L132-L143),
[utils/Makefile#Gen_fmgrtab](../../../../raw/postgres-17/src/backend/utils/Makefile#L47-L53),
[btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L108-L194),
[pageinspect--1.8--1.9.sql#bt_page_stats](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L87-L124),
[pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31).

The pinned regression files provide adjacent engine coverage: deduplication in
`btree_index.sql`, and statistics-setting restrictions for plain, expression and
INCLUDE attributes in `index_including.sql`. They do not constitute an accuracy
test of this wiki's estimator; the fixtures below do that.
[btree_index.sql#deduplication-tests](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L186-L213),
[index_including.sql#statistics-tests](../../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L150-L158).

### Plan review

This section is the source-only review filed on 2026-09-07, before any of the
plans were implemented. Its verdicts are unchanged; what each one became in the
statement is in [What the ten plans changed](#what-the-ten-plans-changed).

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

The review's supporting evidence, in the same order: statistics visibility and the
owner-only default ACL
([acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569),
[acl.c#acldefault](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L813-L821),
[acl.c#aclmask](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L1388-L1445),
[aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863));
`DO` and PL/pgSQL availability for a generated probe
([gram.y#DoStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L9037),
[initdb.c#load_plpgsql](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L1974-L1977));
the forced-flush path
([pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708),
[postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705),
[pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600),
[pgstat.c#flush-intervals](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122),
[pgstat_relation.c#AtEOXact_PgStat_Relations](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L560-L574),
[pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337),
[pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860),
[pgstat.c#pgstat_clear_snapshot](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L786-L800),
[pgstatfuncs.c#snapshot-and-flush-functions](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c#L1680-L1695),
[guc_tables.c#stats_fetch_consistency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974),
[stats.sql#forced-flush](../../../../raw/postgres-17/src/test/regress/sql/stats.sql#L101-L102));
the built-in equal-image inventory and the pattern-opclass collation check
([pg_amproc.dat#bpchar_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L31-L33),
[pg_amproc.dat#text_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L205-L212),
[index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849),
[btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921));
the numeric range error
([numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549));
the out-of-tree build requirement
([installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432),
[installation.sgml#meson-setup](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L2012-L2025));
and the split strategies
([nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335),
[nbtsplitloc.c#single-value-strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416),
[nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033)).

### What the ten plans changed

| Plan | Change in the statement or harness | Status |
|---|---|---|
| 1 | `has_column_privilege` on the index for expression attributes; `attstattarget = 0` reported as disabled; per-attribute missing/hidden/disabled classification; four new caveat strings | implemented |
| 2 | `EXISTS` population probe and `GROUP BY` group probe generated from the catalogs; `reltuples_writer` column; `zero modelled rows` caveat | implemented |
| 3 | `pg_stat_force_next_flush()` barrier before and after every fixture ANALYZE | implemented in the harness |
| 4 | Whole posting tuples and one tail per group priced separately, with fractional-TID interpolation and a class-mean page capacity | implemented |
| 5 | Hard-fit reservation in `leaf_cap`; minus-infinity first item in `int_cap`; the never-closed rightmost page in the level recursion | implemented |
| 6 | `inherited = false` on both `pg_stats` joins and on `pg_stats_ext` | implemented |
| 7 | Three-state `equalimage` column; determinism tested only for `btvarstrequalimage`; scored against `bt_metap` | implemented |
| 8 | `pg_size_pretty(numeric)`, `numeric` projections, new `wasted_space_bytes`; no `bigint` cast anywhere | implemented |
| 9 | Out-of-tree VPATH build; block hashes re-baselined; core regression suite run | implemented |
| 10 | Seven insertion-pattern fixtures scored against `REINDEX INDEX` | implemented |

Two corrections the plans did not anticipate came out of the runs: the
three-byte varlena-header difference in an index expression's recorded width, and
in-index compression of wide keys. Both are covered under
[Widths, expression statistics and compression](#widths-expression-statistics-and-compression).

### Measured acceptance results

Everything in this section was measured on 2026-09-08 on an isolated 17.11 server
built out of tree from `raw/postgres-17/` at pin
`786db8dcf168bd9df8f55047337525ac19118b1c`, with `autovacuum = off`,
`fsync = off`, `shared_buffers = 256MB`, `maintenance_work_mem = 64MB`,
`--locale=C` and the default `BLCKSZ` of 8192. The checkout was never written to.
`make check` passed **225 of 225** tests, and `make -C contrib/pageinspect check`
and `make -C contrib/pgstattuple check` both passed. The published statement
extracted from the previous revision of this page hashed to
`bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357`, matching the
baseline finding 9 recorded, so the `old` columns below are that exact text. The
four fenced blocks now on this page are byte-identical to the text that was run;
their SHA-256 baselines are
`8acd531b7bcd2f2ca679e65024d83bd61debcb4b75bb18f3834a368454d574fd` for the
estimator, `bfa7721f5edae40fd883b5bc0f0776e499716c48cfdbe10d191e95b9f8a3bb0d` for
the probe generator,
`0b03f0c918a669b5402d1e630046bf2f1b9fc71453f54ef7119942d13a43c7ec` for the
geometry harness and
`3e57a5687d15c0725ab5cd1c2cea09b0a18a549ac649b82eee5246d1b75b8777` for the
calibration fixtures.

#### Page geometry against pageinspect

26 key widths from 8 to 2600 bytes, each at `fillfactor` 70, 90 and 100, built as
sorted builds with `deduplicate_items = off` and `PLAIN` storage; the aligned
tuple size was read from `bt_page_items` and the items per page from
`bt_page_stats`.

| Quantity | Published form | Revised form |
|---|---|---|
| Closed-leaf item count, 78 cells | exact in 48, one too high in 30 | exact in 78, no ragged cell |
| Internal-page item count, 45 testable cells | not modelled | exact in 45 |
| Whole-index `relpages`, 78 cells | exact in 26, within one block in 52, worst 15 | exact in 78, worst 0 |

The 30 cells the fillfactor-only form gets wrong are every `fillfactor = 100`
case and the four `fillfactor = 90` cases with aligned tuple sizes 904, 904, 1016
and 1216; in all 30 it is exactly one item too high.

The published `relpages` figures in that table were corrected on 2026-09-09. The
row previously read "exact in 53, within one block in 71, worst 5", which the
re-run does not reproduce: scored with the superseded statement's own closed
forms — leaf `ceil(rows / cap_soft)` and levels `ceil(pages / int_cap)` — the
published model is exact in 26 of 78 cells, within one block in 52, and 15 blocks
off at worst. The 53 figure belongs to a partial variant that keeps the revised
leaf and level recursion and only reverts the internal capacity; six variants
were scored to find it. The revised form's 78 of 78 is unchanged.
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server).

#### Fresh sorted builds end to end

Ten tables sized to about 250 leaf pages each, key widths 8 to 2000 bytes, indexed with
`CREATE INDEX` and analyzed. A fresh sorted build is its own ideal rebuild, so
the correct answer is `0 bytes`.

| Statement | Rows reporting exactly `0 bytes` | Worst reading |
|---|---|---|
| Published | 6 of 10 | `-33.9 %` at a 2000-byte key, `+11.0 %` at 1000 bytes |
| Revised | 10 of 10 | `0.0 %` |

Six further index shapes built fresh in the same database — unique, two-column,
INCLUDE, partial (300,000 of 1,200,000 rows), `fillfactor = 70`, and a NULL-heavy
column whose 100,000 NULLs deduplicate — read `0.0`, `0.0`, `-0.1`, `+0.4`, `0.0`
and `0.0` percent under both statements, so the changes leave the well-behaved
cases alone. On the NULL-heavy index the floor model reads `-29.7 %` against the
deduplication model's exact `0.0 %`.

#### The three deterministic defects

One database, one set of inputs, both statements run as the table owner and as a
role holding only `SELECT` on the tables.

| Fixture | Published | Revised |
|---|---|---|
| `inh_idx`, index on an inheritance parent, freshly built | `-550.8 %` (floor `-225.4 %`) | `0.0 %` |
| `expr_idx`, `lower(txt)` index, freshly built, read by the owner | `-22.2 %` | `0.0 %` |
| `expr_idx` read by a non-owner | absent from the report | present, `statistics not visible to this role` |
| `stz_idx`, expression index with `SET STATISTICS 0`, read by the owner | absent from the report | present, `statistics target zero on an index column`, and `-22.3 %` |
| `ovf_idx`, index `reltuples` forged to `1e30` | `ERROR: bigint out of range`, no rows at all | all rows, `idx_reltuples` printed as `1000000000000000000000000000000` |

Both `old` row counts are of the same ten-index database with the forged
`reltuples` reverted. The published statement drops `stz_idx` for the owner,
9 rows of 10, and **both** `expr_idx` and `stz_idx` for the reader, 8 rows of
10; the revised statement returns all 10 rows to both roles. This page said "9
rows instead of 10" for the reader until the 2026-09-09 re-run, which is wrong:
`stz_idx` has no `pg_stats` row for *any* role, because ANALYZE skips an
attribute whose `attstattarget` is 0, so the published missing-statistics term
suppresses it for the reader as well. No omission is explained anywhere in the
published output.
[analyze.c#examine_attribute](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1019-L1030),
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server).

The revised statement's `-22.3 %` on `stz_idx` is the price of the disabled
attribute, not a defect of the fix: with no statistics row the model falls back
to the 32-byte default width where the stored key is 24 bytes wide. The caveat
names the condition and [Reading the output](#reading-the-output) keeps such a
row out of a rebuild decision, but the number itself is not usable.

The inheritance fixture's two `pg_stats` rows for the same attribute recorded
`avg_width` 21 (`inherited = false`, the parent's own 20-character values) and 77
(`inherited = true`, parent plus a child of 100-character values). For the
privilege fixture, `has_column_privilege('expr_idx', 1, 'SELECT')` is false for
the reader while `has_table_privilege('expr_t', 'SELECT')` is true, and the
reader sees zero `pg_stats` rows for `expr_idx`.

#### Equal-image classification against the metapage

Ten indexes scored against `bt_metap().allequalimage`, which is the value the
build itself computed and stored.

| Model verdict | Indexes | Metapage flag | Agreement |
|---|---|---|---|
| `recognized` | `eq_int_idx`, `eq_text_idx`, `expr_idx`, `inh_idx`, `ovf_idx`, `stz_idx` | true | exact, 6 of 6 |
| `ineligible` | `eq_numeric_idx` (no support function 4), `inc_idx` (INCLUDE) | false | exact, 2 of 2 |
| `unknown` | `eq_custom_f_idx`, `eq_custom_t_idx` (a PL/pgSQL support function 4 returning false and true) | false, true | conservative |

The custom-opclass index whose support function returns true is the one case the
model under-credits: the engine deduplicated it to 1400 kB and the model priced
200,000 separate tuples, reading `-214.9 %` with the caveat
`unrecognized equal-image support function: no credit`. The false-returning twin
reads `0.2 %`, which is right.

#### Posting-list tails

Thirteen indexes over 400,000 rows: twelve uniform duplicate-group sizes chosen to
straddle the 132-TID capacity, and one index deliberately mixing 10-byte and
700-byte keys. Modelled blocks against `relpages`.

**Sign convention.** This table alone reports
`100 * (modelled_blocks / relpages - 1)`, so a positive number means the model
predicts *more* blocks than the build wrote. That is the negation of
`wasted_space_pct` used in every other table on this page, and the two open
questions that quote these figures follow this table's sign. The convention was
unlabelled until the 2026-09-09 re-run, which reproduced the row after
negating it.

| Group size | 1 | 2 | 50 | 131 | 132 | 133 | 200 | 263 | 264 | 265 | 500 | 1000 | mixed |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Published error | 0.0 | -0.5 | 0.3 | -9.3 | 0.9 | **98.8** | **33.1** | 0.9 | 0.6 | -9.9 | 0.8 | 0.6 | -51.2 |
| Revised error | 0.0 | -0.5 | 0.3 | 0.3 | 0.6 | **0.3** | **0.3** | 0.6 | 0.6 | -10.2 | 0.6 | 0.3 | -60.1 |

Mean absolute error falls from 15.92 % to 5.73 %, the count within one point
rises from 8 to 11 of 13, and the two capacity-boundary blowouts at group sizes
133 and 200 are gone. The two remaining outliers are filed as open questions:
group size 265, where each group is two full posting tuples plus a single-TID
tuple, and the mixed-width index, where one averaged key width cannot represent
two posting capacities. `bt_page_items` confirmed 132 TIDs at an item length of
808 bytes on every index whose groups reach the cap, matching
`floor((MAXALIGN_DOWN(812) - 16) / 6) = 132` and `MAXALIGN(16 + 132 * 6) = 808`.

#### The statistics publication barrier

The artifact finding 3 predicted reproduced without being sought. Nine tables
loaded and analyzed in one session left `n_live_tup` at 400,000 on a 200,000-row
table and 240,000 on a 120,000-row table, with `n_mod_since_analyze` at 200,000
and 120,000 **after** ANALYZE — ANALYZE wrote absolute counts and reset the
change counter in shared memory, and the deferred flush then added the pending
insert deltas on top. Two of the nine escaped, their flush having landed before
ANALYZE. Re-running the same script with `SELECT pg_stat_force_next_flush();`
between the load and the ANALYZE gave exact counts and a zero change counter on
all nine. Every fixture on this page uses that barrier.
[pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708),
[postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705),
[pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337).

**The artifact needs one more condition than "one session", and the 2026-09-09
re-run measured which.** An unforced flush is refused until
`PGSTAT_MIN_INTERVAL`, 1000 ms, has passed since the last one, so the load and
the ANALYZE must fall inside a single flush interval for the deltas to still be
pending when ANALYZE writes its absolute counts. Loading nine tables of 200,000
rows took 2.3 s on the 2026-09-09 host, a flush landed in between, and all nine
read exact counts and a zero change counter with no barrier at all. Shrinking
the same script to 20,000-row tables brought the load inside one interval and
reproduced the artifact on 9 of 9: `n_live_tup` 40,000 on a 20,000-row table,
24,000 on the 12,000-row one, and `n_mod_since_analyze` at 20,000 and 12,000
after ANALYZE. Adding the barrier to that fast script returned all nine to exact
counts and zero. So a fixture is not safe because it is single-session; it is
safe because it forces the flush.
[pgstat.c#flush-intervals](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122),
[pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600).

### Calibration by insertion pattern

Seven indexes, each grown by one insertion pattern, then measured, then rebuilt
with `REINDEX INDEX`. `true` is the byte reduction the rebuild actually produced;
`density` is `pgstatindex.avg_leaf_density` before the rebuild.

| Pattern | Density | Fragmentation | True | Published | Revised | Revised floor | Revised, re-read after the rebuild |
|---|---|---|---|---|---|---|---|
| sorted build | 90.05 | 0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| append-only inserts | 90.05 | 0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| random inserts | 66.90 | 50.05 | 25.7 | 25.7 | 25.7 | 25.7 | 0.0 |
| random inserts, 100-byte key | 69.61 | 50 | 23.8 | 18.6 | 23.8 | 23.8 | 0.0 |
| duplicate-heavy inserts | 95.49 | 0 | **-7.5** | -6.7 | -6.7 | -245.2 | 0.8 |
| delete half, then VACUUM | 45.17 | 0 | 49.8 | 49.8 | 49.8 | 49.8 | 0.0 |
| update half, then VACUUM | 60.13 | 0 | 33.3 | 33.3 | 33.3 | 33.3 | 0.0 |

The revised point estimate equals the true rebuild outcome in six of seven
patterns and is 0.8 points optimistic on the seventh. Read as calibration:

- **Append-only and sorted indexes settle at the leaf fillfactor**, 90.05 %
  density, and the estimate is 0.0. On these patterns any positive reading above
  about one point is real bloat, not model error.
- **Random inserts settle below it**, 66.90 % for a four-byte key and 69.61 % for
  a 100-byte key, because ordinary splits are 50:50. The estimate is exact, so
  the reading is the rebuild saving, not a threshold to tune.
- **Duplicate-heavy inserts settle above it**, 95.49 %, because a leaf page full
  of one value uses the single-value strategy at 96 %. A rebuild *grows* this
  index by 7.5 %, and the estimate says so with a negative number. Never rebuild
  on a negative reading.
- **Deleted and updated rows leave the largest true savings**, 49.8 % and 33.3 %,
  and the estimate matches both exactly.
- **The floor column is not a lower bound.** On the duplicate-heavy index it
  reads `-245.2 %` against a true `-7.5 %`, so it must not be used as the
  conservative reading on any index that can deduplicate.

Re-reading each index after its rebuild gives 0.0 on six of seven and 0.8 on the
duplicate-heavy one, so the estimator agrees with itself about a known-clean
index.
[nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335),
[nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033),
[nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L199-L202),
[pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31).

### Re-verified on a rebuilt server

**Every measured claim above was re-run on 2026-09-09 against a second server
built from the same pin, and every one reproduced.** The 2026-09-08 sandbox had
been deleted, so nothing was reused: 17.11 was configured out of tree with
`--with-icu --enable-debug --with-readline --with-zlib`, `make check` passed
**225 of 225**, and the `pageinspect`, `pgstattuple` and `amcheck` suites passed
8, 1 and 3 tests. The cluster was fresh (`--locale=C`, `autovacuum = off`,
`fsync = off`, `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`,
`max_parallel_maintenance_workers = 0`) and `raw/postgres-17/` was never written
to. Both statement texts were extracted from this page and from revision
`f2d73b4` and hashed before use; all five hashes matched their baselines.
[installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432),
[regress.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59),
[regress.sgml#contrib-suites](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195).

| Claim | As filed | Re-measured |
|---|---|---|
| Closed-leaf capacity, 78 cells | 78 exact; published 48, one too high in 30 | identical, and the same 30 cells |
| Internal-page item count | exact in 45 testable cells | exact in 45 |
| Whole-index `relpages`, revised | exact in 78 | exact in 78, worst 0 |
| Fresh sorted builds, revised | `0 bytes` on 10 of 10 | `0 bytes` on 10 of 10 |
| Fresh sorted builds, published | right on 6 of 10, worst `-33.9 %` at 2000 bytes, `+11.0 %` at 1000 | 6 of 10, `-33.9 %`, `+11.0 %`, and `-0.7 %` / `-3.4 %` at 400 and 800 |
| Expression-width defect | `-22.2 %` -> `0.0 %` | `-22.3 %` -> `0.0 %` |
| `bigint` range defect | published aborts with no rows; revised prints `numeric` | identical, 0 rows against 10 |
| Equal-image against `bt_metap` | 6 true, 2 false, 2 conservative | identical on all 10 |
| Posting capacity | 132 TIDs at `itemlen` 808 | 132 at 808, and 131 at 808 |
| Posting tails | mean absolute error 15.92 % -> 5.73 %; within one point 8 -> 11 of 13 | 16.06 % -> 5.77 %; 8 -> 11 of 13 |
| Calibration, seven patterns | every density, true, published, revised, floor and re-read value | digit for digit, all seven rows |
| Compression | 367 against 10,003 blocks, `avg_width` 904 both, `-2625.6 %` against `0.0 %` | identical |
| Probes | `false` on the empty subset, `true` on the forged zero, `Index Only Scan using empty_open` | identical; group probe 5,000 against a modelled 4,998 |
| `scripts/wiki_lint` | reports nothing on this page | whole-repository run: 0 errors, 0 warnings |

Six things did not survive the re-run and are corrected in place: the published
`relpages` cell of
[Page geometry against pageinspect](#page-geometry-against-pageinspect); the
non-owner row count in
[The three deterministic defects](#the-three-deterministic-defects); the
unlabelled sign convention of [Posting-list tails](#posting-list-tails); the
missing timing precondition in
[The statistics publication barrier](#the-statistics-publication-barrier); the
"32-byte tuples" wording under
[In-index compression of wide keys](#in-index-compression-of-wide-keys); and the
platform question, which the record below settles.

**Platform record**, as
[What still needs to be tested](#what-still-needs-to-be-tested) item 10 asked
for: `uname -sm` reports `Linux x86_64`; the server banner reads
`PostgreSQL 17.11 on x86_64-pc-linux-gnu, compiled by gcc ... 13.3.0, 64-bit`;
`pg_control_init()` reports `max_data_alignment` **8** and
`database_block_size` **8192**. So this page's "x86-64 Linux" statement matches
the host that ran both passes, and the geometry constants were measured at the
alignment and block size they assume.
[pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997),
[pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L204).

Three fixture families could not be rebuilt exactly, because this page never
published their row counts, so their magnitudes differ while their mechanisms
reproduce: the inheritance parent read `-461.0 %` (floor `-180.5 %`) with
`avg_width` 21 and 66 instead of `-550.8 %` with 21 and 77; the true-returning
custom-opclass index read `-226.0 %` on a 1352 kB index instead of `-214.9 %` on
1400 kB; and the six further fresh shapes read `0.0`, `0.0`, `0.0`, `+0.5`, `0.0`
and `+0.1` percent, with the NULL-heavy index's floor at `-20.8 %` rather than
`-29.7 %`. See
[Fixture recipes that do not reproduce](#fixture-recipes-that-do-not-reproduce).

### The deduplication gate, scored against the current statement

**The gate group of the mandatory suite is now scored against the recommended
text, and it passes.** 27 fixtures were built on two 500,000-row tables with
5,000 distinct values per key column — the shape this page filed on 2026-08-19 —
in a UTF8 database so that the ICU cases could exist. The oracle is
`bt_metap().allequalimage`, the verdict the build itself computed and stored.
[btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921),
[nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84).

| `equalimage` | metapage | credited | Fixtures |
|---|---|---|---|
| `recognized` | true | yes, 8 | `i_int4`, `i_int8`, `i_text_det`, `i_text_det2`, `i_text_icu_det`, `i_ei_alias`, `i_multi_ok`, `i2_ok` |
| `recognized` | true | no, 3 | `i_dupoff`, `i_text_off`, `i2_off` — `deduplicate_items = off` |
| `ineligible` | false | no, 9 | `i_numeric`, `i_float4`, `i_float8`, `i_inc`, `i_multi_bad`, `i_ei_none`, `i_expr_num`, `i_expr_lower_ci`, `i_text_nondet` |
| `unknown` | false | no, 5 | `i_ei_false`, `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft` |
| `unknown` | true | no, 2 | `i_ei_true`, `i_squat` |

Against the pass criteria of
[How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement):
no index is credited that the metapage says was not deduplicated, 0 of 27;
`equalimage` agrees with the metapage on every one of the 11 `recognized` and 9
`ineligible` rows; and no fixture reads above 30 % on either column, the maximum
being `i_multi_bad` at 28.8 % and the minimum `i_multi_ok` at `-562.1 %`. One
criterion needs correcting: the under-credits are **two**, not one. `i_ei_true`
is the designed case, and `i_squat` — an impostor support function — is the
second, at `-226.4 %` on an index the engine did deduplicate. Both are the
conservative direction.

Block counts came out identical to the 2026-08-24 run: 421 blocks for `i_int4`,
`i_int8`, `i_ei_alias`, `i_ei_true` and `i_squat`, 460 for the three
deterministic `text` fixtures, 459 for `i_multi_ok` and `i2_ok`, and 1931 for
the four test-15 fixtures. Four predictions from
[Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement)
are confirmed and three are refined:

| Prediction | Outcome |
|---|---|
| the credit decision does not move on any fixture | confirmed, 27 of 27 |
| `i_multi_bad` keeps its 28.8 % | confirmed, exactly 28.8 % under both texts |
| tests 13 and 15 stay near 0.2 % and gain `unrecognized equal-image support function: no credit` | confirmed: 0.2 % on all four test-15 fixtures, with the caveat |
| `i_ei_true` keeps its designed under-credit | confirmed at `-226.4 %`, the 2026-08-24 figure |
| "re-baseline all five" positives | the three `text` fixtures moved from `+7.8 %` to `-0.2 %`, so the geometry change *fixed* them rather than shifting them; `i_int4` and `i_int8` moved from `-0.5 %` to `-0.2 %` |
| `i_squat` lands in `unknown` | true only when the impostor is schema-qualified. `FUNCTION 4 btequalimage(oid)` in an operator class resolves through `pg_catalog` first and picks up the built-in, giving `recognized` and a true metapage; `FUNCTION 4 public.btequalimage(oid)` gives the intended `unknown`. This is a fixture-construction trap, not a statement defect |
| the two-column pair's arithmetic changes | it does not change without extended statistics: `(int4, int8)` reads `-320.0 %` under both texts, and `(int4, text)` `-562.1 %` under both. Adding `CREATE STATISTICS ... (ndistinct)` gives 8.1 % under the six-change text — the 2026-08-24 figure — and `0.0 %` under the current one |

That last row is the one operational finding of the group, and it is filed as
[Multicolumn key groups without extended statistics](#multicolumn-key-groups-without-extended-statistics).
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309),
[mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385),
[fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240).

### The collation branch, measured with ICU

**The `ineligible` branch that reads `collisdeterministic` is no longer
source-derived only.** Every earlier run was built `--without-icu`; the
2026-09-09 build enabled it, which is `configure`'s own default.
[installation.sgml#ICU-default](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L170).

| Step | Result |
|---|---|
| `CREATE COLLATION` in a `SQL_ASCII` database | `ERROR: current database's encoding is not supported with this provider`; the fixtures need `TEMPLATE template0 ENCODING 'UTF8'` |
| `CREATE COLLATION nd (provider = icu, locale = 'und-u-ks-level2', deterministic = false)` | `pg_collation.collisdeterministic` false |
| index on a `COLLATE nd` key | metapage `allequalimage` **false**; statement reads `ineligible`; build logs `index "..." cannot use deduplication` |
| index on a deterministic ICU key | metapage **true**; statement reads `recognized`; build logs `index "..." can safely use deduplication` |
| `CREATE INDEX ... (k text_pattern_ops)` on the `COLLATE nd` key | `ERROR: nondeterministic collations are not supported for operator class "text_pattern_ops"` |

So the gate's determinism test tracks the engine on both sides, and the
`text_pattern_ops` refusal this page derived from source happens verbatim. The
853 ICU collations in the fresh cluster were not otherwise exercised.
[varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615),
[pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575),
[nbtutils.c#_bt_allequalimage-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180),
[index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849).

### Reproducing the measurements

Build 17.11 out of tree from the pinned checkout, which must stay read-only, and
run `initdb` with `--locale=C`. The documentation describes the VPATH form for
`configure` and the mandatory build directory for `meson setup`.
[installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432),
[installation.sgml#meson-setup](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L2012-L2025).

The geometry harness, which needs `pageinspect`:

```sql
-- Geometry harness: one index per (key width, fillfactor), measured with
-- pageinspect, scored against the closed forms. PLAIN storage keeps
-- index_form_tuple() from compressing the wide keys.
CREATE EXTENSION IF NOT EXISTS pageinspect;

CREATE TABLE geo_result(keylen int, fillfactor int, rows_loaded int, itupsz int,
  leaf_pages int, closed_pages int, min_items int, max_items int, mode_items int,
  pred_soft int, pred_hard int, pred_cap int, int_pages int, int_min_items int,
  int_max_items int, int_pred_cap int, relpages int);

DO $geo$
DECLARE
  v_lens int[] := ARRAY[8,12,16,24,32,48,64,100,126,128,200,400,600,800,880,884,
                        888,892,896,900,904,1000,1200,1600,2000,2600];
  v_ffs  int[] := ARRAY[90, 100, 70];
  v_len int; v_ff int; v_rows int; v_szg int; v_capg int;
  v_sz int; v_bfull int; v_soft int; v_hard int;
BEGIN
  FOREACH v_ff IN ARRAY v_ffs LOOP
    FOREACH v_len IN ARRAY v_lens LOOP
      v_szg  := ((8 + 4 + v_len + 7) / 8) * 8;
      v_capg := greatest((8144 - (8192 * (100 - v_ff) / 100)) / (v_szg + 4), 1);
      v_rows := v_capg * 25;
      EXECUTE 'DROP TABLE IF EXISTS geo_t';
      EXECUTE 'CREATE TABLE geo_t(k text)';
      EXECUTE 'ALTER TABLE geo_t ALTER COLUMN k SET STORAGE PLAIN';
      EXECUTE format('INSERT INTO geo_t SELECT lpad(i::text, %s, ''0'')'
                     ' FROM generate_series(1, %s) i', v_len, v_rows);
      EXECUTE format('CREATE INDEX geo_i ON geo_t (k)'
                     ' WITH (deduplicate_items = off, fillfactor = %s)', v_ff);
      SELECT itemlen INTO v_sz FROM bt_page_items('geo_i', 1) bi WHERE bi.itemoffset = 2;
      v_bfull := 8192 * (100 - v_ff) / 100;
      v_soft  := (8192 - 48 - v_bfull) / (v_sz + 4);
      v_hard  := (8192 - 48 - 8 - v_sz) / (v_sz + 4);
      INSERT INTO geo_result
      SELECT v_len, v_ff, v_rows, v_sz, lf.leaf_pages, lf.closed_pages,
             lf.min_items, lf.max_items, lf.mode_items,
             v_soft, v_hard, least(v_soft, v_hard),
             ip.int_pages, ip.int_min_items, ip.int_max_items,
             1 + (8192 - 48 - (8192 * 30 / 100) - 12) / (v_sz + 4),
             (SELECT relpages FROM pg_class WHERE relname = 'geo_i')
        FROM (SELECT count(*) AS leaf_pages,
                     count(*) FILTER (WHERE btpo_next <> 0) AS closed_pages,
                     min(live_items - 1) FILTER (WHERE btpo_next <> 0) AS min_items,
                     max(live_items - 1) FILTER (WHERE btpo_next <> 0) AS max_items,
                     mode() WITHIN GROUP (ORDER BY live_items - 1)
                       FILTER (WHERE btpo_next <> 0) AS mode_items
                FROM generate_series(1, (SELECT relpages - 1 FROM pg_class
                                          WHERE relname = 'geo_i')) g(blkno)
                CROSS JOIN LATERAL bt_page_stats('geo_i', g.blkno) s
               WHERE s.type = 'l') lf,
             (SELECT count(*) AS int_pages,
                     min(live_items - 1) FILTER (WHERE btpo_next <> 0) AS int_min_items,
                     max(live_items - 1) FILTER (WHERE btpo_next <> 0) AS int_max_items
                FROM generate_series(1, (SELECT relpages - 1 FROM pg_class
                                          WHERE relname = 'geo_i')) g(blkno)
                CROSS JOIN LATERAL bt_page_stats('geo_i', g.blkno) s
               WHERE s.type IN ('i', 'r')) ip;
    END LOOP;
  END LOOP;
  EXECUTE 'DROP TABLE IF EXISTS geo_t';
END
$geo$;
```

The calibration fixtures, which need `pgstattuple`; score each index by reading
`pg_relation_size`, running `REINDEX INDEX`, and reading the size again:

```sql
-- Calibration fixtures: one insertion pattern per index, 300,000 rows each,
-- scored against REINDEX INDEX. setseed makes the random patterns repeatable.
CREATE EXTENSION IF NOT EXISTS pgstattuple;
CREATE SCHEMA cal;
SELECT setseed(0.42);

-- 1. sorted build: bulk load, then CREATE INDEX (the model's own baseline)
CREATE TABLE cal.sorted(k int);
INSERT INTO cal.sorted SELECT i FROM generate_series(1, 300000) i;
CREATE INDEX sorted_i ON cal.sorted(k);

-- 2. append-only: index first, ascending keys (rightmost splits use fillfactor)
CREATE TABLE cal.append(k int);
CREATE INDEX append_i ON cal.append(k);
INSERT INTO cal.append SELECT i FROM generate_series(1, 300000) i;

-- 3. random inserts: index first, unordered keys (ordinary 50:50 splits)
CREATE TABLE cal.random(k int);
CREATE INDEX random_i ON cal.random(k);
INSERT INTO cal.random SELECT (random() * 1000000000)::int FROM generate_series(1, 300000) i;

-- 4. duplicate-heavy inserts: three values in ascending runs (single-value strategy)
CREATE TABLE cal.dup(k int);
CREATE INDEX dup_i ON cal.dup(k);
INSERT INTO cal.dup SELECT (i / 100000)::int FROM generate_series(1, 299999) i;

-- 5. wide random inserts: 100-byte keys, unordered
CREATE TABLE cal.wide(k text);
ALTER TABLE cal.wide ALTER COLUMN k SET STORAGE PLAIN;
CREATE INDEX wide_i ON cal.wide(k);
INSERT INTO cal.wide SELECT lpad(((random() * 1000000000)::bigint)::text, 100, '0')
  FROM generate_series(1, 120000) i;

-- 6. delete churn: sorted build, delete half, VACUUM
CREATE TABLE cal.del(k int);
INSERT INTO cal.del SELECT i FROM generate_series(1, 300000) i;
CREATE INDEX del_i ON cal.del(k);
DELETE FROM cal.del WHERE k % 2 = 0;
VACUUM cal.del;

-- 7. update churn: sorted build, update half out of place, VACUUM
CREATE TABLE cal.upd(k int, pad text);
INSERT INTO cal.upd SELECT i, 'x' FROM generate_series(1, 300000) i;
CREATE INDEX upd_i ON cal.upd(k);
UPDATE cal.upd SET k = k + 1000000 WHERE k % 2 = 0;
VACUUM cal.upd;

-- ordered publication barrier, then statistics, then the barrier again
SELECT pg_stat_force_next_flush();
ANALYZE cal.sorted, cal.append, cal.random, cal.dup, cal.wide, cal.del, cal.upd;
SELECT pg_stat_force_next_flush();
```

The defect fixtures are an inheritance parent with a child of wider values and an
index on the parent alone; a non-partial `lower(txt)` expression index read by a
role holding only `SELECT` on the table; the same shape with
`ALTER INDEX ... ALTER COLUMN 1 SET STATISTICS 0`; an index whose `pg_class`
`reltuples` is set to `1e30`; and, for the equal-image matrix, indexes on `int4`,
`text`, `numeric`, an `int4` INCLUDE index, and two `int4` opclasses whose
support function 4 is a PL/pgSQL function returning true and false.

The 2026-09-09 re-run recorded the row counts the earlier recipes left out, so
these are the sizes its numbers belong to: ten fresh sorted builds of
`lpad(i::text, L, '0')` at `L` in (8, 16, 32, 64, 100, 200, 400, 800, 1000,
2000), each `((8144 - 819) / (MAXALIGN(12 + L) + 4)) * 250` rows with `PLAIN`
storage; 200,000 rows in every defect and equal-image fixture, the two custom
opclasses over 2,000 groups of 100; 400,000 rows per posting-tail fixture;
60,000 rows of `lpad(i::text, 900, 'x')` for each compression twin; 300,000 rows
per probe fixture with 5,000 groups in the group-probe one; and two 500,000-row
tables with 5,000 distinct values per key column for the gate group, in a
database created `TEMPLATE template0 ENCODING 'UTF8' LC_COLLATE 'C'` so the ICU
collations can be created.

Both statement texts are installed as views with three edits and nothing else:
delete the two `SET` lines, project the internals the scorer reads immediately
before `FROM modelled` — the projection needs a comma appended to
`server_version_num`, or the first added name silently becomes its alias — and
drop the `WHERE actual_bytes > 1024 * 1024 AND NOT suppress_row`, the `ORDER BY`
and the `LIMIT 20`. Privilege cases must run the filed text rather than a view,
because a view executes with its owner's privileges and would hide exactly the
effect under test.
[The current recommended statement](#the-current-recommended-statement),
[system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275).

### What remains unimplemented

- A per-attribute diagnostic projection that reports every input's provenance
  before the size and top-20 filters. The caveat strings now name the condition,
  but not which attribute produced it.
- A capacity model for indexes whose key width varies across groups. This is the
  largest remaining error, `-60.1 %` on the measured fixture.
- Any reading of the compressed stored width of a wide key. The statement can
  only warn.
- Cross-version work. Every number here is 17.11 at the default block size; the
  PostgreSQL 12 comparison this page began with has not been re-run against the
  revised statement.
- The partial-index half of the mandatory suite. Tests 1-17 were scored against
  this statement on 2026-09-09, but tests 18-91 and fixtures 92-121 have not;
  see [Mandatory test review](#mandatory-test-review) and
  [The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement).
- A group count for a multicolumn key with correlated columns. Without a
  `CREATE STATISTICS ... (ndistinct)` object the model multiplies per-column
  distinct counts, which cost `-320.0 %` and `-562.1 %` on two measured fresh
  builds; see
  [Multicolumn key groups without extended statistics](#multicolumn-key-groups-without-extended-statistics).

### Mandatory test review

This section is the source-only review filed earlier on 2026-09-09, before any
of the suite was re-run. Its inventory and gap list stand; the
deduplication-gate group has since been scored, under
[The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement).

**The suite this page calls mandatory had never been run against the statement
this page now recommends.** The asker filed it in three steps: seventeen
deduplication-gate tests on 2026-08-18, seventy-four partial-index tests on
2026-08-19 and the drained-queue test 113 on 2026-08-24, with fixtures 92-112
and 114-121 as the controls for changes A through E. Its contract was that a
statement failing a mandatory test is corrected, not merely reported. Every one
of those tests was last scored on 2026-08-24 against the six-change text whose
SHA-256 is `bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357`.
The 2026-09-07 cleanup removed the suite's tables from this page, and the
2026-09-08 implementation replaced that text with
`wiki_btree_wasted_space_sweep_r2` and measured it on new fixtures only. The
result is a page whose headline numbers come from one fixture family and whose
contractual tests are unscored for the current text.

Two things were checked today without a server. The text between the fence
lines of the four SQL blocks on this page hashes to the four baselines recorded
under [Measured acceptance results](#measured-acceptance-results), so the
statement under review is the one that was measured. And the superseded text is
recoverable: the revision of this page before the rewrite carries one SQL block,
and it hashes to `bffd166e…`, so a side-by-side rerun has both texts.
`scripts/wiki_lint` reports nothing on this page.

| Group | Tests | Fixtures and oracle | Last scored against | State for the current statement |
|---|---|---|---|---|
| Deduplication gate | 1-17 | 28 to 30 indexes on two 500,000-row tables with 5,000 keys each; the engine's `DEBUG1` verdict, `bt_metap().allequalimage` and posting tuples in `bt_page_items` | current text, 2026-09-09, 27 fixtures | **run and passed**; see [The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement) |
| Partial indexes | 18-91 | 74 indexes over about 58 tables, each populated, analysed, indexed, sized, read, rebuilt with `REINDEX INDEX` and sized again; verdict on `wasted_space_pct_floor` | six-change text, 2026-08-24, as a shape rebuild of the lost original scripts | not run; the 2026-09-08 run had one partial fresh build and two population-probe fixtures |
| Change A-D controls | 92-112 | threshold calibration 92-95, non-partial controls 96-99, variable-width `INCLUDE` 100-105, expression statistics 106-112 | six-change text, 2026-08-24 | not run |
| Drained queue and change E | 113a-c, 114-121 | one 1,000,000-row table per state; `reltuples = 0` shapes; a 12.2 twin | six-change text, 2026-08-24 | not run |
| Acceptance fixtures | 2026-09-08 | 78 geometry cells, 16 fresh builds, four defect fixtures read as two roles, 10 equal-image indexes, 13 posting-tail indexes, 9 barrier tables, 4 probe fixtures, 7 calibration patterns | current text | run twice; the 2026-09-09 repeat added ICU and reproduced every figure |
| Engine regression | `make check`, `contrib/pageinspect`, `contrib/pgstattuple`, `contrib/amcheck` | temporary installation of the build | 2026-09-09 build | passed 225 of 225, plus 8, 1 and 3; validates the build, not the estimator |
| Repository checks | `scripts/wiki_lint`, block hashes, Contents anchors | this repository | 2026-09-09 | pass; the whole-repository lint run reports 0 errors and 0 warnings |

`make check` runs the core regression tests against a temporary installation
inside the build tree, and each contrib module's tests run from its own
directory the same way. Those suites exercise the engine, not this wiki's
estimator: nothing in them reads `wasted_space_pct`. The adjacent engine
coverage they do provide is the deduplication block of `btree_index.sql`.
[regress.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59),
[regress.sgml#contrib-suites](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195),
[installation.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L515-L522),
[btree_index.sql#deduplication-tests](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L186-L213).

### Expected verdicts under the current statement

This section derives, from the two SQL texts, what the rerun should find. Nothing
in it is a measurement. The current text differs from the six-change text in five
ways that touch a mandatory test: the equal-image verdict is three-valued and the
determinism test applies only to `btvarstrequalimage` keys; the leaf, internal
and level arithmetic changed; posting tails are priced separately; an index
expression's width loses three bytes; and the missing-statistics term excludes
hidden and disabled attributes while five new caveat strings exist. Every
exclusion term from changes A through E is carried unchanged, and `live_rows`
is still the two-arm `CASE` that prices a `reltuples` of 0 as an empty index.
[The current recommended statement](#the-current-recommended-statement),
[What the ten plans changed](#what-the-ten-plans-changed).

**Deduplication gate, tests 1-17.** The gate reads the same catalog facts as
before, so the credit decision should not move on any fixture. What moves is
the label and the caveat. The engine's rule is unchanged: an `INCLUDE` index is
refused before any lookup, each key's support function 4 is looked up and then
called, and the build logs one `DEBUG1` line per index except for `INCLUDE`
indexes, which return before the message.
[nbtutils.c#_bt_allequalimage-INCLUDE](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147),
[nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183),
[nbtutils.c#_bt_allequalimage-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180),
[nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150).

| Tests | Fixtures | Six-change gate | Expected `equalimage` | Credit | Expected change in the reading |
|---|---|---|---|---|---|
| 1, 2, 3, 7, 14 | `i_int4`, `i_int8`, `i_text_det`, `i_text_icu_det`, `i_multi_ok`, `i2_ok`, `i_ei_alias` | true | `recognized` | yes | the 8.0 % on both `text` fixtures and the −320.0 % / 8.1 % two-column pair were arithmetic, and the tail pricing and geometry changed that arithmetic; re-baseline all five |
| 4, 9 | `i_text_nondet`, `i_expr_lower_ci` | false | `ineligible` | no | none expected; the collation test now fires only because `text_ops` registers `btvarstrequalimage` |
| 5, 6, 8, 9, 12 | `i_numeric`, `i_float4`, `i_float8`, `i_multi_bad`, `i_expr_num`, `i_ei_none` | false | `ineligible` | no | none expected; `i_multi_bad` keeps its 28.8 %, see [Integer-truncated widths across an alignment boundary](#integer-truncated-widths-across-an-alignment-boundary) |
| 10 | `i_inc` | false | `ineligible` | no | none expected |
| 11 | `i_dupoff`, `i_text_off`, `i2_off` | true, `dedup_applies` false | `recognized`, `dedup_applies` false | no | none expected; the column now says `recognized` on an index that is not credited |
| 13, 15 | `i_ei_false`, `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft` | false | `unknown` | no | readings stay near 0.2 % and gain `unrecognized equal-image support function: no credit` |
| 14 | `i_ei_true` | false by design | `unknown` | no | the designed under-credit remains; the 2026-09-08 twin `eq_custom_t_idx` read −214.9 % |
| 16 | `i_squat`, the replaced built-in, `i_text_det2` | false / false / true | `unknown` / `unknown` / `recognized` | no / no / yes | conservative on the first two; `i_text_det2` stays credited because `prosrc` survives a rename |
| 17 | ten fixtures on 12.2 | every gate false | not derivable | — | the text must first be shown to execute there; see [Cross-version execution of the revised statement](#cross-version-execution-of-the-revised-statement) |

Test 14's `i_ei_alias` is the case that separates `prosrc` from `proname`: its
function is `LANGUAGE internal AS 'btequalimage'` in schema `public`, and the
current gate's `proc_always` test reads exactly `lanname = 'internal'` and
`prosrc = 'btequalimage'`, so the index is `recognized`. Test 16's `i_squat`
has a SQL function under the built-in's name, which fails `lanname =
'internal'` and lands in `unknown`. The engine resolves a `LANGUAGE internal`
function by `prosrc`, which is why the alias deduplicates and the impostor
does not.
[fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240),
[datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L424-L438),
[varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615),
[The current recommended statement](#the-current-recommended-statement).

**Partial indexes, tests 18-91.** The five exclusion terms are the same
expressions, so the 36 withheld and 38 reported rows of the last run should
reproduce, and the eight false negatives (65, 67, 86-91) should remain, because
they are input errors the statement cannot see. The numbers move: on a 276-block
fixture the hard-fit reservation and the never-closed rightmost page change the
floor by at most a page or two, and every deduplicating fixture (22, 34, 36, 37,
40, 42, 44, 48, 51) gets a new point estimate from the tail pricing. Two
predictions are worth writing down before the run. Test 36, one key group of
100,000 TIDs, was exact under the old rounding and should stay exact, because
`floor(100000 / 132)` full posting tuples plus one tail is what the build
writes. And test 47 stays withheld, because `any_varlena_include` is unchanged.
[nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1026-L1050),
[nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513),
[Page and posting geometry](#page-and-posting-geometry).

**Controls 92-121.** The table lists every control whose output is expected to
differ from the 2026-08-24 run, and the two that are expected to fail again.

| Fixture | Shape | 2026-08-24 result | Expected now | Why |
|---|---|---|---|---|
| `x109` (109) | plain index, key column with `SET STATISTICS 0` | 64.9 %, no caveat, reported | 64.9 %, reported, with `statistics target zero on an index column` | the caveat is new; [Reading the output](#reading-the-output) does not list it, so the row is still alertable |
| `i103` (103) | partial, wide plain key, unique values | 84.1 %, no caveat, reported | the same | nothing in the current text reads a subset's width; this is the surviving critical false positive |
| `p118` (118) | subset measured empty, then 50,000 rows arrive | 99.3 %, `status = ok`, no caveat | 99.3 % with `zero modelled rows: validate with a population probe`; the generator emits an `EXISTS` probe that should return `true` | `live_rows = 0` now raises the caveat, and the probe was measured to catch a forged zero |
| `p120` (120) | a 300-row sample missed a 2,000-row subset | 87.5 %, below the 1 MB filter | the same with the caveat | as above; the sample size is `300 * attstattarget` |
| 113b, 113c, `p75` | drained subsets | 100.0, 100.0, 99.6 | the same with the caveat; the probe should return `false` | a true zero is confirmed rather than believed |
| `nz_k` (121) | non-partial, emptied, vacuumed, reloaded | 100.0 with `row-count sources disagree` | the same plus `zero modelled rows` | both conditions hold |
| `nzb_k`, `i_trunc` (121) | rebuilt or truncated while empty | `unmeasured: reltuples unknown` | the same | a new relfilenode writes `reltuples = -1`, and a build that counts zero rows leaves it |
| `np97`, `x110`, `x106`, `x111` | expression index, no statistics row | withheld by change D | withheld | the term is unchanged |
| `x108` | expression index, never-analysed table | reported with `never analyzed` | the same | unchanged |
| 92-95, 96, 98, 99, 100-102, 104, 105, 107, 112 | change B thresholds and the C/D controls | as filed | states unchanged, numbers re-baselined | the terms are unchanged; the arithmetic is not |

Fixture 118 is the one the asker adopted knowingly on 2026-08-24, and it is the
one whose classification the current text changes most: the row still reports,
but a reader following [Reading the output](#reading-the-output) does not
promote a `zero modelled rows` row without the probe, and the probe answers. The
index never grows in that fixture because an `UPDATE` whose new row fails the
predicate skips the partial index, which is why the whole file is dead. All of
this is prediction to confirm, not result.
[Validation probes](#validation-probes),
[execIndexing.c#partial-predicate-skip](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L384-L386),
[relcache.c#RelationSetNewRelfilenumber-reltuples](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3951-L3952),
[index.c#index_update_stats-sentinel](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2836),
[analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894).

**The 2026-09-08 fixtures.** These were the only measurements of the current
text when this section was written, and their limits were the rerun's first
targets. The build was `--without-icu`, so no fixture exercised a
nondeterministic collation and the `ineligible` branch that reads
`collisdeterministic` was source-derived only. The run was owner-only except for
the four defect fixtures, and it had no partial-index suite and no 12.2 leg. The
2026-09-09 rerun closed the ICU gap and the platform question, and left the
partial-index suite and the 12.2 leg open; the measured outcomes are under
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server) and
[The collation branch, measured with ICU](#the-collation-branch-measured-with-icu).
[installation.sgml#--without-icu](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1209-L1216),
[pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575),
[Measured acceptance results](#measured-acceptance-results).

### What still needs to be tested

In priority order. Items 1 through 4 are the mandatory contract; the rest are
gaps this review found.

1. **The numbered suite against the current text.** Tests 1-17 were scored on
   2026-09-09 and passed; tests 18-91, fixtures 92-112, test 113 and fixtures
   114-121 remain, rebuilt from the page history and scored by the procedure
   below. The pass criteria are step 9 of that procedure, amended by
   [The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement)
   to allow two under-credits.
2. **Attribution of every moved row.** Install the superseded text
   (`bffd166e…`) beside the current one and run `EXCEPT` in both directions over
   the columns both views project, on the same fixture state, before any
   `REINDEX`. A row that moves must be explained by one of the five changes
   named above; an unexplained move is a defect.
3. **ICU.** Done on 2026-09-09 for the gate cases: the `ineligible` branch, its
   deterministic twin and the `text_pattern_ops` refusal are measured under
   [The collation branch, measured with ICU](#the-collation-branch-measured-with-icu).
   What remains is the partial-index legs, tests 51 and 52, and a non-C locale
   for the cluster rather than for one column.
   [installation.sgml#ICU-default](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L170),
   [installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193),
   [index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849).
4. **The 12.2 leg of test 17.** The current text reads `pg_stats_ext.inherited`,
   which this version's view defines. Whether the pinned 12 view defines it is
   a version-local question this page cannot answer; the 12.2 leg therefore
   starts by executing the exact text and recording the outcome, before any
   fixture is built. The v12 page carries that version's build and publication
   notes.
   [system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290),
   [The v12 publication protocol](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md#the-v12-publication-protocol),
   [V12 catalog, build and output compatibility](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md#v12-catalog-build-and-output-compatibility).
5. **Role coverage.** `stats_hidden` now decides both a caveat and whether the
   missing-statistics term fires, and `pg_stats` filters an expression
   attribute's row by the index's owner-only ACL. Run the expression fixtures
   (`p48` to `p50`, `x106` to `x112`, `np97` and the four `i_expr_*` indexes) as
   a role holding only `SELECT` on the tables, and record which rows appear and
   with which caveat.
   [system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275),
   [acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569).
6. **Fixture contract.** Before a fixture is scored it asserts its intended row
   count, predicate membership, deletion fraction and duplicate groups. Two
   known recipe defects are corrected first: `p75` drains its subset completely
   where its comment says 90 %, and `CREATE INDEX np99 ON np99` cannot run
   because an index and a table share a namespace. `pg_stat_force_next_flush()`
   precedes every `ANALYZE` and `VACUUM`, which the barrier measurement on this
   page made a rule.
   [pgstatfuncs.c#snapshot-and-flush-functions](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c#L1680-L1695),
   [stats.sql#forced-flush](../../../../raw/postgres-17/src/test/regress/sql/stats.sql#L101-L102),
   [The statistics publication barrier](#the-statistics-publication-barrier).
7. **The scoring column.** The partial-index contract scores
   `wasted_space_pct_floor`; the calibration on this page found the floor
   unusable as a lower bound on a deduplicating index. The rerun records both
   columns and classifies each row twice; which column carries the verdict is
   the asker's decision, filed under
   [Scoring column for the partial-index contract](#scoring-column-for-the-partial-index-contract).
8. **Untested configurations.** A `--with-blocksize=16` or `32` build, because
   every geometry constant in the statement is derived from `block_size` and no
   run has checked the arithmetic at another size; a parallel build, which
   `btbuild` starts when `ii_ParallelWorkers > 0` and which still ends in
   `_bt_leafbuild`; a `CREATE INDEX CONCURRENTLY` and a `REINDEX CONCURRENTLY`
   build, which both reach `index_build` through `index_concurrently_build`; a
   partitioned table, whose per-partition indexes enter the candidate set as
   `relkind = 'i'` while the partitioned index itself is `'I'` and excluded,
   and whose `ANALYZE` expands to the partitions; and a non-C locale.
   `MAXIMUM_ALIGNOF` other than 8 and big-endian hardware are not available in
   this environment and stay open.
   [installation.sgml#--with-blocksize](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1472-L1482),
   [nbtsort.c#btbuild-parallel](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L389-L392),
   [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571),
   [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1533-L1539),
   [indexcmds.c#DefineIndex-concurrent-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1682),
   [indexcmds.c#ReindexRelationConcurrently-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4009),
   [pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173),
   [vacuum.c#expand_vacuum_rel-partitions](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L963-L982),
   [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259).
9. **Harness typing.** The result table of the partial-index harness declared
   `modelled_rows`, `key_groups` and `idx_reltuples` as `bigint`; the current
   text projects them as `numeric`, and the `ovf_idx` defect fixture would then
   fail the harness rather than the statement. Declare them `numeric` and add
   `wasted_space_bytes`, `equalimage` and `reltuples_writer` columns.
   [numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549).
10. **Platform record.** Done on 2026-09-09: `Linux x86_64`,
    `max_data_alignment` 8, `database_block_size` 8192, and the configure line
    recorded under
    [Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server). Keep
    storing the two `pg_control_init()` values with every result row so a future
    non-default build is self-identifying.
    [pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997),
    [pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L204).
11. **Cost.** No timing exists for the current text. The current text adds the
    `opc` join, the per-class `classfit` and `classsize` split, and two
    privilege function calls per attribute over the six-change text. Time six
    interleaved pairs against the superseded text on the rebuilt fixture
    database and on a database of several hundred indexes, and file the
    distributions rather than a single number.

### How to run the suite against the current statement

Every step below was designed against the pinned source and the harness this
page filed on 2026-08-19 and 2026-08-24; none of it was executed for this
review. The checkout under `raw/postgres-17/` stays read-only: the build is a
VPATH build in a directory under `.wiki-runtime/tmp/`, which is the form the
documentation describes.
[installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432).

**1. Build and check.** Configure with ICU, which is on by default, and with
debugging symbols; run the core suite and the three contrib suites the oracles
need. On a host where ICU is not on the default search path, pass `ICU_CFLAGS`
and `ICU_LIBS` as the documentation shows.
[installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193),
[installation.sgml#--enable-debug](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1530-L1540),
[regress.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59),
[regress.sgml#contrib-suites](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195).

```sh
mkdir -p .wiki-runtime/tmp/btree17r/build && cd .wiki-runtime/tmp/btree17r/build
../../../../raw/postgres-17/configure --prefix="$PWD/../install" \
    --enable-debug --without-readline --without-zlib
make -j8 && make check
for m in pageinspect pgstattuple amcheck; do
  make -C contrib/$m && make -C contrib/$m check
done
make install
for m in pageinspect pgstattuple amcheck; do make -C contrib/$m install; done
```

**2. Cluster.** Initialise with `--locale=C`, then start with the settings
below. The context column is each GUC's definition in the pinned table, and it
decides how a later change is applied.
[initdb.sgml#--locale](../../../../raw/postgres-17/doc/src/sgml/ref/initdb.sgml#L281-L291).

| Setting | Value | Context | Scope of a change |
|---|---|---|---|
| `autovacuum` | `off` | `PGC_SIGHUP` | reload; set before the first fixture so no background `ANALYZE` repairs one mid-test |
| `fsync` | `off` | `PGC_SIGHUP` | reload |
| `shared_buffers` | `512MB` | `PGC_POSTMASTER` | restart |
| `maintenance_work_mem` | `256MB` | `PGC_USERSET` | session or transaction |
| `max_parallel_maintenance_workers` | `0` for the suite, `2` for the parallel-build case | `PGC_USERSET` | session or transaction |
| `client_min_messages` | `debug1` around `CREATE INDEX` in the gate harness | `PGC_USERSET` | session or transaction |
| `default_statistics_target` | `1` for fixture 120 only | `PGC_USERSET` | session or transaction |
| `statement_timeout`, `lock_timeout` | `600s` and `2s` in the harness session | `PGC_USERSET` | session or transaction |

[guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1453),
[guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1097-L1100),
[guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262-L2265),
[guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2469),
[guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3410-L3413),
[guc_tables.c#client_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4777-L4780),
[guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2074),
[guc_tables.c#statement_timeout-and-lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631).

**3. Statement texts.** Extract the estimator block from this page and the
superseded block from the page's previous revision, verify both hashes, and
install each as a view with the harness edits this page documents: drop the
1 MB filter, the `ORDER BY` and the `LIMIT 20`; drop `AND NOT suppress_row` and
project `suppress_row` instead; and project the internals the scorer reads
(`expected_blocks`, `floor_blocks`, `actual_bytes`, `live_rows`, `slot`,
`leaf_cap`, `nmax`, `dedup_applies`, `is_partial`, `equalimage_state`,
`stats_row_missing`, `dedup_credited`, `stats_stale`). Run both exact texts
once as filed, filter and `LIMIT` intact, to prove they execute. The extraction
below was run for this review and reproduced both hashes.

```sh
awk '/^```sql$/{n++; inb=1; next} /^```$/{inb=0; next} inb && n==1' \
  wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md > est_r2.sql
shasum -a 256 est_r2.sql   # 8acd531b7bcd2f2ca679e65024d83bd61debcb4b75bb18f3834a368454d574fd
git show f2d73b4:wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md \
  | awk '/^```sql$/{n++; inb=1; next} /^```$/{inb=0; next} inb && n==1' > est_old.sql
shasum -a 256 est_old.sql  # bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357
```

**4. Fixtures.** Rebuild them from the page history. The deduplication-gate
harness is verbatim in revision `33fe5a4` of this page under "The harness,
runnable"; the partial-index shapes are the requirement tables under "The
seventy-four partial-index tests, and the verdict on each"; the 92-121 recipes
are described in the change A through E sections of the same revision. Apply
the two recipe corrections, give every fixture an assertion of its intended
population, and call `pg_stat_force_next_flush()` before every `ANALYZE` and
`VACUUM`. In `CREATE INDEX`, `WITH (fillfactor = ...)` precedes `WHERE`.
[gram.y#IndexStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L8093-L8095).

**5. Score.** One procedure per index, in this order: assert the population,
record `pg_relation_size`, read both views, `REINDEX INDEX`, record the size
again. The `res` table is the 2026-08-19 harness's with `modelled_rows`,
`key_groups` and `idx_reltuples` as `numeric` and three added columns,
`wasted_space_bytes numeric`, `equalimage text` and `reltuples_writer text`.
The verdict query classifies each row on both columns, replaces the
2026-08-19 rule's unclassified band with an explicit five-point margin, and
adds an `alertable` column that applies this page's reading rule: a row is
alertable only when its `caveats` string contains none of `never analyzed`,
`row-count sources disagree`, `statistics not visible`, `zero modelled rows`
and `wide compressible key`.

```sql
SET /* wiki_btree_mandatory_verdict_statement_timeout */ statement_timeout = '30s';
SET /* wiki_btree_mandatory_verdict_lock_timeout */ lock_timeout = '2s';

SELECT /* wiki_btree_mandatory_verdict */
       num, idx, blocks_before, blocks_after, a.actual, wsp, wspf,
       v.verdict_point, v.verdict_floor,
       (caveats IS NULL OR caveats !~ '(never analyzed|row-count sources disagree|statistics not visible|zero modelled rows|wide compressible key)')
                                                          AS alertable,
       caveats, equalimage
  FROM res
  CROSS JOIN LATERAL (
        SELECT round(100.0 * (size_before - size_after) / greatest(size_before, 1), 1) AS actual) a
  CROSS JOIN LATERAL (
        SELECT CASE WHEN wsp IS NULL THEN 'UNMEASURED'
                    WHEN wsp >= 50 AND a.actual < 10 THEN 'CRITICAL FALSE POSITIVE'
                    WHEN wsp >= 50 AND a.actual < 45 THEN 'FALSE POSITIVE'
                    WHEN wsp >= 50 AND wsp - a.actual > 5 THEN 'FALSE POSITIVE'
                    WHEN wsp <  45 AND a.actual >= 50 THEN 'FALSE NEGATIVE'
                    ELSE 'PASS' END                        AS verdict_point,
               CASE WHEN wspf IS NULL THEN 'UNMEASURED'
                    WHEN wspf >= 50 AND a.actual < 10 THEN 'CRITICAL FALSE POSITIVE'
                    WHEN wspf >= 50 AND a.actual < 45 THEN 'FALSE POSITIVE'
                    WHEN wspf >= 50 AND wspf - a.actual > 5 THEN 'FALSE POSITIVE'
                    WHEN wspf <  45 AND a.actual >= 50 THEN 'FALSE NEGATIVE'
                    ELSE 'PASS' END                        AS verdict_floor) v
 ORDER BY num;
```

Oracles are read beside each row and never scored: `bt_metap().allequalimage`
against `equalimage`, the `DEBUG1` line from the build, `count(tids) > 0` over
`bt_page_items` for posting lists, `pgstatindex` for density, and
`bt_index_check` after any support-function mutation in test 16, because it
raises when the metapage disagrees with the current catalog.
[pageinspect--1.8--1.9.sql#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L73-L82),
[pageinspect--1.8--1.9.sql#bt_page_items](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L109-L118),
[btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921),
[pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31),
[amcheck--1.0--1.1.sql#bt_index_check](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.0--1.1.sql#L12-L28),
[verify_nbtree.c#metapage-equalimage-check](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L380-L396).

**6. Attribution.** Before the first `REINDEX`, run `SELECT ... FROM est_r2
EXCEPT SELECT ... FROM est_old` and its reverse over the columns both views
project, and keep every returned row with the change that explains it. This is
the same proof the 2026-08-20 and 2026-08-24 runs used, and it is what turns a
new verdict table into a regression result.

**7. Probes.** Run the generator from [Validation probes](#validation-probes)
on the fixture database, execute each emitted statement, and record its result
beside the row. Expected: `true` for fixtures 118 and 120, `false` for 113b,
113c and `p75`, and a group count within the sampling error of `key_groups`
for every recognized gated index.

**8. The 12.2 leg.** Build the pinned 12 checkout the same way, execute the
exact current text, and record the outcome as a result in its own right. Only
if it executes, run the transformer that drops the constructs 12 lacks and score
the constructible subset against a measured `REINDEX INDEX`.

**9. Pass criteria.** The suite passes for the current text when all of the
following hold:

- Gate group: no index is credited that the metapage says was not
  deduplicated; the only under-credit is `i_ei_true`; `equalimage` agrees with
  `bt_metap().allequalimage` on every `recognized` and `ineligible` row; no
  fixture reads above 30 % on either column.
- Partial group: no critical false positive among reported rows on the chosen
  column; the true detections (68, 74, 77 and the corrected 75) are reported
  within five points; every withheld row names the term that withheld it.
- Controls: 113b and 113c read 100.0; every `modelled_rows = 0` row carries
  `zero modelled rows` and has a recorded probe result; `x109` and `i103` are
  filed as residual false positives rather than passed silently.
- Every `EXCEPT` row is attributed to one of the five changes.
- Both exact texts execute as filed, and no row raises an error.
- `make check` and the three contrib checks pass; both block hashes match;
  `scripts/wiki_lint` reports no new issue.

**10. Filing.** The measured verdicts replace the expected columns in
[Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement),
with the platform record from item 10 above. The sandbox lives under
`.wiki-runtime/tmp/` and is deleted or kept as the asker directs.

## Context Reviewed

- PostgreSQL 17 pin `786db8dcf168bd9df8f55047337525ac19118b1c`; the source checkout is read-only and was never written to.
- The revised statement's catalog inputs, CTE dependencies, formulas, output projection, exclusions and timeout settings, and the superseded statement extracted from the previous revision of this page for the side-by-side runs.
- The sorted-build caller/callee path and its page arithmetic (`nbtsort.c`, `nbtdedup.c`, `bufpage.c`, `nbtree.h`), tuple formation and varlena header handling (`indextuple.c`, `heaptuple.c`, `varatt.h`, `heaptoast.h`, `htup_details.h`), index attribute construction (`index.c`), expression-statistics selection and the two ANALYZE passes (`analyze.c`), statistics visibility and privilege resolution (`system_views.sql`, `acl.c`, `aclchk.c`), cumulative-statistics flush ordering (`postgres.c`, `pgstat.c`, `pgstat_relation.c`), split strategies (`nbtsplitloc.c`), size formatting and numeric range errors (`dbsize.c`, `numeric.c`), and the generated catalog/function boundary.
- Plan review on 2026-09-07, same pin, retained above.
- Implementation and measurement run on 2026-09-08, same pin: 17.11 built out of tree under `.wiki-runtime/tmp/btree17/` (`--without-readline --without-zlib --without-icu`), an isolated cluster on port 55437 with `autovacuum = off`, `fsync = off`, `shared_buffers = 256MB` and `--locale=C`; `make check` 225 of 225 tests passed, `contrib/pageinspect` and `contrib/pgstattuple` checks passed; six fixture databases covering page geometry, fresh sorted builds, the three deterministic defects, the equal-image matrix, posting-list tails, validation probes and the insertion-pattern calibration. `pageinspect` and `pgstattuple` were installed in the disposable cluster only. The sandbox was deleted after filing, so reproducing any number means rebuilding from the pin and re-running the published SQL.
- Mandatory test review on 2026-09-09, same pin, no server built or started: the revisions of this page before the 2026-09-07 cleanup (`33fe5a4`, `f8265ad`) and before the 2026-09-08 rewrite (`f2d73b4`) for the suite's requirement tables, harnesses and fixture recipes; the log entries of 2026-08-18, 2026-08-19, 2026-08-20, 2026-08-24 and 2026-09-08 for each run's provenance; the four fenced blocks on this page hashed against their recorded baselines, and the superseded block recovered from `f2d73b4` and hashed against `bffd166e…`; the installation and regression documentation for VPATH builds, ICU, block size, `make check` and contrib suites; the GUC contexts of every cluster setting the protocol names; and the equal-image, concurrent-build, parallel-build, partition-expansion, statistics-sample and `reltuples`-sentinel paths cited in the review.
- Full re-verification on 2026-09-09, same pin: every source citation on this page re-read against `raw/postgres-17/` (476 links over 68 files, 164 distinct ranges, all resolving and all supporting their labels), every `## Contents` entry and page-internal anchor re-checked, and every measured claim re-run on a second isolated server. 17.11 was built out of tree under `.wiki-runtime/tmp/btreerev/` with `--with-icu --enable-debug --with-readline --with-zlib`; `make check` passed 225 of 225 and the `pageinspect`, `pgstattuple` and `amcheck` checks passed 8, 1 and 3; the cluster ran `--locale=C`, `autovacuum = off`, `fsync = off`, `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`, `max_parallel_maintenance_workers = 0` at the default `BLCKSZ`, with eleven fixture databases covering page geometry, fresh sorted builds, the three deterministic defects read as two roles, the equal-image matrix, posting tails, the statistics barrier, the probes, in-index compression, the 27-fixture deduplication gate and the ICU collation cases. Both statement texts were extracted and hashed before use. `raw/postgres-17/` was never written to and stayed clean at the pin; the build and cluster were deleted after filing, leaving only the fixture scripts under `.wiki-runtime/tmp/btree-rev-harness/`.

## Evidence Map

| Claim group | Primary evidence |
|---|---|
| Candidate and count inputs | [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L26-L62), [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66), [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703). |
| Sizes, relation locks and missing-relation behavior | [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371). |
| Leaf and internal page capacity | [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629), [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L907-L923), [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854), [nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735), [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1147). |
| The never-closed rightmost page of each level | [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1062-L1128), [nbtsort.c#_bt_slideleft](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L682-L700). |
| Posting-list capacity and sizing | [nbtsort.c#maxpostingsize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1304-L1305), [nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513), [nbtdedup.c#_bt_dedup_start_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L432-L474), [nbtdedup.c#_bt_form_posting-size](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L879-L884). |
| Equal-image eligibility and its three states | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183), [datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L424-L438), [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84), [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921). |
| Width, expression header and compression | [analyze.c#compute_scalar_stats-width](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2420-L2426), [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L234-L242), [varatt.h#VARATT_CAN_MAKE_SHORT](../../../../raw/postgres-17/src/include/varatt.h#L257-L262), [indextuple.c#index_form_tuple-compression](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138), [heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68). |
| Inheritance passes and statistics visibility | [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259), [analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647), [system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275), [acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569), [aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863). |
| Probe construction and grouping semantics | [pg_proc.dat#pg_get_expr](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8219-L8220), [nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296), [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905), [indxpath.c#check_index_predicates](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3244-L3350). |
| Flush ordering and the counter artifact | [postgres.c#idle-stats-flush](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L4634-L4705), [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600), [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337). |
| Output formatting, range errors and split strategies | [dbsize.c#pg_size_pretty-sign](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L569-L600), [numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549), [nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033). |
| Harness oracles | [btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L108-L194), [pageinspect--1.8--1.9.sql#bt_page_stats](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L87-L124), [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31). |
| Core-SQL contract and model-specific choices | [The current recommended statement](#the-current-recommended-statement). These expressions are the wiki's model, not a PostgreSQL engine guarantee. |
| Mandatory test inventory and what the engine suites cover | [regress.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59), [regress.sgml#contrib-suites](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195), [installation.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L515-L522). |
| Expected deduplication-gate verdicts | [nbtutils.c#_bt_allequalimage-INCLUDE](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147), [nbtutils.c#_bt_allequalimage-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180), [fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150). |
| Expected control verdicts | [execIndexing.c#partial-predicate-skip](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L384-L386), [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3951-L3952), [index.c#index_update_stats-sentinel](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2836), [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894). |
| Rerun protocol: build, cluster settings and oracles | [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432), [installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193), [installation.sgml#--enable-debug](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1530-L1540), [initdb.sgml#--locale](../../../../raw/postgres-17/doc/src/sgml/ref/initdb.sgml#L281-L291), [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1453), [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262-L2265), [pageinspect--1.8--1.9.sql#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L73-L82), [pageinspect--1.8--1.9.sql#bt_page_items](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L109-L118), [amcheck--1.0--1.1.sql#bt_index_check](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.0--1.1.sql#L12-L28), [verify_nbtree.c#metapage-equalimage-check](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L380-L396). |
| Untested build paths and relation shapes | [installation.sgml#--with-blocksize](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1472-L1482), [nbtsort.c#btbuild-parallel](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L389-L392), [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1533-L1539), [indexcmds.c#ReindexRelationConcurrently-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4009), [pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173), [vacuum.c#expand_vacuum_rel-partitions](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L963-L982). |
| Cross-version, width and platform open questions | [system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290), [pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L41-L50), [analyze.c#stawidth](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540), [pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997). |
| The statistics-flush interval that gates the publication artifact | [pgstat.c#flush-intervals](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122), [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L584-L600), [pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708). |
| Gate scoring oracle and the internal-function resolution the impostor case turns on | [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84), [nbtutils.c#_bt_allequalimage-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180), [fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240). |
| The measured nondeterministic-collation branch | [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2615), [pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575), [index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849), [installation.sgml#ICU-default](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L170). |
| Multicolumn key groups and the extended-statistics escape | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309), [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385), [The current recommended statement](#the-current-recommended-statement). |

## Open Questions

These are the limitations that survived the 2026-09-08 implementation run and
the 2026-09-09 re-verification, each with the number that measured it. The
2026-09-09 pass added
[Multicolumn key groups without extended statistics](#multicolumn-key-groups-without-extended-statistics)
and replaced the platform question with
[Fixture recipes that do not reproduce](#fixture-recipes-that-do-not-reproduce).

### Mixed key widths in one index

A single averaged key width cannot represent an index whose groups have different
base tuple sizes, because the posting capacity `floor((808 - itupsz) / 6)` depends
on that size: 130 TIDs for a 10-byte key and 16 for a 700-byte key. An index built
half from each read `-60.1 %` against `relpages` under the revised statement and
`-51.2 %` under the published one, so the two-size tail pricing made this case
slightly worse while fixing the uniform cases. A width-distribution model would
need per-group widths that no catalog records. Until then, treat any duplicate-heavy
index with a widely varying key as unmeasured.
[nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513),
[Measured acceptance results](#measured-acceptance-results).

### Alternating posting and singleton tuples

A class whose groups are two full posting tuples plus one single-TID tuple packs
12 items per leaf page where the class-mean capacity predicts 13, because whether
a page closes on the fillfactor limit depends on the posting overhead of whichever
tuple happens to be last, and that alternates. The measured error is `-10.2 %` at
group size 265 and `-9.9 %` under the published statement, so this one is not new.
Modelling it needs the *sequence* of tuple sizes on a page, not their mean.
[nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854),
[nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1026-L1050).

### In-index compression of wide keys

`index_form_tuple` compresses a varlena key wider than `TOAST_INDEX_TARGET`
whose storage is `extended` or `main`, and nothing in the catalogs records the
compressed width. On 60,000 rows of a 900-character repetitive key the index
stored a 32-byte compressed datum in a 40-byte tuple and occupied 367 blocks,
while the same data under `plain` storage stored 912-byte tuples in 10,003
blocks; `pg_stats.avg_width` was 904 for both, so the estimator read
`-2625.6 %` against an exact `0.0 %`. The 2026-09-09 re-run reproduced both
block counts, both widths and both readings, and measured the item length
directly: every item on a leaf page of the compressed index is 40 bytes,
`MAXALIGN(8 + 32)`, which is what corrects this page's earlier "32-byte
tuples". The statement warns with
`wide compressible key: stored width may be over-stated` but cannot correct the
number. A fix would need a sampled `pg_column_size` probe of the index expression,
which is not a catalog read.
[indextuple.c#index_form_tuple-compression](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138),
[heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68).

### Most-common-value class frequencies

The single-key model splits rows into a NULL class, one class per most-common
value, and a remainder. On a uniform 400,000-row index of 8,002 groups of 50 rows
each, the sampled MCV frequencies implied 146 to 160 rows per group for the five
MCV classes — three times the truth. Their weight is small, so the total error
stayed at 0.3 %, but a skewed index with heavy MCVs has no such protection, and
nothing on this page measures that case. The group probe is the available check.
[analyze.c#compute_scalar_stats-width](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2420-L2426),
[Validation probes](#validation-probes).

### Custom operator classes

An opclass whose support function 4 is not one of the two internal functions
lands in `unknown` and receives no deduplication credit, even when the engine
calls it and deduplicates. The measured cost was `-214.9 %` on an index the
engine packed to 1400 kB, `-226.0 %` on the 2026-09-09 rebuild of the same shape
and `-226.4 %` on the gate group's `i_ei_true`. Executing the function from SQL
would decide the case — `OidFunctionCall1Coll` is what the engine does — but a
catalog-only statement must not call an arbitrary user function, so the state
stays three-valued. The harness oracle is the metapage flag, which only a
rebuilt index carries.

An impostor is the same case: an operator class registering a non-internal
function under a built-in's name also lands in `unknown`, which is the
conservative answer the 2026-08-18 contract asks for. Constructing that fixture
needs care, because `FUNCTION 4 btequalimage(oid)` in `CREATE OPERATOR CLASS`
resolves through `pg_catalog` and finds the real function; only a
schema-qualified `public.btequalimage(oid)` registers the impostor.
[nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183),
[btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921),
[The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement).

### Expression width heuristic

Subtracting three bytes from an index expression's recorded width assumes the
expression returns a freshly built four-byte-header datum, which is what
`lower()`, `upper()` and `md5()` do. An expression that returns an already-short
datum unchanged would be under-counted by up to three bytes per attribute. The
run measured only the four-byte-header case, where the correction is exact.
[heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L234-L242),
[varatt.h#VARATT_CAN_MAKE_SHORT](../../../../raw/postgres-17/src/include/varatt.h#L257-L262).

### Untested configurations

Nothing here was measured at a non-default `BLCKSZ`, at a `MAXIMUM_ALIGNOF` other
than 8, or on a big-endian machine, and parallel index builds, `CREATE INDEX
CONCURRENTLY`, `REINDEX CONCURRENTLY`, partitioned tables and a non-C cluster
locale were outside both runs. Every number on this page is a 17.11 build on
x86-64 Linux with `--locale=C`, at `max_data_alignment` 8 and
`database_block_size` 8192; the 2026-09-09 pass recorded those two values from
`pg_control_init()` rather than assuming them. The nondeterministic-collation
gap is closed: that pass built with ICU and measured the `collisdeterministic`
branch on both sides, in a UTF8 database, under
[The collation branch, measured with ICU](#the-collation-branch-measured-with-icu).
[index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849),
[pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575).

### Multicolumn key groups without extended statistics

**A duplicate-heavy multicolumn index whose key columns are correlated is the
statement's largest unflagged error, and the 2026-09-09 pass measured it.** For
a key of more than one column the model has no per-class split, so it takes one
class of `live_rows` rows over `groups_est` groups, and `groups_est` is the
product of the per-column distinct counts clamped to the row count. On two
500,000-row fresh builds with 5,000 distinct values in each key column, that
product saturates the clamp, `tids_per_tuple` comes back 1.0, and the model
prices 500,000 separate tuples against an index the engine deduplicated to 459
blocks: `-320.0 %` for `(int4, int8)` and `-562.1 %` for `(int4, text)`, under
the current text and the superseded one alike.

The fix already exists in the statement and is not automatic. Adding
`CREATE STATISTICS ... (ndistinct)` on the key columns and analysing makes the
`extstat` stage read the whole-key entry — 4,993 against a true 5,000 — which
takes the same index from `-320.0 %` to `0.0 %` and raises the caveat
`key groups from extended statistics`. Until then, read a large negative
reading on a multicolumn index as a missing statistics object rather than as
index bloat, and note that no caveat says so. The alternative would be a
correlation-aware group estimate from per-column statistics, which no catalog
supports.
[The current recommended statement](#the-current-recommended-statement),
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309),
[mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385),
[The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement).

### Partial-index populations and zero counts

The population probe validates whether a subset is empty, but not the widths,
NULL pattern or expression results over that subset, and an empty sample remains
inconclusive about a subset that acquired rows after the last ANALYZE. The four
partial-index suppression conditions still drop rows from the report rather than
reporting them with a caveat, so a partial index with real savings can still be
invisible. Neither gap is closed.
[analyze.c#sample-membership](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975),
[autovacuum.c#analyze-threshold](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3095).

### Statistics publication and test ordering

The barrier is now used by every fixture, and the artifact it prevents is
measured, including the timing precondition the 2026-09-09 pass added: the load
and the ANALYZE must fall inside one `PGSTAT_MIN_INTERVAL` of 1000 ms, or an
unforced flush lands between them and the artifact never appears. What is not
established is how a production reader should order itself against an unknown
writer: `pg_stat_force_next_flush()` forces the *calling* backend's pending
statistics, not another session's, so a report taken while other backends hold
unflushed deltas can still mix an absolute ANALYZE write with a later additive
flush. The observer's own snapshot needs `pg_stat_clear_snapshot()` only inside
a transaction block.
[pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708),
[pgstat.c#pgstat_clear_snapshot](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L786-L800),
[guc_tables.c#stats_fetch_consistency](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974).

### Alert thresholds and rebuild savings

The calibration covers seven patterns on two key widths at one row count. It does
not cover mixed workloads, indexes under concurrent write load, bottom-up deletion
reclaiming space between the reading and the rebuild, or whether a saving persists
after the workload resumes. The `floor` column is now known to be unusable as a
conservative bound on a deduplicating index (`-245.2 %` against a true `-7.5 %`),
and no replacement lower bound has been derived.
[Calibration by insertion pattern](#calibration-by-insertion-pattern),
[nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335).

### The mandatory suite and the current statement

The asker's 2026-08-18 contract says a statement that fails a mandatory test is
corrected, not reported. Half the suite has now been put to the current text:
tests 1-17 were scored on 2026-09-09 and passed, with two under-credits rather
than the one the pass criteria allowed. Tests 18-91 and fixtures 92-121 were
last scored against the superseded six-change text on 2026-08-24, so the
partial-index contract remains unenforced for the current text. The expected
outcomes are derived under
[Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement)
and the procedure is under
[How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement);
until it runs, two fixtures are expected to remain critical false positives on
the report (`i103`, and `x109` unless the reading rule learns the
`statistics target zero on an index column` caveat), and the deduplication
verdicts of tests 13 through 16 carry a caveat they did not carry before.
[Mandatory test review](#mandatory-test-review).

### Scoring column for the partial-index contract

The asker chose `wasted_space_pct_floor` as the verdict column on 2026-08-19
because that column has no duplication term, and the first run confirmed the
choice: nine point-estimate false positives did not reach the floor. The
calibration on this page then measured the floor at `-245.2 %` against a true
`-7.5 %` on a duplicate-heavy index, and the reading guidance now names
`wasted_space_pct` with `equalimage` and `caveats`. The two rules disagree on
any index that deduplicates. The protocol records both columns and both
verdicts; which one carries the contract is the asker's decision, and the
pass criteria above are written for whichever column is chosen.
[Calibration by insertion pattern](#calibration-by-insertion-pattern),
[Reading the output](#reading-the-output).

### Cross-version execution of the revised statement

The current text was written and measured on 17.11 only. Its `extstat` stage
selects `pg_stats_ext` rows with `inherited = false`, and `inherited` is a
column this version's view defines from `stxdinherit`. Test 17 and the
"12 through 17" contract of the earlier statement require the text to execute
on the pinned 12.2 server; nothing on this page shows that it does, and the v12
companion page has not been updated for the revised text. The first result of
the 12.2 leg is therefore the parse outcome itself, and a text that does not
execute there fails test 17 before any fixture is built.
[system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290),
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309),
[What remains unimplemented](#what-remains-unimplemented).

### Integer-truncated widths across an alignment boundary

`stawidth` is an `int32`, and for a variable-width column `ANALYZE` assigns it
`total_width / nonnull_cnt`, so a column whose values average 4.9996 bytes
records 4. On the `(int4, numeric)` fixture `i_multi_bad` the model priced a
16-byte tuple where the build stores 24, and the reading was 28.8 % on 17.11 and
12.2 alike, with the gate closed on both. The current text still reads
`avg_width` for a variable-width key, so the error is expected to survive the
rerun. A remedy would need a fractional width that no catalog holds, or a
sampled `pg_column_size`, which is not a catalog read.
[analyze.c#stawidth](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540),
[pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L41-L50),
[Page and posting geometry](#page-and-posting-geometry).

### Fixture recipes that do not reproduce

Three fixture families are described on this page without the row counts their
numbers depend on, so the 2026-09-09 pass could reproduce their mechanisms but
not their magnitudes: the inheritance parent (`-461.0 %` measured against a
filed `-550.8 %`, from `avg_width` 21 and 66 against 21 and 77), the
true-returning custom-opclass index (`-226.0 %` on 1352 kB against `-214.9 %` on
1400 kB), and the six further fresh shapes, whose INCLUDE, partial and
NULL-heavy readings came out `0.0`, `+0.5` and `+0.1` against a filed `-0.1`,
`+0.4` and `0.0`, with the NULL-heavy floor at `-20.8 %` against `-29.7 %`. Each
gap is a missing recipe, not a contradicted claim: every one of these is
sampling-dependent through `pg_stats`, and the direction and mechanism
reproduced in all three. The recipes recorded under
[Reproducing the measurements](#reproducing-the-measurements) close this for
every fixture built on or after 2026-09-09; the earlier three would have to be
re-derived.

The platform question this section replaced is settled: the host reports
`Linux x86_64`, `max_data_alignment` 8 and `database_block_size` 8192, which
matches this page's "x86-64 Linux" statement. See
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server).
[pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997),
[pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L204),
[analyze.c#stawidth](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540).

## Source References

- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L784-L855)
- [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1349)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L371)
- [nbtsort.c#soft-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854)
- [nbtsort.c#_bt_sortaddtup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L713-L735)
- [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L262)
- [guc_tables.c#statement_timeout-and-lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631)
- [pg_proc.dat#size-functions](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7507)
- [pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371)
- [dbsize.c#pg_size_pretty-sign](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L569-L600)
- [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L975)
- [system_views.sql#pg_stats-visibility](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L268-L275)
- [indexfsm.c#index-FSM](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L65)
- [pg_index.h#pg_index](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L26-L62)
- [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L66)
- [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)
- [pg_amproc.dat#text_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L205-L212)
- [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309)
- [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385)
- [analyze.c#expression-attributes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L448-L478)
- [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L249-L259)
- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671)
- [nbtsort.c#posting-size-limit](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1284-L1308)
- [nbtsort.c#group-boundaries](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1310-L1349)
- [nbtdedup.c#_bt_dedup_save_htid-cap](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L510-L513)
- [nbtdedup.c#_bt_form_posting-size](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L879-L884)
- [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1062-L1128)
- [nbtsort.c#_bt_slideleft](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L682-L700)
- [nbtsort.c#BTPageState](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L229-L252)
- [index.c#ambuild-call](../../../../raw/postgres-17/src/backend/catalog/index.c#L3048-L3053)
- [nbtree.c#ambuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L128-L129)
- [nbtsort.c#btbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L290-L328)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571)
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
- [pg_locale.c#pg_locale_deterministic](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1567-L1575)
- [fmgr.c#internal-function-resolution](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)
- [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84)
- [nbtsort.c:1126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1126)
- [analyze.c#compute_scalar_stats-width](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2420-L2426)
- [analyze.c#stawidth](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540)
- [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L234-L242)
- [varatt.h#VARATT_CAN_MAKE_SHORT](../../../../raw/postgres-17/src/include/varatt.h#L257-L262)
- [indextuple.c#index_form_tuple-compression](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L116-L138)
- [heaptoast.h#TOAST_INDEX_TARGET](../../../../raw/postgres-17/src/include/access/heaptoast.h#L63-L68)
- [htup_details.h#MaxHeapTupleSize](../../../../raw/postgres-17/src/include/access/htup_details.h#L553-L563)
- [pg_type.h#TYPSTORAGE_EXTENDED](../../../../raw/postgres-17/src/include/catalog/pg_type.h#L307-L310)
- [index.c#ConstructTupleDescriptor-attstorage](../../../../raw/postgres-17/src/backend/catalog/index.c#L353-L360)
- [analyze.c#update_attstats-stainherit](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1647)
- [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L211)
- [acl.c#column_privilege_check](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L2538-L2569)
- [acl.c#acldefault](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L813-L821)
- [aclchk.c#pg_class_aclmask_ext](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L3396-L3412)
- [aclchk.c#grant-refuses-indexes](../../../../raw/postgres-17/src/backend/catalog/aclchk.c#L1858-L1863)
- [pg_attribute.h#attstattarget](../../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L168-L176)
- [analyze.c#examine_attribute](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1019-L1030)
- [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629)
- [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L907-L923)
- [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L214)
- [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1147)
- [nbtsort.c#page-boundary](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L874-L935)
- [nbtree.h#fillfactors](../../../../raw/postgres-17/src/include/access/nbtree.h#L199-L202)
- [nbtsort.c#maxpostingsize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1304-L1305)
- [nbtdedup.c#_bt_dedup_start_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L432-L474)
- [analyze.c#index-relstats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703)
- [nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1347-L1362)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L870-L920)
- [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3099)
- [analyze.c#index-statistics-write](../../../../raw/postgres-17/src/backend/commands/analyze.c#L588-L602)
- [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L845-L884)
- [execnodes.h#IndexInfo](../../../../raw/postgres-17/src/include/nodes/execnodes.h#L183-L193)
- [autovacuum.c#analyze-reloptions](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017)
- [autovacuum.c#analyze-threshold](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3095)
- [pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-17/src/include/pgstat.h#L399-L429)
- [pg_proc.dat#pg_get_expr](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8219-L8220)
- [ruleutils.c#pg_get_expr_ext](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L2648-L2662)
- [nodeSubplan.c#ExecScanSubPlan-EXISTS](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L293-L296)
- [indxpath.c#check_index_predicates](../../../../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3244-L3350)
- [nbtsort.c#_bt_load-grouping](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1314-L1316)
- [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4875-L4905)
- [datum.c:271](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L271)
- [pg_class.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L21-L22)
- [pg_index.h#generated-header](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L21-L22)
- [catalog/Makefile#genbki](../../../../raw/postgres-17/src/include/catalog/Makefile#L132-L143)
- [utils/Makefile#Gen_fmgrtab](../../../../raw/postgres-17/src/backend/utils/Makefile#L47-L53)
- [btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L108-L194)
- [pageinspect--1.8--1.9.sql#bt_page_stats](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L87-L124)
- [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31)
- [btree_index.sql#deduplication-tests](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L186-L213)
- [index_including.sql#statistics-tests](../../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L150-L158)
- [acl.c#aclmask](../../../../raw/postgres-17/src/backend/utils/adt/acl.c#L1388-L1445)
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
- [pg_amproc.dat#bpchar_ops-equalimage](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L31-L33)
- [index.c#pattern-ops-collation-check](../../../../raw/postgres-17/src/backend/catalog/index.c#L826-L849)
- [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921)
- [numeric.c#bigint-out-of-range](../../../../raw/postgres-17/src/backend/utils/adt/numeric.c#L4546-L4549)
- [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432)
- [installation.sgml#meson-setup](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L2012-L2025)
- [nbtsplitloc.c#split-policy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L278-L335)
- [nbtsplitloc.c#single-value-strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416)
- [nbtsplitloc.c#_bt_strategy-single-value](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L1020-L1033)
- [regress.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59)
- [regress.sgml#contrib-suites](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195)
- [installation.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L515-L522)
- [installation.sgml#ICU-default](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L170)
- [installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193)
- [installation.sgml#--without-icu](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1209-L1216)
- [installation.sgml#--with-blocksize](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1472-L1482)
- [installation.sgml#--enable-debug](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1530-L1540)
- [initdb.sgml#--locale](../../../../raw/postgres-17/doc/src/sgml/ref/initdb.sgml#L281-L291)
- [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1453)
- [guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1097-L1100)
- [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262-L2265)
- [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2469)
- [guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3410-L3413)
- [guc_tables.c#client_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4777-L4780)
- [guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2074)
- [nbtutils.c#_bt_allequalimage-INCLUDE](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147)
- [nbtutils.c#_bt_allequalimage-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180)
- [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)
- [execIndexing.c#partial-predicate-skip](../../../../raw/postgres-17/src/backend/executor/execIndexing.c#L384-L386)
- [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3951-L3952)
- [index.c#index_update_stats-sentinel](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2836)
- [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894)
- [system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290)
- [pageinspect--1.8--1.9.sql#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L73-L82)
- [pageinspect--1.8--1.9.sql#bt_page_items](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L109-L118)
- [amcheck--1.0--1.1.sql#bt_index_check](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.0--1.1.sql#L12-L28)
- [verify_nbtree.c#metapage-equalimage-check](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L380-L396)
- [gram.y#IndexStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L8093-L8095)
- [nbtsort.c#btbuild-parallel](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L389-L392)
- [index.c#index_concurrently_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L1533-L1539)
- [indexcmds.c#DefineIndex-concurrent-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1682)
- [indexcmds.c#ReindexRelationConcurrently-build](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4009)
- [pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173)
- [vacuum.c#expand_vacuum_rel-partitions](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L963-L982)
- [pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L41-L50)
- [pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997)
- [pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L204)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [versions](../../../versions.md)
