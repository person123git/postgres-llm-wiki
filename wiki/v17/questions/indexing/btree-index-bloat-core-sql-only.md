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
  - [The portable extended-statistics filter](#the-portable-extended-statistics-filter)
  - [Reproducing the measurements](#reproducing-the-measurements)
  - [What remains unimplemented](#what-remains-unimplemented)
  - [Mandatory test review](#mandatory-test-review)
  - [Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement)
  - [What still needs to be tested](#what-still-needs-to-be-tested)
- [Measurement Script](#measurement-script)
  - [How to use the suite scripts](#how-to-use-the-suite-scripts)
  - [How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement)
  - [The two suite scripts, and the rules they follow](#the-two-suite-scripts-and-the-rules-they-follow)
  - [The PostgreSQL 17 suite script](#the-postgresql-17-suite-script)
  - [The PostgreSQL 12 leg script](#the-postgresql-12-leg-script)
  - [Reading the results of a run](#reading-the-results-of-a-run)
  - [What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09)
  - [What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured)
  - [Measurement-script section review](#measurement-script-section-review)
  - [The stop stage, repaired](#the-stop-stage-repaired)
  - [What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64)
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
  - [Fixture verdicts that depend on the ANALYZE sample](#fixture-verdicts-that-depend-on-the-analyze-sample)
  - [Statistics publication and test ordering](#statistics-publication-and-test-ordering)
  - [Alert thresholds and rebuild savings](#alert-thresholds-and-rebuild-savings)
  - [The mandatory suite and the current statement](#the-mandatory-suite-and-the-current-statement)
  - [Scoring column for the partial-index contract](#scoring-column-for-the-partial-index-contract)
  - [Cross-version execution of the revised statement](#cross-version-execution-of-the-revised-statement)
  - [Integer-truncated widths across an alignment boundary](#integer-truncated-widths-across-an-alignment-boundary)
  - [Fixture recipes that do not reproduce](#fixture-recipes-that-do-not-reproduce)
  - [Attribution row counts differ between hosts](#attribution-row-counts-differ-between-hosts)
  - [Fixture statements are marked disposable, not tagged](#fixture-statements-are-marked-disposable-not-tagged)
  - [The 12 leg's settings have no citable apply scope here](#the-12-legs-settings-have-no-citable-apply-scope-here)
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

Sixth prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, for the question "Testing the PostgreSQL 12
> Core-SQL B-Tree Bloat Method on PostgreSQL 17", add a section with two
> scripts, one for version 17 and one for version 12, holding all the tests, so
> that the section can be reused during reviews and improvements of the
> statement. Use Bash and SQL only.

The original read `follow agents.md, in postgresql 17,  for question: Testing
the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified) ,
add to the question a section with two scripts for version 17 and version 12
with all the tests so it can be reused during reviews and improvements of the
statement. use bash and sql only.`: `agents.md` for AGENTS.md, lowercase
`postgresql`, a double space after the first comma, `for question:` without an
article, a space before a comma, the `(unverified)` title hint treated as part
of the title, an unclear "it" for the section, and `bash and sql` for Bash and
SQL. The asker chose to build and run both legs, to cover every test family the
page names, and to keep the scripts in this page rather than in a second file.
The work is filed under
[The two suite scripts, and the rules they follow](#the-two-suite-scripts-and-the-rules-they-follow),
[The PostgreSQL 17 suite script](#the-postgresql-17-suite-script),
[The PostgreSQL 12 leg script](#the-postgresql-12-leg-script) and
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09).

Seventh prompt, corrected and restated with the asker's agreement:

> Also add to the section information on how to understand the results of
> running the scripts.

The original read `add to section also information on how to understand the
results of the script execution.`: no article before `section`, no sentence
capitalisation, and `also` placed mid-sentence. The asker chose a reader's
guide rather than a per-stage reference with example output. It is filed under
[Reading the results of a run](#reading-the-results-of-a-run).

Eighth prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, for the question "Testing the PostgreSQL 12
> Core-SQL B-Tree Bloat Method on PostgreSQL 17", review the measurement script
> section.

The original read `follow agents.md, in postgresql 17,  for question: Testing
the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified) ,
review the measurement script section`: `agents.md` for AGENTS.md, lowercase
`postgresql`, a double space after the first comma, `for question:` without an
article, a space before the comma after the title, the `(unverified)` title
hint treated as part of the title, and no sentence capitalisation or terminal
period. The asker chose a **read-only audit** with no server run, a
**restructure of the page to comply** with `MANDATORY Measurement Script`, and
**repair in place** for what the audit found. The review is filed under
[Measurement Script](#measurement-script): the usage information under
[How to use the suite scripts](#how-to-use-the-suite-scripts) and the findings
under
[Measurement-script section review](#measurement-script-section-review).

Ninth prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, review the question "Testing the
> PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17", and propose a
> fix for the failed cross-version execution test: the exact filed statement is
> refused on the pinned 12.2 server with
> `ERROR: column se.inherited does not exist` at
> `LINE 116: AND se.inherited = false`.

The original read `follow agents.md, in postgresql 17,  review question: Testing
the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified) ,
propose a fix for :  Not a pass — PostgreSQL 12.2 / The cross-version execution
test fails outright / The exact filed statement is refused on the pinned 12.2
server:`: `agents.md` for AGENTS.md, lowercase `postgresql`, a double space
after the first comma and after `for :`, a space before the comma after the
title and before that colon, `review question:` without an article, the
`(unverified)` title hint treated as part of the title, the quoted review
pasted as three fragments, and no sentence capitalisation or terminal period.
The asker chose the single-portable-text fix over keeping the filed text beside
a documented 12-only variant, and chose to implement it and re-run the affected
stages rather than only propose it. The work is filed under
[The portable extended-statistics filter](#the-portable-extended-statistics-filter)
and
[What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured).

Tenth prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, review the question "Testing the
> PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17".

The original read `follow agents.md, in postgresql 17, review  question: #
Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
(unverified)`: `agents.md` for AGENTS.md, lowercase `postgresql`, a double
space after `review`, `review question:` without an article, a stray `#`
before the title, the `(unverified)` title hint treated as part of the title,
and no sentence capitalisation or terminal period. The asker chose a
**targeted script run** over a claim-by-claim citation pass: fix the known
script defects, re-run the stages the 2026-09-10 pass had not re-run under the
repaired script text, build and check the 12 leg so that its full runtime is
measured, and **repair in place**. The work is filed under
[The stop stage, repaired](#the-stop-stage-repaired) and
[What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64).

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

**The whole suite is now two runnable scripts, and both legs have been run.**
[The PostgreSQL 17 suite script](#the-postgresql-17-suite-script) and
[The PostgreSQL 12 leg script](#the-postgresql-12-leg-script) are Bash and SQL
only, extract the statement texts from this page, and reproduce every family
above plus the partial-index half of the mandatory suite that had never been
scored against this text. On 112 numbered fixtures the current text reports 59
rows and withholds 53, every withheld row naming the term that withheld it, and
**three reported critical false positives survive this page's own reading rule**:
a forged stale partial `reltuples`, the wide-key partial index `i103` at 84.1 %,
and the zero-statistics-target index `x109` at 62.5 %.
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09).

**The cross-version question was answered no on 2026-09-09, and the fix filed
on 2026-09-10 makes it yes.** The text filed until then was refused by the
pinned 12 server with `ERROR: column se.inherited does not exist`, because that
`extstat` filter names a `pg_stats_ext` column the server does not have. The
filed text now reads the same flag through `row_to_json(se) ->> 'inherited'`,
which parses whether or not the column exists, and **the exact filed text is
now accepted unmodified on 12.2, with the 12 leg's transformer reporting
`transform_edits=0`**. On 17.11 the change is inert: `EXCEPT` in both
directions over every column both texts project returns 0 rows on all six
fixture databases, 186 rows in total.
[The portable extended-statistics filter](#the-portable-extended-statistics-filter),
[system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290),
[pg_proc.dat#row_to_json](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8975-L8977),
[jsonfuncs.c#json_object_field_text](../../../../raw/postgres-17/src/backend/utils/adt/jsonfuncs.c#L881-L895).

**The two scripts now sit in a top-level [Measurement Script](#measurement-script)
section with the usage information the rule requires, and a read-only audit of
that section fixed thirteen defects in it.** The audit changed no measured
number and started no server; it re-derived all five SHA-256 baselines from this
page in pure Bash, parsed both scripts, and re-read the 37 source citations in
the section. The material findings: an `awk` and `shasum` recipe that the
Bash-and-SQL-only rule forbids and the scripts had already replaced, a `cost`
stage missing from the 17 script's own stage list, a `KEEP` variable documented
but never read, three cluster settings written without their apply scopes named,
`psql` calls whose errors could not raise the exit status, an unguarded
`rm -rf` on an environment variable, and fixture blocks that were never marked
disposable. The repaired scripts were re-run in part that day and in full the
next.
[Measurement-script section review](#measurement-script-section-review),
[startup.c#single-query-action](../../../../raw/postgres-17/src/bin/psql/startup.c#L377-L386),
[mainloop.c#die_on_error](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L587-L594).

**Both scripts were then run end to end from an empty sandbox on a second
platform, Darwin arm64, and every verdict count of the Linux run reproduced.**
That pass first repaired the one defect the scripts still carried, a `stop`
stage that used `pg_ctl -m immediate`, which skips the shutdown checkpoint and
forces crash recovery on the next start; both legs now stop with `-m fast -w`
and confirm the teardown before `clean` deletes anything. On this host the 17
leg's full run takes 2 min 26 s and the 12 leg's 1 min 52 s, so the 12 leg's
full-run runtime is recorded at last; and the one number that would not
reproduce is the attribution row count, 31 rows in each direction against the
filed 26.
[The stop stage, repaired](#the-stop-stage-repaired),
[What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64),
[postmaster.c#process_pm_shutdown_request-immediate](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2307-L2342),
[xlogrecovery.c#not-properly-shut-down](../../../../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L922-L949).

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
    -- The inherited flag is read through row_to_json() instead of being named
    -- as a column, so one text also parses where pg_stats_ext has no inherited
    -- column: there the key is absent, ->> returns NULL, and the coalesce
    -- admits the row, which is the only ANALYZE pass such a server records.
    SELECT k.idxoid, max(e.nd) AS ext_ndistinct
      FROM keyatts k
      JOIN idx i           ON i.idxoid = k.idxoid
      JOIN pg_stats_ext se ON se.schemaname = i.schemaname
                          AND se.tablename = i.tablename
                          AND coalesce((row_to_json(se) ->> 'inherited')::boolean,
                                       false) = false
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
| `keyatts`, `extstat` | Look for a whole-key distinct-count entry from the non-inherited pass, identified through `row_to_json(se) ->> 'inherited'` rather than by naming the column; use the maximum matching visible estimate. | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309), [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L355-L385), [The portable extended-statistics filter](#the-portable-extended-statistics-filter). |
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

Both `pg_stats` joins and the `pg_stats_ext` join keep to the non-inherited
pass. The `pg_stats` joins say `inherited = false`; the `pg_stats_ext` join
reads the same flag as `coalesce((row_to_json(se) ->> 'inherited')::boolean,
false) = false`, so that one text parses on a server whose view has no such
column, and there admits the single pass it does record. See
[The portable extended-statistics filter](#the-portable-extended-statistics-filter).
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
| 6 | `inherited = false` on both `pg_stats` joins and on `pg_stats_ext` | implemented; since 2026-09-10 the `pg_stats_ext` half reads the flag through `row_to_json()` so one text runs on both majors |
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

### The portable extended-statistics filter

**One line changed, and the statement now runs unmodified on both majors.** The
`extstat` CTE used to name a column:

```text
AND se.inherited = false
```

It now reads the same flag out of the row instead:

```text
AND coalesce((row_to_json(se) ->> 'inherited')::boolean,
             false) = false
```

Nothing else about the statement moved, and the page proves that rather than
asserting it: the `extstat` stage rebuilds the previous text from the filed one
by undoing exactly this edit, and the reconstruction hashes to
`8acd531b7bcd2f2ca679e65024d83bd61debcb4b75bb18f3834a368454d574fd`, the SHA-256
the previous text was filed under.
[The PostgreSQL 17 suite script](#the-postgresql-17-suite-script).

#### Why naming the column cannot work in one text

A column reference is resolved when the statement is parsed, so no runtime
guard can protect it. `current_setting('server_version_num')`, a `CASE`, or an
`OR` that is never reached all fail the same way: the server rejects the
statement before it evaluates anything. The refusal this page recorded on
2026-09-09 arrives with no row read and no fixture built.

On this version the column exists, and `pg_stats_ext` derives it from the data
row's own key:

| Piece | Where |
|---|---|
| `sd.stxdinherit AS inherited` in the view | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309), [system_views.sql:290](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290) |
| the catalog column, and its place in the data row's unique key | [pg_statistic_ext_data.h:35](../../../../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L35), [pg_statistic_ext_data.h:57](../../../../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L57) |
| `ANALYZE` runs the plain pass, then a second pass when the table has children | [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L246-L259) |
| each pass builds extended statistics under its own `inh` flag, stored as `stxdinherit` | [analyze.c#BuildRelationExtStatistics-call](../../../../raw/postgres-17/src/backend/commands/analyze.c#L604-L606), [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L111-L114), [extended_stats.c#statext_store](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L790-L791) |

Reading the base catalogs instead is not an option for this statement: an
unprivileged reader may query `pg_statistic_ext` but not
`pg_statistic_ext_data`, which the view exists to mediate.
[system_views.sql#pg_statistic_ext_data-revoke](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L382-L383).

#### Why the replacement is exact where the column exists

`row_to_json` takes a `record` and emits one JSON key per non-dropped
attribute, named by `attname`; a boolean attribute is written as bare `true` or
`false`. `->>` is `json_object_field_text`, which returns `NULL` — not an error
— when the key is absent, and `'true'`/`'false'` cast back to boolean because
`boolin` accepts exactly those spellings.

| Step | Evidence |
|---|---|
| `row_to_json(record)` returns `json`, volatility stable | [pg_proc.dat#row_to_json](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8975-L8977) |
| one key per attribute, `attisdropped` skipped, key from `NameStr(att->attname)` | [json.c#composite_to_json](../../../../raw/postgres-17/src/backend/utils/adt/json.c#L546-L579) |
| a boolean datum is written as `true` or `false` | [json.c#datum_to_json-bool](../../../../raw/postgres-17/src/backend/utils/adt/json.c#L212-L221) |
| `->>` on `json` is `json_object_field_text` | [pg_operator.dat#json-arrow-text](../../../../raw/postgres-17/src/include/catalog/pg_operator.dat#L3160-L3162), [pg_proc.dat#json_object_field_text](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L9078-L9081) |
| a missing key yields SQL NULL | [jsonfuncs.c#json_object_field_text](../../../../raw/postgres-17/src/backend/utils/adt/jsonfuncs.c#L881-L895) |
| `'true'` and `'false'` parse to boolean | [bool.c#parse_bool_with_len](../../../../raw/postgres-17/src/backend/utils/adt/bool.c#L36-L58), [bool.c#boolin](../../../../raw/postgres-17/src/backend/utils/adt/bool.c#L126-L150) |

So where the column exists the predicate is the same predicate, and where it
does not the `coalesce` admits the row — which is the correct reading on a
server that records one `ANALYZE` pass, because that pass is the plain one. The
first half of that sentence is measured below; the second half is measured on
the 12 leg, whose server reports thirteen `pg_stats_ext` columns and none named
`inherited`.
[What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured).

#### What the old transformer cost, measured

The 12 leg used to make the text run by deleting the filter line. That widened
the CTE instead of preserving it, which this page carried as an open question
until now. The `extstat` stage settles it by scoring three texts — the filed
one, the previous one, and the widened one — on four fixtures in one database,
against a measured `REINDEX INDEX`.

The mechanism first. Because `extstat` takes `max(e.nd)`, admitting the
inherited row can only raise the whole-key distinct estimate, so the widened
text can only over-state the number of key groups, under-state deduplication,
and therefore under-report bloat. The two passes the fixtures store, whole-key
`n_distinct` from each:

| Fixture | Own pass | Inherited pass |
|---|---|---|
| `xpar`, low-cardinality child | 20 | 3,498 |
| `xpar2`, the same with 60 % of the parent deleted and vacuumed | 8 | 1,404 |
| `xpar3`, high-cardinality child | 20 | 28,610 |
| `xflat`, no children | 20 | none written |

The parent's own index contains only the parent's own rows, so the own pass is
the correct input in every row of that table. Scored:

| Fixture | Measured `REINDEX` | Filed text | Previous text | Widened text | `key_groups`, filed vs widened |
|---|---|---|---|---|---|
| `xpar_ab` | 0.0 | 0.8 | 0.8 | **2.3** | 20 vs 3,498 |
| `xpar2_ab` | 59.7 | 59.7 | 59.7 | **60.1** | 8 vs 1,404 |
| `xpar3_ab` | 0.0 | 0.8 | 0.8 | **-33.7** | 20 vs 28,610 |
| `xflat_ab` | 0.0 | 0.8 | 0.8 | 0.8 | 20 vs 20 |

Three findings, and the third is the reason this section reports a range rather
than a headline:

1. **The filed text and the previous text agree on every row of every fixture**,
   here and in the equivalence run below.
2. **The widened text differs on every inheritance parent and on none of the
   controls**, always in the under-reporting direction.
3. **The size of that error depends on the shape, not on the size of the
   `key_groups` error.** A 175x wrong group count moves the reading by 1.5
   points on `xpar_ab`, because a posting list's TID payload dominates the
   index either way; the same fixture with a high-cardinality child moves it
   34.5 points, because there the widened estimate crosses the boundary at
   which the model stops crediting deduplication at all and prices singleton
   tuples. The inherited estimate is also sample-dependent — three runs of the
   same `xpar` fixture read 3,500, 3,481 and 3,498 — so the widened text's
   error is not reproducible to the decimal, while the filed text's 0.8 is.
   [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894).

The `wspf` floor column reads `-219.8` on all three fresh fixtures under all
three texts. That is the known floor behaviour on a deduplicating index, not
an effect of this change; see
[Scoring column for the partial-index contract](#scoring-column-for-the-partial-index-contract).

#### What it costs to run

`row_to_json(se)` serialises a whole `pg_stats_ext` row, including the
`most_common_vals` arrays the view's lateral already builds, once per candidate
join. Six interleaved runs of the two exact texts on the `suite` database, 325
B-tree indexes and three extended-statistics objects, measured a mean of
95.3 ms for the filed text against 91.6 ms for the previous one, with the
ranges overlapping (75.4-111.0 against 84.9-100.0) and the single fastest run
of the twelve belonging to the filed text. On this database the difference is
inside the noise; a database with many large `pg_mcv_list` objects would be the
place to re-measure it.
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L301-L307).

#### What the fix does not change

The two `pg_stats` joins still name `inherited` directly, and deliberately:
`pg_stats` projects it from `pg_statistic.stainherit` on this version, the 12
leg executes those joins as filed, and the refusal was only ever raised against
`pg_stats_ext`. Changing them would add cost and risk for nothing measured. The
statement tag also stays `wiki_btree_wasted_space_sweep_r2`, because the model
is unchanged and the two texts are measurably identical here; what identifies
the new text is its SHA-256 baseline, `646df923…`, which both scripts check.
[system_views.sql#pg_stats-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L194).

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
- A block size other than 8192. Every number here is 17.11 at the default block
  size. The cross-version gap is closed:
  [What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09)
  records the 12.2 leg, including the one construct that stops the text from
  executing there.
- Row-count fidelity of the reconstructed fixtures. Tests 18-91 and fixtures
  92-121 have now been scored against this statement, but from shapes rebuilt
  out of the published requirement tables rather than the original scripts, so
  a cell whose value depends on the exact population differs; test 36 is the
  clearest case. See
  [What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09).
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
4. **The 12.2 leg of test 17.** Done, twice over. The text filed until
   2026-09-10 named `pg_stats_ext.inherited`, and the 12.2 server refused it;
   the filed text now reads that flag through `row_to_json()` and **executes
   there unmodified**, with the 12 leg's transformer reporting zero edits. The
   leg still starts by executing the exact text and recording the outcome
   before any fixture is built, and its `extstat` stage now also re-runs the
   refused text to keep the refusal reproducible. The v12 page carries that
   version's build and publication notes.
   [system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290),
   [The portable extended-statistics filter](#the-portable-extended-statistics-filter),
   [What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured),
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

## Measurement Script

Two scripts produce the numbers this page reports, one per version leg:
`btree_bloat_suite_v17.sh` for 17.11 and `btree_bloat_suite_v12.sh` for the
12.2 cross-version leg. Both are filed in full below, in Bash and SQL only, and
both were run end to end on 2026-09-09 on Linux x86_64 and again, from an
empty sandbox and under the script text now filed, on 2026-09-10 on Darwin
arm64.

- The 17 leg backs
  [What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09)
  in full, and re-measures every family the earlier runs reported: page
  geometry, fresh sorted builds, the deduplication gate, the acceptance
  fixtures, the insertion-pattern calibration, the 112-fixture numbered suite,
  the `EXCEPT` attribution, the validation probes and the cost comparison.
- The 12 leg backs the cross-version half of the same section and
  [Cross-version execution of the revised statement](#cross-version-execution-of-the-revised-statement).
- The numbers in
  [Measured acceptance results](#measured-acceptance-results),
  [Calibration by insertion pattern](#calibration-by-insertion-pattern),
  [Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server),
  [The deduplication gate, scored against the current statement](#the-deduplication-gate-scored-against-the-current-statement)
  and [The collation branch, measured with ICU](#the-collation-branch-measured-with-icu)
  were measured on 2026-09-08 and 2026-09-09 by the predecessor harnesses these
  two scripts replace. The scripts cover the same families and reproduced those
  figures on 2026-09-09; a re-measurement of any of them re-runs these scripts
  rather than rebuilding a harness.

### How to use the suite scripts

| Item | The 17 leg, `btree_bloat_suite_v17.sh` | The 12 leg, `btree_bloat_suite_v12.sh` |
|---|---|---|
| Purpose | measures the current estimator on a 17.11 server built from this page's pin: page geometry, fresh builds, the deduplication gate, the acceptance fixtures, the calibration patterns, the 112 numbered fixtures, attribution, probes, the scoring pass against a measured `REINDEX INDEX`, and statement cost | answers whether the exact filed text executes on the pinned 12.2 checkout, records the refusal verbatim, then transforms, fixtures and scores the constructible subset |
| Invocation | `bash btree_bloat_suite_v17.sh [stage ...]`, run from the repository root | `bash btree_bloat_suite_v12.sh [stage ...]`, run from the repository root |
| Stages | 17 stages plus `stop` and `clean`; see [the 17 leg's stages](#the-17-legs-stages) | 10 stages plus `stop` and `clean`; see [the 12 leg's stages](#the-12-legs-stages) |
| Environment | 7 variables, all with defaults; see [what the scripts read from the environment](#what-the-scripts-read-from-the-environment) | 7 variables, all with defaults; same table |
| Prerequisites | see [Prerequisites](#prerequisites) | the same, plus `-DTRUE=1 -DFALSE=0` in `EXTRA_CFLAGS` on a host whose ICU headers no longer define those macros |
| Output | under `$SANDBOX/out`; **open `criteria.txt` first**, and see [Reading the results of a run](#reading-the-results-of-a-run) for the file map and the result tables. The build and check diagnostics are copied there too, so they survive the build tree | under the same `$SANDBOX/out`; **open `v12_facts.txt` first**, then `verdicts12.txt`; this leg's copies carry a `12` in the name |
| Runtime | about eleven minutes for a full run on the Linux host recorded under [Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server), most of it the build and the four regression suites, and about ninety seconds for `suite attribution probes score criteria` from a built tree; on the Darwin arm64 host recorded under [What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64), 2 min 26 s for the full run and 65 s for those five stages from a built tree | 1 min 52 s for the full run on the Darwin host, most of it the 12.2 build and its `make check`; from a built tree and a running cluster, 0.3 s for `exact transform facts` and 52.7 s for `fixtures score extstat report` on the Linux host |
| Cleanup | `bash btree_bloat_suite_v17.sh clean` stops the server cleanly with `pg_ctl -m fast -w stop`, confirms the teardown (no `postmaster.pid`, no postgres process on the data directory, an empty socket directory) and only then deletes `$SANDBOX`; `stop` does the first two and keeps everything. **`out/` is inside `$SANDBOX`, so copy it out before `clean`** — nothing else preserves a run's results | `bash btree_bloat_suite_v12.sh clean` stops the 12 server the same way and deletes only that leg's `build12`, `install12`, `data12` and `sock12`, because the 17 leg owns the shared `out/` and `sql/`. Run the 17 leg's `clean` last to remove the sandbox entirely |

Save the two fenced blocks below as `btree_bloat_suite_v17.sh` and
`btree_bloat_suite_v12.sh`, then run them from the repository root, because
`WIKI_ROOT` defaults to `$PWD` and both scripts resolve this page, both pinned
checkouts and the sandbox beneath it. The 17 leg also runs `git show` inside the
repository to recover the superseded statement text.

```sh
bash btree_bloat_suite_v17.sh                       # every stage, in order
bash btree_bloat_suite_v17.sh suite score criteria  # selected stages
bash btree_bloat_suite_v12.sh exact                 # just the 12.2 parse result
bash btree_bloat_suite_v17.sh clean                 # stop and delete the sandbox
```

#### The 17 leg's stages

Every stage is idempotent and re-runnable on its own once the stages it needs
have run. The default order is the order of this table.

| Stage | What it does | Needs first |
|---|---|---|
| `build` | configures the pinned checkout out of tree under `$SANDBOX/build17`, installs into `$SANDBOX/install17`, then builds and installs `pageinspect`, `pgstattuple` and `amcheck`; skips everything when the binary already exists. It then copies `configure.log`, `make.log` and `install.log` into `out/`, because `clean` deletes the build tree | nothing |
| `check` | `make check` plus the three contrib checks, one result line each into `out/checks.txt`, then copies every `check_*.log` into `out/` and any `regression.diffs` as `out/diffs_*.txt`; before 2026-09-10 a failed suite left only its one-line summary once the sandbox was gone | `build` |
| `cluster` | `initdb --locale=C --encoding=UTF8`, writes the settings below into `postgresql.conf`, starts on `PORT`, records `uname -sm`, `max_data_alignment` and `database_block_size` into `out/platform.txt`, and creates the six UTF8 databases `geo`, `cal`, `gate`, `acc`, `suite` and `xstat` | `build` |
| `texts` | extracts the four `sql` blocks of this page and the superseded text from `OLD_REV`, checks all five SHA-256 baselines, runs both exact texts as filed, and installs the two harness views in five databases | `cluster` |
| `geometry` | the 78 (key width, fillfactor) cells, scored against `pageinspect` | `texts` |
| `calibration` | the seven insertion patterns, each scored against its own `REINDEX INDEX` | `texts` |
| `gate` | the deduplication-gate fixtures, with `bt_metap().allequalimage` and the build's `DEBUG1` verdicts as oracles | `texts` |
| `acceptance` | fresh sorted builds, the deterministic defects read as two roles, in-index compression, posting tails, the probe fixtures and the statistics barrier | `texts` |
| `suite` | resets the `suite` schema, reinstalls both views, and builds the 112 numbered fixtures with their population assertions; it plans and scores nothing | `texts` |
| `extstat` | rebuilds the two texts the portable `extstat` filter replaced — `est_pre`, which must hash to `BASEPRE`, and `est_wide` — runs both, then compares all three over every database and scores them on an inheritance parent, a bloated inheritance parent and a childless control, into `out/extstat.txt`. It rebuilds indexes only in `xstat`, so the `suite` fixtures are still untouched when `attribution` runs | `texts`; it follows `suite` in the default order so that its equivalence counts and cost pairs see the populated `suite` database. Until 2026-09-10 it ran right after `texts`, where a fresh full run found `suite` empty |
| `attribution` | `EXCEPT` in both directions between the two texts, taken before any rebuild | `suite` |
| `probes` | runs the probe generator on `suite` and `acc` and executes every statement it emits, still before any rebuild | `suite` |
| `score` | `CALL score_all()`: per fixture assert the population, read both views, `REINDEX INDEX`, re-read the size; then writes `out/verdicts.txt` | `suite` |
| `cost` | six interleaved timings of the two exact texts, and the size of the database they ran against | `texts` |
| `criteria` | the six pass-criteria blocks into `out/criteria.txt`, then block 7, which matches every `ERROR`, `FATAL` and `PANIC` line in `out/server.log` against the two errors this suite provokes on purpose and dies on anything left over | `check`, `texts`, `gate`, `attribution`, `score` |
| `report` | lists what landed in `out/` | nothing |
| `stop` | stops the server with `pg_ctl -m fast -w stop`, so the checkpointer writes a shutdown checkpoint and the next start needs no recovery, then confirms the teardown: no `postmaster.pid`, no postgres process on the data directory, an empty socket directory. It dies rather than report a stop that did not happen. Until 2026-09-10 it used `-m immediate`, which skips the checkpoint and forces crash recovery on restart | `cluster` |
| `clean` | `stop`, then deletes `$SANDBOX` after checking it is inside `$WIKI_ROOT/.wiki-runtime/tmp/`; because `stop` dies on a failed teardown, `clean` never deletes a live cluster | nothing |

#### The 12 leg's stages

| Stage | What it does | Needs first |
|---|---|---|
| `build` | 12.2 out of tree under `$SANDBOX/build12` with `CFLAGS="$EXTRA_CFLAGS"`, plus the same three contrib modules, then copies its `configure.log`, `make.log` and `install.log` into `out/` as `configure12.log`, `make12.log` and `install12.log` | nothing |
| `check` | the 12.2 core and contrib suites into `out/checks12.txt`, then copies each `check_*.log` into `out/` as `check12_*.log` and any `regression.diffs` as `diffs12_*.txt` | `build` |
| `cluster` | `initdb --locale=C --encoding=UTF8`, the same cluster settings without `log_min_messages`, started on `PORT12`, and the `leg12` database | `build` |
| `exact` | extracts `sql` block 1, checks its hash, runs the text **unmodified**, and records `exact_text=executes` or `exact_text=refused` plus the first error lines in `out/v12_facts.txt` | `cluster` |
| `transform` | applies one recorded edit per refused construct, re-runs, writes `transform_edits`, and installs the harness view; it dies rather than guess when a construct is still refused | `exact` |
| `facts` | records `server_version_num`, block size, alignment, whether `pg_stat_force_next_flush()` exists, the `pg_stats_ext` columns, the registered B-tree support-function numbers, and whether `WITH (deduplicate_items = off)` is accepted | `cluster` |
| `fixtures` | builds the constructible subset, one writer session per step, polling `pg_stat_all_tables` for publication instead of forcing a flush | `transform` |
| `score` | the same measured-`REINDEX INDEX` scoring, into `out/verdicts12.txt` | `fixtures` |
| `extstat` | rebuilds the text filed before the portable `extstat` filter, checks it against `BASEPRE`, records that this server still refuses it, then scores the filed text against the widened one on an inheritance parent, a bloated inheritance parent and a childless control, into `out/extstat12.txt` | `transform` |
| `report` | appends the same server-error check to `out/v12_facts.txt` — the two errors this leg provokes on purpose are allowed, anything left over kills the run — and prints the file | `facts` |
| `stop` | stops the 12 server with `pg_ctl -m fast -w stop` and confirms the same three teardown facts, dying on any of them; `-m immediate` until 2026-09-10 | `cluster` |
| `clean` | `stop`, then deletes this leg's four directories after the same containment check | nothing |

#### What the scripts read from the environment

| Variable | Default | Read by | Meaning |
|---|---|---|---|
| `WIKI_ROOT` | `$PWD` | both | the repository root; everything else is resolved beneath it |
| `PAGE` | `$WIKI_ROOT/wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md` | both | the page the `sql` blocks are extracted from |
| `SANDBOX` | `$WIKI_ROOT/.wiki-runtime/tmp/btree-suite` | both | build, install, data, socket, SQL and output directories; the only tree either script writes |
| `JOBS` | `4` | both | `make -j` parallelism |
| `SRC` | `$WIKI_ROOT/raw/postgres-17` | 17 leg | the pinned 17 checkout, read only |
| `PORT` | `55437` | 17 leg | the 17 cluster's port |
| `OLD_REV` | `f2d73b4` | 17 leg | the revision of this page holding the superseded statement |
| `SRC12` | `$WIKI_ROOT/raw/postgres-12` | 12 leg | the pinned 12 checkout, read only |
| `PORT12` | `55412` | 12 leg | the 12 cluster's port |
| `EXTRA_CFLAGS` | `-O2 -g -DTRUE=1 -DFALSE=0` | 12 leg | `CFLAGS` for the 12.2 build; the two macro definitions are only needed against ICU 68 or newer |

Both scripts export `PGPORT`, `PGHOST` and `PGDATABASE` for their own `psql`
calls; those are written, not read, so a value in the caller's environment is
overridden rather than honoured.

#### Prerequisites

- A C toolchain and `make`. The recorded run used gcc 13.3.0 on `Linux x86_64`.
  Both legs build their own server; no installed PostgreSQL is used or needed.
- Development headers for ICU, readline and zlib, because both legs configure
  `--with-icu --with-readline --with-zlib` and the gate's nondeterministic
  collation cases need ICU. On a host where ICU is off the default search path,
  pass `ICU_CFLAGS` and `ICU_LIBS` to `configure`.
  [installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193),
  [installation.sgml#--enable-debug](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L1530-L1540).
- The two pinned checkouts, at `raw/postgres-17` and `raw/postgres-12`. Both
  stay read-only: each build is a VPATH build under `.wiki-runtime/tmp/`.
  [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432).
- `git`, with `OLD_REV` (`f2d73b4`) reachable, since the 17 leg recovers the
  superseded statement with `git show`.
- `sha256sum`, and a Bash new enough for arrays and `${var:-default}`. Nothing
  else: no Python, no `awk`, no `perl`, no `jq`.
- Free TCP ports 55437 and 55412, or `PORT`/`PORT12` set to free ones. Both
  clusters listen on a Unix socket inside the sandbox and set
  `listen_addresses = ''`, so the port is reserved but never bound on TCP.
- `pageinspect`, `pgstattuple` and `amcheck`, built and installed from the same
  tree as the server, into the disposable cluster only. The gate reads
  `bt_metap()` and `bt_page_items()`, the acceptance stage reads `pgstatindex`.
  [pageinspect--1.8--1.9.sql#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L73-L82),
  [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31).
- `initdb --locale=C`, which the geometry and gate fixtures assume, and UTF8
  databases, which ICU requires: a `SQL_ASCII` database answers
  `current database's encoding is not supported with this provider`.
  [initdb.sgml#--locale](../../../../raw/postgres-17/doc/src/sgml/ref/initdb.sgml#L281-L291),
  [installation.sgml#ICU-default](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L170).
- Disk for two source builds, two clusters and the fixtures. The 2026-09-09 run
  left 325 B-tree indexes over 85,017 blocks in the 17 leg's `suite` database
  alone.
- On macOS, measured on 2026-09-10: Apple's command-line tools supply the
  compiler, `make`, `bison`, `flex` and `perl`, and `sha256sum` ships at
  `/sbin`. Homebrew's `icu4c` is keg-only and that host had no `pkg-config`,
  so export `ICU_CFLAGS="-I<prefix>/include"` and
  `ICU_LIBS="-L<prefix>/lib -licui18n -licuuc -licudata"` before either
  script; `configure` reads both from the environment and then needs no
  `pkg-config`. The documentation's warning that System Integrity Protection
  breaks `make check` unless `make install` runs first does not bite, because
  both `build` stages install before `check` runs.
  [installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193),
  [installation.sgml#SIP](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L3611-L3618).

Every statement either script sends is disposable. The fixtures create and drop
tables, indexes, operator classes, collations, a login role and, in the `suite`
database, the whole `public` schema; the `clean` stages delete the cluster.
Never point `PORT`, `PORT12` or `PGHOST` at a cluster anyone cares about.

### How to run the suite against the current statement

Every step below was designed against the pinned source and the harness this
page filed on 2026-08-19 and 2026-08-24. The two scripts filed under
[The PostgreSQL 17 suite script](#the-postgresql-17-suite-script) and
[The PostgreSQL 12 leg script](#the-postgresql-12-leg-script) implement this
protocol, and both were executed on 2026-09-09; run them rather than these steps
by hand. The checkout under `raw/postgres-17/` stays read-only: the build is a
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
below reproduced both hashes on 2026-09-09, and again during the
[Measurement-script section review](#measurement-script-section-review).

Extraction is Bash only, using the same `md_block` helper the scripts define, so
no `awk` and no `shasum` — the first is forbidden by `MANDATORY Measurement
Script` and the second is a Perl program. `sha256sum` from coreutils is the
digest tool both scripts use.

```sh
# Paste the md_block function from the 17 leg script below into the shell first;
# stage_texts does exactly this, for all four blocks at once.
page=wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md
md_block sql 1 "$page" > est_r2.sql
sha256sum est_r2.sql   # 8acd531b7bcd2f2ca679e65024d83bd61debcb4b75bb18f3834a368454d574fd
git show "f2d73b4:$page" > old_page.md
md_block sql 1 old_page.md > est_old.sql
sha256sum est_old.sql  # bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357
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
with the platform record item 2 writes to `out/platform.txt`: `uname -sm`,
`max_data_alignment` and `database_block_size`, the three facts every geometry
constant assumes. The sandbox lives under `.wiki-runtime/tmp/` and is deleted or
kept as the asker directs.

### The two suite scripts, and the rules they follow

**Both scripts are filed in full below, and both were run end to end on
2026-09-09 before this section was written.** They turn
[How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement)
into two files. `btree_bloat_suite_v17.sh` builds 17.11 out of tree from the
pin, runs the engine suites, and then runs every test family this page names —
page geometry, insertion-pattern calibration, the deduplication gate, the
acceptance fixtures, the numbered suite of 112 partial-index and control
fixtures, the `EXCEPT` attribution, the validation probes, the scoring pass and
the cost comparison. `btree_bloat_suite_v12.sh` runs step 8, the cross-version
leg, against the pinned 12 checkout. They are Bash and SQL only: no Python, no
`awk`, no external harness, so a reviewer needs a compiler, a shell and this
page.

Eleven rules hold in both scripts. The last two were added by the
[Measurement-script section review](#measurement-script-section-review).

| Rule | How the scripts keep it |
|---|---|
| The pinned checkout stays read only | every artifact goes under `SANDBOX`, default `.wiki-runtime/tmp/btree-suite`, and the build is the VPATH form the documentation describes. [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L427-L432) |
| The statement under test is never retyped | `md_block` extracts a fenced block from this Markdown file in pure Bash, and `git show` recovers the superseded text from revision `f2d73b4`; all five SHA-256 baselines are checked before use |
| The exact filed text runs first | both texts execute with the 1 MB filter, the `ORDER BY` and the `LIMIT 20` intact, before any harness view exists |
| Only the three documented edits are applied | `harness_view` drops the two `SET` lines, projects the internals the scorer reads, and drops the filter, the order and the limit. Nothing else is rewritten. [The current recommended statement](#the-current-recommended-statement) |
| Statistics are published before they are read | `pg_stat_force_next_flush()` precedes every `ANALYZE` and `VACUUM` on the 17 leg, and no fixture is read inside the transaction that built it. [pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708), [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337) |
| Attribution and probes run before the first rebuild | the fixture stage only builds and plans; `EXCEPT` and the probe generator run next; the scoring pass is the first thing that rebuilds an index. [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2829), [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3597) |
| Every fixture asserts its own population | the `plan` table carries a counting query and the intended row count, and `verdicts.contract_ok` is false when they disagree |
| Oracles are read beside each row and never scored | `bt_metap().allequalimage` for the gate, the build's `DEBUG1` line, and posting lists in `bt_page_items`. [btreefuncs.c#bt_metap-allequalimage](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L916-L921), [nbtutils.c#_bt_allequalimage-debug](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180), [pageinspect--1.8--1.9.sql#bt_page_items](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L109-L118) |
| Stages are selectable and idempotent | `bash btree_bloat_suite_v17.sh gate score` runs two stages; a second run reuses the build and the cluster, and the suite stage resets its schema and reinstalls both views before rebuilding its fixtures |
| No `psql` error passes without a non-zero status | every helper carries `-X -v ON_ERROR_STOP=1`. `-X` keeps a stray `~/.psqlrc` out of the result; `ON_ERROR_STOP` is what makes a failed statement inside a `-f` script exit non-zero at all, because `MainLoop` only sets a failure status when `die_on_error` is set from it. [mainloop.c:376](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L376), [mainloop.c#die_on_error](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L587-L594), [psql-ref.sgml#Exit-Status](../../../../raw/postgres-17/doc/src/sgml/ref/psql-ref.sgml#L627-L636) |
| Nothing outside the sandbox is deleted | both `clean` stages check that the directory they are about to remove is inside `$WIKI_ROOT/.wiki-runtime/tmp/` and refuse otherwise, so a stray `SANDBOX` cannot turn `rm -rf` loose |

Neither script contains a Markdown fence: `md_block` assembles the three
backticks from `printf '\140'`, so each script can live inside the fenced block
that publishes it and still extract blocks from this page.

**The order of this page's fenced `sql` blocks is load-bearing.** `md_block sql
N` counts fenced `sql` blocks from the top of the file, so blocks 1 to 4 are the
estimator, the probe generator, the geometry harness and the calibration
harness, in that order. Inserting a new `sql` block above any of them silently
repoints the scripts at the wrong text. The five SHA-256 baselines are the
guard: `out/hashes.txt` shows `DIFFER` and every number below it is about a
different statement. When this page gains SQL, put the block after the
calibration harness, or re-baseline deliberately.

The stages, in default order:

| Script | Stages |
|---|---|
| `btree_bloat_suite_v17.sh` | `build check cluster texts geometry calibration gate acceptance suite extstat attribution probes score cost criteria report`, plus `stop` and `clean` |
| `btree_bloat_suite_v12.sh` | `build check cluster exact transform facts fixtures score extstat report`, plus `stop` and `clean` |

Until 2026-09-10 this table omitted the `extstat` stage that both scripts had
run by default since that morning's filter change; the stage tables under
[How to use the suite scripts](#how-to-use-the-suite-scripts) had it, this
one did not. The same pass moved the 17 leg's `extstat` from right after
`texts` to right after `suite`; see
[The stop stage, repaired](#the-stop-stage-repaired).

The cluster the 17 script writes uses the settings
[How to run the suite against the current statement](#how-to-run-the-suite-against-the-current-statement)
prescribes. They are written into `postgresql.conf` before the first start, so
each is in effect from startup; the apply scope below is what a *later* change
to that setting would need.

| Setting | Value | Context | Scope of a later change |
|---|---|---|---|
| `autovacuum` | `off` | `PGC_SIGHUP` | reload. [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1453) |
| `fsync` | `off` | `PGC_SIGHUP` | reload. [guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1097-L1100) |
| `listen_addresses` | `''`, so the cluster is reachable only through the sandbox socket | `PGC_POSTMASTER` | restart. [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4437-L4441) |
| `port` | `PORT`, default `55437`, and `PORT12`, default `55412` | `PGC_POSTMASTER` | restart. [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2394-L2397) |
| `logging_collector` | `off`, so `pg_ctl -l` keeps every line in one file the run can grep | `PGC_POSTMASTER` | restart. [guc_tables.c#logging_collector](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1641-L1644) |
| `shared_buffers` | `512MB` | `PGC_POSTMASTER` | restart. [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262-L2265) |
| `maintenance_work_mem` | `256MB` | `PGC_USERSET` | session or transaction. [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2469) |
| `max_parallel_maintenance_workers` | `0` | `PGC_USERSET` | session or transaction. [guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3410-L3413) |
| `log_min_messages` | `debug1` | `PGC_SUSET` | session, and only for a superuser or a role granted `SET` on it. [guc_tables.c#log_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4873-L4877) |
| `unix_socket_directories` | the sandbox socket directory | `PGC_POSTMASTER` | restart. [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4426-L4430) |
| `client_min_messages` | `debug1` around the gate builds | `PGC_USERSET` | session or transaction. [guc_tables.c#client_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4777-L4780) |
| `default_statistics_target` | `1` for fixture 120 only | `PGC_USERSET` | session or transaction. [guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2074) |
| `statement_timeout`, `lock_timeout` | `600s`/`900s` and `2s` in the harness sessions | `PGC_USERSET` | session or transaction. [guc_tables.c#statement_timeout-and-lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631) |

Every context and scope above is the PostgreSQL 17 definition, read from the
pinned 17 checkout. The 12 leg writes the same setting names except
`log_min_messages`, all of them into `postgresql.conf` before its first start,
so the run itself never needs an apply scope there; the scopes those settings
have on 12.2 are not stated on this page, because a v17 page may not cite a v12
checkout. See
[The 12 leg's settings have no citable apply scope here](#the-12-legs-settings-have-no-citable-apply-scope-here).

`initdb --locale=C` fixes the collation the geometry and gate fixtures assume,
and the ICU cases need a UTF8 database, which is why every database is created
`TEMPLATE template0 ENCODING 'UTF8'`.
[initdb.sgml#--locale](../../../../raw/postgres-17/doc/src/sgml/ref/initdb.sgml#L281-L291),
[installation.sgml#ICU-default](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L170).

### The PostgreSQL 17 suite script

Run it from the repository root. `bash btree_bloat_suite_v17.sh` runs every
stage; `bash btree_bloat_suite_v17.sh clean` stops the server and deletes the
sandbox. The whole run took about eleven minutes on the host recorded under
[Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server), of which
the build and the four regression suites are most of the time; from a built
tree, `suite attribution probes score criteria` is about ninety seconds.

```bash
#!/usr/bin/env bash
#
# btree_bloat_suite_v17.sh - the whole test suite of the PostgreSQL 17 wiki page
# "Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17",
# in bash and SQL only.
#
# It builds 17.11 out of tree from the pinned checkout, runs the engine
# regression suites, starts an isolated cluster, installs the page's estimator
# and the superseded text as views, builds every fixture family the page names,
# scores each one against a measured REINDEX INDEX, attributes every moved row,
# runs the validation probes and prints the pass criteria.
#
# The pinned checkout is read only: everything this script writes lives under
# $SANDBOX (default .wiki-runtime/tmp/btree-suite).
#
# Usage, from the repository root:
#   bash btree_bloat_suite_v17.sh                 # all stages
#   bash btree_bloat_suite_v17.sh build check     # selected stages
#   bash btree_bloat_suite_v17.sh clean           # stop and delete the sandbox
#
# Stages: build check cluster texts geometry calibration gate acceptance
#         suite extstat attribution probes score cost criteria report
#         stop clean
#
# Environment: WIKI_ROOT PAGE SRC SANDBOX PORT JOBS OLD_REV
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-17}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/btree-suite}"
PORT="${PORT:-55437}"
JOBS="${JOBS:-4}"
OLD_REV="${OLD_REV:-f2d73b4}"          # revision holding the superseded text

BUILD="$SANDBOX/build17"; INST="$SANDBOX/install17"; DATA="$SANDBOX/data17"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql"; SOCK="$SANDBOX/sock"; BIN="$INST/bin"
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE=postgres

# SHA-256 baselines of the four fenced SQL blocks of the page, in page order.
BASE1=646df923635182809f1a139e2f7f9367e94b6e0eaf79ed66fc37697a82d5d706  # estimator
BASE2=bfa7721f5edae40fd883b5bc0f0776e499716c48cfdbe10d191e95b9f8a3bb0d  # probes
BASE3=0b03f0c918a669b5402d1e630046bf2f1b9fc71453f54ef7119942d13a43c7ec  # geometry
BASE4=3e57a5687d15c0725ab5cd1c2cea09b0a18a549ac649b82eee5246d1b75b8777  # calibration
BASEOLD=bffd166e44a4e81c181df3d9a10bfb547a6dcaf7349c2cd055578f35050d1357
# The estimator text as filed before the portable extstat filter of 2026-09-10.
# The extstat stage rebuilds it from the current text and must reproduce this.
BASEPRE=8acd531b7bcd2f2ca679e65024d83bd61debcb4b75bb18f3834a368454d574fd

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# psql helpers.  -X ignores ~/.psqlrc; ON_ERROR_STOP is on every helper, because
# without it a failed statement inside a -f script leaves the exit status 0.
q()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$1" -c "$2"; }        # command
f()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$1" -f "$2"; }        # file
s()  { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$1" -c "$2"; }    # scalar
t()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off \
                   -d "$1" -c "$2"; }                                 # table

# md_block <fence-language> <n> <file>: print the nth fenced block, bash only.
# The fence is assembled from printf '\140' so that this script contains no
# literal Markdown fence and can therefore live inside one.
md_block() {
  local lang=$1 want=$2 file=$3 n=0 inb=0 line tick fence
  tick=$(printf '\140'); fence="$tick$tick$tick"
  while IFS= read -r line; do
    if [ "$inb" = 1 ]; then
      if [ "$line" = "$fence" ]; then inb=0; [ "$n" = "$want" ] && return 0; continue; fi
      [ "$n" = "$want" ] && printf '%s\n' "$line"
    elif [ "$line" = "$fence$lang" ]; then
      n=$((n + 1)); inb=1
    fi
  done < "$file"
}

# harness_view <sql-file> <view> <extra projection>: the three documented edits.
# Drop the two SET lines, project the internals the scorer reads, and drop the
# 1 MB filter, the suppress_row filter, the ORDER BY and the LIMIT.
harness_view() {
  local file=$1 view=$2 extra=$3 line
  printf 'DROP VIEW IF EXISTS %s;\nCREATE VIEW %s AS\n' "$view" "$view"
  while IFS= read -r line; do
    case $line in
      "SET /* wiki_btree_wasted_space"*)     continue ;;
      "       server_version_num")           printf '       server_version_num,\n%s\n' "$extra"; continue ;;
      " WHERE actual_bytes > 1024 * 1024"*)  continue ;;
      " ORDER BY (actual_bytes"*)            continue ;;
      " LIMIT 20;")                          printf ';\n'; continue ;;
    esac
    printf '%s\n' "$line"
  done < "$file"
}

# Internals both texts define, and the four the current text adds.
INTERNALS='       expected_blocks, floor_blocks, actual_bytes, live_rows, slot,
       leaf_cap, int_cap, nmax, leaf_pages, tids, dedup_applies, is_partial,
       has_expressions, stats_row_missing, dedup_credited, stats_stale,
       suppress_row, ext_used, any_no_stats, any_stats_hidden,
       any_varlena_include'
INTERNALS_R2='       itupsz, any_stats_disabled, any_compressible, equalimage_state'

# ---------------------------------------------------------------- build ------
stage_build() {
  say "build 17.11 out of tree from $SRC"
  [ -x "$BIN/postgres" ] && { note "already built, skipping"; return 0; }
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC; set SRC or run from the repository root"
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --enable-debug \
      --with-icu --with-readline --with-zlib > configure.log 2>&1 ) \
    || die "configure failed, see $BUILD/configure.log"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || die "make failed, see $BUILD/make.log"
  local m
  for m in pageinspect pgstattuple amcheck; do
    ( cd "$BUILD" && make -C "contrib/$m" -j"$JOBS" >> install.log 2>&1 \
        && make -C "contrib/$m" install >> install.log 2>&1 ) || die "contrib/$m failed"
  done
  # clean deletes $BUILD, and $OUT is what a reviewer copies out, so keep the
  # build diagnostics in $OUT.  Without this a failed build leaves nothing to
  # read once the sandbox is gone.
  cp "$BUILD/configure.log" "$BUILD/make.log" "$BUILD/install.log" "$OUT/" 2>/dev/null
  note "$("$BIN/postgres" --version)"
}

# ---------------------------------------------------------------- check ------
stage_check() {
  say "engine regression suites"
  : > "$OUT/checks.txt"
  ( cd "$BUILD" && make check > check_core.log 2>&1 )
  printf 'core=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_core.log" | tail -1)" \
    >> "$OUT/checks.txt"
  local m
  for m in pageinspect pgstattuple amcheck; do
    ( cd "$BUILD" && make -C "contrib/$m" check > "check_$m.log" 2>&1 )
    printf '%s=%s %s\n' "$m" "$?" \
      "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_$m.log" | tail -1)" \
      >> "$OUT/checks.txt"
  done
  # A one-line summary cannot diagnose a failure, and the logs and diffs live
  # in $BUILD, which clean deletes.  Copy both where they survive.
  cp "$BUILD"/check_*.log "$OUT/" 2>/dev/null
  local d
  for d in "$BUILD/src/test/regress" "$BUILD"/contrib/*; do
    [ -f "$d/regression.diffs" ] \
      && cp "$d/regression.diffs" "$OUT/diffs_$(basename "$d").txt"
  done
  cat "$OUT/checks.txt" >&2
}

# ---------------------------------------------------------------- cluster ----
stage_cluster() {
  say "isolated cluster on port $PORT"
  # Returning early here would skip the database loop below, so a cluster left
  # running by an earlier run would never gain a database a new stage needs.
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    note "already running"
  else
    if [ ! -d "$DATA" ]; then
      mkdir -p "$SOCK"
      "$BIN/initdb" -D "$DATA" --locale=C --encoding=UTF8 > "$OUT/initdb.log" 2>&1 \
        || die "initdb failed"
      cat >> "$DATA/postgresql.conf" <<CONF
listen_addresses = ''
unix_socket_directories = '$SOCK'
port = $PORT
autovacuum = off
fsync = off
shared_buffers = '512MB'
maintenance_work_mem = '256MB'
max_parallel_maintenance_workers = 0
log_min_messages = debug1
logging_collector = off
CONF
    fi
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w start > /dev/null \
      || die "server start failed"
  fi
  note "$(s postgres 'select /* wiki_btree_suite_version */ version()')"
  s postgres "select /* wiki_btree_suite_platform */
                     'max_data_alignment=' || max_data_alignment ||
              ' database_block_size=' || database_block_size from pg_control_init()" \
    | tee "$OUT/platform.txt" >&2
  printf 'uname: %s\n' "$(uname -sm)" >> "$OUT/platform.txt"
  local db
  for db in geo cal gate acc suite xstat; do
    s postgres "select /* wiki_btree_suite_database_exists */ 1
                  from pg_database where datname='$db'" | grep -q 1 \
      || "$BIN/createdb" -T template0 -E UTF8 --locale=C "$db"
  done
}

# ---------------------------------------------------------------- texts ------
stage_texts() {
  say "statement texts, hashes and harness views"
  [ -f "$PAGE" ] || die "no page at $PAGE; set PAGE or run from the repository root"
  md_block sql 1 "$PAGE" > "$SQLD/est_r2.sql"
  md_block sql 2 "$PAGE" > "$SQLD/probegen.sql"
  md_block sql 3 "$PAGE" > "$SQLD/geometry.sql"
  md_block sql 4 "$PAGE" > "$SQLD/calibration.sql"
  ( cd "$WIKI_ROOT" && git show "$OLD_REV:wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md" ) \
    > "$SQLD/old_page.md" 2>/dev/null || die "cannot read revision $OLD_REV"
  md_block sql 1 "$SQLD/old_page.md" > "$SQLD/est_old.sql"

  : > "$OUT/hashes.txt"
  local n f base got
  n=0
  for f in est_r2 probegen geometry calibration est_old; do
    n=$((n + 1))
    case $n in 1) base=$BASE1;; 2) base=$BASE2;; 3) base=$BASE3;; 4) base=$BASE4;; 5) base=$BASEOLD;; esac
    got=$(sha256sum < "$SQLD/$f.sql" | cut -d' ' -f1)
    if [ "$got" = "$base" ]; then printf '%-12s match  %s\n' "$f" "$got" >> "$OUT/hashes.txt"
    else printf '%-12s DIFFER %s (baseline %s)\n' "$f" "$got" "$base" >> "$OUT/hashes.txt"; fi
  done
  cat "$OUT/hashes.txt" >&2

  # Both exact texts must execute as filed, filter and LIMIT intact.
  local db
  for db in suite acc; do
    f "$db" "$SQLD/est_r2.sql" > "$OUT/exact_r2_$db.txt" 2>&1 \
      && note "exact current text runs on $db" || die "exact current text failed on $db"
  done
  f suite "$SQLD/est_old.sql" > "$OUT/exact_old_suite.txt" 2>&1 \
    && note "exact superseded text runs on suite" || note "exact superseded text FAILED on suite"

  harness_view "$SQLD/est_r2.sql"  est_r2  "$INTERNALS,
$INTERNALS_R2" > "$SQLD/view_r2.sql"
  harness_view "$SQLD/est_old.sql" est_old "$INTERNALS" > "$SQLD/view_old.sql"
  for db in geo cal gate acc suite; do
    f "$db" "$SQLD/view_r2.sql"  || die "est_r2 view failed on $db"
    f "$db" "$SQLD/view_old.sql" || die "est_old view failed on $db"
  done
}

# ---------------------------------------------------------------- extstat ----
# The portable inherited filter of the extstat CTE, against the two readings it
# replaces.  Both are rebuilt from the filed text, one edit apart:
#   est_pre   the flag named as a column, AND se.inherited = false, which is
#             the text filed before 2026-09-10 and must hash to BASEPRE
#   est_wide  est_pre with that line deleted, which is what the 12 leg's
#             transformer produced before this change
# Pass: est_r2 equals est_pre row for row in every database, and est_wide
# differs on the inheritance parent, whose own ANALYZE pass is the only one
# describing the rows the parent's own index actually contains.
# In the default order this stage runs after suite, so that its equivalence
# counts and its cost pairs see the populated suite database; until 2026-09-10
# it ran right after texts, where a fresh full run found suite empty.
stage_extstat() {
  say "extstat: the portable inherited filter against the two readings it replaces"
  [ -s "$SQLD/est_r2.sql" ] || die "no $SQLD/est_r2.sql; run the texts stage first"
  local line got
  : > "$SQLD/est_pre.sql"
  while IFS= read -r line; do
    case $line in
      "    -- The inherited flag is read through row_to_json()"*) continue ;;
      "    -- as a column, so one text also parses where"*)       continue ;;
      "    -- column: there the key is absent"*)                  continue ;;
      "    -- admits the row, which is the only ANALYZE pass"*)   continue ;;
      "                          AND coalesce((row_to_json(se)"*)
        printf '                          AND se.inherited = false\n' \
          >> "$SQLD/est_pre.sql"; continue ;;
      "                                       false) = false")    continue ;;
    esac
    printf '%s\n' "$line" >> "$SQLD/est_pre.sql"
  done < "$SQLD/est_r2.sql"
  got=$(sha256sum < "$SQLD/est_pre.sql" | cut -d' ' -f1)
  : > "$SQLD/est_wide.sql"
  while IFS= read -r line; do
    [ "$line" = "                          AND se.inherited = false" ] && continue
    printf '%s\n' "$line" >> "$SQLD/est_wide.sql"
  done < "$SQLD/est_pre.sql"
  { printf 'est_pre  %s %s\n' \
      "$([ "$got" = "$BASEPRE" ] && printf match || printf DIFFER)" "$got"
    printf 'est_wide        %s\n' "$(sha256sum < "$SQLD/est_wide.sql" | cut -d' ' -f1)"
  } > "$OUT/extstat.txt"
  [ "$got" = "$BASEPRE" ] || note "est_pre does not match BASEPRE; the reconstruction is stale"

  # Both reconstructions must execute as filed on a server that has the column.
  f xstat "$SQLD/est_pre.sql"  > "$OUT/extstat_pre.txt"  2>&1 \
    || die "the reconstructed previous text does not execute here"
  f xstat "$SQLD/est_wide.sql" > "$OUT/extstat_wide.txt" 2>&1 \
    || die "the widened text does not execute here"
  harness_view "$SQLD/est_pre.sql"  est_pre  "$INTERNALS,
$INTERNALS_R2" > "$SQLD/view_pre.sql"
  harness_view "$SQLD/est_wide.sql" est_wide "$INTERNALS,
$INTERNALS_R2" > "$SQLD/view_wide.sql"
  local db
  for db in geo cal gate acc suite xstat; do
    f "$db" "$SQLD/view_r2.sql"   || die "est_r2 view failed on $db"
    f "$db" "$SQLD/view_pre.sql"  || die "est_pre view failed on $db"
    f "$db" "$SQLD/view_wide.sql" || die "est_wide view failed on $db"
  done

  # Disposable fixtures: the block below drops and creates tables, statistics
  # objects and indexes in the xstat database of the sandbox cluster.  It is
  # not meant for a database anyone cares about.
  f xstat /dev/stdin <<'SQL'
SET /* wiki_btree_extstat_client_min_messages */ client_min_messages = warning;
DROP TABLE IF EXISTS xchi, xpar, xchi2, xpar2, xchi3, xpar3, xflat,
                     xstat_res CASCADE;

-- A. an inheritance parent carrying a whole-key ndistinct object.  ANALYZE
--    writes one pg_statistic_ext_data row per pass, and a legacy inheritance
--    parent's index holds only the parent's own rows, so the own pass is the
--    correct input and the inherited pass describes rows it does not contain.
CREATE TABLE xpar(a int, b int, c int);
CREATE TABLE xchi(a int, b int, c int) INHERITS (xpar);
CREATE STATISTICS xpar_nd (ndistinct) ON a, b FROM xpar;
INSERT INTO xpar SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i;
INSERT INTO xchi SELECT i % 500, i % 700, i FROM generate_series(1, 300000) i;
CREATE INDEX xpar_ab ON xpar (a, b);
ANALYZE xpar;
SELECT pg_stat_force_next_flush();

-- B. the same shape with 60 % of the parent's rows deleted and vacuumed, so a
--    real 60 % of the index is reclaimable and every text can be scored
--    against a measured REINDEX INDEX rather than against a fresh build.
CREATE TABLE xpar2(a int, b int, c int);
CREATE TABLE xchi2(a int, b int, c int) INHERITS (xpar2);
CREATE STATISTICS xpar2_nd (ndistinct) ON a, b FROM xpar2;
INSERT INTO xpar2 SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i;
INSERT INTO xchi2 SELECT i % 500, i % 700, i FROM generate_series(1, 300000) i;
CREATE INDEX xpar2_ab ON xpar2 (a, b);
DELETE FROM xpar2 WHERE c % 5 < 3;
VACUUM xpar2;
ANALYZE xpar2;
SELECT pg_stat_force_next_flush();

-- D. the same parent with a high-cardinality child, which is the shape that
--    makes the difference large.  The inherited pass then estimates about one
--    distinct pair per row, so a text reading it credits no deduplication at
--    all and prices singleton tuples instead of posting ones.
CREATE TABLE xpar3(a int, b int, c int);
CREATE TABLE xchi3(a int, b int, c int) INHERITS (xpar3);
CREATE STATISTICS xpar3_nd (ndistinct) ON a, b FROM xpar3;
INSERT INTO xpar3 SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i;
INSERT INTO xchi3 SELECT i, i, i FROM generate_series(1, 300000) i;
CREATE INDEX xpar3_ab ON xpar3 (a, b);
ANALYZE xpar3;
SELECT pg_stat_force_next_flush();

-- C. control: the same object on a table with no children, so ANALYZE writes
--    one pass and all three texts must agree.
CREATE TABLE xflat(a int, b int, c int);
CREATE STATISTICS xflat_nd (ndistinct) ON a, b FROM xflat;
INSERT INTO xflat SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i;
CREATE INDEX xflat_ab ON xflat (a, b);
ANALYZE xflat;
SELECT pg_stat_force_next_flush();

CREATE TABLE xstat_res(idx text, txt text, blocks int, wsp numeric,
                       wspf numeric, key_groups numeric, ext_used bool,
                       caveats text, actual numeric);
DO $x$
DECLARE ix text; v text; e record; sb bigint; sa bigint;
BEGIN
  FOREACH ix IN ARRAY ARRAY['xpar_ab','xpar2_ab','xpar3_ab','xflat_ab'] LOOP
    sb := pg_relation_size(ix::regclass);
    FOREACH v IN ARRAY ARRAY['est_r2','est_pre','est_wide'] LOOP
      EXECUTE format('SELECT wasted_space_pct AS wsp,
                             wasted_space_pct_floor AS wspf,
                             key_groups, ext_used, caveats
                        FROM %I WHERE indexname = %L', v, ix) INTO e;
      INSERT INTO xstat_res(idx, txt, blocks, wsp, wspf, key_groups,
                            ext_used, caveats)
        VALUES (ix, v, sb / 8192, e.wsp, e.wspf, e.key_groups, e.ext_used,
                e.caveats);
    END LOOP;
  END LOOP;
  FOREACH ix IN ARRAY ARRAY['xpar_ab','xpar2_ab','xpar3_ab','xflat_ab'] LOOP
    sb := pg_relation_size(ix::regclass);
    EXECUTE format('REINDEX INDEX %I', ix);
    sa := pg_relation_size(ix::regclass);
    UPDATE xstat_res SET actual = round(100.0 * (sb - sa) / greatest(sb, 1), 1)
     WHERE idx = ix;
  END LOOP;
END $x$;
SQL

  # 1. equivalence: the filed text against the text it replaced, every column
  #    both views project, in every database the suite builds.  ext_used
  #    counts the rows the extstat CTE actually fed, because a database with
  #    no extended-statistics object cannot tell the two texts apart.
  printf 'equivalence, est_r2 against est_pre\n' >> "$OUT/extstat.txt"
  for db in geo cal gate acc suite xstat; do
    printf '%-6s r2_minus_pre=%s pre_minus_r2=%s rows=%s ext_used=%s\n' "$db" \
      "$(s "$db" 'SELECT /* wiki_btree_extstat_r2_minus_pre */ count(*)
                    FROM (SELECT * FROM est_r2 EXCEPT SELECT * FROM est_pre) d')" \
      "$(s "$db" 'SELECT /* wiki_btree_extstat_pre_minus_r2 */ count(*)
                    FROM (SELECT * FROM est_pre EXCEPT SELECT * FROM est_r2) d')" \
      "$(s "$db" 'SELECT /* wiki_btree_extstat_row_count */ count(*) FROM est_r2')" \
      "$(s "$db" 'SELECT /* wiki_btree_extstat_ext_used */ count(*)
                    FROM est_r2 WHERE ext_used')" \
      >> "$OUT/extstat.txt"
  done
  # 2. the two ANALYZE passes the fixture stores, and what each says.
  t xstat "SELECT /* wiki_btree_extstat_passes */
                  tablename, statistics_name, inherited,
                  ((n_distinct::text)::json ->> '1, 2')::numeric AS nd_whole_key
             FROM pg_stats_ext WHERE schemaname = 'public'
            ORDER BY tablename, inherited" >> "$OUT/extstat.txt" 2>&1
  # 3. the three texts scored against a measured REINDEX INDEX.
  t xstat "SELECT /* wiki_btree_extstat_scored */
                  idx, txt, blocks, actual, wsp, wspf, key_groups, ext_used
             FROM xstat_res ORDER BY idx, txt" >> "$OUT/extstat.txt" 2>&1
  # 4. cost: row_to_json() serialises a whole pg_stats_ext row per candidate
  #    join, so the two texts are timed interleaved on the largest database.
  printf 'cost, six interleaved pairs on suite\n' >> "$OUT/extstat.txt"
  local i
  for i in 1 2 3 4 5 6; do
    printf 'pair %s r2  %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
           -c '\timing on' -f "$SQLD/est_r2.sql" 2>&1 \
         | grep -E '^Time:' | tail -1)" >> "$OUT/extstat.txt"
    printf 'pair %s pre %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
           -c '\timing on' -f "$SQLD/est_pre.sql" 2>&1 \
         | grep -E '^Time:' | tail -1)" >> "$OUT/extstat.txt"
  done
  cat "$OUT/extstat.txt" >&2
}

# ---------------------------------------------------------------- geometry ---
stage_geometry() {
  say "page geometry against pageinspect, 78 cells"
  # Disposable fixtures: this stage creates and drops relations in the geo
  # database of the sandbox cluster.  Never point it at a real cluster.
  q geo 'CREATE EXTENSION IF NOT EXISTS pageinspect'
  q geo 'DROP TABLE IF EXISTS geo_result'
  f geo "$SQLD/geometry.sql" || die "geometry harness failed"
  t geo "SELECT /* wiki_btree_geometry_score */
           count(*) AS cells,
           count(*) FILTER (WHERE mode_items = pred_cap)          AS leaf_exact,
           count(*) FILTER (WHERE mode_items = least(pred_soft, pred_hard) + 1) AS leaf_one_high,
           count(*) FILTER (WHERE int_max_items IS NULL
                              OR int_max_items <= int_pred_cap)   AS int_within_cap,
           count(*) FILTER (WHERE relpages = leaf_pages + int_pages + 1) AS relpages_exact
         FROM geo_result" > "$OUT/geometry.txt" 2>&1
  t geo "SELECT /* wiki_btree_geometry_cells */
                keylen, fillfactor, itupsz, leaf_pages, int_pages, relpages,
                mode_items, pred_cap, min_items, max_items
           FROM geo_result ORDER BY fillfactor, keylen" >> "$OUT/geometry.txt" 2>&1
  head -12 "$OUT/geometry.txt" >&2
}

# ------------------------------------------------------------- calibration ---
stage_calibration() {
  say "calibration by insertion pattern, scored against REINDEX INDEX"
  # Disposable fixtures: the cal schema is dropped and rebuilt on every run.
  q cal 'DROP SCHEMA IF EXISTS cal CASCADE'
  f cal "$SQLD/calibration.sql" || die "calibration fixtures failed"
  f cal /dev/stdin <<'SQL'
DROP TABLE IF EXISTS cal_res;
CREATE TABLE cal_res(pattern text, idx text, size_before bigint, size_after bigint,
                     actual numeric, wsp numeric, wspf numeric, tids numeric,
                     equalimage text, caveats text);
DO $cal$
DECLARE r record; sb bigint; sa bigint; e record;
BEGIN
  FOR r IN SELECT c.relname AS idx FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE n.nspname = 'cal' AND c.relkind = 'i' ORDER BY c.relname LOOP
    sb := pg_relation_size(('cal.' || quote_ident(r.idx))::regclass);
    SELECT * INTO e FROM est_r2 WHERE indexname = r.idx;
    EXECUTE format('REINDEX INDEX cal.%I', r.idx);
    sa := pg_relation_size(('cal.' || quote_ident(r.idx))::regclass);
    INSERT INTO cal_res VALUES (r.idx, r.idx, sb, sa,
      round(100.0 * (sb - sa) / greatest(sb, 1), 1),
      e.wasted_space_pct, e.wasted_space_pct_floor, e.tids_per_tuple,
      e.equalimage, e.caveats);
  END LOOP;
END $cal$;
SQL
  t cal 'SELECT /* wiki_btree_calibration_report */
                pattern, size_before/8192 AS blocks_before, size_after/8192 AS blocks_after,
                actual, wsp, wspf, tids, equalimage, caveats
           FROM cal_res ORDER BY pattern' > "$OUT/calibration.txt" 2>&1
  cat "$OUT/calibration.txt" >&2
}

# ---------------------------------------------------------------- gate -------
stage_gate() {
  say "deduplication gate, tests 1-17, 28 fixtures on two 500,000-row tables"
  q gate 'CREATE EXTENSION IF NOT EXISTS pageinspect'
  q gate 'CREATE EXTENSION IF NOT EXISTS amcheck'
  # Disposable fixtures: the block below drops and recreates tables, operator
  # classes, collations and a public.btequalimage impostor in the gate database
  # of the sandbox cluster.  It is not meant for a database anyone cares about.
  PGOPTIONS='-c client_min_messages=debug1' \
    "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d gate -f /dev/stdin \
    > "$OUT/gate_build.log" 2>&1 <<'SQL'
DROP TABLE IF EXISTS t CASCADE;
DROP TABLE IF EXISTS t2 CASCADE;
DROP OPERATOR CLASS IF EXISTS int4_ei_true USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS int4_ei_false USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS int4_ei_none USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS int4_ei_alias USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS int8_ei_true USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS int8_ei_false USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS text_squat USING btree CASCADE;
DROP OPERATOR CLASS IF EXISTS text_renamed USING btree CASCADE;

CREATE COLLATION IF NOT EXISTS ci   (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
CREATE COLLATION IF NOT EXISTS cdet (provider = icu, locale = 'und');

CREATE OR REPLACE FUNCTION ei_true(oid)  RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;
CREATE OR REPLACE FUNCTION ei_false(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT false $$;
CREATE OR REPLACE FUNCTION ei_alias(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btequalimage';
-- test 16, the impostor: a SQL function wearing the built-in's name.  It must
-- be schema-qualified in the operator class or pg_catalog wins the lookup.
CREATE OR REPLACE FUNCTION public.btequalimage(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;
-- test 16, the rename: prosrc still names the built-in, so the gate credits it.
CREATE OR REPLACE FUNCTION ei_renamed(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btvarstrequalimage';

CREATE OPERATOR CLASS int4_ei_true FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_true(oid);
CREATE OPERATOR CLASS int4_ei_false FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_false(oid);
CREATE OPERATOR CLASS int4_ei_none FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4);
CREATE OPERATOR CLASS int4_ei_alias FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_alias(oid);
CREATE OPERATOR CLASS int8_ei_true FOR TYPE int8 USING btree AS
  OPERATOR 1 <(int8,int8), OPERATOR 2 <=(int8,int8), OPERATOR 3 =(int8,int8),
  OPERATOR 4 >=(int8,int8), OPERATOR 5 >(int8,int8),
  FUNCTION 1 btint8cmp(int8,int8), FUNCTION 4 ei_true(oid);
CREATE OPERATOR CLASS int8_ei_false FOR TYPE int8 USING btree AS
  OPERATOR 1 <(int8,int8), OPERATOR 2 <=(int8,int8), OPERATOR 3 =(int8,int8),
  OPERATOR 4 >=(int8,int8), OPERATOR 5 >(int8,int8),
  FUNCTION 1 btint8cmp(int8,int8), FUNCTION 4 ei_false(oid);
CREATE OPERATOR CLASS text_squat FOR TYPE text USING btree AS
  OPERATOR 1 <(text,text), OPERATOR 2 <=(text,text), OPERATOR 3 =(text,text),
  OPERATOR 4 >=(text,text), OPERATOR 5 >(text,text),
  FUNCTION 1 bttextcmp(text,text), FUNCTION 4 public.btequalimage(oid);
CREATE OPERATOR CLASS text_renamed FOR TYPE text USING btree AS
  OPERATOR 1 <(text,text), OPERATOR 2 <=(text,text), OPERATOR 3 =(text,text),
  OPERATOR 4 >=(text,text), OPERATOR 5 >(text,text),
  FUNCTION 1 bttextcmp(text,text), FUNCTION 4 ei_renamed(oid);

CREATE TABLE t AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s, ((i % 5000)::numeric) AS n,
       (i % 5000)::float4 AS f4, (i % 5000)::float8 AS f8, (i % 7)::int4 AS d
  FROM generate_series(1, 500000) i;
CREATE TABLE t2 AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;

SET client_min_messages = debug1;          -- logs the engine's own verdict
CREATE INDEX i_int4          ON t (a);                                   -- 1
CREATE INDEX i_int8          ON t (b);                                   -- 2
CREATE INDEX i_text_det      ON t (s);                                   -- 3
CREATE INDEX i_text_det2     ON t (s text_renamed);                      -- 16
CREATE INDEX i_text_icu_det  ON t (s COLLATE cdet);                      -- 3
CREATE INDEX i_text_nondet   ON t (s COLLATE ci);                        -- 4
CREATE INDEX i_numeric       ON t (n);                                   -- 5
CREATE INDEX i_float4        ON t (f4);                                  -- 6
CREATE INDEX i_float8        ON t (f8);                                  -- 6
CREATE INDEX i_multi_ok      ON t (a, b);                                -- 7
CREATE INDEX i_multi_bad     ON t (a, n);                                -- 8
CREATE INDEX i_expr_lower_ci ON t ((lower(s)) COLLATE ci);               -- 9
CREATE INDEX i_expr_num      ON t ((a::numeric));                        -- 9
CREATE INDEX i_inc           ON t (a) INCLUDE (d);                       -- 10
CREATE INDEX i_dupoff        ON t (a) WITH (deduplicate_items = off);    -- 11
CREATE INDEX i_text_off      ON t (s) WITH (deduplicate_items = off);    -- 11
CREATE INDEX i_ei_none       ON t (a int4_ei_none);                      -- 12
CREATE INDEX i_ei_false      ON t (a int4_ei_false);                     -- 13
CREATE INDEX i_ei_true       ON t (a int4_ei_true);                      -- 14
CREATE INDEX i_ei_alias      ON t (a int4_ei_alias);                     -- 14
CREATE INDEX i_mixed_tf      ON t (a int4_ei_true, b int8_ei_false);     -- 15
CREATE INDEX i_mixed_ft      ON t (a int4_ei_false, b int8_ei_true);     -- 15
CREATE INDEX i_squat         ON t (s text_squat);                        -- 16
CREATE UNIQUE INDEX i_uniq   ON t (u);
CREATE INDEX i2_ok           ON t2 (a, b);                               -- 7
CREATE INDEX i2_off          ON t2 (s) WITH (deduplicate_items = off);   -- 11
CREATE INDEX i2_tf           ON t2 (a int4_ei_true, b int8_ei_false);    -- 15
CREATE INDEX i2_ft           ON t2 (a int4_ei_false, b int8_ei_true);    -- 15
RESET client_min_messages;
SELECT pg_stat_force_next_flush();
ANALYZE t, t2;
SELECT pg_stat_force_next_flush();
SQL
  [ $? -eq 0 ] || { tail -20 "$OUT/gate_build.log" >&2; die "gate fixtures failed"; }

  # test 4 and the pattern-opclass refusal, measured rather than derived.  The
  # refusal is the expected outcome, so the file says so above the error text;
  # a bare ERROR line at the top of a result file reads like a failure.
  q gate "CREATE INDEX i_pattern_nondet ON t (s COLLATE ci text_pattern_ops)" \
    > "$OUT/gate_pattern.txt" 2>&1 && note "text_pattern_ops accepted (unexpected)" \
    || note "text_pattern_ops refused as expected: $(tail -1 "$OUT/gate_pattern.txt")"
  { printf 'expected: test 4, text_pattern_ops refuses a nondeterministic collation\n'
    cat "$OUT/gate_pattern.txt"; } > "$OUT/gate_pattern.tmp" \
    && mv "$OUT/gate_pattern.tmp" "$OUT/gate_pattern.txt"

  f gate /dev/stdin <<'SQL'
DROP TABLE IF EXISTS gate_res;
CREATE TABLE gate_res AS
SELECT e.indexname, e.equalimage, e.wasted_space_pct AS wsp,
       e.wasted_space_pct_floor AS wspf, e.dedup_applies, e.tids_per_tuple,
       e.actual_bytes / 8192 AS blocks, e.caveats,
       (bt_metap(e.indexname)).allequalimage AS metapage,
       EXISTS (SELECT 1 FROM bt_page_items(e.indexname, 1) bi WHERE bi.tids IS NOT NULL)
                                                              AS posting_written
  FROM est_r2 e
 WHERE e.schemaname = 'public'
 ORDER BY e.indexname;
SQL
  t gate "SELECT /* wiki_btree_gate_rows */
                 indexname, blocks, equalimage, metapage, dedup_applies AS credited,
                 posting_written, wsp, wspf, tids_per_tuple, caveats
            FROM gate_res ORDER BY indexname" > "$OUT/gate.txt" 2>&1
  t gate "SELECT /* wiki_btree_gate_counters */
                 count(*) AS fixtures,
                 count(*) FILTER (WHERE dedup_applies AND NOT metapage) AS over_credit,
                 count(*) FILTER (WHERE equalimage = 'recognized' AND NOT metapage) AS recognized_wrong,
                 count(*) FILTER (WHERE equalimage = 'ineligible' AND metapage)     AS ineligible_wrong,
                 count(*) FILTER (WHERE equalimage = 'unknown' AND metapage)        AS under_credit,
                 max(greatest(wsp, wspf)) AS worst_reading
            FROM gate_res" >> "$OUT/gate.txt" 2>&1
  grep -c 'can safely use deduplication' "$OUT/gate_build.log" \
    | xargs printf 'DEBUG1 can safely use deduplication: %s\n' >> "$OUT/gate.txt"
  grep -c 'cannot use deduplication' "$OUT/gate_build.log" \
    | xargs printf 'DEBUG1 cannot use deduplication:      %s\n' >> "$OUT/gate.txt"
  tail -14 "$OUT/gate.txt" >&2
}

# ------------------------------------------------------------- acceptance ----
stage_acceptance() {
  say "acceptance fixtures: fresh builds, defects, compression, posting tails, probes, barrier"
  q acc 'CREATE EXTENSION IF NOT EXISTS pageinspect'
  q acc 'CREATE EXTENSION IF NOT EXISTS pgstattuple'
  # Disposable fixtures: the block below drops and recreates tables, forges two
  # pg_class rows and drops and recreates the wiki_reader login role in the
  # sandbox cluster.  It is not meant for a database anyone cares about.
  f acc /dev/stdin <<'SQL'
SET client_min_messages = warning;
DROP TABLE IF EXISTS fresh_res, tail_res, cmp_res, bar_res CASCADE;

-- 1. ten fresh sorted builds, key widths 8 to 2000, PLAIN storage.  The
--    fixtures are built here and read in the next command, because a reading
--    taken inside the building transaction sees the table's statistics as they
--    were before its own ANALYZE.
DROP TABLE IF EXISTS fresh_plan;
CREATE TABLE fresh_plan(keylen int, rows_loaded int, idx text);
DO $fresh$
DECLARE l int; n int;
BEGIN
  FOREACH l IN ARRAY ARRAY[8,16,32,64,100,200,400,800,1000,2000] LOOP
    n := ((8144 - 819) / (((12 + l + 7) / 8) * 8 + 4)) * 250;
    EXECUTE format('DROP TABLE IF EXISTS fr%s', l);
    EXECUTE format('CREATE TABLE fr%s(k text)', l);
    EXECUTE format('ALTER TABLE fr%s ALTER COLUMN k SET STORAGE PLAIN', l);
    EXECUTE format('INSERT INTO fr%s SELECT lpad(i::text, %s, ''0'') FROM generate_series(1, %s) i', l, l, n);
    EXECUTE format('ANALYZE fr%s', l);
    EXECUTE format('CREATE INDEX fr_i%s ON fr%s (k)', l, l);
    INSERT INTO fresh_plan VALUES (l, n, 'fr_i' || l);
  END LOOP;
END $fresh$;
SELECT pg_stat_force_next_flush();
CREATE TABLE fresh_res AS
SELECT p.keylen, p.rows_loaded, e.actual_bytes / 8192 AS blocks,
       e.wasted_space_pct AS wsp, e.wasted_space_pct_floor AS wspf,
       e.wasted_space_bytes AS wasted_bytes, o.wasted_space_pct AS old_wsp,
       e.caveats
  FROM fresh_plan p
  JOIN est_r2  e ON e.indexname = p.idx
  JOIN est_old o ON o.indexname = p.idx
 ORDER BY p.keylen;

-- 2. the three deterministic defects.
--    a. an inheritance parent: two pg_stats rows per attribute.
DROP TABLE IF EXISTS inh_c, inh_p CASCADE;
CREATE TABLE inh_p(s text);
CREATE TABLE inh_c(LIKE inh_p) INHERITS (inh_p);
INSERT INTO inh_p SELECT lpad(i::text, 20, '0') FROM generate_series(1, 200000) i;
INSERT INTO inh_c SELECT lpad(i::text, 76, '0') FROM generate_series(1, 200000) i;
CREATE INDEX inh_i ON ONLY inh_p (s);
SELECT pg_stat_force_next_flush();
ANALYZE inh_p;                    -- writes both the inherited and own passes
SELECT pg_stat_force_next_flush();

--    b. an expression index whose statistics only the owner can read.
DROP TABLE IF EXISTS expr_t CASCADE;
CREATE TABLE expr_t(txt text);
INSERT INTO expr_t SELECT lpad(i::text, 20, 'a') FROM generate_series(1, 200000) i;
CREATE INDEX expr_i ON expr_t (lower(txt));
SELECT pg_stat_force_next_flush();
ANALYZE expr_t;
SELECT pg_stat_force_next_flush();
DO $role$
BEGIN
  IF EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'wiki_reader') THEN
    EXECUTE 'DROP OWNED BY wiki_reader';       -- also drops privileges granted
    EXECUTE 'DROP ROLE wiki_reader';
  END IF;
END $role$;
CREATE ROLE wiki_reader LOGIN;
GRANT USAGE ON SCHEMA public TO wiki_reader;
GRANT SELECT ON expr_t TO wiki_reader;

--    c. an index whose reltuples is past the bigint range.
DROP TABLE IF EXISTS ovf_t CASCADE;
CREATE TABLE ovf_t(k int);
INSERT INTO ovf_t SELECT i FROM generate_series(1, 200000) i;
CREATE INDEX ovf_idx ON ovf_t (k);
SELECT pg_stat_force_next_flush();
ANALYZE ovf_t;
SELECT pg_stat_force_next_flush();
UPDATE pg_class SET reltuples = 1e30 WHERE relname = 'ovf_idx';

--    d. statistics target zero on an index column.
DROP TABLE IF EXISTS st0_t CASCADE;
CREATE TABLE st0_t(k text);
INSERT INTO st0_t SELECT lpad(i::text, 20, '0') FROM generate_series(1, 200000) i;
ALTER TABLE st0_t ALTER COLUMN k SET STATISTICS 0;
CREATE INDEX st0_i ON st0_t (k);
SELECT pg_stat_force_next_flush();
ANALYZE st0_t;
SELECT pg_stat_force_next_flush();

-- 3. in-index compression of a wide key: the same 900-byte values, two storages.
CREATE TABLE cmp_res(storage text, avg_width int, blocks int, wsp numeric, caveats text);
DROP TABLE IF EXISTS cmp_x, cmp_p CASCADE;
CREATE TABLE cmp_x(k text);                       -- extended: index compresses
CREATE TABLE cmp_p(k text);                       -- plain: it cannot
ALTER TABLE cmp_p ALTER COLUMN k SET STORAGE PLAIN;
INSERT INTO cmp_x SELECT lpad(i::text, 900, 'x') FROM generate_series(1, 60000) i;
INSERT INTO cmp_p SELECT lpad(i::text, 900, 'x') FROM generate_series(1, 60000) i;
SELECT pg_stat_force_next_flush();
ANALYZE cmp_x, cmp_p;
SELECT pg_stat_force_next_flush();
CREATE INDEX cmp_x_i ON cmp_x (k);
CREATE INDEX cmp_p_i ON cmp_p (k);
INSERT INTO cmp_res
SELECT CASE WHEN e.indexname = 'cmp_x_i' THEN 'extended' ELSE 'plain' END,
       (SELECT avg_width FROM pg_stats WHERE tablename = e.tablename AND attname = 'k'),
       e.actual_bytes / 8192, e.wasted_space_pct, e.caveats
  FROM est_r2 e WHERE e.indexname IN ('cmp_x_i', 'cmp_p_i');

-- 4. posting tails: thirteen duplicate classes with different tail sizes.
DROP TABLE IF EXISTS tail_plan;
CREATE TABLE tail_plan(rows_per_group int, idx text);
DO $tail$
DECLARE g int;
BEGIN
  FOREACH g IN ARRAY ARRAY[1,2,3,5,8,13,32,66,131,132,133,264,400] LOOP
    EXECUTE format('DROP TABLE IF EXISTS pt%s', g);
    EXECUTE format('CREATE TABLE pt%s AS SELECT (i / %s)::int AS k FROM generate_series(1, 400000) i', g, g);
    EXECUTE format('ANALYZE pt%s', g);
    EXECUTE format('CREATE INDEX pt_i%s ON pt%s (k)', g, g);
    INSERT INTO tail_plan VALUES (g, 'pt_i' || g);
  END LOOP;
END $tail$;
SELECT pg_stat_force_next_flush();
CREATE TABLE tail_res AS
SELECT p.rows_per_group, e.actual_bytes / 8192 AS blocks,
       e.wasted_space_pct AS wsp, e.wasted_space_pct_floor AS wspf,
       o.wasted_space_pct AS old_wsp, e.tids_per_tuple AS tids
  FROM tail_plan p
  JOIN est_r2  e ON e.indexname = p.idx
  JOIN est_old o ON o.indexname = p.idx
 ORDER BY p.rows_per_group;

-- 5. probe fixtures: an empty subset, a forged zero, and 5,000 real groups.
DROP TABLE IF EXISTS pr_empty, pr_forged, pr_groups, pr_num CASCADE;
CREATE TABLE pr_empty(k int, open bool);
INSERT INTO pr_empty SELECT i, false FROM generate_series(1, 300000) i;
CREATE INDEX empty_open ON pr_empty (k) WHERE open;
CREATE TABLE pr_forged(k int, open bool);
INSERT INTO pr_forged SELECT i, true FROM generate_series(1, 300000) i;
CREATE INDEX forged_open ON pr_forged (k) WHERE open;
CREATE TABLE pr_groups(k int);
INSERT INTO pr_groups SELECT (i % 5000)::int FROM generate_series(1, 300000) i;
CREATE INDEX groups_k ON pr_groups (k);
CREATE TABLE pr_num(n numeric);
INSERT INTO pr_num SELECT (i % 5000)::numeric FROM generate_series(1, 300000) i;
CREATE INDEX num_n ON pr_num (n);
SELECT pg_stat_force_next_flush();
ANALYZE pr_empty, pr_forged, pr_groups, pr_num;
SELECT pg_stat_force_next_flush();
UPDATE pg_class SET reltuples = 0 WHERE relname = 'forged_open';

-- 6. the statistics publication barrier: the same load, with and without it.
CREATE TABLE bar_res(leg text, n_mod_since_analyze bigint, n_live_tup bigint);
DROP TABLE IF EXISTS bar_a, bar_b CASCADE;
CREATE TABLE bar_a(k int);
INSERT INTO bar_a SELECT i FROM generate_series(1, 200000) i;
ANALYZE bar_a;                                  -- no flush first
CREATE TABLE bar_b(k int);
INSERT INTO bar_b SELECT i FROM generate_series(1, 200000) i;
SELECT pg_stat_force_next_flush();              -- the barrier
ANALYZE bar_b;
SELECT pg_stat_force_next_flush();
INSERT INTO bar_res
SELECT CASE WHEN relname = 'bar_a' THEN 'no barrier' ELSE 'barrier' END,
       n_mod_since_analyze, n_live_tup
  FROM pg_stat_all_tables WHERE relname IN ('bar_a', 'bar_b');
SQL
  [ $? -eq 0 ] || die "acceptance fixtures failed"

  t acc 'SELECT /* wiki_btree_fresh_builds */ * FROM fresh_res ORDER BY keylen' \
    > "$OUT/acceptance.txt" 2>&1
  t acc 'SELECT /* wiki_btree_posting_tails */ * FROM tail_res ORDER BY rows_per_group' \
    >> "$OUT/acceptance.txt" 2>&1
  t acc 'SELECT /* wiki_btree_compression */ * FROM cmp_res ORDER BY storage' \
    >> "$OUT/acceptance.txt" 2>&1
  t acc 'SELECT /* wiki_btree_stats_barrier */ * FROM bar_res ORDER BY leg' \
    >> "$OUT/acceptance.txt" 2>&1
  t acc "SELECT /* wiki_btree_defect_fixtures */
                indexname, actual_bytes/8192 AS blocks, status, wasted_space_pct AS wsp,
                wasted_space_pct_floor AS wspf, caveats
           FROM est_r2 WHERE indexname IN ('inh_i','expr_i','ovf_idx','st0_i')
          ORDER BY indexname" >> "$OUT/acceptance.txt" 2>&1

  say "the three deterministic defects, current text against the superseded one"
  t acc "SELECT /* wiki_btree_defects_side_by_side */
                e.indexname, e.actual_bytes/8192 AS blocks,
                o.wasted_space_pct AS old_wsp, e.wasted_space_pct AS r2_wsp,
                e.caveats
           FROM est_r2 e JOIN est_old o USING (indexname)
          WHERE e.indexname IN ('inh_i','expr_i','st0_i') ORDER BY 1" \
    > "$OUT/defects.txt" 2>&1
  { printf '\n-- superseded text over the whole database (bigint range defect)\n'
    f acc "$SQLD/est_old.sql" 2>&1
    printf '\n-- current text over the whole database, exact as filed\n'
    f acc "$SQLD/est_r2.sql" 2>&1
    printf '\n-- current text as a role holding only SELECT on the tables\n'
    ( export PGUSER=wiki_reader; f acc "$SQLD/est_r2.sql" 2>&1 )
  } >> "$OUT/defects.txt" 2>&1
  tail -6 "$OUT/acceptance.txt" >&2
  head -8 "$OUT/defects.txt" >&2
}

# ---------------------------------------------------------------- suite ------
stage_suite() {
  say "the numbered suite: 74 partial-index fixtures and controls 92-121"
  # A clean schema makes the stage idempotent; the views go back in first.
  q suite 'DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public'
  f suite "$SQLD/view_r2.sql"  || die "est_r2 view failed"
  f suite "$SQLD/view_old.sql" || die "est_old view failed"
  cat > "$SQLD/fixtures_suite.sql" <<'FIXTURES'
-- The numbered suite: tests 18-91 (partial indexes) and controls 92-121.
-- One fixture per requirement, built in the prescribed order, planned here and
-- scored later so that the EXCEPT attribution and the probes both run before
-- the first REINDEX.  pg_stat_force_next_flush() precedes every ANALYZE and
-- every VACUUM, and WITH (fillfactor = ...) precedes WHERE in CREATE INDEX.
--
-- Disposable fixtures.  Everything below creates, forges and drops objects in
-- the suite database of the sandbox cluster, whose public schema stage_suite
-- has just dropped.  It is not meant for a database anyone cares about.
SET /* wiki_btree_suite_client_min_messages */ client_min_messages = warning;
SET /* wiki_btree_suite_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btree_suite_lock_timeout */ lock_timeout = '2s';
SET /* wiki_btree_suite_maintenance_work_mem */ maintenance_work_mem = '256MB';


CREATE TABLE plan(num int, leg text DEFAULT '', req text, idx text,
                  rowsql text, want_rows bigint, note text,
                  PRIMARY KEY (num, leg));

CREATE TABLE res(num int, leg text, req text, idx text,
                 size_before bigint, size_after bigint,
                 blocks_before int, blocks_after int,
                 status text, wsp numeric, wspf numeric, wasted_bytes numeric,
                 caveats text, equalimage text, reltuples_writer text,
                 modelled_rows numeric, key_groups numeric, tids numeric,
                 idx_reltuples numeric, exp_blocks numeric, floor_blocks numeric,
                 slot numeric, leaf_cap numeric, nmax numeric,
                 dedup_applies bool, is_partial bool, has_expressions bool,
                 suppress_row bool, stats_row_missing bool, dedup_credited bool,
                 stats_stale bool, any_varlena_include bool,
                 old_wsp numeric, old_wspf numeric,
                 true_rows bigint, want_rows bigint, note text,
                 PRIMARY KEY (num, leg));

CREATE OR REPLACE FUNCTION plan_add(n int, r text, i text, q text DEFAULT NULL,
                                    w bigint DEFAULT NULL, lg text DEFAULT '',
                                    nt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS
$$ INSERT INTO plan(num, leg, req, idx, rowsql, want_rows, note)
   VALUES (n, lg, r, i, q, w, nt) $$;

CREATE OR REPLACE PROCEDURE score_all() LANGUAGE plpgsql AS $sc$
DECLARE p record; e record; sb bigint; sa bigint; tr bigint;
        ow numeric; owf numeric;
BEGIN
  FOR p IN SELECT * FROM plan ORDER BY num, leg LOOP
    tr := NULL; ow := NULL; owf := NULL;
    IF p.rowsql IS NOT NULL THEN EXECUTE p.rowsql INTO tr; END IF;
    sb := pg_relation_size(p.idx::regclass);
    SELECT * INTO e FROM est_r2 WHERE indexname = p.idx;
    IF NOT FOUND THEN RAISE EXCEPTION 'estimator returned no row for %', p.idx; END IF;
    BEGIN
      SELECT o.wasted_space_pct, o.wasted_space_pct_floor INTO ow, owf
        FROM est_old o WHERE o.indexname = p.idx;
    EXCEPTION WHEN OTHERS THEN ow := NULL; owf := NULL;
    END;
    EXECUTE format('REINDEX INDEX %I', p.idx);
    sa := pg_relation_size(p.idx::regclass);
    INSERT INTO res VALUES (p.num, p.leg, p.req, p.idx, sb, sa, sb / 8192, sa / 8192,
      e.status, e.wasted_space_pct, e.wasted_space_pct_floor, e.wasted_space_bytes,
      e.caveats, e.equalimage, e.reltuples_writer, e.modelled_rows, e.key_groups,
      e.tids_per_tuple, e.idx_reltuples, e.expected_blocks, e.floor_blocks,
      e.slot, e.leaf_cap, e.nmax, e.dedup_applies, e.is_partial, e.has_expressions,
      e.suppress_row, e.stats_row_missing, e.dedup_credited, e.stats_stale,
      e.any_varlena_include, ow, owf, tr, p.want_rows, p.note);
  END LOOP;
END $sc$;

CREATE VIEW verdicts AS
SELECT r.num, r.leg, r.idx, r.req, r.blocks_before, r.blocks_after, a.actual,
       r.wsp, r.wspf, r.old_wsp, r.old_wspf, v.verdict_point, v.verdict_floor,
       (r.caveats IS NULL OR r.caveats !~
        '(never analyzed|row-count sources disagree|statistics not visible|zero modelled rows|wide compressible key)')
                                                          AS alertable,
       NOT r.suppress_row                                 AS reported,
       CASE WHEN NOT r.suppress_row                             THEN NULL
            WHEN r.is_partial AND r.stats_row_missing           THEN 'A: no statistics row'
            WHEN r.is_partial AND r.dedup_credited              THEN 'A: duplicates from table statistics'
            WHEN r.is_partial AND r.stats_stale                 THEN 'B: changed since ANALYZE'
            WHEN r.is_partial AND r.any_varlena_include         THEN 'C: variable-width INCLUDE'
            WHEN r.has_expressions AND r.stats_row_missing      THEN 'D: expression, no statistics row'
            ELSE 'unexplained' END                        AS withheld_by,
       (r.want_rows IS NULL OR r.true_rows = r.want_rows) AS contract_ok,
       r.true_rows, r.want_rows, r.modelled_rows, r.idx_reltuples, r.status,
       r.caveats, r.equalimage, r.tids, r.note
  FROM res r
  CROSS JOIN LATERAL (
        SELECT round(100.0 * (r.size_before - r.size_after)
                     / greatest(r.size_before, 1), 1) AS actual) a
  CROSS JOIN LATERAL (
        SELECT CASE WHEN r.wsp IS NULL                       THEN 'UNMEASURED'
                    WHEN r.wsp >= 50 AND a.actual < 10       THEN 'CRITICAL FALSE POSITIVE'
                    WHEN r.wsp >= 50 AND a.actual < 45       THEN 'FALSE POSITIVE'
                    WHEN r.wsp >= 50 AND r.wsp - a.actual > 5 THEN 'FALSE POSITIVE'
                    WHEN r.wsp <  45 AND a.actual >= 50      THEN 'FALSE NEGATIVE'
                    ELSE 'PASS' END                          AS verdict_point,
               CASE WHEN r.wspf IS NULL                      THEN 'UNMEASURED'
                    WHEN r.wspf >= 50 AND a.actual < 10      THEN 'CRITICAL FALSE POSITIVE'
                    WHEN r.wspf >= 50 AND a.actual < 45      THEN 'FALSE POSITIVE'
                    WHEN r.wspf >= 50 AND r.wspf - a.actual > 5 THEN 'FALSE POSITIVE'
                    WHEN r.wspf <  45 AND a.actual >= 50     THEN 'FALSE NEGATIVE'
                    ELSE 'PASS' END                          AS verdict_floor) v;

-- ============================================================ 18-21 =========
-- Predicate selectivity.  One 1,000,000-row table, distinct bigint keys.
CREATE TABLE pt1 AS
SELECT i::bigint AS k, (i % 100)::int AS sel FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pt1; SELECT pg_stat_force_next_flush();
CREATE INDEX p18 ON pt1 (k) WHERE sel < 20;
CREATE INDEX p19 ON pt1 (k) WHERE sel < 1;
CREATE INDEX p20 ON pt1 (k) WHERE sel < 10;
CREATE INDEX p21 ON pt1 (k) WHERE sel < 80;
SELECT plan_add(18, 'baseline, subset distribution = table (20%)', 'p18',
                'SELECT count(*) FROM pt1 WHERE sel < 20', 200000);
SELECT plan_add(19, 'very selective, ~1%', 'p19',
                'SELECT count(*) FROM pt1 WHERE sel < 1', 10000);
SELECT plan_add(20, 'moderately selective, ~10%', 'p20',
                'SELECT count(*) FROM pt1 WHERE sel < 10', 100000);
SELECT plan_add(21, 'large subset, ~80%', 'p21',
                'SELECT count(*) FROM pt1 WHERE sel < 80', 800000);

-- ============================================================ 22-33 =========
CREATE TABLE pd22 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd22; SELECT pg_stat_force_next_flush();
CREATE INDEX p22 ON pd22 (k) WHERE hot;
SELECT plan_add(22, 'highly duplicated subset, unique outside', 'p22',
                'SELECT count(*) FROM pd22 WHERE hot', 100000);

CREATE TABLE pd23 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE ((i / 5) % 100)::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd23; SELECT pg_stat_force_next_flush();
CREATE INDEX p23 ON pd23 (k) WHERE hot;
SELECT plan_add(23, 'highly unique subset, duplicated outside', 'p23',
                'SELECT count(*) FROM pd23 WHERE hot', 100000);

CREATE TABLE pd24 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50000)::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd24; SELECT pg_stat_force_next_flush();
CREATE INDEX p24 ON pd24 (k) WHERE hot;
SELECT plan_add(24, 'n_distinct radically different in the subset', 'p24',
                'SELECT count(*) FROM pd24 WHERE hot', 100000);

CREATE TABLE pd25 AS SELECT (i % 100 = 0) AS hot,
       CASE WHEN i % 100 = 0 THEN ((i / 100) % 997)::int ELSE (i % 5)::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd25; SELECT pg_stat_force_next_flush();
CREATE INDEX p25 ON pd25 (k) WHERE hot;
SELECT plan_add(25, 'MCV distribution differs inside the subset', 'p25',
                'SELECT count(*) FROM pd25 WHERE hot', 5000);

CREATE TABLE pd26 AS SELECT (i % 50 = 0) AS hot,
       CASE WHEN i % 50 = 0 THEN (1000000 + i)::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd26; SELECT pg_stat_force_next_flush();
CREATE INDEX p26 ON pd26 (k) WHERE hot;
SELECT plan_add(26, 'table-wide MCVs absent inside the subset', 'p26',
                'SELECT count(*) FROM pd26 WHERE hot', 10000);

CREATE TABLE pd27 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 AND i % 100 <> 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd27; SELECT pg_stat_force_next_flush();
CREATE INDEX p27 ON pd27 (k) WHERE hot;
SELECT plan_add(27, 'NULL-heavy subset, non-NULL outside', 'p27',
                'SELECT count(*) FROM pd27 WHERE hot', 100000);

CREATE TABLE pd28 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN i::bigint ELSE NULL END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd28; SELECT pg_stat_force_next_flush();
CREATE INDEX p28 ON pd28 (k) WHERE hot;
SELECT plan_add(28, 'NULL-free subset, NULL-heavy table (bigint)', 'p28',
                'SELECT count(*) FROM pd28 WHERE hot', 25000);

CREATE TABLE pd29 AS
SELECT CASE WHEN i % 5 = 0 THEN NULL ELSE lpad(i::text, 20, '0') END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd29; SELECT pg_stat_force_next_flush();
CREATE INDEX p29 ON pd29 (s) WHERE s IS NULL;
SELECT plan_add(29, 'all-NULL partial index, WHERE s IS NULL', 'p29',
                'SELECT count(*) FROM pd29 WHERE s IS NULL', 100000);

CREATE TABLE pd30 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd30; SELECT pg_stat_force_next_flush();
CREATE INDEX p30 ON pd30 (s) WHERE hot;
SELECT plan_add(30, 'subset values wider than outside (13 against 204 bytes)', 'p30',
                'SELECT count(*) FROM pd30 WHERE hot', 25000);

CREATE TABLE pd31 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN lpad((i % 9)::text, 12, 'n')
            ELSE repeat('W', 190) || lpad(i::text, 10, '0') END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd31; SELECT pg_stat_force_next_flush();
CREATE INDEX p31 ON pd31 (s) WHERE hot;
SELECT plan_add(31, 'subset values narrower than outside', 'p31',
                'SELECT count(*) FROM pd31 WHERE hot', 25000);

-- 32 is the page's published recipe, verbatim.
CREATE TABLE pw32 AS
SELECT (i % 50 = 0) AS hot,
       CASE WHEN i % 50 = 0 THEN repeat('W', 390) || lpad(i::text, 10, '0')
            ELSE repeat('n', 18) || (i % 9)::text END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pw32; SELECT pg_stat_force_next_flush();
CREATE INDEX p32 ON pw32 (s) WHERE hot;
SELECT plan_add(32, 'extreme width mismatch (27 against 404 bytes)', 'p32',
                'SELECT count(*) FROM pw32 WHERE hot', 10000);

CREATE TABLE pd33 AS SELECT (i % 5 = 0) AS hot,
       lpad(i::text, 10 + (i % 40), 'x') AS s FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd33; SELECT pg_stat_force_next_flush();
CREATE INDEX p33 ON pd33 (s) WHERE hot;
SELECT plan_add(33, 'variable-width values, same range inside and out', 'p33',
                'SELECT count(*) FROM pd33 WHERE hot', 100000);

-- ============================================================ 34-39 =========
CREATE TABLE pd34 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd34; SELECT pg_stat_force_next_flush();
CREATE INDEX p34 ON pd34 (k) WHERE hot;
SELECT plan_add(34, 'dedup-heavy subset, 1000 rows per key', 'p34',
                'SELECT count(*) FROM pd34 WHERE hot', 100000);

CREATE TABLE pd35 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd35; SELECT pg_stat_force_next_flush();
CREATE INDEX p35 ON pd35 (k) WHERE hot;
SELECT plan_add(35, 'duplicate-heavy table, unique subset', 'p35',
                'SELECT count(*) FROM pd35 WHERE hot', 100000);

CREATE TABLE pd36 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN 42 ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd36; SELECT pg_stat_force_next_flush();
CREATE INDEX p36 ON pd36 (k) WHERE hot;
SELECT plan_add(36, 'one key group, 100,000 TIDs against a 132 cap', 'p36',
                'SELECT count(*) FROM pd36 WHERE hot', 100000);

CREATE TABLE pd37 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd37; SELECT pg_stat_force_next_flush();
CREATE INDEX p37 ON pd37 (k) WHERE hot;
SELECT plan_add(37, 'NULL deduplication, every subset key NULL', 'p37',
                'SELECT count(*) FROM pd37 WHERE hot', 100000);

CREATE TABLE pd38 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd38; SELECT pg_stat_force_next_flush();
CREATE INDEX p38 ON pd38 (k) WITH (deduplicate_items = off) WHERE hot;
SELECT plan_add(38, 'deduplicate_items = off', 'p38',
                'SELECT count(*) FROM pd38 WHERE hot', 100000);

CREATE TABLE pd39 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd39; SELECT pg_stat_force_next_flush();
CREATE UNIQUE INDEX p39 ON pd39 (k) WHERE hot;
SELECT plan_add(39, 'partial UNIQUE index', 'p39',
                'SELECT count(*) FROM pd39 WHERE hot', 100000);

-- ============================================================ 40-47 =========
CREATE TABLE pd40 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 97)::int  END AS b
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd40; SELECT pg_stat_force_next_flush();
CREATE INDEX p40 ON pd40 (a, b) WHERE hot;
SELECT plan_add(40, 'two-column key correlated only in the subset', 'p40',
                'SELECT count(*) FROM pd40 WHERE hot', 100000);

CREATE TABLE pd41 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 97)::int  ELSE (i % 100)::int END AS b
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd41; SELECT pg_stat_force_next_flush();
CREATE INDEX p41 ON pd41 (a, b) WHERE hot;
SELECT plan_add(41, 'two-column key independent only in the subset', 'p41',
                'SELECT count(*) FROM pd41 WHERE hot', 100000);

CREATE TABLE pd42 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50)::int ELSE i::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50)::int ELSE i::int END AS b
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd42; SELECT pg_stat_force_next_flush();
CREATE INDEX p42 ON pd42 (a, b) WHERE hot;
SELECT plan_add(42, 'multi-column duplicate keys in the subset', 'p42',
                'SELECT count(*) FROM pd42 WHERE hot', 100000);

CREATE TABLE pd43 AS SELECT (i % 5 = 0) AS hot, i::int AS a, (i * 2)::int AS b
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd43; SELECT pg_stat_force_next_flush();
CREATE INDEX p43 ON pd43 (a, b) WHERE hot;
SELECT plan_add(43, 'multi-column unique keys in the subset', 'p43',
                'SELECT count(*) FROM pd43 WHERE hot', 100000);

-- 44: the same correlated shape with and without a CREATE STATISTICS object.
--     Two tables, because one ANALYZE would repair both legs at once.
CREATE TABLE pd44a AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd44a; SELECT pg_stat_force_next_flush();
CREATE INDEX p44a ON pd44a (a, b) WHERE hot;
SELECT plan_add(44, 'multicolumn key, no ndistinct object', 'p44a',
                'SELECT count(*) FROM pd44a WHERE hot', 100000);
CREATE TABLE pd44b AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS pd44b_nd (ndistinct) ON a, b FROM pd44b;
SELECT pg_stat_force_next_flush(); ANALYZE pd44b; SELECT pg_stat_force_next_flush();
CREATE INDEX p44b ON pd44b (a, b) WHERE hot;
SELECT plan_add(44, 'multicolumn key, with CREATE STATISTICS (ndistinct)', 'p44b',
                'SELECT count(*) FROM pd44b WHERE hot', 100000, 'ndistinct');

CREATE TABLE pd45 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 97)::int  ELSE (i % 100)::int END AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS pd45_nd (ndistinct) ON a, b FROM pd45;
SELECT pg_stat_force_next_flush(); ANALYZE pd45; SELECT pg_stat_force_next_flush();
CREATE INDEX p45 ON pd45 (a, b) WHERE hot;
SELECT plan_add(45, 'extended statistics wrong for the subset', 'p45',
                'SELECT count(*) FROM pd45 WHERE hot', 100000);

CREATE TABLE pd46 AS SELECT (i % 5 = 0) AS hot, i::int AS k, (i % 7)::int AS pay
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pd46; SELECT pg_stat_force_next_flush();
CREATE INDEX p46 ON pd46 (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(46, 'partial index with INCLUDE columns', 'p46',
                'SELECT count(*) FROM pd46 WHERE hot', 100000);

CREATE TABLE pi47 AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS payload
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pi47; SELECT pg_stat_force_next_flush();
CREATE INDEX p47 ON pi47 (k) INCLUDE (payload) WHERE hot;
SELECT plan_add(47, 'wide INCLUDE values inside the subset', 'p47',
                'SELECT count(*) FROM pi47 WHERE hot', 25000);

-- ============================================================ 48-55 =========
-- Expression legs come in twins: the '' leg has no statistics row for the
-- expression, the 'after analyze' leg has one.
CREATE TABLE pe48 AS SELECT (i % 5 = 0) AS active,
       CASE WHEN i % 5 = 0 THEN 'NAME' || lpad(((i / 5) % 20)::text, 6, '0')
            ELSE 'name' || lpad((i % 100)::text, 6, '0') END AS name
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pe48; SELECT pg_stat_force_next_flush();
CREATE INDEX p48 ON pe48 (lower(name)) WHERE active;
SELECT plan_add(48, 'partial expression index, lower(name) WHERE active', 'p48',
                'SELECT count(*) FROM pe48 WHERE active', 100000);
CREATE TABLE pe48b AS SELECT (i % 5 = 0) AS active,
       CASE WHEN i % 5 = 0 THEN 'NAME' || lpad(((i / 5) % 20)::text, 6, '0')
            ELSE 'name' || lpad((i % 100)::text, 6, '0') END AS name
  FROM generate_series(1, 500000) i;
CREATE INDEX p48b ON pe48b (lower(name)) WHERE active;
SELECT pg_stat_force_next_flush(); ANALYZE pe48b; SELECT pg_stat_force_next_flush();
SELECT plan_add(48, 'the same after one ANALYZE with the index in place', 'p48b',
                'SELECT count(*) FROM pe48b WHERE active', 100000, 'after analyze');

CREATE TABLE pe49 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pe49; SELECT pg_stat_force_next_flush();
CREATE INDEX p49 ON pe49 (upper(s)) WHERE hot;
SELECT plan_add(49, 'expression width mismatch in the subset', 'p49',
                'SELECT count(*) FROM pe49 WHERE hot', 25000);
CREATE TABLE pe49b AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX p49b ON pe49b (upper(s)) WHERE hot;
SELECT pg_stat_force_next_flush(); ANALYZE pe49b; SELECT pg_stat_force_next_flush();
SELECT plan_add(49, 'the same after one ANALYZE with the index in place', 'p49b',
                'SELECT count(*) FROM pe49b WHERE hot', 25000, 'after analyze');

CREATE TABLE pe50 AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pe50; SELECT pg_stat_force_next_flush();
CREATE INDEX p50 ON pe50 (upper(s)) WHERE hot;   -- real width 101, fallback 32
SELECT plan_add(50, 'missing expression statistics, 32-byte fallback', 'p50',
                'SELECT count(*) FROM pe50 WHERE hot', 100000);
CREATE TABLE pe50b AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX p50b ON pe50b (upper(s)) WHERE hot;
SELECT pg_stat_force_next_flush(); ANALYZE pe50b; SELECT pg_stat_force_next_flush();
SELECT plan_add(50, 'the same after one ANALYZE with the index in place', 'p50b',
                'SELECT count(*) FROM pe50b WHERE hot', 100000, 'after analyze');

CREATE COLLATION suite_det    (provider = icu, locale = 'und');
CREATE COLLATION suite_nondet (provider = icu, locale = 'und-u-ks-level2',
                               deterministic = false);
CREATE TABLE pc51 AS SELECT (i % 5 = 0) AS hot,
       'key' || lpad(((i / 5) % 100)::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc51; SELECT pg_stat_force_next_flush();
CREATE INDEX p51 ON pc51 (s COLLATE suite_det) WHERE hot;
CREATE INDEX p52 ON pc51 (s COLLATE suite_nondet) WHERE hot;
SELECT plan_add(51, 'deterministic ICU collation', 'p51',
                'SELECT count(*) FROM pc51 WHERE hot', 100000);
SELECT plan_add(52, 'nondeterministic ICU collation', 'p52',
                'SELECT count(*) FROM pc51 WHERE hot', 100000);

CREATE TABLE pf AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pf; SELECT pg_stat_force_next_flush();
CREATE INDEX p53 ON pf (k) WHERE hot;
CREATE INDEX p54 ON pf (k) WITH (fillfactor = 100) WHERE hot;
CREATE INDEX p55 ON pf (k) WITH (fillfactor = 70)  WHERE hot;
SELECT plan_add(53, 'default fillfactor 90', 'p53', 'SELECT count(*) FROM pf WHERE hot', 100000);
SELECT plan_add(54, 'fillfactor = 100',      'p54', 'SELECT count(*) FROM pf WHERE hot', 100000);
SELECT plan_add(55, 'fillfactor = 70',       'p55', 'SELECT count(*) FROM pf WHERE hot', 100000);

-- ============================================================ 56-63 =========
CREATE TABLE ps AS
SELECT (i % 5 = 0) AS flag,
       CASE WHEN i % 5 = 0 THEN 'OPEN' ELSE 'CLOSED' END AS status,
       timestamptz '2020-01-01' + (i * interval '1 minute') AS created,
       CASE WHEN i % 5 = 0 THEN NULL ELSE i::int END AS nk,
       i::int AS k, (i % 1000)::int AS k2
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE ps; SELECT pg_stat_force_next_flush();
CREATE INDEX p56 ON ps (k) WHERE flag;
CREATE INDEX p57 ON ps (k) WHERE status = 'OPEN';
CREATE INDEX p58 ON ps (k) WHERE created >= timestamptz '2020-09-01';
CREATE INDEX p59 ON ps (k) WHERE nk IS NULL;
CREATE INDEX p60 ON ps (k) WHERE nk IS NOT NULL;
CREATE INDEX p61 ON ps (k) WHERE flag AND status = 'OPEN';
CREATE INDEX p62 ON ps (k) WHERE k < 100000;
CREATE INDEX p63 ON ps (k2) WHERE k >= 400000;
SELECT plan_add(56, 'boolean predicate, WHERE flag', 'p56', 'SELECT count(*) FROM ps WHERE flag', 100000);
SELECT plan_add(57, 'equality predicate, status = ''OPEN''', 'p57', 'SELECT count(*) FROM ps WHERE status = ''OPEN''', 100000);
SELECT plan_add(58, 'range predicate, created >= ...', 'p58', 'SELECT count(*) FROM ps WHERE created >= timestamptz ''2020-09-01''', NULL);
SELECT plan_add(59, 'IS NULL predicate on a non-key column', 'p59', 'SELECT count(*) FROM ps WHERE nk IS NULL', 100000);
SELECT plan_add(60, 'IS NOT NULL predicate', 'p60', 'SELECT count(*) FROM ps WHERE nk IS NOT NULL', 400000);
SELECT plan_add(61, 'multi-column predicate', 'p61', 'SELECT count(*) FROM ps WHERE flag AND status = ''OPEN''', 100000);
SELECT plan_add(62, 'predicate correlated with the indexed value', 'p62', 'SELECT count(*) FROM ps WHERE k < 100000', 99999);
SELECT plan_add(63, 'predicate negatively correlated with the value', 'p63', 'SELECT count(*) FROM ps WHERE k >= 400000', 100001);

-- ============================================================ 64-69 =========
-- 64: stale statistics after inserts into the subset.
CREATE TABLE pc64 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc64; SELECT pg_stat_force_next_flush();
CREATE INDEX p64 ON pc64 (k) WHERE hot;
INSERT INTO pc64 SELECT true, 500000 + i FROM generate_series(1, 200000) i;
SELECT pg_stat_force_next_flush();
SELECT plan_add(64, 'stale statistics after inserts into the subset', 'p64',
                'SELECT count(*) FROM pc64 WHERE hot', 300000);

-- 65: stale statistics after deletes, no VACUUM.
CREATE TABLE pc65 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc65; SELECT pg_stat_force_next_flush();
CREATE INDEX p65 ON pc65 (k) WHERE hot;
DELETE FROM pc65 WHERE hot AND k % 50 <> 0;
SELECT pg_stat_force_next_flush();
SELECT plan_add(65, 'stale statistics after deletes, no VACUUM', 'p65',
                'SELECT count(*) FROM pc65 WHERE hot', 10000);

-- 66: rows entering the index (false -> true).
CREATE TABLE pc66 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc66; SELECT pg_stat_force_next_flush();
CREATE INDEX p66 ON pc66 (k) WHERE hot;
UPDATE pc66 SET hot = true WHERE NOT hot AND k % 5 = 1;
SELECT pg_stat_force_next_flush();
SELECT plan_add(66, 'rows entering the index (false -> true)', 'p66',
                'SELECT count(*) FROM pc66 WHERE hot', 200000);

-- 67: rows leaving the index (true -> false).
CREATE TABLE pc67 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc67; SELECT pg_stat_force_next_flush();
CREATE INDEX p67 ON pc67 (k) WHERE hot;
UPDATE pc67 SET hot = false WHERE hot AND k % 50 <> 0;
SELECT pg_stat_force_next_flush();
SELECT plan_add(67, 'rows leaving the index (true -> false)', 'p67',
                'SELECT count(*) FROM pc67 WHERE hot', 10000);

-- 68: heavy predicate churn, then VACUUM + ANALYZE.
CREATE TABLE pc68 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc68; SELECT pg_stat_force_next_flush();
CREATE INDEX p68 ON pc68 (k) WHERE hot;
UPDATE pc68 SET hot = true  WHERE k % 3 = 0;
UPDATE pc68 SET hot = false WHERE k % 3 = 0;
UPDATE pc68 SET hot = true  WHERE k % 3 = 1;
UPDATE pc68 SET hot = false WHERE k % 3 = 1;
UPDATE pc68 SET hot = (k % 10 = 0);
SELECT pg_stat_force_next_flush();
VACUUM pc68;
SELECT pg_stat_force_next_flush(); ANALYZE pc68; SELECT pg_stat_force_next_flush();
SELECT plan_add(68, 'heavy predicate churn, then VACUUM + ANALYZE', 'p68',
                'SELECT count(*) FROM pc68 WHERE hot', 50000);

-- 69: stale reltuples, VACUUM but no ANALYZE.
CREATE TABLE pc69 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pc69; SELECT pg_stat_force_next_flush();
CREATE INDEX p69 ON pc69 (k) WHERE hot;
DELETE FROM pc69 WHERE hot AND k % 50 <> 0;
SELECT pg_stat_force_next_flush();
VACUUM pc69;
SELECT pg_stat_force_next_flush();
SELECT plan_add(69, 'stale reltuples, VACUUM but no ANALYZE', 'p69',
                'SELECT count(*) FROM pc69 WHERE hot', 10000);

-- ============================================================ 70-77 =========
CREATE TABLE pb AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb; SELECT pg_stat_force_next_flush();
CREATE INDEX p70 ON pb (k) WHERE hot;
SELECT plan_add(70, 'freshly created partial index', 'p70',
                'SELECT count(*) FROM pb WHERE hot', 100000);
CREATE INDEX p71 ON pb (k) WHERE hot;
REINDEX INDEX p71;
SELECT plan_add(71, 'freshly REINDEXed partial index', 'p71',
                'SELECT count(*) FROM pb WHERE hot', 100000);

CREATE TABLE pb72 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb72; SELECT pg_stat_force_next_flush();
CREATE INDEX p72 ON pb72 (k) WHERE hot;
DELETE FROM pb72 WHERE hot AND (k / 5) % 4 = 0;
SELECT pg_stat_force_next_flush(); VACUUM pb72;
SELECT pg_stat_force_next_flush(); ANALYZE pb72; SELECT pg_stat_force_next_flush();
SELECT plan_add(72, '25% of the subset deleted', 'p72', 'SELECT count(*) FROM pb72 WHERE hot', 75000);

CREATE TABLE pb73 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb73; SELECT pg_stat_force_next_flush();
CREATE INDEX p73 ON pb73 (k) WHERE hot;
DELETE FROM pb73 WHERE hot AND (k / 5) % 2 = 0;
SELECT pg_stat_force_next_flush(); VACUUM pb73;
SELECT pg_stat_force_next_flush(); ANALYZE pb73; SELECT pg_stat_force_next_flush();
SELECT plan_add(73, '50% of the subset deleted', 'p73', 'SELECT count(*) FROM pb73 WHERE hot', 50000);

CREATE TABLE pb74 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb74; SELECT pg_stat_force_next_flush();
CREATE INDEX p74 ON pb74 (k) WHERE hot;
DELETE FROM pb74 WHERE hot AND (k / 5) % 4 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM pb74;
SELECT pg_stat_force_next_flush(); ANALYZE pb74; SELECT pg_stat_force_next_flush();
SELECT plan_add(74, '75% of the subset deleted', 'p74', 'SELECT count(*) FROM pb74 WHERE hot', 25000);

-- 75 is the corrected recipe: 90% of the subset, not the whole of it.
CREATE TABLE pb75 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb75; SELECT pg_stat_force_next_flush();
CREATE INDEX p75 ON pb75 (k) WHERE hot;
DELETE FROM pb75 WHERE hot AND (k / 5) % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM pb75;
SELECT pg_stat_force_next_flush(); ANALYZE pb75; SELECT pg_stat_force_next_flush();
SELECT plan_add(75, '90% of the subset deleted (corrected recipe)', 'p75',
                'SELECT count(*) FROM pb75 WHERE hot', 10000);

CREATE TABLE pb76 AS SELECT (i % 5 = 0) AS hot, i::int AS k, 'x'::text AS pad
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb76; SELECT pg_stat_force_next_flush();
CREATE INDEX p76 ON pb76 (k) WHERE hot;
UPDATE pb76 SET k = k + 1000000 WHERE hot;
SELECT pg_stat_force_next_flush(); VACUUM pb76;
SELECT pg_stat_force_next_flush(); ANALYZE pb76; SELECT pg_stat_force_next_flush();
SELECT plan_add(76, 'bloated through indexed-key UPDATEs', 'p76',
                'SELECT count(*) FROM pb76 WHERE hot', 100000);

CREATE TABLE pb77 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE pb77; SELECT pg_stat_force_next_flush();
CREATE INDEX p77 ON pb77 (k) WHERE hot;
DELETE FROM pb77 WHERE hot AND k < 475000;          -- contiguous 95%
SELECT pg_stat_force_next_flush(); VACUUM pb77;
SELECT pg_stat_force_next_flush(); ANALYZE pb77; SELECT pg_stat_force_next_flush();
SELECT plan_add(77, 'many empty and deleted B-tree pages', 'p77',
                'SELECT count(*) FROM pb77 WHERE hot', 5001);

-- ============================================================ 78-85 =========
-- Critical-false-positive constructions.  Every index is freshly built.
CREATE TABLE f78t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 290) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f78t; SELECT pg_stat_force_next_flush();
CREATE INDEX f78 ON f78t (s) WHERE hot;
SELECT plan_add(78, 'predicate-conditioned width mismatch', 'f78',
                'SELECT count(*) FROM f78t WHERE hot', 25000);

CREATE TABLE f79t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('t', 300) || lpad(i::text, 4, '0')
            ELSE NULL END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f79t; SELECT pg_stat_force_next_flush();
CREATE INDEX f79 ON f79t (s) WHERE hot;
SELECT plan_add(79, 'predicate-conditioned NULL mismatch', 'f79',
                'SELECT count(*) FROM f79t WHERE hot', 25000);

CREATE TABLE f80t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f80t; SELECT pg_stat_force_next_flush();
CREATE INDEX f80 ON f80t (k) WHERE hot;
SELECT plan_add(80, 'predicate-conditioned n_distinct mismatch', 'f80',
                'SELECT count(*) FROM f80t WHERE hot', 100000);

CREATE TABLE f81t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 20)::int ELSE 7 END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f81t; SELECT pg_stat_force_next_flush();
CREATE INDEX f81 ON f81t (k) WHERE hot;
SELECT plan_add(81, 'predicate-conditioned MCV mismatch', 'f81',
                'SELECT count(*) FROM f81t WHERE hot', 100000);

CREATE TABLE f82t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 89)::int  END AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS f82_nd (ndistinct) ON a, b FROM f82t;
SELECT pg_stat_force_next_flush(); ANALYZE f82t; SELECT pg_stat_force_next_flush();
CREATE INDEX f82 ON f82t (a, b) WHERE hot;
SELECT plan_add(82, 'predicate-conditioned multi-column correlation', 'f82',
                'SELECT count(*) FROM f82t WHERE hot', 100000);

CREATE TABLE f83t AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f83t; SELECT pg_stat_force_next_flush();
CREATE INDEX f83 ON f83t (md5(s), lower(s)) WHERE hot;    -- no statistics row
SELECT plan_add(83, 'missing index/expression statistics', 'f83',
                'SELECT count(*) FROM f83t WHERE hot', 100000);

CREATE TABLE f84t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f84t; SELECT pg_stat_force_next_flush();
CREATE INDEX f84 ON f84t (k) WHERE hot;
UPDATE pg_class SET reltuples = 5000 WHERE relname = 'f84';   -- stale partial count
SELECT plan_add(84, 'stale partial-index reltuples', 'f84',
                'SELECT count(*) FROM f84t WHERE hot', 100000);

CREATE TABLE f85t AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f85t; SELECT pg_stat_force_next_flush();
UPDATE f85t SET s = repeat('W', 200) || s WHERE hot;   -- table statistics now stale
SELECT pg_stat_force_next_flush(); VACUUM f85t; SELECT pg_stat_force_next_flush();
CREATE INDEX f85 ON f85t (s) WHERE hot;
SELECT plan_add(85, 'stale table statistics', 'f85',
                'SELECT count(*) FROM f85t WHERE hot', 100000);

-- ============================================================ 86-91 =========
-- Critical-false-negative constructions: genuinely bloated, VACUUMed, ANALYZEd.
CREATE TABLE f86t AS SELECT (i % 5 = 0) AS hot, ((i / 5) % 100)::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f86t; SELECT pg_stat_force_next_flush();
CREATE INDEX f86 ON f86t (k) WHERE hot;
DELETE FROM f86t WHERE hot AND k >= 25;
SELECT pg_stat_force_next_flush(); VACUUM f86t;
SELECT pg_stat_force_next_flush(); ANALYZE f86t; SELECT pg_stat_force_next_flush();
SELECT plan_add(86, 'duplicate concentration inside the subset', 'f86',
                'SELECT count(*) FROM f86t WHERE hot', 25000);

CREATE TABLE f87t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f87t; SELECT pg_stat_force_next_flush();
CREATE INDEX f87 ON f87t (k) WHERE hot;
DELETE FROM f87t WHERE hot AND k IS NOT NULL;
SELECT pg_stat_force_next_flush(); VACUUM f87t;
SELECT pg_stat_force_next_flush(); ANALYZE f87t; SELECT pg_stat_force_next_flush();
SELECT plan_add(87, 'NULL concentration inside the subset', 'f87',
                'SELECT count(*) FROM f87t WHERE hot', 25000);

CREATE TABLE f88t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN lpad((i % 9)::text, 9, '0')
            ELSE repeat('W', 390) || lpad(i::text, 10, '0') END AS s
  FROM generate_series(1, 200000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f88t; SELECT pg_stat_force_next_flush();
CREATE INDEX f88 ON f88t (s) WHERE hot;
DELETE FROM f88t WHERE hot AND s > lpad('4', 9, '0');
SELECT pg_stat_force_next_flush(); VACUUM f88t;
SELECT pg_stat_force_next_flush(); ANALYZE f88t; SELECT pg_stat_force_next_flush();
SELECT plan_add(88, 'subset narrower than table statistics', 'f88', NULL, NULL);

CREATE TABLE f89t AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f89t; SELECT pg_stat_force_next_flush();
CREATE INDEX f89 ON f89t (a, b) WHERE hot;
DELETE FROM f89t WHERE hot AND a >= 25;
SELECT pg_stat_force_next_flush(); VACUUM f89t;
SELECT pg_stat_force_next_flush(); ANALYZE f89t; SELECT pg_stat_force_next_flush();
SELECT plan_add(89, 'conditional multi-column correlation', 'f89',
                'SELECT count(*) FROM f89t WHERE hot', 25000);

CREATE TABLE f90t AS SELECT (i % 5 = 0) AS hot, ((i / 5) % 1000)::int AS k
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f90t; SELECT pg_stat_force_next_flush();
CREATE INDEX f90 ON f90t (k) WHERE hot;
DELETE FROM f90t WHERE hot AND k >= 250;
SELECT pg_stat_force_next_flush(); VACUUM f90t;
SELECT pg_stat_force_next_flush(); ANALYZE f90t; SELECT pg_stat_force_next_flush();
SELECT plan_add(90, 'real deduplication stronger than predicted', 'f90',
                'SELECT count(*) FROM f90t WHERE hot', 25000);

CREATE TABLE f91t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN lpad((i % 9)::text, 9, '0')
            ELSE repeat('W', 390) || lpad(i::text, 10, '0') END AS s,
       i::int AS ord
  FROM generate_series(1, 200000) i;
SELECT pg_stat_force_next_flush(); ANALYZE f91t; SELECT pg_stat_force_next_flush();
CREATE INDEX f91 ON f91t (s) WHERE hot;
DELETE FROM f91t WHERE hot AND ord < 190000;         -- contiguous 95%
SELECT pg_stat_force_next_flush(); VACUUM f91t;
SELECT pg_stat_force_next_flush(); ANALYZE f91t; SELECT pg_stat_force_next_flush();
SELECT plan_add(91, 'many deleted pages plus an over-predicting model', 'f91',
                'SELECT count(*) FROM f91t WHERE hot', 2001);

-- ============================================================ 92-95 =========
-- Change B threshold calibration: a genuinely reclaimable partial index,
-- disturbed by a known number of row changes, with and without reloptions.
CREATE TABLE b92t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE b92t; SELECT pg_stat_force_next_flush();
CREATE INDEX b92 ON b92t (k) WHERE hot;
DELETE FROM b92t WHERE hot AND (k / 5) % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM b92t;
SELECT pg_stat_force_next_flush(); ANALYZE b92t; SELECT pg_stat_force_next_flush();
UPDATE b92t SET k = k WHERE k % 500 = 0;              -- 1,000 rows changed
SELECT pg_stat_force_next_flush();
SELECT plan_add(92, '1,000 rows updated under the GUC threshold', 'b92',
                'SELECT count(*) FROM b92t WHERE hot', 10000);

CREATE TABLE b93t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE b93t; SELECT pg_stat_force_next_flush();
CREATE INDEX b93 ON b93t (k) WHERE hot;
DELETE FROM b93t WHERE hot AND (k / 5) % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM b93t;
SELECT pg_stat_force_next_flush(); ANALYZE b93t; SELECT pg_stat_force_next_flush();
UPDATE b93t SET k = k WHERE k % 2 = 0;                -- above the trigger
SELECT pg_stat_force_next_flush();
SELECT plan_add(93, 'rows updated above the GUC threshold', 'b93',
                'SELECT count(*) FROM b93t WHERE hot', 10000);

CREATE TABLE b94t (hot bool, k int)
  WITH (autovacuum_analyze_threshold = 100, autovacuum_analyze_scale_factor = 0);
INSERT INTO b94t SELECT (i % 5 = 0), i FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE b94t; SELECT pg_stat_force_next_flush();
CREATE INDEX b94 ON b94t (k) WHERE hot;
DELETE FROM b94t WHERE hot AND (k / 5) % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM b94t;
SELECT pg_stat_force_next_flush(); ANALYZE b94t; SELECT pg_stat_force_next_flush();
UPDATE b94t SET k = k WHERE k % 500 = 0;              -- 1,000 > the reloption
SELECT pg_stat_force_next_flush();
SELECT plan_add(94, '1,000 rows updated, table reloption threshold 100', 'b94',
                'SELECT count(*) FROM b94t WHERE hot', 10000);

CREATE TABLE b95t (hot bool, k int)
  WITH (autovacuum_analyze_threshold = 200000, autovacuum_analyze_scale_factor = 1);
INSERT INTO b95t SELECT (i % 5 = 0), i FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE b95t; SELECT pg_stat_force_next_flush();
CREATE INDEX b95 ON b95t (k) WHERE hot;
DELETE FROM b95t WHERE hot AND (k / 5) % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM b95t;
SELECT pg_stat_force_next_flush(); ANALYZE b95t; SELECT pg_stat_force_next_flush();
UPDATE b95t SET k = k WHERE k % 2 = 0;                -- below the reloption
SELECT pg_stat_force_next_flush();
SELECT plan_add(95, 'many rows updated, table reloption threshold 200,000', 'b95',
                'SELECT count(*) FROM b95t WHERE hot', 10000);

-- ============================================================ 96-99 =========
-- Non-partial controls: the partial-only exclusions must not reach them.
CREATE TABLE np AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE np; SELECT pg_stat_force_next_flush();
CREATE INDEX np96 ON np (k);
CREATE INDEX np97 ON np (upper(s));                   -- no statistics row
SELECT plan_add(96, 'plain index, fresh statistics', 'np96', 'SELECT count(*) FROM np', 500000);
SELECT plan_add(97, 'expression index, no statistics row', 'np97', 'SELECT count(*) FROM np', 500000);

CREATE TABLE np98t AS SELECT i::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE np98t; SELECT pg_stat_force_next_flush();
CREATE INDEX np98 ON np98t (k);
INSERT INTO np98t SELECT 500000 + i FROM generate_series(1, 300000) i;
SELECT pg_stat_force_next_flush();
SELECT plan_add(98, 'plain index, stale row counts after 300,000 inserts', 'np98',
                'SELECT count(*) FROM np98t', 800000);

-- 99: the corrected recipe.  An index and a table cannot share a name, so the
-- table is np99t and the index np99.
CREATE TABLE np99t AS SELECT (i % 1000)::int AS k FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE np99t; SELECT pg_stat_force_next_flush();
CREATE INDEX np99 ON np99t (k);
DELETE FROM np99t WHERE k >= 60;
SELECT pg_stat_force_next_flush(); VACUUM np99t;
SELECT pg_stat_force_next_flush(); ANALYZE np99t; SELECT pg_stat_force_next_flush();
SELECT plan_add(99, 'duplicate-heavy index, genuinely reclaimable', 'np99',
                'SELECT count(*) FROM np99t', 30000);

-- =========================================================== 100-105 ========
-- The variable-width INCLUDE family.
CREATE TABLE i100t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE i100t; SELECT pg_stat_force_next_flush();
CREATE INDEX i100 ON i100t (k) INCLUDE (pay) WHERE hot;
DELETE FROM i100t WHERE hot AND (k / 5) % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM i100t;
SELECT pg_stat_force_next_flush(); ANALYZE i100t; SELECT pg_stat_force_next_flush();
SELECT plan_add(100, 'partial + INCLUDE (text), 90% of the subset deleted', 'i100',
                'SELECT count(*) FROM i100t WHERE hot', 10000);

CREATE TABLE i101t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE i101t; SELECT pg_stat_force_next_flush();
CREATE INDEX i101 ON i101t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(101, 'partial + INCLUDE (text), same width inside and outside', 'i101',
                'SELECT count(*) FROM i101t WHERE hot', 100000);

CREATE TABLE i102t AS SELECT i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE i102t; SELECT pg_stat_force_next_flush();
CREATE INDEX i102 ON i102t (k) INCLUDE (pay);
SELECT plan_add(102, 'non-partial + wide INCLUDE (text), freshly built', 'i102',
                'SELECT count(*) FROM i102t', 500000);

CREATE TABLE i103t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad(i::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE i103t; SELECT pg_stat_force_next_flush();
CREATE INDEX i103 ON i103t (s) WHERE hot;
SELECT plan_add(103, 'partial + wide key column, unique values, no caveat', 'i103',
                'SELECT count(*) FROM i103t WHERE hot', 25000);

CREATE TABLE i104t AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN lpad((i % 9)::text, 12, 'n')
            ELSE repeat('W', 190) || lpad(i::text, 10, '0') END AS pay
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE i104t; SELECT pg_stat_force_next_flush();
CREATE INDEX i104 ON i104t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(104, 'partial + INCLUDE (text) narrower inside the subset', 'i104',
                'SELECT count(*) FROM i104t WHERE hot', 25000);

CREATE TABLE i105t AS SELECT (i % 20 = 0) AS hot, i::int AS k, (i % 7)::int AS n,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS pay
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE i105t; SELECT pg_stat_force_next_flush();
CREATE INDEX i105 ON i105t (k) INCLUDE (n, pay) WHERE hot;
SELECT plan_add(105, 'partial + INCLUDE (int, text), mixed non-key widths', 'i105',
                'SELECT count(*) FROM i105t WHERE hot', 25000);

-- =========================================================== 106-112 ========
-- The expression-statistics family.
CREATE TABLE x106t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE x106t; SELECT pg_stat_force_next_flush();
CREATE INDEX x106 ON x106t (upper(s));
DELETE FROM x106t WHERE k % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM x106t; SELECT pg_stat_force_next_flush();
SELECT plan_add(106, 'expression index, no statistics row, 90% deleted', 'x106',
                'SELECT count(*) FROM x106t', 50000);

CREATE TABLE x107t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE x107t; SELECT pg_stat_force_next_flush();
CREATE INDEX x107 ON x107t (upper(s));
DELETE FROM x107t WHERE k % 10 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM x107t;
SELECT pg_stat_force_next_flush(); ANALYZE x107t; SELECT pg_stat_force_next_flush();
SELECT plan_add(107, 'the same, with one ANALYZE after the build', 'x107',
                'SELECT count(*) FROM x107t', 50000);

CREATE TABLE x108t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX x108 ON x108t (upper(s));                -- table never analysed
SELECT pg_stat_force_next_flush();
SELECT plan_add(108, 'expression index on a never-analysed table', 'x108',
                'SELECT count(*) FROM x108t', 500000);

CREATE TABLE x109t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ALTER TABLE x109t ALTER COLUMN s SET STATISTICS 0;
SELECT pg_stat_force_next_flush(); ANALYZE x109t; SELECT pg_stat_force_next_flush();
CREATE INDEX x109 ON x109t (s);
SELECT plan_add(109, 'plain index, key column with SET STATISTICS 0', 'x109',
                'SELECT count(*) FROM x109t', 500000);

CREATE TABLE x110t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE x110t; SELECT pg_stat_force_next_flush();
CREATE INDEX x110 ON x110t (k, upper(s));             -- mixed key, no stats row
SELECT plan_add(110, 'mixed key (k, upper(s)), no statistics row', 'x110',
                'SELECT count(*) FROM x110t', 500000);

CREATE TABLE x111t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE x111t; SELECT pg_stat_force_next_flush();
CREATE INDEX x111 ON x111t (left(s, 3));              -- narrow expression
SELECT plan_add(111, 'narrow expression left(s, 3), no statistics row', 'x111',
                'SELECT count(*) FROM x111t', 500000);

CREATE TABLE x112t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush(); ANALYZE x112t; SELECT pg_stat_force_next_flush();
CREATE INDEX x112 ON x112t (upper(s)) WHERE hot;      -- partial expression
SELECT plan_add(112, 'partial expression index, no statistics row', 'x112',
                'SELECT count(*) FROM x112t WHERE hot', 100000);

-- =========================================================== 113-121 ========
-- The drained queue in three states.
CREATE TABLE q113a AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE q113a; SELECT pg_stat_force_next_flush();
CREATE INDEX p113a ON q113a (id) WHERE state = 'pending';
UPDATE q113a SET state = 'done';
SELECT pg_stat_force_next_flush();
SELECT plan_add(113, 'drained queue, nothing run', 'p113a',
                'SELECT count(*) FROM q113a WHERE state = ''pending''', 0, 'a');

CREATE TABLE q113b AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE q113b; SELECT pg_stat_force_next_flush();
CREATE INDEX p113b ON q113b (id) WHERE state = 'pending';
UPDATE q113b SET state = 'done';
SELECT pg_stat_force_next_flush(); VACUUM q113b;
SELECT pg_stat_force_next_flush(); ANALYZE q113b; SELECT pg_stat_force_next_flush();
SELECT plan_add(113, 'drained queue, VACUUM + ANALYZE', 'p113b',
                'SELECT count(*) FROM q113b WHERE state = ''pending''', 0, 'b');

CREATE TABLE q113c AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE q113c; SELECT pg_stat_force_next_flush();
CREATE INDEX p113c ON q113c (id) WHERE state = 'pending';
UPDATE q113c SET state = 'done';
SELECT pg_stat_force_next_flush(); ANALYZE q113c; SELECT pg_stat_force_next_flush();
SELECT plan_add(113, 'drained queue, ANALYZE only', 'p113c',
                'SELECT count(*) FROM q113c WHERE state = ''pending''', 0, 'c');

-- 114: a genuine, fully repaired detection on the same queue shape.
CREATE TABLE q114 AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE q114; SELECT pg_stat_force_next_flush();
CREATE INDEX p114 ON q114 (id) WHERE state = 'pending';
UPDATE q114 SET state = 'done' WHERE id % 100 <> 0;
SELECT pg_stat_force_next_flush(); VACUUM q114;
SELECT pg_stat_force_next_flush(); ANALYZE q114; SELECT pg_stat_force_next_flush();
SELECT plan_add(114, 'queue drained to 1%, VACUUM + ANALYZE', 'p114',
                'SELECT count(*) FROM q114 WHERE state = ''pending''', 10000);

-- 115: index built on an analysed empty table, then loaded.
CREATE TABLE q115(id int, state text);
SELECT pg_stat_force_next_flush(); ANALYZE q115; SELECT pg_stat_force_next_flush();
CREATE INDEX p115 ON q115 (id) WHERE state = 'pending';
INSERT INTO q115 SELECT i, 'pending' FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush();
SELECT plan_add(115, 'index built on an analysed empty table, then loaded', 'p115',
                'SELECT count(*) FROM q115 WHERE state = ''pending''', 1000000);

-- 116: a subset that is genuinely empty and was measured empty.
CREATE TABLE q116 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p116 ON q116 (id) WHERE state = 'pending';
SELECT pg_stat_force_next_flush(); ANALYZE q116; SELECT pg_stat_force_next_flush();
SELECT plan_add(116, 'subset empty from the start and measured empty', 'p116',
                'SELECT count(*) FROM q116 WHERE state = ''pending''', 0);

-- 117: drained, then VACUUM only.
CREATE TABLE q117 AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE q117; SELECT pg_stat_force_next_flush();
CREATE INDEX p117 ON q117 (id) WHERE state = 'pending';
UPDATE q117 SET state = 'done';
SELECT pg_stat_force_next_flush(); VACUUM q117; SELECT pg_stat_force_next_flush();
SELECT plan_add(117, 'drained, then VACUUM only', 'p117',
                'SELECT count(*) FROM q117 WHERE state = ''pending''', 0);

-- 118: the subset was empty at the last ANALYZE, then 50,000 rows arrived.
CREATE TABLE q118 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p118 ON q118 (id) WHERE state = 'pending';
SELECT pg_stat_force_next_flush(); ANALYZE q118; SELECT pg_stat_force_next_flush();
INSERT INTO q118 SELECT 1000000 + i, 'pending' FROM generate_series(1, 50000) i;
SELECT pg_stat_force_next_flush();
SELECT plan_add(118, 'subset measured empty, then 50,000 rows arrive', 'p118',
                'SELECT count(*) FROM q118 WHERE state = ''pending''', 50000);

-- 119: fixture 118 after one ANALYZE.
CREATE TABLE q119 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p119 ON q119 (id) WHERE state = 'pending';
SELECT pg_stat_force_next_flush(); ANALYZE q119; SELECT pg_stat_force_next_flush();
INSERT INTO q119 SELECT 1000000 + i, 'pending' FROM generate_series(1, 50000) i;
SELECT pg_stat_force_next_flush(); ANALYZE q119; SELECT pg_stat_force_next_flush();
SELECT plan_add(119, 'fixture 118 after one ANALYZE', 'p119',
                'SELECT count(*) FROM q119 WHERE state = ''pending''', 50000);

-- 120: the ANALYZE sample missed the subset entirely.
CREATE TABLE q120 AS SELECT i::int AS id,
       CASE WHEN i <= 2000 THEN 'pending' ELSE 'done' END::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p120 ON q120 (id) WHERE state = 'pending';
SET default_statistics_target = 1;
SELECT pg_stat_force_next_flush(); ANALYZE q120; SELECT pg_stat_force_next_flush();
RESET default_statistics_target;
SELECT plan_add(120, 'a 300-row sample missed a 2,000-row subset', 'p120',
                'SELECT count(*) FROM q120 WHERE state = ''pending''', 2000);

-- 121: a stale zero on non-partial indexes, three recipes.
CREATE TABLE nz AS SELECT i::int AS k FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE nz; SELECT pg_stat_force_next_flush();
CREATE INDEX nz_k ON nz (k);
DELETE FROM nz;
SELECT pg_stat_force_next_flush(); VACUUM nz; SELECT pg_stat_force_next_flush();
INSERT INTO nz SELECT i FROM generate_series(1, 500000) i;   -- no ANALYZE
SELECT pg_stat_force_next_flush();
SELECT plan_add(121, 'emptied, vacuumed, reloaded without ANALYZE', 'nz_k',
                'SELECT count(*) FROM nz', 500000, 'nz_k');

CREATE TABLE nzb AS SELECT i::int AS k FROM generate_series(1, 1000000) i;
SELECT pg_stat_force_next_flush(); ANALYZE nzb; SELECT pg_stat_force_next_flush();
CREATE INDEX nzb_k ON nzb (k);
DELETE FROM nzb;
SELECT pg_stat_force_next_flush(); VACUUM nzb; SELECT pg_stat_force_next_flush();
REINDEX INDEX nzb_k;                                   -- rebuilt while empty
INSERT INTO nzb SELECT i FROM generate_series(1, 500000) i;
SELECT pg_stat_force_next_flush();
SELECT plan_add(121, 'the same, plus a REINDEX while the table is empty', 'nzb_k',
                'SELECT count(*) FROM nzb', 500000, 'nzb_k');

CREATE TABLE trunc_t AS SELECT i::int AS k FROM generate_series(1, 300000) i;
SELECT pg_stat_force_next_flush(); ANALYZE trunc_t; SELECT pg_stat_force_next_flush();
CREATE INDEX i_trunc ON trunc_t (k);
TRUNCATE trunc_t;
INSERT INTO trunc_t SELECT i FROM generate_series(1, 300000) i;  -- no ANALYZE
SELECT pg_stat_force_next_flush();
SELECT plan_add(121, 'TRUNCATE then reload without ANALYZE', 'i_trunc',
                'SELECT count(*) FROM trunc_t', 300000, 'i_trunc');

SELECT count(*) AS planned_fixtures FROM plan;
FIXTURES
  f suite "$SQLD/fixtures_suite.sql" || die "suite fixtures failed"
  note "$(s suite 'SELECT count(*) || '\'' planned fixtures'\'' FROM plan')"
}

# ------------------------------------------------------------ attribution ----
stage_attribution() {
  say "attribution: EXCEPT in both directions, before any REINDEX"
  f suite /dev/stdin > "$OUT/attribution.txt" 2>&1 <<'SQL'
\pset pager off
SELECT /* wiki_btree_attribution_r2_minus_old */ 'r2 minus old' AS direction, * FROM (
  SELECT indexname, status, wasted_space_pct, wasted_space_pct_floor, caveats,
         key_groups, modelled_rows, idx_reltuples, suppress_row
    FROM est_r2
  EXCEPT
  SELECT indexname, status, wasted_space_pct, wasted_space_pct_floor, caveats,
         key_groups, modelled_rows, idx_reltuples, suppress_row
    FROM est_old) d ORDER BY indexname;
SELECT /* wiki_btree_attribution_old_minus_r2 */ 'old minus r2' AS direction, * FROM (
  SELECT indexname, status, wasted_space_pct, wasted_space_pct_floor, caveats,
         key_groups, modelled_rows, idx_reltuples, suppress_row
    FROM est_old
  EXCEPT
  SELECT indexname, status, wasted_space_pct, wasted_space_pct_floor, caveats,
         key_groups, modelled_rows, idx_reltuples, suppress_row
    FROM est_r2) d ORDER BY indexname;
SQL
  tail -20 "$OUT/attribution.txt" >&2
}

# ---------------------------------------------------------------- probes -----
stage_probes() {
  say "validation probes, generated and executed, before any REINDEX"
  local db
  for db in suite acc; do
    "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$db" -f "$SQLD/probegen.sql" \
      > "$OUT/probes_gen_$db.txt" 2>&1
    : > "$OUT/probes_$db.txt"
    local name kind sql
    while IFS='|' read -r name kind sql; do
      [ -n "${sql:-}" ] || continue
      printf '%s|%s|%s\n' "$name" "$kind" "$(s "$db" "$sql" | tr '\n' ' ')" >> "$OUT/probes_$db.txt"
    done < "$OUT/probes_gen_$db.txt"
    note "$db: $(wc -l < "$OUT/probes_$db.txt") probes executed"
  done
  cat "$OUT/probes_suite.txt" "$OUT/probes_acc.txt" >&2
}

# ---------------------------------------------------------------- score ------
stage_score() {
  say "score: assert population, read both texts, REINDEX INDEX, re-read the size"
  f suite /dev/stdin <<'SQL'
SET /* wiki_btree_score_statement_timeout */ statement_timeout = '600s';
SET /* wiki_btree_score_lock_timeout */ lock_timeout = '2s';
CALL /* wiki_btree_score_all */ score_all();
SQL
  [ $? -eq 0 ] || die "scoring failed"
  t suite "SELECT /* wiki_btree_verdict_rows */ * FROM verdicts" > "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btree_verdict_floor_counts */
                  verdict_floor, count(*) FROM verdicts GROUP BY 1 ORDER BY 2 DESC" \
    >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btree_verdict_point_counts */
                  verdict_point, count(*) FROM verdicts GROUP BY 1 ORDER BY 2 DESC" \
    >> "$OUT/verdicts.txt" 2>&1
  tail -20 "$OUT/verdicts.txt" >&2
}

# ---------------------------------------------------------------- cost -------
stage_cost() {
  say "cost: six interleaved pairs of the two exact texts"
  : > "$OUT/cost.txt"
  local i
  for i in 1 2 3 4 5 6; do
    printf 'pair %s r2  %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
           -c '\timing on' -f "$SQLD/est_r2.sql" 2>&1 \
         | grep -E '^Time:' | tail -1)" >> "$OUT/cost.txt"
    printf 'pair %s old %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
           -c '\timing on' -f "$SQLD/est_old.sql" 2>&1 \
         | grep -E '^Time:' | tail -1)" >> "$OUT/cost.txt"
  done
  s suite "select /* wiki_btree_cost_database_size */
                  count(*) || ' B-tree indexes over ' ||
           sum(pg_relation_size(c.oid)) / 8192 || ' blocks'
             from pg_class c join pg_am a on a.oid = c.relam
            where a.amname = 'btree' and c.relkind = 'i'" >> "$OUT/cost.txt"
  cat "$OUT/cost.txt" >&2
}

# ------------------------------------------------- expected server errors ---
# Every server-side error this suite provokes is deliberate: the gate's
# nondeterministic-collation refusal, and the superseded text's bigint
# overflow on the ovf fixture.  Anything else in the log is a real failure, so
# match the log against this list and count what is left over.  Each logged
# error carries the statement that raised it, because log_min_error_statement
# defaults to error, so a leftover can be read back to its statement.
EXPECTED_ERRORS=(
  'nondeterministic collations are not supported for operator class "text_pattern_ops"'
  'bigint out of range'
)
UNEXPECTED_ERRORS=0
check_server_errors() {
  local log=$1 line e known
  UNEXPECTED_ERRORS=0
  [ -f "$log" ] || { printf '   no %s to read\n' "$log"; return 0; }
  while IFS= read -r line; do
    known=1
    for e in "${EXPECTED_ERRORS[@]}"; do
      case $line in *"$e"*) known=0; break ;; esac
    done
    [ $known -eq 0 ] || { UNEXPECTED_ERRORS=$((UNEXPECTED_ERRORS + 1))
                          printf '   unexpected: %s\n' "$line"; }
  done < <(grep -E '(ERROR|FATAL|PANIC):' "$log")
  printf '   allowed=%s unexpected_server_errors=%s\n' \
         "${#EXPECTED_ERRORS[@]}" "$UNEXPECTED_ERRORS"
}

# ---------------------------------------------------------------- criteria ---
stage_criteria() {
  say "pass criteria"
  { printf '1. gate group\n'
    t gate "SELECT /* wiki_btree_criteria_gate */
                   count(*) FILTER (WHERE dedup_applies AND NOT metapage) AS over_credit,
                   count(*) FILTER (WHERE equalimage = 'recognized' AND NOT metapage)
                          + count(*) FILTER (WHERE equalimage = 'ineligible' AND metapage)
                                                                          AS metapage_disagreements,
                   count(*) FILTER (WHERE equalimage = 'unknown' AND metapage) AS under_credits,
                   max(greatest(wsp, wspf))                                    AS worst_reading
              FROM gate_res"
    printf '2. partial group and controls\n'
    t suite "SELECT /* wiki_btree_criteria_partial */
                    count(*) FILTER (WHERE reported AND verdict_floor = 'CRITICAL FALSE POSITIVE') AS crit_fp_floor,
                    count(*) FILTER (WHERE reported AND verdict_point = 'CRITICAL FALSE POSITIVE') AS crit_fp_point,
                    count(*) FILTER (WHERE verdict_floor = 'FALSE NEGATIVE')   AS false_negatives,
                    count(*) FILTER (WHERE NOT reported)                       AS withheld,
                    count(*) FILTER (WHERE NOT reported AND withheld_by IS NULL) AS withheld_unexplained,
                    count(*) FILTER (WHERE NOT contract_ok)                    AS contract_failures
               FROM verdicts"
    printf '3. drained subsets and zero rows\n'
    t suite "SELECT /* wiki_btree_criteria_zero_rows */
                    num, leg, idx, wsp, wspf, actual, modelled_rows, caveats
               FROM verdicts WHERE modelled_rows = 0 OR num IN (113, 118, 120) ORDER BY num, leg"
    printf '4. attribution\n'
    grep -c '^ ' "$OUT/attribution.txt" 2>/dev/null | xargs printf '   EXCEPT output lines: %s\n'
    printf '5. exact texts\n'
    ls -l "$OUT"/exact_*.txt | while read -r l; do printf '   %s\n' "$l"; done
    printf '6. engine and repository checks\n'
    cat "$OUT/checks.txt" 2>/dev/null
    cat "$OUT/hashes.txt" 2>/dev/null
  } > "$OUT/criteria.txt" 2>&1
  { printf '7. server errors\n'
    check_server_errors "$OUT/server.log"; } >> "$OUT/criteria.txt" 2>&1
  cat "$OUT/criteria.txt" >&2
  [ "$UNEXPECTED_ERRORS" -eq 0 ] \
    || die "$UNEXPECTED_ERRORS unexpected server error(s); see block 7 of $OUT/criteria.txt"
}

# ---------------------------------------------------------------- report -----
stage_report() {
  say "report written to $OUT"
  ls -1 "$OUT" >&2
}

# ---------------------------------------------------------------- stop -------
# Shut the server down cleanly.  -m fast disconnects clients and lets the
# checkpointer write a shutdown checkpoint, so the next start needs no
# recovery; -m immediate, which this stage used until 2026-09-10, skips that
# and forces crash recovery on restart.  The stop is then confirmed the way
# the teardown rule asks, and the stage dies rather than report a stop that
# did not happen, so clean never deletes a live cluster.
stage_stop() {
  say "stop the server cleanly"
  [ -x "$BIN/pg_ctl" ] || { note "no server binary under $BIN, nothing to stop"; return 0; }
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null 2>&1 || die "pg_ctl -m fast stop failed"
    tail -3 "$OUT/server.log" 2>/dev/null | grep -q 'database system is shut down' \
      && note "server.log: database system is shut down"
  else
    note "not running"
  fi
  [ -e "$DATA/postmaster.pid" ] && die "$DATA/postmaster.pid still exists"
  if command -v pgrep > /dev/null 2>&1 && pgrep -f -- "-D $DATA" > /dev/null 2>&1; then
    die "a postgres process still runs on $DATA"
  fi
  [ -z "$(ls -A "$SOCK" 2>/dev/null)" ] || die "socket directory $SOCK is not empty"
  note "confirmed: no postmaster.pid, no postgres process on $DATA, socket directory empty"
}

# Containment check before any rm -rf: SANDBOX comes from the environment, so
# refuse to delete anything outside this repository's .wiki-runtime/tmp tree.
inside_tmp() {
  case ${1%/} in
    "$WIKI_ROOT/.wiki-runtime/tmp"/?*) return 0 ;;
    *) return 1 ;;
  esac
}
stage_clean() {
  stage_stop
  inside_tmp "$SANDBOX" || die "refusing to delete $SANDBOX outside $WIKI_ROOT/.wiki-runtime/tmp/"
  rm -rf "$SANDBOX"; say "sandbox deleted"
}

main() {
  local stages=("$@")
  [ ${#stages[@]} -eq 0 ] && stages=(build check cluster texts geometry calibration \
                                     gate acceptance suite extstat attribution \
                                     probes score cost criteria report)
  local st
  for st in "${stages[@]}"; do
    case $st in
      build|check|cluster|texts|extstat|geometry|calibration|gate|acceptance|suite|\
      attribution|probes|score|cost|criteria|report|stop|clean) "stage_$st" ;;
      *) die "unknown stage: $st" ;;
    esac
  done
}

main "$@"
```

### The PostgreSQL 12 leg script

This is step 8, and its first result is the parse outcome of the unmodified
text, recorded before any fixture exists. Nothing in the script assumes what
PostgreSQL 12 does: the server-version number, the block size and alignment,
whether `pg_stat_force_next_flush()` exists, which columns `pg_stats_ext`
exposes, which B-tree support-function numbers are registered and whether the
`deduplicate_items` reloption is accepted are all discovered at run time and
written to `out/v12_facts.txt`. Because the flush function is not available
there, the leg publishes counters the way the shipped 12 statistics test does:
the writing backend exits and the observer polls in fresh sessions until the
counter it is waiting for appears, with a timeout treated as a failed
precondition rather than as a result.

```bash
#!/usr/bin/env bash
#
# btree_bloat_suite_v12.sh - the 12.2 leg of the same suite, in bash and SQL
# only.  It is step 8 of the page's protocol: build the pinned 12 checkout,
# execute the exact current statement text, record the outcome as a result in
# its own right, and only then transform, fixture and score.
#
# Nothing here assumes what PostgreSQL 12 does.  Every version-local fact the
# leg depends on is discovered at run time and written to $OUT/v12_facts.txt:
# whether the exact text parses, which construct rejects it, whether
# pg_stat_force_next_flush() exists, and whether a B-tree operator class
# offers a support function 4.  The pinned checkouts stay read only.
#
# Usage, from the repository root:
#   bash btree_bloat_suite_v12.sh                  # all stages
#   bash btree_bloat_suite_v12.sh exact            # just the parse result
#
# Stages: build check cluster exact transform facts fixtures score extstat
#         report stop clean
#
# Environment: WIKI_ROOT PAGE SRC12 SANDBOX PORT12 JOBS EXTRA_CFLAGS
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md}"
SRC12="${SRC12:-$WIKI_ROOT/raw/postgres-12}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/btree-suite}"
PORT12="${PORT12:-55412}"
JOBS="${JOBS:-4}"
# ICU dropped the TRUE/FALSE macros in ICU 68; a 12.2 tree configured
# --with-icu against a newer ICU needs them back.  Empty this variable on a
# host whose ICU still defines them, or drop --with-icu instead.
EXTRA_CFLAGS="${EXTRA_CFLAGS:--O2 -g -DTRUE=1 -DFALSE=0}"

BUILD="$SANDBOX/build12"; INST="$SANDBOX/install12"; DATA="$SANDBOX/data12"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql"; SOCK="$SANDBOX/sock12"; BIN="$INST/bin"
DB=leg12
export PGPORT="$PORT12" PGHOST="$SOCK" PGDATABASE=postgres

BASE1=646df923635182809f1a139e2f7f9367e94b6e0eaf79ed66fc37697a82d5d706
# The estimator text as filed before the portable extstat filter of 2026-09-10.
# The extstat stage rebuilds it from the current text, must reproduce this
# hash, and must find this server refusing it.
BASEPRE=8acd531b7bcd2f2ca679e65024d83bd61debcb4b75bb18f3834a368454d574fd

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# -X ignores ~/.psqlrc; ON_ERROR_STOP is on every helper, because without it a
# failed statement inside a -f script leaves the exit status 0.
q() { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }          # writer
s() { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }      # scalar
t() { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" -c "$1"; }  # table
fl() { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -f "$1"; }         # file

# The fence is assembled from printf '\140' so that this script contains no
# literal Markdown fence and can therefore live inside one.
md_block() {
  local lang=$1 want=$2 file=$3 n=0 inb=0 line tick fence
  tick=$(printf '\140'); fence="$tick$tick$tick"
  while IFS= read -r line; do
    if [ "$inb" = 1 ]; then
      if [ "$line" = "$fence" ]; then inb=0; [ "$n" = "$want" ] && return 0; continue; fi
      [ "$n" = "$want" ] && printf '%s\n' "$line"
    elif [ "$line" = "$fence$lang" ]; then
      n=$((n + 1)); inb=1
    fi
  done < "$file"
}

harness_view() {
  local file=$1 view=$2 extra=$3 line
  printf 'DROP VIEW IF EXISTS %s;\nCREATE VIEW %s AS\n' "$view" "$view"
  while IFS= read -r line; do
    case $line in
      "SET /* wiki_btree_wasted_space"*)     continue ;;
      "       server_version_num")           printf '       server_version_num,\n%s\n' "$extra"; continue ;;
      " WHERE actual_bytes > 1024 * 1024"*)  continue ;;
      " ORDER BY (actual_bytes"*)            continue ;;
      " LIMIT 20;")                          printf ';\n'; continue ;;
    esac
    printf '%s\n' "$line"
  done < "$file"
}

INTERNALS='       expected_blocks, floor_blocks, actual_bytes, live_rows, slot,
       leaf_cap, nmax, leaf_pages, tids, dedup_applies, is_partial,
       has_expressions, stats_row_missing, dedup_credited, stats_stale,
       suppress_row, ext_used, equalimage_state'

# There is no pg_stat_force_next_flush() before PostgreSQL 15, so this leg
# publishes counters the way the shipped 12 statistics test does: the writing
# backend exits, and the observer polls in fresh sessions until the counter it
# is waiting for appears.  A timeout is a failed precondition, not a result.
wait_for() {                       # wait_for <sql returning boolean> <label>
  local i
  for i in $(seq 1 120); do
    [ "$(s "SELECT ($1)::text")" = "true" ] && return 0
    sleep 0.5
  done
  printf '!! statistics did not publish within 60s: %s\n' "$2" >&2
  return 1
}
loaded()   { wait_for "(SELECT n_live_tup FROM pg_stat_all_tables WHERE relname = '$1') >= $2" "$1 loaded"; }
analyzed() { wait_for "(SELECT last_analyze IS NOT NULL FROM pg_stat_all_tables WHERE relname = '$1')" "$1 analyzed"; }
vacuumed() { wait_for "(SELECT last_vacuum IS NOT NULL FROM pg_stat_all_tables WHERE relname = '$1')" "$1 vacuumed"; }

# ---------------------------------------------------------------- build ------
stage_build() {
  say "build 12.2 out of tree from $SRC12"
  [ -x "$BIN/postgres" ] && { note "already built, skipping"; return 0; }
  [ -x "$SRC12/configure" ] || die "no pinned checkout at $SRC12; set SRC12 or run from the repository root"
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" --enable-debug \
      --with-icu --with-readline --with-zlib CFLAGS="$EXTRA_CFLAGS" \
      > configure.log 2>&1 ) || die "configure failed, see $BUILD/configure.log"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { grep -m3 'error:' "$BUILD/make.log" >&2; die "make failed"; }
  local m
  for m in pageinspect pgstattuple amcheck; do
    ( cd "$BUILD" && make -C "contrib/$m" -j"$JOBS" >> install.log 2>&1 \
        && make -C "contrib/$m" install >> install.log 2>&1 ) || die "contrib/$m failed"
  done
  # clean deletes $BUILD, so keep this leg's build diagnostics in $OUT.  The
  # 17 leg owns the shared out/, hence the 12 suffix on every copied name.
  local l
  for l in configure make install; do
    cp "$BUILD/$l.log" "$OUT/${l}12.log" 2>/dev/null
  done
  note "$("$BIN/postgres" --version)"
}

stage_check() {
  say "engine regression suites, 12.2"
  : > "$OUT/checks12.txt"
  ( cd "$BUILD" && make check > check_core.log 2>&1 )
  printf 'core=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_core.log" | tail -1)" \
    >> "$OUT/checks12.txt"
  local m
  for m in pageinspect pgstattuple amcheck; do
    ( cd "$BUILD" && make -C "contrib/$m" check > "check_$m.log" 2>&1 )
    printf '%s=%s %s\n' "$m" "$?" \
      "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_$m.log" | tail -1)" \
      >> "$OUT/checks12.txt"
  done
  # Same reason as the build stage: a one-line summary cannot diagnose a
  # failure, and the logs and diffs go with $BUILD.
  local l d
  for l in core pageinspect pgstattuple amcheck; do
    cp "$BUILD/check_$l.log" "$OUT/check12_$l.log" 2>/dev/null
  done
  for d in "$BUILD/src/test/regress" "$BUILD"/contrib/*; do
    [ -f "$d/regression.diffs" ] \
      && cp "$d/regression.diffs" "$OUT/diffs12_$(basename "$d").txt"
  done
  cat "$OUT/checks12.txt" >&2
}

stage_cluster() {
  say "isolated 12.2 cluster on port $PORT12"
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then note "already running"
  else
    if [ ! -d "$DATA" ]; then
      mkdir -p "$SOCK"
      "$BIN/initdb" -D "$DATA" --locale=C --encoding=UTF8 > "$OUT/initdb12.log" 2>&1 \
        || die "initdb failed"
      cat >> "$DATA/postgresql.conf" <<CONF
listen_addresses = ''
unix_socket_directories = '$SOCK'
port = $PORT12
autovacuum = off
fsync = off
shared_buffers = '512MB'
maintenance_work_mem = '256MB'
max_parallel_maintenance_workers = 0
CONF
    fi
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server12.log" -w start > /dev/null || die "start failed"
  fi
  # The helpers connect to $DB, which does not exist yet on a fresh cluster, so
  # this one check connects to postgres rather than failing its way to createdb.
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d postgres \
      -c "select /* wiki_btree_leg12_database_exists */ 1
            from pg_database where datname='$DB'" | grep -q 1 \
    || "$BIN/createdb" -T template0 -E UTF8 --locale=C "$DB"
  note "$(s 'select /* wiki_btree_leg12_version */ version()')"
}

# ---------------------------------------------------------------- exact ------
# The first result of this leg is the parse outcome of the unmodified text.
stage_exact() {
  say "the exact current text, unmodified, on 12.2"
  [ -f "$PAGE" ] || die "no page at $PAGE; set PAGE or run from the repository root"
  md_block sql 1 "$PAGE" > "$SQLD/est_r2.sql"
  local got; got=$(sha256sum < "$SQLD/est_r2.sql" | cut -d' ' -f1)
  [ "$got" = "$BASE1" ] && note "text hash matches the baseline" \
                        || note "text hash DIFFERS from the baseline: $got"
  if "$BIN/psql" -X -v ON_ERROR_STOP=1 -d "$DB" -f "$SQLD/est_r2.sql" \
       > "$OUT/v12_exact.txt" 2>&1; then
    printf 'exact_text=executes\n' > "$OUT/v12_facts.txt"
    note "the exact text EXECUTES on 12.2"
  else
    printf 'exact_text=refused\n' > "$OUT/v12_facts.txt"
    grep -E 'ERROR|LINE' "$OUT/v12_exact.txt" | head -4 | while read -r l; do
      printf 'exact_error=%s\n' "$l" >> "$OUT/v12_facts.txt"; done
    note "the exact text is REFUSED: $(grep -m1 ERROR "$OUT/v12_exact.txt")"
  fi
}

# ------------------------------------------------------------- transform -----
# One documented edit per construct the server refuses, applied line by line so
# the diff against the filed text is auditable.  Each edit is recorded.
#
# Since 2026-09-10 the filed text reads pg_stats_ext.inherited through
# row_to_json() instead of naming it, so this stage has nothing left to edit
# and transform_edits comes back 0.  The rule below is kept as a regression
# guard: if a future revision names the column again, this leg records the
# edit rather than failing, and the extstat stage measures what it costs.
stage_transform() {
  say "transformer: drop the constructs this server refuses"
  local line dropped=0
  : > "$SQLD/est_v12.sql"
  while IFS= read -r line; do
    case $line in
      # A server whose pg_stats_ext exposes no inherited column refuses this
      # reference.  Dropping the line widens the extstat CTE to whatever rows
      # the view does expose, which is why the filed text no longer names it.
      "                          AND se.inherited = false")
        dropped=$((dropped + 1))
        printf -- '-- dropped: %s\n' "$line" >> "$SQLD/est_v12.sql"; continue ;;
    esac
    printf '%s\n' "$line" >> "$SQLD/est_v12.sql"
  done < "$SQLD/est_r2.sql"
  printf 'transform_edits=%s\n' "$dropped" >> "$OUT/v12_facts.txt"
  if "$BIN/psql" -X -v ON_ERROR_STOP=1 -d "$DB" -f "$SQLD/est_v12.sql" \
       > "$OUT/v12_transformed.txt" 2>&1; then
    printf 'transformed_text=executes\n' >> "$OUT/v12_facts.txt"
    note "the transformed text executes after $dropped edit(s)"
  else
    printf 'transformed_text=refused\n' >> "$OUT/v12_facts.txt"
    grep -m1 ERROR "$OUT/v12_transformed.txt" >&2
    die "the transformer is incomplete; add the next refused construct"
  fi
  harness_view "$SQLD/est_v12.sql" est12 "$INTERNALS" > "$SQLD/view_v12.sql"
  fl "$SQLD/view_v12.sql" || die "harness view failed"
}

# ---------------------------------------------------------------- facts ------
stage_facts() {
  say "version-local facts this leg depends on, measured not assumed"
  { printf 'server_version_num=%s\n' \
      "$(s "SELECT /* wiki_btree_leg12_server_version */ current_setting('server_version_num')")"
    printf 'block_size=%s max_data_alignment=%s\n' \
      "$(s "SELECT /* wiki_btree_leg12_block_size */ current_setting('block_size')")" \
      "$(s 'SELECT /* wiki_btree_leg12_alignment */ max_data_alignment FROM pg_control_init()')"
    printf 'has_force_next_flush=%s\n' \
      "$(s "SELECT /* wiki_btree_leg12_has_flush */
                   (to_regprocedure('pg_stat_force_next_flush()') IS NOT NULL)::text")"
    printf 'pg_stats_ext_columns=%s\n' \
      "$(s "SELECT /* wiki_btree_leg12_stats_ext_columns */
                   string_agg(attname, ',' ORDER BY attnum) FROM pg_attribute
             WHERE attrelid = 'pg_stats_ext'::regclass AND attnum > 0")"
    printf 'btree_support_procs=%s\n' \
      "$(s "SELECT /* wiki_btree_leg12_support_procs */
                   string_agg(DISTINCT amprocnum::text, ',' ORDER BY amprocnum::text)
              FROM pg_amproc ap JOIN pg_opfamily f ON f.oid = ap.amprocfamily
              JOIN pg_am a ON a.oid = f.opfmethod WHERE a.amname = 'btree'")"
  } >> "$OUT/v12_facts.txt"
  # Whether the reloption exists is answered by trying it, not by asserting it.
  # Disposable fixture: dedup_probe is created and dropped in the sandbox.
  q "DROP TABLE IF EXISTS dedup_probe" > /dev/null 2>&1
  q "CREATE TABLE dedup_probe(k int)" > /dev/null 2>&1
  if q "CREATE INDEX dedup_probe_i ON dedup_probe (k) WITH (deduplicate_items = off)" \
       > /dev/null 2>&1; then
    printf 'deduplicate_items_reloption=accepted\n' >> "$OUT/v12_facts.txt"
  else
    printf 'deduplicate_items_reloption=rejected\n' >> "$OUT/v12_facts.txt"
  fi
  q "DROP TABLE IF EXISTS dedup_probe" > /dev/null 2>&1
  cat "$OUT/v12_facts.txt" >&2
}

# ---------------------------------------------------------------- fixtures ---
stage_fixtures() {
  say "the constructible fixture subset, one writer session per step"
  # Disposable fixtures: every statement from here to the end of the stage
  # creates or drops objects in the leg12 database of the sandbox cluster.
  fl /dev/stdin <<'SQL'
SET /* wiki_btree_leg12_client_min_messages */ client_min_messages = warning;
DROP VIEW IF EXISTS verdicts12;
DROP TABLE IF EXISTS res12, plan12 CASCADE;
CREATE TABLE plan12(num int, req text, idx text, rowsql text, want_rows bigint,
                    PRIMARY KEY (num));
CREATE TABLE res12(num int, req text, idx text, size_before bigint, size_after bigint,
                   blocks_before int, blocks_after int, status text,
                   wsp numeric, wspf numeric, caveats text, equalimage text,
                   modelled_rows numeric, key_groups numeric, tids numeric,
                   idx_reltuples numeric, dedup_applies bool, is_partial bool,
                   suppress_row bool, true_rows bigint, want_rows bigint,
                   PRIMARY KEY (num));
CREATE OR REPLACE FUNCTION plan_add(n int, r text, i text, q text DEFAULT NULL,
                                    w bigint DEFAULT NULL) RETURNS void
LANGUAGE sql AS $$ INSERT INTO plan12 VALUES (n, r, i, q, w) $$;
CREATE OR REPLACE PROCEDURE score_all() LANGUAGE plpgsql AS $sc$
DECLARE p record; e record; sb bigint; sa bigint; tr bigint;
BEGIN
  FOR p IN SELECT * FROM plan12 ORDER BY num LOOP
    tr := NULL;
    IF p.rowsql IS NOT NULL THEN EXECUTE p.rowsql INTO tr; END IF;
    sb := pg_relation_size(p.idx::regclass);
    SELECT * INTO e FROM est12 WHERE indexname = p.idx;
    IF NOT FOUND THEN RAISE EXCEPTION 'estimator returned no row for %', p.idx; END IF;
    EXECUTE format('REINDEX INDEX %I', p.idx);
    sa := pg_relation_size(p.idx::regclass);
    INSERT INTO res12 VALUES (p.num, p.req, p.idx, sb, sa, sb / 8192, sa / 8192,
      e.status, e.wasted_space_pct, e.wasted_space_pct_floor, e.caveats,
      e.equalimage, e.modelled_rows, e.key_groups, e.tids_per_tuple,
      e.idx_reltuples, e.dedup_applies, e.is_partial, e.suppress_row,
      tr, p.want_rows);
  END LOOP;
END $sc$;
CREATE VIEW verdicts12 AS
SELECT r.num, r.idx, r.req, r.blocks_before, r.blocks_after, a.actual,
       r.wsp, r.wspf,
       CASE WHEN r.wspf IS NULL                          THEN 'UNMEASURED'
            WHEN r.wspf >= 50 AND a.actual < 10          THEN 'CRITICAL FALSE POSITIVE'
            WHEN r.wspf >= 50 AND a.actual < 45          THEN 'FALSE POSITIVE'
            WHEN r.wspf >= 50 AND r.wspf - a.actual > 5  THEN 'FALSE POSITIVE'
            WHEN r.wspf <  45 AND a.actual >= 50         THEN 'FALSE NEGATIVE'
            ELSE 'PASS' END                              AS verdict_floor,
       NOT r.suppress_row AS reported,
       (r.want_rows IS NULL OR r.true_rows = r.want_rows) AS contract_ok,
       r.equalimage, r.dedup_applies, r.modelled_rows, r.idx_reltuples,
       r.status, r.caveats
  FROM res12 r
  CROSS JOIN LATERAL (
        SELECT round(100.0 * (r.size_before - r.size_after)
                     / greatest(r.size_before, 1), 1) AS actual) a;
SQL

  # -- 1-3: fresh sorted builds at three key widths -------------------------
  local l n
  for l in 8 100 1000; do
    n=$(( ((8144 - 819) / ((((12 + l + 7) / 8) * 8) + 4)) * 250 ))
    q "DROP TABLE IF EXISTS fr$l CASCADE"
    q "CREATE TABLE fr$l(k text); ALTER TABLE fr$l ALTER COLUMN k SET STORAGE PLAIN"
    q "INSERT INTO fr$l SELECT lpad(i::text, $l, '0') FROM generate_series(1, $n) i"
    loaded "fr$l" "$n"; q "ANALYZE fr$l"; analyzed "fr$l"
    q "CREATE INDEX fr_i$l ON fr$l (k)"
    q "SELECT plan_add(${l}, 'fresh sorted build, ${l}-byte key', 'fr_i$l',
                       'SELECT count(*) FROM fr$l', $n)"
  done

  # -- 1001: duplicate-heavy index, the deduplication difference ------------
  q "DROP TABLE IF EXISTS dup CASCADE"
  q "CREATE TABLE dup AS SELECT (i % 1000)::int AS k FROM generate_series(1, 500000) i"
  loaded dup 500000; q "ANALYZE dup"; analyzed dup
  q "CREATE INDEX dup_k ON dup (k)"
  q "SELECT plan_add(1001, 'duplicate-heavy index, 500 rows per key', 'dup_k',
                     'SELECT count(*) FROM dup', 500000)"

  # -- 1002-1005: the partial family ----------------------------------------
  local f
  for f in 1002 1003 1004; do
    q "DROP TABLE IF EXISTS pb$f CASCADE"
    q "CREATE TABLE pb$f AS SELECT (i % 5 = 0) AS hot, i::int AS k
         FROM generate_series(1, 500000) i"
    loaded "pb$f" 500000; q "ANALYZE pb$f"; analyzed "pb$f"
    q "CREATE INDEX p$f ON pb$f (k) WHERE hot"
  done
  q "SELECT plan_add(1002, 'freshly built partial index', 'p1002',
                     'SELECT count(*) FROM pb1002 WHERE hot', 100000)"
  q "DELETE FROM pb1003 WHERE hot AND (k / 5) % 2 = 0"
  q "VACUUM pb1003"; vacuumed pb1003; q "ANALYZE pb1003"
  q "SELECT plan_add(1003, '50% of the subset deleted', 'p1003',
                     'SELECT count(*) FROM pb1003 WHERE hot', 50000)"
  q "DELETE FROM pb1004 WHERE hot AND (k / 5) % 10 <> 0"
  q "VACUUM pb1004"; vacuumed pb1004; q "ANALYZE pb1004"
  q "SELECT plan_add(1004, '90% of the subset deleted', 'p1004',
                     'SELECT count(*) FROM pb1004 WHERE hot', 10000)"

  q "DROP TABLE IF EXISTS q1005 CASCADE"
  q "CREATE TABLE q1005 AS SELECT i::int AS id, 'pending'::text AS state
       FROM generate_series(1, 1000000) i"
  loaded q1005 1000000; q "ANALYZE q1005"; analyzed q1005
  q "CREATE INDEX p1005 ON q1005 (id) WHERE state = 'pending'"
  q "UPDATE q1005 SET state = 'done'"
  q "VACUUM q1005"; vacuumed q1005; q "ANALYZE q1005"
  q "SELECT plan_add(1005, 'drained queue, VACUUM + ANALYZE', 'p1005',
                     'SELECT count(*) FROM q1005 WHERE state = ''pending''', 0)"

  # -- 1006-1007: wide keys and a wide INCLUDE column ------------------------
  q "DROP TABLE IF EXISTS wide CASCADE"
  q "CREATE TABLE wide AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad(i::text, 12, 'n') END AS s
       FROM generate_series(1, 500000) i"
  loaded wide 500000; q "ANALYZE wide"; analyzed wide
  q "CREATE INDEX w_key ON wide (s) WHERE hot"
  q "CREATE INDEX w_inc ON wide (k) INCLUDE (s) WHERE hot"
  q "SELECT plan_add(1006, 'partial index on a wide key inside the subset', 'w_key',
                     'SELECT count(*) FROM wide WHERE hot', 25000)"
  q "SELECT plan_add(1007, 'partial index with a wide INCLUDE column', 'w_inc',
                     'SELECT count(*) FROM wide WHERE hot', 25000)"

  # -- 1008-1009: expression indexes with and without statistics -------------
  q "DROP TABLE IF EXISTS ex CASCADE"
  q "CREATE TABLE ex AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
       FROM generate_series(1, 500000) i"
  loaded ex 500000; q "ANALYZE ex"; analyzed ex
  q "CREATE INDEX ex_nostats ON ex (upper(s))"
  q "SELECT plan_add(1008, 'expression index, no statistics row', 'ex_nostats',
                     'SELECT count(*) FROM ex', 500000)"
  q "DROP TABLE IF EXISTS ex2 CASCADE"
  q "CREATE TABLE ex2 AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
       FROM generate_series(1, 500000) i"
  loaded ex2 500000
  q "CREATE INDEX ex_stats ON ex2 (upper(s))"
  q "ANALYZE ex2"; analyzed ex2
  q "SELECT plan_add(1009, 'expression index, analysed after the build', 'ex_stats',
                     'SELECT count(*) FROM ex2', 500000)"

  # -- 1010-1012: the reltuples-zero family ----------------------------------
  q "DROP TABLE IF EXISTS nz CASCADE"
  q "CREATE TABLE nz AS SELECT i::int AS k FROM generate_series(1, 1000000) i"
  loaded nz 1000000; q "ANALYZE nz"; analyzed nz
  q "CREATE INDEX nz_k ON nz (k)"
  q "DELETE FROM nz"; q "VACUUM nz"; vacuumed nz
  q "INSERT INTO nz SELECT i FROM generate_series(1, 500000) i"
  q "SELECT plan_add(1010, 'emptied, vacuumed, reloaded without ANALYZE', 'nz_k',
                     'SELECT count(*) FROM nz', 500000)"
  q "DROP TABLE IF EXISTS nzb CASCADE"
  q "CREATE TABLE nzb AS SELECT i::int AS k FROM generate_series(1, 1000000) i"
  loaded nzb 1000000; q "ANALYZE nzb"; analyzed nzb
  q "CREATE INDEX nzb_k ON nzb (k)"
  q "DELETE FROM nzb"; q "VACUUM nzb"; vacuumed nzb
  q "REINDEX INDEX nzb_k"
  q "INSERT INTO nzb SELECT i FROM generate_series(1, 500000) i"
  q "SELECT plan_add(1011, 'the same, plus a REINDEX while empty', 'nzb_k',
                     'SELECT count(*) FROM nzb', 500000)"
  q "DROP TABLE IF EXISTS trunc_t CASCADE"
  q "CREATE TABLE trunc_t AS SELECT i::int AS k FROM generate_series(1, 300000) i"
  loaded trunc_t 300000; q "ANALYZE trunc_t"; analyzed trunc_t
  q "CREATE INDEX i_trunc ON trunc_t (k)"
  q "TRUNCATE trunc_t"
  q "INSERT INTO trunc_t SELECT i FROM generate_series(1, 300000) i"
  q "SELECT plan_add(1012, 'TRUNCATE then reload without ANALYZE', 'i_trunc',
                     'SELECT count(*) FROM trunc_t', 300000)"

  # -- 1013-1016: the equal-image gate, three types and a nondeterministic
  #    collation.  What the gate answers here is a result, not an assumption.
  q "DROP TABLE IF EXISTS gate CASCADE"
  q "DROP COLLATION IF EXISTS nd12"
  q "CREATE COLLATION nd12 (provider = icu, locale = 'und-u-ks-level2',
                            deterministic = false)" \
    || note "nondeterministic ICU collation refused on this build"
  q "CREATE TABLE gate AS
       SELECT (i % 5000)::int AS a, (i % 5000)::numeric AS n,
              'key' || lpad((i % 5000)::text, 8, '0') AS s
         FROM generate_series(1, 500000) i"
  loaded gate 500000; q "ANALYZE gate"; analyzed gate
  q "CREATE INDEX g_int4 ON gate (a)"
  q "CREATE INDEX g_num  ON gate (n)"
  q "CREATE INDEX g_text ON gate (s)"
  q "CREATE INDEX g_nd   ON gate (s COLLATE nd12)" \
    || note "index on a nondeterministic collation refused"
  q "SELECT plan_add(1013, 'int4 key, gate verdict', 'g_int4', 'SELECT count(*) FROM gate', 500000)"
  q "SELECT plan_add(1014, 'numeric key, gate verdict', 'g_num', 'SELECT count(*) FROM gate', 500000)"
  q "SELECT plan_add(1015, 'text key, gate verdict', 'g_text', 'SELECT count(*) FROM gate', 500000)"
  s "SELECT 1 FROM pg_class WHERE relname = 'g_nd'" | grep -q 1 && \
    q "SELECT plan_add(1016, 'nondeterministic collation, gate verdict', 'g_nd',
                       'SELECT count(*) FROM gate', 500000)"
  note "$(s 'SELECT count(*) || '\'' fixtures planned'\'' FROM plan12')"
}

# ---------------------------------------------------------------- score ------
stage_score() {
  say "score the 12.2 subset against a measured REINDEX INDEX"
  fl /dev/stdin <<'SQL'
SET /* wiki_btree_leg12_statement_timeout */ statement_timeout = '600s';
SET /* wiki_btree_leg12_lock_timeout */ lock_timeout = '2s';
CALL /* wiki_btree_leg12_score_all */ score_all();
SQL
  [ $? -eq 0 ] || die "scoring failed"
  t "SELECT /* wiki_btree_leg12_verdict_rows */
            num, idx, blocks_before, blocks_after, actual, wsp, wspf,
            verdict_floor, reported, contract_ok, equalimage, dedup_applies,
            status, caveats FROM verdicts12 ORDER BY num" > "$OUT/verdicts12.txt" 2>&1
  t "SELECT /* wiki_btree_leg12_verdict_counts */
            verdict_floor, count(*) FROM verdicts12 GROUP BY 1 ORDER BY 2 DESC" \
    >> "$OUT/verdicts12.txt" 2>&1
  t "SELECT /* wiki_btree_leg12_gate_summary */
            equalimage, count(*), bool_or(dedup_applies) AS any_credited
       FROM verdicts12 GROUP BY 1" >> "$OUT/verdicts12.txt" 2>&1
  cat "$OUT/verdicts12.txt" >&2
}

# --------------------------------------------------------------- extstat -----
# What the portable inherited filter buys on this server, and what naming the
# column still costs here.  Two texts are rebuilt from the filed one:
#   est_pre   AND se.inherited = false, the text filed before 2026-09-10; it
#             must hash to BASEPRE and this server must refuse it
#   est_wide  est_pre with that line deleted, the old transformer output; here
#             it should agree with the filed text, because this server records
#             one ANALYZE pass and the filter it lost had nothing to remove
stage_extstat() {
  say "extstat: the portable filter, the text it replaced, and the widened one"
  [ -s "$SQLD/est_r2.sql" ] || die "no $SQLD/est_r2.sql; run the exact stage first"
  local line got
  : > "$SQLD/est_pre.sql"
  while IFS= read -r line; do
    case $line in
      "    -- The inherited flag is read through row_to_json()"*) continue ;;
      "    -- as a column, so one text also parses where"*)       continue ;;
      "    -- column: there the key is absent"*)                  continue ;;
      "    -- admits the row, which is the only ANALYZE pass"*)   continue ;;
      "                          AND coalesce((row_to_json(se)"*)
        printf '                          AND se.inherited = false\n' \
          >> "$SQLD/est_pre.sql"; continue ;;
      "                                       false) = false")    continue ;;
    esac
    printf '%s\n' "$line" >> "$SQLD/est_pre.sql"
  done < "$SQLD/est_r2.sql"
  got=$(sha256sum < "$SQLD/est_pre.sql" | cut -d' ' -f1)
  [ "$got" = "$BASEPRE" ] && note "est_pre matches BASEPRE" \
                          || note "est_pre DIFFERS from BASEPRE: $got"
  : > "$SQLD/est_wide.sql"
  while IFS= read -r line; do
    [ "$line" = "                          AND se.inherited = false" ] && continue
    printf '%s\n' "$line" >> "$SQLD/est_wide.sql"
  done < "$SQLD/est_pre.sql"

  { printf 'extstat_pre_hash=%s\n' "$got"
    if "$BIN/psql" -X -v ON_ERROR_STOP=1 -d "$DB" -f "$SQLD/est_pre.sql" \
         > "$OUT/v12_pre.txt" 2>&1; then
      printf 'pre_text=executes\n'
    else
      printf 'pre_text=refused\n'
      printf 'pre_text_note=expected: the pre-fix text is kept to prove the refusal the filed text removed\n'
      grep -E 'ERROR|LINE' "$OUT/v12_pre.txt" | head -2 \
        | while read -r line; do printf 'pre_error=%s\n' "$line"; done
    fi
    if "$BIN/psql" -X -v ON_ERROR_STOP=1 -d "$DB" -f "$SQLD/est_wide.sql" \
         > "$OUT/v12_wide.txt" 2>&1; then
      printf 'wide_text=executes\n'
    else
      printf 'wide_text=refused\n'
    fi
  } >> "$OUT/v12_facts.txt"
  harness_view "$SQLD/est_wide.sql" est_wide12 "$INTERNALS" > "$SQLD/view_wide12.sql"
  fl "$SQLD/view_wide12.sql" || die "widened harness view failed"

  # Disposable fixtures: the statements below drop and create tables,
  # statistics objects and indexes in the leg12 database of the sandbox
  # cluster.  They are not meant for a database anyone cares about.
  q "DROP TABLE IF EXISTS xchi, xpar, x2chi, x2par, x3chi, x3par, xflat,
                          xstat_res CASCADE"
  local p
  for p in x x2; do
    q "CREATE TABLE ${p}par(a int, b int, c int)"
    q "CREATE TABLE ${p}chi(a int, b int, c int) INHERITS (${p}par)"
    q "CREATE STATISTICS ${p}par_nd (ndistinct) ON a, b FROM ${p}par"
    q "INSERT INTO ${p}par SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i"
    q "INSERT INTO ${p}chi SELECT i % 500, i % 700, i FROM generate_series(1, 300000) i"
    q "CREATE INDEX ${p}par_ab ON ${p}par (a, b)"
    loaded "${p}par" 300000
  done
  q "DELETE FROM x2par WHERE c % 5 < 3"
  q "VACUUM x2par"; vacuumed x2par
  q "ANALYZE xpar";  analyzed xpar
  q "ANALYZE x2par"; analyzed x2par
  # The high-cardinality child: the shape whose inherited pass, on a server
  # that has one, moves the reading furthest.
  q "CREATE TABLE x3par(a int, b int, c int)"
  q "CREATE TABLE x3chi(a int, b int, c int) INHERITS (x3par)"
  q "CREATE STATISTICS x3par_nd (ndistinct) ON a, b FROM x3par"
  q "INSERT INTO x3par SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i"
  q "INSERT INTO x3chi SELECT i, i, i FROM generate_series(1, 300000) i"
  q "CREATE INDEX x3par_ab ON x3par (a, b)"
  loaded x3par 300000; q "ANALYZE x3par"; analyzed x3par
  q "CREATE TABLE xflat(a int, b int, c int)"
  q "CREATE STATISTICS xflat_nd (ndistinct) ON a, b FROM xflat"
  q "INSERT INTO xflat SELECT i % 10, i % 20, i FROM generate_series(1, 300000) i"
  q "CREATE INDEX xflat_ab ON xflat (a, b)"
  loaded xflat 300000; q "ANALYZE xflat"; analyzed xflat

  fl /dev/stdin <<'SQL'
CREATE TABLE xstat_res(idx text, txt text, blocks int, wsp numeric,
                       wspf numeric, key_groups numeric, ext_used bool,
                       caveats text, actual numeric);
DO $x$
DECLARE ix text; v text; e record; sb bigint; sa bigint;
BEGIN
  FOREACH ix IN ARRAY ARRAY['xpar_ab','x2par_ab','x3par_ab','xflat_ab'] LOOP
    sb := pg_relation_size(ix::regclass);
    FOREACH v IN ARRAY ARRAY['est12','est_wide12'] LOOP
      EXECUTE format('SELECT wasted_space_pct AS wsp,
                             wasted_space_pct_floor AS wspf,
                             key_groups, ext_used, caveats
                        FROM %I WHERE indexname = %L', v, ix) INTO e;
      INSERT INTO xstat_res(idx, txt, blocks, wsp, wspf, key_groups,
                            ext_used, caveats)
        VALUES (ix, v, sb / 8192, e.wsp, e.wspf, e.key_groups, e.ext_used,
                e.caveats);
    END LOOP;
  END LOOP;
  FOREACH ix IN ARRAY ARRAY['xpar_ab','x2par_ab','x3par_ab','xflat_ab'] LOOP
    sb := pg_relation_size(ix::regclass);
    EXECUTE format('REINDEX INDEX %I', ix);
    sa := pg_relation_size(ix::regclass);
    UPDATE xstat_res SET actual = round(100.0 * (sb - sa) / greatest(sb, 1), 1)
     WHERE idx = ix;
  END LOOP;
END $x$;
SQL

  { printf 'rows_per_statistics_object\n'
    t "SELECT /* wiki_btree_leg12_extstat_passes */
              tablename, statistics_name, count(*) AS view_rows,
              max(((n_distinct::text)::json ->> '1, 2')::numeric) AS nd_whole_key
         FROM pg_stats_ext WHERE schemaname = 'public'
        GROUP BY 1, 2 ORDER BY 1"
    printf 'scored against a measured REINDEX INDEX\n'
    t "SELECT /* wiki_btree_leg12_extstat_scored */
              idx, txt, blocks, actual, wsp, wspf, key_groups, ext_used
         FROM xstat_res ORDER BY idx, txt"
    printf 'filed against widened, every projected column\n'
    t "SELECT /* wiki_btree_leg12_extstat_equivalence */
              (SELECT count(*) FROM (SELECT * FROM est12
                                     EXCEPT SELECT * FROM est_wide12) d)
                AS filed_minus_wide,
              (SELECT count(*) FROM (SELECT * FROM est_wide12
                                     EXCEPT SELECT * FROM est12) d)
                AS wide_minus_filed,
              (SELECT count(*) FROM est12) AS rows_read"
  } > "$OUT/extstat12.txt" 2>&1
  cat "$OUT/extstat12.txt" >&2
}

# Both errors this leg provokes are deliberate: the reloption probe that
# discovers deduplicate_items is unknown here, and the pre-fix text kept to
# prove the refusal the filed text removed.  Anything else fails the run.
EXPECTED_ERRORS=(
  'unrecognized parameter "deduplicate_items"'
  'column se.inherited does not exist'
)
UNEXPECTED_ERRORS=0
check_server_errors() {
  local log=$1 line e known
  UNEXPECTED_ERRORS=0
  [ -f "$log" ] || { printf '   no %s to read\n' "$log"; return 0; }
  while IFS= read -r line; do
    known=1
    for e in "${EXPECTED_ERRORS[@]}"; do
      case $line in *"$e"*) known=0; break ;; esac
    done
    [ $known -eq 0 ] || { UNEXPECTED_ERRORS=$((UNEXPECTED_ERRORS + 1))
                          printf '   unexpected: %s\n' "$line"; }
  done < <(grep -E '(ERROR|FATAL|PANIC):' "$log")
  printf '   allowed=%s unexpected_server_errors=%s\n' \
         "${#EXPECTED_ERRORS[@]}" "$UNEXPECTED_ERRORS"
}

stage_report() {
  say "12.2 leg written to $OUT"
  { printf 'server_errors\n'
    check_server_errors "$OUT/server12.log"; } >> "$OUT/v12_facts.txt" 2>&1
  cat "$OUT/v12_facts.txt" >&2
  [ "$UNEXPECTED_ERRORS" -eq 0 ] \
    || die "$UNEXPECTED_ERRORS unexpected server error(s); see the end of $OUT/v12_facts.txt"
}

# Clean stop, confirmed, as in the 17 leg: -m fast rather than the -m immediate
# this stage used until 2026-09-10, then no postmaster.pid, no postgres process
# on the data directory and an empty socket directory, or the stage dies.
stage_stop() {
  say "stop the 12.2 server cleanly"
  [ -x "$BIN/pg_ctl" ] || { note "no server binary under $BIN, nothing to stop"; return 0; }
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null 2>&1 || die "pg_ctl -m fast stop failed"
    tail -3 "$OUT/server12.log" 2>/dev/null | grep -q 'database system is shut down' \
      && note "server12.log: database system is shut down"
  else
    note "not running"
  fi
  [ -e "$DATA/postmaster.pid" ] && die "$DATA/postmaster.pid still exists"
  if command -v pgrep > /dev/null 2>&1 && pgrep -f -- "-D $DATA" > /dev/null 2>&1; then
    die "a postgres process still runs on $DATA"
  fi
  [ -z "$(ls -A "$SOCK" 2>/dev/null)" ] || die "socket directory $SOCK is not empty"
  note "confirmed: no postmaster.pid, no postgres process on $DATA, socket directory empty"
}

# Containment check before any rm -rf, as in the 17 leg.  This stage removes
# only this leg's four directories: out/ and sql/ belong to the 17 leg, whose
# own clean stage deletes the whole sandbox.
inside_tmp() {
  case ${1%/} in
    "$WIKI_ROOT/.wiki-runtime/tmp"/?*) return 0 ;;
    *) return 1 ;;
  esac
}
stage_clean() {
  stage_stop
  local d
  for d in "$BUILD" "$INST" "$DATA" "$SOCK"; do
    inside_tmp "$d" || die "refusing to delete $d outside $WIKI_ROOT/.wiki-runtime/tmp/"
  done
  rm -rf "$BUILD" "$INST" "$DATA" "$SOCK"
  say "12.2 build, install, data and socket directories deleted; out/ and sql/ kept"
}

main() {
  local stages=("$@")
  [ ${#stages[@]} -eq 0 ] && stages=(build check cluster exact transform facts \
                                     fixtures score extstat report)
  local st
  for st in "${stages[@]}"; do
    case $st in
      build|check|cluster|exact|transform|facts|fixtures|score|extstat|report|stop|clean)
        "stage_$st" ;;
      *) die "unknown stage: $st" ;;
    esac
  done
}

main "$@"
```

### Reading the results of a run

**Open `out/criteria.txt` first: it is the whole verdict on one screen, and
every other file exists to explain one of its numbers.** A run writes its output
under `$SANDBOX/out`, one or more files per stage, and leaves its result tables
in the databases so they can be queried directly.

| File | Holds |
|---|---|
| `criteria.txt` | seven blocks: the gate counters, the partial-group counters, every row whose modelled row count is zero, the size of the `EXCEPT` output, the exact-text runs, the build and hash checks, and the server-error check |
| `hashes.txt` | the five text hashes against their baselines. A `DIFFER` line means the page changed and every number below it is about a different statement |
| `checks.txt`, `checks12.txt` | `make check` and the three contrib suites, per leg |
| `configure*.log`, `make*.log`, `install*.log`, `check*_*.log`, `diffs*_*.txt` | the build and regression diagnostics, copied out of the build tree so they outlive it. A `12` in the name marks the 12 leg. `diffs*` exist only when a suite failed |
| `platform.txt` | `uname -sm`, `max_data_alignment` and `database_block_size`, which every geometry constant assumes |
| `geometry.txt` | the 78-cell scorecard, then one row per (key width, fillfactor) cell |
| `gate.txt` | one row per gate fixture with `equalimage`, the metapage, whether credit was given and whether posting lists were written, then the counters and the two `DEBUG1` tallies |
| `acceptance.txt` | the fresh-build, posting-tail, compression and barrier tables, then the four defect fixtures |
| `defects.txt` | the current text beside the superseded one on the three deterministic defects, then the whole database read as the owner and as a role holding only `SELECT` |
| `calibration.txt` | the seven insertion patterns, each scored against its own `REINDEX INDEX` |
| `verdicts.txt` | every scored fixture of the numbered suite, then the verdict counts on both columns |
| `attribution.txt` | the `EXCEPT` output in both directions, taken before the first rebuild |
| `probes_gen_*.txt`, `probes_*.txt` | the probe statements the generator emitted, and the answer each returned |
| `cost.txt` | six interleaved timings of the two exact texts, and the size of the database they ran against |
| `v12_facts.txt`, `v12_exact.txt`, `verdicts12.txt` | the 12 leg: the discovered facts, the verbatim parse outcome, and the scored subset |
| `server.log`, `server12.log`, `gate_build.log` | server output, including the build's own `DEBUG1` deduplication verdicts |

The tables stay queryable after the run, and each one keys its rows
differently, which is worth knowing before writing a `WHERE` clause:

| Table | Database | Keyed by |
|---|---|---|
| `verdicts`, `res` | `suite` | `num`, `leg`, `idx` |
| `gate_res` | `gate` | `indexname` |
| `geo_result` | `geo` | `keylen`, `fillfactor` |
| `cal_res` | `cal` | `pattern` |
| `fresh_res` | `acc` | `keylen` |
| `tail_res` | `acc` | `rows_per_group` |
| `cmp_res` | `acc` | `storage` |
| `bar_res` | `acc` | `leg` |
| `verdicts12` | `leg12` | `num`, `idx` |

#### One scored row, column by column

Every fixture of the numbered suite produces one `verdicts` row. Read it in
this order.

| Column | What it is |
|---|---|
| `blocks_before`, `blocks_after` | the index's size in blocks before and after the scoring pass rebuilt it. `pg_relation_size` measures the main fork only. [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371) |
| `actual` | **the truth column**: `100 * (before - after) / before`, the reclaim a real `REINDEX INDEX` produced. Everything else is scored against it. [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2829), [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3597) |
| `wsp`, `wspf` | what the statement said: the point estimate and the floor, read *before* the rebuild. [Reading the output](#reading-the-output) |
| `old_wsp`, `old_wspf` | the same two numbers from the superseded text, for side-by-side comparison |
| `reported` | false when an exclusion term withheld the row, so the report a reader runs would never show it |
| `withheld_by` | which term did it: `A` for a missing statistics row or duplicates taken from table statistics, `B` for a table changed since its `ANALYZE`, `C` for a variable-width `INCLUDE`, `D` for a non-partial expression index with no statistics row. `unexplained` means the row is withheld by a term this harness does not know about, which is a defect in one of the two |
| `alertable` | whether this page's own reading rule would let the row become a rebuild decision: true unless `caveats` contains `never analyzed`, `row-count sources disagree`, `statistics not visible`, `zero modelled rows` or `wide compressible key` |
| `contract_ok` | whether the fixture built what it intended. False is a **harness** fault, not an estimator result: the row's numbers describe a different fixture from the one the requirement names |
| `verdict_point`, `verdict_floor` | the classification below, computed once per column |
| `caveats`, `equalimage`, `modelled_rows`, `idx_reltuples`, `tids` | the statement's own explanation of the reading |

#### The five verdicts

Both verdict columns use the same rule, one on `wsp` and one on `wspf`:

| Verdict | Condition | What it means |
|---|---|---|
| `PASS` | none of the below | the estimate and the measured reclaim agree closely enough to act on |
| `CRITICAL FALSE POSITIVE` | estimate >= 50 and `actual` < 10 | the statement called for a rebuild that would have reclaimed nothing. This is the failure that matters operationally |
| `FALSE POSITIVE` | estimate >= 50 and `actual` < 45, or estimate exceeds `actual` by more than 5 points | the statement over-stated a real saving |
| `FALSE NEGATIVE` | estimate < 45 and `actual` >= 50 | the statement hid a rebuild worth doing |
| `UNMEASURED` | the estimate is NULL | the statement declined to model the index, which for these fixtures means a `reltuples` sentinel of -1 after a rebuild or a `TRUNCATE`. [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3951-L3952), [index.c#index_update_stats-sentinel](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2836) |

A verdict is only as interesting as `reported` and `alertable` make it. A
critical false positive that is withheld costs a reader nothing; the same
verdict on a row that is both reported and alertable is a defect the statement
owes a fix for. That three-way reading — verdict, `reported`, `alertable` — is
the one this page scores itself on.

#### What a clean run looks like

| Check | Clean value |
|---|---|
| `hashes.txt` | five `match` lines |
| `checks.txt` | four `=0` lines, `All 225` on the core suite |
| Exact texts | three files under `out/exact_*`, none containing `ERROR` |
| Geometry | `leaf_exact` and `relpages_exact` both equal `cells` |
| Gate | `over_credit` 0, `metapage_disagreements` 0, `worst_reading` under 30 |
| Numbered suite | `withheld_unexplained` 0, `contract_failures` 0 |
| Probes | one line per emitted probe, and `false` on every subset the fixture drained, `true` on every subset it refilled |
| Attribution | every `EXCEPT` row explainable by one of the five documented changes; an unexplained row is a regression |
| 12 leg | `transformed_text=executes`, and `transform_edits` no larger than the page documents |
| Server errors | block 7 of `criteria.txt` reads `unexpected_server_errors=0`, and the same line closes `v12_facts.txt` on the 12 leg. Each leg allows exactly the errors it provokes on purpose — two on 17.11, two on 12.2 — and dies on anything else, so a stray error can no longer hide among them |

Three things that look like failures and are not. A **withheld** row is the
exclusion terms working, not a miss. A **negative** percentage is
over-prediction — the model expects the rebuild to be *larger* than the file,
which is conservative and never triggers an alert. And a **zero** `blocks_after`
difference on a fresh build is the point of that fixture, not a null result.

#### When something goes wrong

| Symptom | Where to look, and what it means |
|---|---|
| A stage dies with `!!` | the message names the stage. `configure`/`make` failures land in `build17/configure.log` and `build17/make.log`; a server that will not start lands in `out/server.log` |
| `DIFFER` in `hashes.txt` | the page's SQL changed since the baselines were recorded. Re-baseline deliberately; do not compare the run against older numbers |
| `estimator returned no row for X` | the scoring procedure could not find the index in the harness view. The view was not installed, or the fixture did not create the index |
| `contract_ok` false | the fixture, not the statement. Fix the recipe or the intended count before reading its verdict |
| `withheld_by = unexplained` | the statement withheld a row for a reason the harness cannot name — either a new exclusion term or a stale harness |
| A verdict that moves between runs | check `modelled_rows` first. Fixture 120 is known to flip with the `ANALYZE` sample; see [Fixture verdicts that depend on the ANALYZE sample](#fixture-verdicts-that-depend-on-the-analyze-sample) |
| Caveats appearing on fresh fixtures | a statistics-publication ordering fault, not an estimator one: a reading taken inside the transaction that built the fixture sees the table's statistics as they were before its own `ANALYZE`. [pgstat.c#pgstat_force_next_flush](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L700-L708), [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L289-L337) |
| The 12 leg refuses a construct the transformer does not handle | `stage_transform` stops and says so. Add the refused construct as one more documented edit; the count in `v12_facts.txt` is the honest measure of how far the text is from portable |

Two files the table above does not cover, both written by the `extstat` stages:
`out/extstat.txt` holds the 17 leg's text hashes, the per-database equivalence
counts, the two `ANALYZE` passes of each fixture, the three-text scorecard and
the timing pairs; `out/extstat12.txt` holds the 12 leg's rows per statistics
object, its two-text scorecard and its equivalence count. `out/v12_facts.txt`
gains `pre_text=`, `pre_text_note=` and `wide_text=` lines from the same stage;
the note is there because `pre_text=refused` is the expected result and reads
like a failure without it.

The next section is this reading applied to one run.

### What the two scripts measured on 2026-09-09

This is the previous run's record, superseded as the last-run record by
[What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured);
its cross-version finding is the one the 2026-09-10 fix reverses. **Date**
2026-09-09; **pin**
`786db8dcf168bd9df8f55047337525ac19118b1c` for the 17 leg and
`45b88269a353ad93744772791feb6d01bc7e1e42` for the 12 leg; **servers** 17.11 and
12.2 built from those two checkouts; **platform** `Linux x86_64`,
`max_data_alignment` 8, `database_block_size` 8192, gcc 13.3.0. The numbers
below were produced by the script text as it stood that day, before the
[Measurement-script section review](#measurement-script-section-review) repaired
it; the repaired text was run in full on 2026-09-10, see
[What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64).

Everything below is output from the two scripts as filed above, on the host
recorded under [Re-verified on a rebuilt server](#re-verified-on-a-rebuilt-server).
17.11 was configured `--enable-debug --with-icu --with-readline --with-zlib`
and its suites passed **All 225** core tests plus 8, 1 and 3 for `pageinspect`,
`pgstattuple` and `amcheck`. The five text hashes all matched their baselines,
and both exact texts executed as filed.
[regress.sgml#make-check](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59),
[regress.sgml#contrib-suites](../../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195).

| Family | Result |
|---|---|
| Page geometry, 78 cells | items per closed leaf page equal the closed form in **78 of 78**; the whole-index `relpages` equals the model in **78 of 78**; no internal level exceeds the modelled fanout |
| Fresh sorted builds, 10 key widths | the current text reports **0 bytes on 10 of 10**; the superseded text is right on 6, and reads `-0.7`, `-3.4`, `+11.0` and `-33.9` percent at 400, 800, 1000 and 2000 bytes |
| Deduplication gate, 28 fixtures | **0 over-credits**; `equalimage` agrees with `bt_metap().allequalimage` on all 12 `recognized` and 9 `ineligible` rows; 2 under-credits (`i_ei_true`, `i_squat`), both conservative; worst reading 28.8 % (`i_multi_bad`) |
| Posting tails, 13 classes | mean absolute error **11.38 % -> 0.38 %**; within one point **8 -> 11 of 13** |
| In-index compression | 367 blocks against 10,003 for the same 60,000 900-byte values, `avg_width` 904 in both; `-2625.6 %` with the caveat against `0.0 %` |
| Statistics barrier | with the barrier, `n_mod_since_analyze` 0 and `n_live_tup` 200,000; without it, 200,000 and 400,000 |
| Calibration, 7 patterns | sorted 0.0/0.0, append 0.0/0.0, random 25.7 against a measured 25.7, duplicate-heavy `-6.7` against `-7.5` with the floor at `-245.2`, wide random 23.8/23.8, delete churn 49.8/49.8, update churn 33.3/33.3 |
| Numbered suite, 112 fixtures | 59 reported, 53 withheld, and **every withheld row names the term that withheld it** — 34 change A, 10 change B, 5 change C, 4 change D; **0 contract failures** |
| Attribution | 26 rows differ in each direction over 114 indexes: 23 are a moved number and 3 are a caveat string the current text adds |
| Probes | 73 emitted and executed; the five population probes answer `false` for `p113b`, `p113c` and `p117` and `true` for `p115` and `p118`; the group probe counts 5,000 against a modelled 4,996 |
| Cost | six interleaved pairs over 325 B-tree indexes and 85,017 blocks: 61.5-67.4 ms for the current text against 52.5-56.1 ms for the superseded one |

**The gate group scores the same way it did on the 27-fixture run.** The 28th
fixture is the unique index `i_uniq`, which the metapage marks equal-image and
the statement leaves uncredited, exactly as the three `deduplicate_items = off`
fixtures are. The two under-credits are the designed `i_ei_true` and the
impostor `i_squat`; both are indexes the engine did deduplicate and the model
priced without credit.
[nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183),
[nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L67-L84).

**The partial-index half of the suite has now been scored against the current
text.** On the floor column the 112 fixtures come out 80 PASS, 20 critical
false positives, 7 false negatives, 3 false positives and 2 unmeasured; on the
point estimate 76, 26, 5, 3 and 2. Most of those failures are withheld rather
than reported, which is what the exclusion terms are for. Among the 59 rows the
report would actually show:

| Group | Rows |
|---|---|
| Critical false positives | 6: `f84` 94.2, `i103` 84.1, `x108` 62.5, `x109` 62.5, `p118` 99.3, `p120` 87.5, every one against a measured 0.0 |
| ... of those, alertable under this page's reading rule | **3**: `f84`, `i103`, `x109`. `x108` carries `never analyzed`, and `p118` and `p120` carry `zero modelled rows`, all of which [Reading the output](#reading-the-output) tells a reader not to promote |
| True detections | 12, every one a PASS: 68 (87.4 against 87.4), 74 (73.6/74.3), 75 (88.8/89.1), 76 (50.3/49.9), 77 (94.2/94.2), 92 (89.1/89.1), 95 (89.1/89.1), 99 (81.3/93.6), 107 (90.0/90.0), 113b and 113c (100.0/100.0), 114 (98.9/98.9) |
| False negatives | 1 reported (`f91`, `-254.1` against a measured 89.2); the other six are withheld |
| Unmeasured | 2, both change-E shapes: `nzb_k` and `i_trunc`, whose rebuilt-or-truncated indexes carry the `reltuples` sentinel. [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3951-L3952), [index.c#index_update_stats-sentinel](../../../../raw/postgres-17/src/backend/catalog/index.c#L2835-L2836) |

Five predictions of
[Expected verdicts under the current statement](#expected-verdicts-under-the-current-statement)
are confirmed and two are corrected:

| Prediction | Outcome |
|---|---|
| `i103` stays a reported critical false positive | confirmed at **84.1 %**, the 2026-08-24 figure to the decimal |
| `x109` reports with the new `statistics target zero on an index column` caveat | confirmed; it is one of the three EXCEPT rows that differ by caveat alone |
| fixture 118 reports 99.3 % with `zero modelled rows`, and the probe catches it | confirmed on both counts: 99.3 % against a measured 0.0, and the emitted `EXISTS` probe returns `true` |
| 113b and 113c read 100.0 | confirmed, against a measured 100.0 |
| test 36's one key group of 100,000 TIDs stays exact | **not reproduced**: this reconstruction reads `-217.2 %`, because its table-wide `n_distinct` describes the 400,000 rows outside the predicate rather than the one value inside it. The mechanism the original fixture demonstrated is unchanged; the fixture is not the same fixture |
| every withheld row names its term | confirmed, 53 of 53 |
| `p120` reads 87.5 % | **confirmed but nondeterministic**: three runs of the same script read 87.5, `-50.0` and 87.5, because a `default_statistics_target` of 1 samples 300 rows and either misses the 2,000-row subset or does not. [analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894) |

**The 12.2 leg answers the cross-version question, and the answer is no.** The
exact current text does not execute on the pinned 12 checkout. It is refused
with `ERROR: column se.inherited does not exist`, pointing at the `extstat`
stage, because this version's `pg_stats_ext` view defines `inherited` from
`stxdinherit` and the server the leg ran against exposes no such column.
[system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290),
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309).

One transformer edit — deleting that one filter line — makes the text run
there. What the leg then recorded, all of it measured rather than assumed:

| Recorded | Value |
|---|---|
| `server_version_num`, block size, alignment | 120002, 8192, 8 |
| Engine suites of that build | All 192 core tests, plus 5, 1 and 2 for the three contrib modules |
| `pg_stat_force_next_flush()` | absent, so the leg polls instead |
| `pg_stats_ext` columns | thirteen, none of them `inherited` |
| B-tree support-function numbers registered | 1, 2, 3 |
| `WITH (deduplicate_items = off)` | rejected |
| Gate verdict on all 19 fixtures | `ineligible`, and nothing credited |
| Fresh sorted builds at 8, 100 and 1000 bytes | 254, 258 and 353 blocks, the same block counts as 17.11, each read as 0.0 % |
| Partial family | 50 % deleted 49.6 against a measured 49.6; 90 % deleted 89.9 against 89.1; drained queue 100.0 against 100.0 |
| The wide-key partial index | 84.1 % against a measured 0.0, the same critical false positive `i103` is on 17.11 |
| `nzb_k` and `i_trunc` | 99.9 % each, where 17.11 reports `unmeasured: reltuples unknown` for the same two recipes |

The 12.2 build needed `-DTRUE=1 -DFALSE=0` in `CFLAGS` on this host, because
its ICU headers no longer define those macros; the script carries that in
`EXTRA_CFLAGS` with a comment, and a host with older ICU headers can empty it.

### What the 2026-09-10 targeted re-run measured

This was the last-run record until the full re-run later the same day,
recorded under
[What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64).
**Date** 2026-09-10; **pin**
`786db8dcf168bd9df8f55047337525ac19118b1c` for the 17 leg and
`45b88269a353ad93744772791feb6d01bc7e1e42` for the 12 leg; **servers** the same
17.11 and 12.2 clusters the 2026-09-09 run built under
`.wiki-runtime/tmp/btree-suite/`, on ports 55437 and 55412; **platform**
`Linux x86_64`, `max_data_alignment` 8, `database_block_size` 8192, gcc 13.3.0,
default `BLCKSZ`, `--locale=C`, UTF8 databases. Neither server was rebuilt and
neither regression suite was re-run, so `out/checks.txt` and
`out/checks12.txt` still hold the 2026-09-09 results.

**Both servers were stopped and the sandbox was deleted on 2026-09-10.** Each
server was shut down first with `pg_ctl -D <datadir> -m fast stop`, which wrote
a shutdown checkpoint and logged `database system is shut down`, leaving no
`postmaster.pid`, no postgres process, an empty socket directory and no
listener on either port. `.wiki-runtime/tmp/btree-suite/` was then removed,
taking both builds, both installs and both cluster data directories with it.
Nothing measured on this page survives on disk, so reproducing any number means
rebuilding both legs from their pins and re-running the scripts under
`## Measurement Script` from the start. The scripts lose nothing to the
deletion: the two copies that ran were byte-identical to the two `bash` blocks
on this page, checked before the removal.

**Scope: the stages the statement change can reach, not the whole suite.** The
17 leg ran `cluster texts extstat`; the 12 leg ran
`exact transform facts fixtures score extstat report`. The 17 leg's
`geometry`, `calibration`, `gate`, `acceptance`, `suite`, `attribution`,
`probes`, `score`, `cost` and `criteria` stages were **not** re-run, on the
argument the `extstat` stage measures directly: the two texts return identical
rows in every database those stages populate. That is evidence the edit is
inert, not a re-validation of the numbers themselves, which still date from
2026-09-09.

Runtimes on this host, from a built tree and a running cluster: 0.3 s for the
12 leg's `exact transform facts`, 52.7 s for its
`fixtures score extstat report`, and 7.4 s for the 17 leg's `extstat`.

The 17 leg:

| Check | Result |
|---|---|
| `hashes.txt` | five `match` lines, `est_r2` now `646df923…` |
| Exact filed text | runs on `suite` and `acc`; the superseded text still runs on `suite` |
| `est_pre` rebuilt from the filed text | `8acd531b…`, matching `BASEPRE` — the previous text, byte for byte |
| `est_wide`, the old transformer's output | `615d1674…` |
| Equivalence, `EXCEPT` both ways over every projected column | `0` and `0` in all six databases: `geo` 0 rows, `cal` 7, `gate` 28, `acc` 33, `suite` 114, `xstat` 4 |
| Of those 186 rows, how many the `extstat` CTE actually fed | 7: three in `suite`, four in `xstat`. The other five databases hold no extended-statistics object, so they test the parse and the join, not the filter |
| Three texts scored on four fixtures | filed = previous on 4 of 4; widened differs on all three inheritance parents and on none of the controls. [The portable extended-statistics filter](#the-portable-extended-statistics-filter) |
| Cost, six interleaved pairs on `suite` | filed 89.211, 106.909, 75.400, 94.072, 111.028, 95.151 ms (mean 95.3); previous 85.216, 98.689, 99.962, 84.907, 86.859, 93.872 ms (mean 91.6) |

The 12 leg, and this is the result the fix was filed for:

| Check | 2026-09-09 | 2026-09-10 |
|---|---|---|
| The exact filed text, unmodified | `exact_text=refused` | **`exact_text=executes`** |
| Transformer edits needed | 1 | **0**, `transformed_text=executes` |
| The previous text, rebuilt and re-run | not tested | `pre_text=refused`, `ERROR: column se.inherited does not exist` at `LINE 116: AND se.inherited = false`, from a text hashing to `BASEPRE` |
| The widened text | executes | `wide_text=executes` |
| `pg_stats_ext` columns | thirteen, none `inherited` | unchanged: `schemaname, tablename, statistics_schemaname, statistics_name, statistics_owner, attnames, kinds, n_distinct, dependencies, most_common_vals, most_common_val_nulls, most_common_freqs, most_common_base_freqs` |
| `server_version_num`, block size, alignment | 120002, 8192, 8 | unchanged |
| `pg_stat_force_next_flush()`, support procs, `deduplicate_items` | absent; 1,2,3; rejected | unchanged |
| Rows per statistics object on the four new fixtures | not tested | one each, whole-key `n_distinct` 20, 8, 20, 20 — one pass, so nothing for a filter to remove |
| Filed against widened, `EXCEPT` both ways | not tested | `0` and `0` over 25 rows, and identical readings on all four fixtures |
| The 19 numbered fixtures | 13 PASS, 5 critical false positives, 1 false positive | **the same counts**, all 19 `ineligible` with nothing credited |

Two 12-leg numbers moved, both by less than half a point and both
sample-dependent: the 50 %-deleted partial index reads 49.3 against a measured
49.6 where it read 49.6 on 2026-09-09, and the 90 %-deleted one reads 89.5
against a measured 89.1 where it read 89.9. The fresh sorted builds are
unchanged at 254, 258 and 353 blocks and 0.0 %; the drained queue is unchanged
at 100.0 against 100.0; `w_key` is unchanged at 84.1 against a measured 0.0;
`nzb_k` and `i_trunc` are unchanged at 99.9.

The four new fixtures read 827 blocks each on 12.2 against 258 on 17.11 for the
same rows and the same keys, and the 12 leg credits no deduplication anywhere,
which is consistent with its `equalimage` verdict of `ineligible` on all 19
numbered fixtures and its refusal of `WITH (deduplicate_items = off)`. Those
are measurements of that server; this page cites only the 17 checkout, so the
engine reason belongs on the v12 page.
[The v12 publication protocol](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md#the-v12-publication-protocol).

### Measurement-script section review

**This section was audited against `MANDATORY Measurement Script` on 2026-09-09,
read-only, with no server started. Nothing measured changed; thirteen defects in
the section were repaired in place, and the biggest one was structural: the two
scripts were filed as subsections of `## Answer`, where the rule requires one
top-level `## Measurement Script` section between `## Answer` and
`## Context Reviewed`, listed in `## Contents`.** The scripts themselves were
already Bash and SQL only, stage-selectable, sandboxed and hash-checked, which
is why the rule cites this page as its precedent; what the audit found was
paperwork drift, three unnamed GUCs, and four places where an error or a stray
variable could pass without being caught.

What the audit verified before changing anything:

| Check | Result |
|---|---|
| The four `sql` baselines, re-derived from this page with the scripts' own `md_block` logic in pure Bash | `8acd531b…`, `bfa7721f…`, `0b03f0c9…`, `3e57a568…`, all four matching `BASE1`-`BASE4` |
| The superseded text, recovered with `git show f2d73b4:` and hashed | `bffd166e…`, matching `BASEOLD` |
| Both script bodies, parsed | `bash -n` clean on the 1,793-line and 474-line texts as filed, and again on the 1,843-line and 507-line texts after the repairs below |
| Stage lists against the dispatchers | the 17 leg's `main` and `case` both carry 16 stages plus `stop` and `clean`; the 12 leg's carry 9 plus the same two |
| Source citations inside the section | 37 citations, every one resolving, in bounds, and inside `raw/postgres-17/`; the 7 whose label token is not literally in range were read by hand and each supports its label |
| Both pinned checkouts | clean at `786db8dcf16` and `45b88269a35`, never written to |
| Isolation | `set -uo pipefail` in both, every artifact under `.wiki-runtime/tmp/`, out-of-tree VPATH builds, own socket directory, ports 55437 and 55412, `listen_addresses = ''` |

The thirteen repairs:

| # | Defect | Repair |
|---|---|---|
| 1 | The scripts sat under `## Answer`; the rule requires a top-level `## Measurement Script` section after `## Answer` and before `## Context Reviewed`, listed in `## Contents` | the section exists, with the scripts, the protocol, the reading guide, the last-run record and this review under it. The subsection headings did not change, so every existing link into them still resolves |
| 2 | No usage information: no per-stage description, no environment defaults, no prerequisites, no read-first pointer for the 12 leg, no runtime for it | [How to use the suite scripts](#how-to-use-the-suite-scripts) states all eight items the rule names, with a stage table per leg, a variable-and-default table, and a prerequisites list |
| 3 | Step 3 of the protocol published an `awk` extraction and `shasum -a 256`. `MANDATORY Measurement Script` forbids `awk` and `perl`, and `shasum` is a Perl program | the recipe is now the scripts' own `md_block` plus `sha256sum`, which is what `stage_texts` has always run |
| 4 | The 17 script's header comment omitted the `cost` stage that its dispatcher, its `case` and the stage table all carry | `cost` added to the comment |
| 5 | The same comment advertised a `KEEP` variable the script never reads | removed from the comment; nothing read it, so nothing else changed |
| 6 | `s()` and `t()` ran `psql` without `ON_ERROR_STOP`, and `stage_cost` ran a two-action `psql -c '\timing on' -f file`. Without it a failed statement in a `-f` script leaves the status 0, so a timing could be recorded for a statement that errored | every helper in both scripts now carries `-X -v ON_ERROR_STOP=1`, including the two `stage_cost` invocations and the probe generator. [mainloop.c:376](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L376), [mainloop.c#die_on_error](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L587-L594), [startup.c#single-query-action](../../../../raw/postgres-17/src/bin/psql/startup.c#L377-L386), [psql-ref.sgml#Exit-Status](../../../../raw/postgres-17/doc/src/sgml/ref/psql-ref.sgml#L627-L636) |
| 7 | `stage_clean` ran `rm -rf "$SANDBOX"` on a value taken from the environment, with no containment check | both `clean` stages call `inside_tmp`, which accepts only a path strictly inside `$WIKI_ROOT/.wiki-runtime/tmp/` and refuses the `tmp` directory itself, the repository root and lookalikes such as `tmpx` |
| 8 | Nothing checked that `$PAGE` and the pinned checkout exist. With `set -u` but no `-e`, a missing page produced empty SQL files and a confusing failure three stages later | `stage_build`/`stage_texts` and the 12 leg's `stage_build`/`stage_exact` die with the variable to set |
| 9 | `stage_probes` declared `local line kind sql` but read into `name`, leaving `name` a global and `line` unused | the declaration is now `local name kind sql` |
| 10 | `stage_attribution` ended with `grep -c '^ [a-z]' ... > /dev/null`, whose result went nowhere | removed; `stage_criteria` already counts that file |
| 11 | The cluster settings table named nine settings, but the scripts also write `listen_addresses`, `port` and `logging_collector`, and `MANDATORY GUC Changes` wants a context and apply scope for every setting a script sets | three rows added, all `PGC_POSTMASTER`, so restart. [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4437-L4441), [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2394-L2397), [guc_tables.c#logging_collector](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1641-L1644) |
| 12 | The rule wants fixture statements marked disposable; no fixture block said so, while the `suite` stage drops the whole `public` schema and the acceptance stage drops and recreates a login role | every fixture block now opens with a disposability banner naming the database it writes, and the usage section says the same in one paragraph |
| 13 | `md_block sql N` indexes this page's `sql` blocks positionally, and nothing said so; a new `sql` block above the calibration harness would silently repoint both scripts | the rules table now carries that constraint and names `out/hashes.txt` as the guard |

Three findings were left as they are, deliberately:

- **Fixture DDL carries no statement tag.** `MANDATORY Production SQL` asks for
  an inline tag after the leading verb, and `MANDATORY Measurement Script`
  applies it to "the statements the script sends". The audit added 47 tags to
  the statements whose output this page publishes — every reporting query, both
  timeout `SET` pairs, the two `CALL score_all()` invocations and the
  version/platform probes — and left about 740 fixture `CREATE`, `INSERT`,
  `ANALYZE` and `DROP` lines untagged, on the reading that the same rule treats
  fixtures separately by asking that they be marked disposable instead.
  See [Fixture statements are marked disposable, not tagged](#fixture-statements-are-marked-disposable-not-tagged).
- **The 12 leg's cluster settings have no apply scope on this page**, because a
  v17 page may not cite a v12 checkout.
  See [The 12 leg's settings have no citable apply scope here](#the-12-legs-settings-have-no-citable-apply-scope-here).
- **The 12 leg's runtime was never recorded.** Its from-a-built-tree half was
  measured later that day and its full-run figure on 2026-09-10; see
  [What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64).

Every repair above is either a comment, a documentation table, or a fail-closed
guard on a path that previously had none; none of them changes a statement, a
fixture, a threshold or a stage. Even so, the scripts were **not** re-run, so
the figures under
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09)
predate the script text now filed, and
`verified_by_agent` stays `not yet`.
That gap was closed on 2026-09-10 by the full re-run recorded under
[What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64).

### The stop stage, repaired

**Both scripts stopped their server with `pg_ctl -m immediate`, which is not a
clean shutdown, and printed "server stopped" whether or not anything had been
running. Both `stop` stages were rewritten on 2026-09-10 to stop with
`-m fast -w` and to confirm the teardown before returning, and `clean`, which
calls `stop` first, now refuses to delete a data directory it could not
confirm stopped.** The 2026-09-10 teardown rule found the defect while
stopping the two servers the 2026-09-09 run had left behind, and reported it
rather than fixing it; this pass fixes it and then re-runs both legs end to
end under the repaired text.

What the two modes do, read from the 17 source. On a fast shutdown request the
postmaster logs `received fast shutdown request`, moves to `PM_STOP_BACKENDS`
so that every child is sent `SIGTERM`, and lets its state machine take the
next step; when the checkpointer's own shutdown request is pending it calls
`ShutdownXLOG()`, which writes a checkpoint flagged
`CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_IMMEDIATE`, and the next start finds the
control file in `DB_SHUTDOWNED` and skips recovery. On an immediate request
the postmaster sends every child `SIGQUIT` and, in its own words, exits
"without attempt to properly shut down the data base system"; the next start
then logs `database system was not properly shut down; automatic recovery in
progress`. `pg_ctl`'s usage text says the same in one line each: `fast` is
"quit directly, with proper shutdown (default)", `immediate` is "quit without
complete shutdown; will lead to recovery on restart".
[postmaster.c#process_pm_shutdown_request-fast](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2266-L2305),
[postmaster.c#process_pm_shutdown_request-immediate](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2307-L2342),
[checkpointer.c#ShutdownRequestPending](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L584-L600),
[xlog.c#ShutdownXLOG](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6580-L6621),
[xlogrecovery.c#not-properly-shut-down](../../../../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L922-L949),
[pg_ctl.c#shutdown-modes-usage](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L2006-L2011),
[pg_ctl-ref.sgml#shutdown-modes](../../../../raw/postgres-17/doc/src/sgml/ref/pg_ctl-ref.sgml#L186-L198).

What the repaired stage does, in order, and why each check is enough:

| Step | Check | Evidence |
|---|---|---|
| 1 | `pg_ctl status` on the data directory, tried only when `postmaster.pid` exists; a sandbox that was never started, or is already stopped, is reported as `not running` and nothing is sent | `status` prints `no server running` and exits 3 when there is no live postmaster. [pg_ctl.c#do_status](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L1336-L1388), [pg_ctl-ref.sgml#status-exit-status](../../../../raw/postgres-17/doc/src/sgml/ref/pg_ctl-ref.sgml#L221-L227) |
| 2 | `pg_ctl -D <data> -m fast -w stop`; the stage dies if it fails | `fast` is the documented default mode, and with `-w` `do_stop` sends the signal, waits in `wait_for_postmaster_stop()`, and exits 1 rather than print `server stopped` when the postmaster does not go away. [pg_ctl-ref.sgml#-m](../../../../raw/postgres-17/doc/src/sgml/ref/pg_ctl-ref.sgml#L310-L320), [pg_ctl.c#do_stop](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L1015-L1065) |
| 3 | the last three lines of the server log are searched for `database system is shut down`, and the stage notes it when found | that line is written by the postmaster's lock-file removal callback, which the source calls "the last externally visible action of a postmaster". [miscinit.c#UnlinkLockFiles](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L1170-L1194) |
| 4 | no `postmaster.pid` in the data directory; the stage dies if one remains | the same callback unlinks it. [miscinit.c#UnlinkLockFiles](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L1170-L1194), [miscinit.c#DIRECTORY_LOCK_FILE](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L60) |
| 5 | no process whose command line carries `-D <data>`, checked with `pgrep -f` when `pgrep` exists; the stage dies on a match | `pg_ctl` starts the postmaster as `exec "<postgres>" -D "<data>" ...`, so a match is a postmaster still holding that directory. [pg_ctl.c#start_postmaster-command](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L489-L494), [pg_ctl.c#pgdata_opt](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L2278) |
| 6 | the socket directory is empty; the stage dies otherwise | the postmaster unlinks every socket file it created at shutdown. [pqcomm.c#RemoveSocketFiles](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L846-L861) |

The 12 leg's stage is the same text against `server12.log`, `data12` and
`sock12`. Neither stage changes a statement, a fixture, a threshold or a
measured number; what changes is that a run can no longer end with a cluster
that needs recovery, and that a failed stop is an error rather than a note.
The stage tables under [The 17 leg's stages](#the-17-legs-stages) and
[The 12 leg's stages](#the-12-legs-stages) record the old mode beside the new
one.

Two paperwork drifts were fixed in the same pass. The "stages, in default
order" table under
[The two suite scripts, and the rules they follow](#the-two-suite-scripts-and-the-rules-they-follow)
omitted `extstat` for both legs, although both scripts have run it by default
since that morning's filter change, and the last-run marker on
[What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured)
now points at the run below.

Two more script defects surfaced in the first full run of the day and were
fixed before the second. The 12 leg's `cluster` stage tested whether its
`leg12` database existed through a helper that connects to that very
database, so on a fresh cluster the test failed with
`psql: error: could not connect to server: FATAL: database "leg12" does not
exist` and fell through to `createdb`; it worked by accident and printed an
error. The check now connects to `postgres`. And the 17 leg ran `extstat`
right after `texts`, so in a fresh full run its equivalence counts and its
cost pairs saw an empty `suite` database (`rows=0`) where the targeted re-run
had seen 114 rows; the stage now follows `suite` in the default order, and the
full run below reports the same 186 rows, 7 of them fed by the changed CTE,
as the targeted re-run did.

### What the 2026-09-10 full re-run measured on Darwin arm64

**This is the last-run record. Both scripts were run end to end, twice, from
an empty sandbox, on a second platform, under the repaired script text, and
every verdict count of the 2026-09-09 Linux run reproduced.** The first pair
of runs carried the `stop` repair alone; the second pair, whose numbers are
filed here, carried the two further fixes above as well. The two pairs agreed
on every count and differed only in the sample-dependent cells named below.

**Date** 2026-09-10. **Pins** `786db8dcf168bd9df8f55047337525ac19118b1c` for
the 17 leg and `45b88269a353ad93744772791feb6d01bc7e1e42` for the 12 leg.
**Platform** `uname -sm` `Darwin arm64`; macOS 26.6.2; the server banners read
`PostgreSQL 17.11 on aarch64-apple-darwin25.6.0, compiled by Apple clang
version 21.0.0 (clang-2100.3.34.2), 64-bit` and
`PostgreSQL 12.2 on arm-apple-darwin25.6.0`, same compiler; ICU 78.3 from
Homebrew's `icu4c@78`, reached through `ICU_CFLAGS` and `ICU_LIBS` because
the host has no `pkg-config`; GNU bash 5.3.15; `sha256sum` from `/sbin`;
ten CPU cores and 16 GB of memory, with `JOBS=8`. `pg_control_init()`
reports `max_data_alignment` **8** and `database_block_size` **8192**, the
same values as the Linux host, so the geometry constants were measured at the
alignment and block size they assume. Both clusters ran `--locale=C` with
UTF8 databases and the settings under
[The two suite scripts, and the rules they follow](#the-two-suite-scripts-and-the-rules-they-follow);
both builds used `--enable-debug --with-icu --with-readline --with-zlib`,
the 12 leg with `CFLAGS="-O2 -g -DTRUE=1 -DFALSE=0"`. Neither checkout was
written to; both are clean at their pins.
[pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997),
[installation.sgml#ICU_CFLAGS](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L184-L193).

**Runtimes on this host.** The 17 leg's full run, from configure through
`report`, took **2 min 26 s** (`real 2m26.017s`; the first run of the day
2m26.072s). The 12 leg's full run took **1 min 52 s** (`real 1m51.476s`;
first run 1m51.927s), which is the figure the open question on the 12 leg's
runtime was missing; that question is closed. From a built tree and a running cluster, the
17 leg's `suite attribution probes score criteria` took **65 s**
(`1:05.34 total`). The Linux host's figures were about eleven minutes and
about ninety seconds.

**Teardown.** Each leg's `clean` stage ran after its results were copied out:
both servers stopped with `pg_ctl -m fast -w stop`, each `server.log` ended
in `database system is shut down`, and each stage confirmed no
`postmaster.pid`, no postgres process on its data directory and an empty
socket directory before deleting anything. The sandbox
`.wiki-runtime/tmp/btree-suite/` was 8.4 GB (`data17` 6.3 GB, `data12`
1.8 GB, `build17` 216 MB, `build12` 139 MB, `install17` 38 MB, `install12`
30 MB) and is gone; afterwards `pgrep` found no postgres process, ports 55437
and 55412 were free, and `.wiki-runtime/tmp/` held only
`btree-suite-scripts/`, 1.8 MB of script copies, run logs and the `out/`
text of both runs, kept as generated artifacts under `.wiki-runtime/`.
[miscinit.c#UnlinkLockFiles](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L1170-L1194),
[pqcomm.c#RemoveSocketFiles](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L846-L861).

The 17 leg, family by family, against the 2026-09-09 Linux run:

| Family | Linux x86_64, 2026-09-09 | Darwin arm64, 2026-09-10 |
|---|---|---|
| Engine suites | All 225 core tests; `pageinspect` 8, `pgstattuple` 1, `amcheck` 3 | identical: `core=0 All 225 tests passed`, 8, 1 and 3 |
| Text hashes | five `match` | five `match`; `est_pre` rebuilt to `BASEPRE` `8acd531b…`, `est_wide` `615d1674…`; both exact texts run as filed on `suite` and `acc` |
| Page geometry, 78 cells | 78 of 78 leaf-exact, 78 of 78 `relpages` | 78 of 78, none one-high, 78 of 78 within the internal cap, 78 of 78 `relpages` |
| Fresh sorted builds, 10 widths | `0 bytes` on 10 of 10; superseded `-0.7`, `-3.4`, `+11.0`, `-33.9` at 400, 800, 1000, 2000 | identical, at the same block counts 254 to 375 |
| Deduplication gate, 28 fixtures | 0 over-credits, 0 metapage disagreements, 2 under-credits, worst 28.8 % | identical; `DEBUG1` 16 "can safely use" and 13 "cannot use"; `text_pattern_ops` refuses the nondeterministic collation verbatim |
| Posting tails, 13 classes | mean absolute error 11.38 % -> 0.38 %; within one point 8 -> 11 of 13 | 11.42 % -> 0.35 %; 7 -> 12 of 13, the mean and the counts taken over the 13-row `tail_res` table the script prints; the first run of the day read 11.64 % -> 0.52 % and 7 -> 11 |
| In-index compression | 367 against 10,003 blocks, `avg_width` 904 both, `-2625.6 %` against `0.0 %` | identical |
| Statistics barrier | 0 and 200,000 with the barrier; 200,000 and 400,000 without | identical |
| Three deterministic defects | inheritance parent `-461.0 %` -> `0.0 %`; expression index `0.0`; `st0_i` `-22.3` with its caveat; superseded text aborts on the `bigint` range | `inh_i` `-417.8 %` -> `0.0 %` at `avg_width` 21 and 58; `expr_i` `0.0`; `st0_i` `-22.3` with `statistics target zero on an index column`; the superseded text aborts with `ERROR: bigint out of range` while the current one prints `1000000000000000000000000000000`; the non-owner run reports its rows with `statistics not visible to this role` |
| Calibration, 7 patterns | every value | digit for digit: densities and readings 0.0/0.0, 0.0/0.0, 25.7/25.7, 23.8/23.8, `-7.5`/`-6.7` with the floor at `-245.2`, 49.8/49.8, 33.3/33.3 |
| Numbered suite, 112 fixtures | floor 80 PASS / 20 critical false positives / 7 false negatives / 3 false positives / 2 unmeasured; point 76 / 26 / 5 / 3 / 2; 53 withheld, 0 unexplained, 0 contract failures | floor 81 / 19 / 7 / 3 / 2 and point 77 / 25 / 5 / 3 / 2, the one moved row being fixture 120 at `-50.0 %`; the first run of the day read it at 87.5 % and matched the Linux counts exactly; 53 withheld, 0 unexplained, 0 contract failures |
| Reported critical false positives | `f84` 94.2, `i103` 84.1, `x108` 62.5, `x109` 62.5, `p118` 99.3, `p120` 87.5 | the same five at the same values; `p120` at `-50.0` this run and 87.5 in the first |
| True detections | 12, all PASS | 12, all PASS: 68 at 87.5 against 87.4, 74 at 74.3/74.3, 75 at 89.5/89.1, 76 at 50.3/49.9, 77 at 94.2/94.2, 92 at 89.1/89.1, 95 at 89.9/89.1, 99 at 93.6/93.6 with the floor at 81.3, 107 at 90.0/90.0, 113b and 113c at 100.0/100.0, 114 at 98.9/98.9 |
| `f91` and `p36` | `-254.1` against 89.2; `-217.2` | `-248.6` against 89.2 (first run `-256.8`); `-217.2` |
| Attribution | 26 rows in each direction: 23 a moved number, 3 a caveat string | **31** in each direction: 21 a moved number, 10 differing in the caveat string alone; the first run 33, as 22 and 11. See [Attribution row counts differ between hosts](#attribution-row-counts-differ-between-hosts) |
| Probes | 73 emitted and executed; `false` for `p113b`, `p113c`, `p117`, `true` for `p115`, `p118`; 5,000 groups against a modelled 4,996 | 73 on `suite` and 31 on `acc`, every one executed; the same five answers, `forged_open` `true`; `groups_k` 5,000 against a modelled 4,997 |
| Cost, `suite` | 61.5-67.4 ms against 52.5-56.1 ms over 325 indexes and 85,017 blocks | 36.7-49.4 ms (five of six between 36.7 and 37.6) against 29.0-30.4 ms over 325 indexes and 85,023 blocks |
| `extstat` | 0 and 0 over 186 rows, 7 fed; scorecard 0.8 / 59.7 / 0.8 against widened 2.3 / 60.1 / `-33.7`; 95.3 against 91.6 ms | 0 and 0 over the same 186 rows (`geo` 0, `cal` 7, `gate` 28, `acc` 33, `suite` 114, `xstat` 4), 7 fed; the same scorecard, with inherited `n_distinct` 3490, 1404 and 28606; cost pairs 39.9-49.1 against 39.8-41.0 ms |

The 12 leg reproduced everything the targeted re-run recorded: the exact
filed text executes with `transform_edits=0`; `server_version_num` 120002,
block size 8192, alignment 8, no `pg_stat_force_next_flush()`, thirteen
`pg_stats_ext` columns with no `inherited`, support functions 1, 2 and 3,
`deduplicate_items` rejected; the previous text rebuilt to `8acd531b…` and
refused at `LINE 116` with `column se.inherited does not exist`; the widened
text agreeing with the filed one, 0 and 0 over 25 rows; the four
extended-statistics fixtures at 827 blocks each, one view row per statistics
object; and 19 fixtures at 13 PASS, 5 critical false positives and 1 false
positive, all `ineligible`, nothing credited, with the fresh builds at 254,
258 and 353 blocks and `0.0 %`, `w_key` at 84.1 against a measured 0.0, and
`nz_k` at 100.0 against 66.6. Two cells moved with the sample: `p1003` read
49.6 against 49.6, and `p1004` 90.2 against 89.1 (89.1 in the first run).

What moved between the two Darwin runs, and between Darwin and Linux, is
exactly the sample-dependent set: fixture 120's flip, the posting-tail
decimals, the inheritance parent's inherited `avg_width` (58 here, 66 and 77
in the two Linux passes), `b95` (88.4 then 89.9 against 89.1), `p76` (50.8
then 50.3 against 49.9), `f91`, the moved-row half of the attribution, and
the 12 leg's `p1004`. No verdict count other than fixture 120's changed
between any two runs, on either platform.
[analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894),
[analyze.c#compute_scalar_stats-width](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2420-L2426).

## Context Reviewed

- PostgreSQL 17 pin `786db8dcf168bd9df8f55047337525ac19118b1c`; the source checkout is read-only and was never written to.
- The revised statement's catalog inputs, CTE dependencies, formulas, output projection, exclusions and timeout settings, and the superseded statement extracted from the previous revision of this page for the side-by-side runs.
- The sorted-build caller/callee path and its page arithmetic (`nbtsort.c`, `nbtdedup.c`, `bufpage.c`, `nbtree.h`), tuple formation and varlena header handling (`indextuple.c`, `heaptuple.c`, `varatt.h`, `heaptoast.h`, `htup_details.h`), index attribute construction (`index.c`), expression-statistics selection and the two ANALYZE passes (`analyze.c`), statistics visibility and privilege resolution (`system_views.sql`, `acl.c`, `aclchk.c`), cumulative-statistics flush ordering (`postgres.c`, `pgstat.c`, `pgstat_relation.c`), split strategies (`nbtsplitloc.c`), size formatting and numeric range errors (`dbsize.c`, `numeric.c`), and the generated catalog/function boundary.
- Plan review on 2026-09-07, same pin, retained above.
- Implementation and measurement run on 2026-09-08, same pin: 17.11 built out of tree under `.wiki-runtime/tmp/btree17/` (`--without-readline --without-zlib --without-icu`), an isolated cluster on port 55437 with `autovacuum = off`, `fsync = off`, `shared_buffers = 256MB` and `--locale=C`; `make check` 225 of 225 tests passed, `contrib/pageinspect` and `contrib/pgstattuple` checks passed; six fixture databases covering page geometry, fresh sorted builds, the three deterministic defects, the equal-image matrix, posting-list tails, validation probes and the insertion-pattern calibration. `pageinspect` and `pgstattuple` were installed in the disposable cluster only. The sandbox was deleted after filing, so reproducing any number means rebuilding from the pin and re-running the published SQL.
- Mandatory test review on 2026-09-09, same pin, no server built or started: the revisions of this page before the 2026-09-07 cleanup (`33fe5a4`, `f8265ad`) and before the 2026-09-08 rewrite (`f2d73b4`) for the suite's requirement tables, harnesses and fixture recipes; the log entries of 2026-08-18, 2026-08-19, 2026-08-20, 2026-08-24 and 2026-09-08 for each run's provenance; the four fenced blocks on this page hashed against their recorded baselines, and the superseded block recovered from `f2d73b4` and hashed against `bffd166e…`; the installation and regression documentation for VPATH builds, ICU, block size, `make check` and contrib suites; the GUC contexts of every cluster setting the protocol names; and the equal-image, concurrent-build, parallel-build, partition-expansion, statistics-sample and `reltuples`-sentinel paths cited in the review.
- Full re-verification on 2026-09-09, same pin: every source citation on this page re-read against `raw/postgres-17/` (476 links over 68 files, 164 distinct ranges, all resolving and all supporting their labels), every `## Contents` entry and page-internal anchor re-checked, and every measured claim re-run on a second isolated server. 17.11 was built out of tree under `.wiki-runtime/tmp/btreerev/` with `--with-icu --enable-debug --with-readline --with-zlib`; `make check` passed 225 of 225 and the `pageinspect`, `pgstattuple` and `amcheck` checks passed 8, 1 and 3; the cluster ran `--locale=C`, `autovacuum = off`, `fsync = off`, `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`, `max_parallel_maintenance_workers = 0` at the default `BLCKSZ`, with eleven fixture databases covering page geometry, fresh sorted builds, the three deterministic defects read as two roles, the equal-image matrix, posting tails, the statistics barrier, the probes, in-index compression, the 27-fixture deduplication gate and the ICU collation cases. Both statement texts were extracted and hashed before use. `raw/postgres-17/` was never written to and stayed clean at the pin; the build and cluster were deleted after filing, leaving only the fixture scripts under `.wiki-runtime/tmp/btree-rev-harness/`.

- Measurement-script section review on 2026-09-09, same pin, read-only with no server built or started: `AGENTS.md`'s `MANDATORY Measurement Script`, `MANDATORY Production SQL`, `MANDATORY GUC Changes`, `MANDATORY Table of Contents` and `MANDATORY Citations` rules against this section as filed; both script bodies re-extracted from this page with their own `md_block` logic and parsed with `bash -n` (1,793 and 474 lines); the four `sql` blocks and the superseded text from revision `f2d73b4` re-hashed against all five baselines; the 37 source citations inside the section re-read against `raw/postgres-17/`, including the seven whose label token is not literally inside the range; `psql`'s exit-status handling for `-c` and `-f` actions (`startup.c`, `mainloop.c`, `psql-ref.sgml`); the GUC definitions of `listen_addresses`, `port` and `logging_collector`; and the containment guard tested against nine paths, including `/`, `$HOME`, the repository root, the `tmp` directory itself and a `tmpx` lookalike. Both pinned checkouts were clean at their pins throughout, and no sandbox, cluster or build was created.

- Suite-script run on 2026-09-09, same pin, both legs built and executed: 17.11 out of tree under `.wiki-runtime/tmp/btree-suite/build17` (`--enable-debug --with-icu --with-readline --with-zlib`), `make check` All 225 tests plus 8, 1 and 3 for `pageinspect`, `pgstattuple` and `amcheck`; a cluster at `--locale=C`, `autovacuum = off`, `fsync = off`, `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`, `max_parallel_maintenance_workers = 0`, default `BLCKSZ`, with five UTF8 databases for geometry, calibration, the deduplication gate, the acceptance fixtures and the 112-fixture numbered suite. 12.2 was built out of tree from this repository's pinned 12 checkout the same way, needing `-DTRUE=1 -DFALSE=0` for this host's ICU headers, and passed All 192 core tests plus 5, 1 and 2. Both checkouts stayed read-only and clean at their pins; the two scripts, their SQL and their output lived under the git-ignored `.wiki-runtime/tmp/btree-suite/`, deleted on 2026-09-10. The estimator, probe-generator, geometry and calibration blocks were re-extracted from this page after the edit that added the two script blocks, and all four still hash to their baselines, together with the superseded text from revision `f2d73b4`.

- Full re-run of both suite scripts on 2026-09-10, same pins, on a second platform: the postmaster's fast and immediate shutdown paths, the checkpointer's shutdown checkpoint, the recovery decision at the next start, `pg_ctl`'s `stop`, `status` and `start` command construction, lock-file and socket-file removal (`postmaster.c`, `checkpointer.c`, `xlog.c`, `xlogrecovery.c`, `pg_ctl.c`, `miscinit.c`, `pqcomm.c`, `pg_ctl-ref.sgml`), and the installation notes on ICU flags and macOS System Integrity Protection. Both `stop` stages were rewritten from `pg_ctl -m immediate` to `-m fast -w` with a confirmed teardown, the 12 leg's database-existence check was pointed at `postgres`, and the 17 leg's `extstat` stage was moved after `suite`; both scripts were re-extracted from this page with their own `md_block` logic, parsed with `bash -n` (2,062 and 681 lines), and diffed against the previous extraction so that only those hunks changed. Each leg was then built from scratch and run end to end twice under `.wiki-runtime/tmp/btree-suite/` on Darwin arm64 (macOS 26.6.2, Apple clang 21.0.0, ICU 78.3 via `ICU_CFLAGS`/`ICU_LIBS`, `JOBS=8`): `make check` All 225 plus 8, 1 and 3 on 17.11 and All 192 plus 5, 1 and 2 on 12.2, all five text hashes matching, and every verdict count of the Linux run reproduced. Both servers were stopped by the repaired `stop` stages, the 8.4 GB sandbox was deleted, and both checkouts stayed read-only and clean at their pins; 1.8 MB of script copies, run logs and output text remains under `.wiki-runtime/tmp/btree-suite-scripts/`.

- Portable extended-statistics filter, filed and measured on 2026-09-10, same pin: the `pg_stats_ext` and `pg_stats` view definitions and the `pg_statistic_ext_data` grant boundary (`system_views.sql`); the `stxdinherit` catalog column and its place in the data row's unique key (`pg_statistic_ext_data.h`); the two `ANALYZE` passes and the `inh` flag each one stores (`analyze.c`, `extended_stats.c`); and the whole `row_to_json` -> `->>` -> `boolean` chain, including what a missing key returns (`pg_proc.dat`, `pg_operator.dat`, `json.c`, `jsonfuncs.c`, `bool.c`). Both scripts were edited in place — one new `extstat` stage each, a new `BASEPRE` baseline, `BASE1` re-baselined to `646df923…`, an idempotent `cluster` stage, and a sixth database `xstat` — re-extracted from this page, parsed with `bash -n` (2,036 and 658 lines; this entry first recorded 2,004 and 648, corrected on 2026-09-10 by re-extracting both blocks), and run against the 17.11 and 12.2 clusters the 2026-09-09 run left in place under `.wiki-runtime/tmp/btree-suite/`: the 17 leg's `cluster texts extstat` and the 12 leg's `exact transform facts fixtures score extstat report`. Neither server was rebuilt, neither regression suite was re-run, both checkouts stayed read-only at their pins, and the sandbox was left in place for that pass, then stopped and deleted on 2026-09-10. The reconstruction of the previous statement text reproduced its filed SHA-256 exactly, which is what makes the one-line diff auditable.

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
| Clean shutdown and teardown confirmation in the suite scripts | [postmaster.c#process_pm_shutdown_request-fast](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2266-L2305), [postmaster.c#process_pm_shutdown_request-immediate](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2307-L2342), [xlog.c#ShutdownXLOG](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6580-L6621), [xlogrecovery.c#not-properly-shut-down](../../../../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L922-L949), [pg_ctl.c#do_stop](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L1015-L1065), [miscinit.c#UnlinkLockFiles](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L1170-L1194), [pqcomm.c#RemoveSocketFiles](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L846-L861). |
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
| What the scoring pass rebuilds, and under which lock | [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2829), [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3597). |
| Cluster settings the scripts write, and their apply scopes | [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1453), [guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1097-L1100), [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262-L2265), [guc_tables.c#log_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4873-L4877), [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4426-L4430), [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4437-L4441), [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2394-L2397), [guc_tables.c#logging_collector](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1641-L1644), [initdb.sgml#--locale](../../../../raw/postgres-17/doc/src/sgml/ref/initdb.sgml#L281-L291). |
| Why every `psql` helper carries `ON_ERROR_STOP`, and what a `-c` action returns without it | [mainloop.c:376](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L376), [mainloop.c#die_on_error](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L587-L594), [startup.c#single-query-action](../../../../raw/postgres-17/src/bin/psql/startup.c#L377-L386), [psql-ref.sgml#Exit-Status](../../../../raw/postgres-17/doc/src/sgml/ref/psql-ref.sgml#L627-L636). |
| Where the `inherited` flag of `pg_stats_ext` comes from, and which `ANALYZE` pass writes it | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309), [system_views.sql:290](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290), [pg_statistic_ext_data.h:35](../../../../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L35), [pg_statistic_ext_data.h:57](../../../../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L57), [analyze.c#analyze_rel-passes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L246-L259), [analyze.c#BuildRelationExtStatistics-call](../../../../raw/postgres-17/src/backend/commands/analyze.c#L604-L606), [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L111-L114), [extended_stats.c#statext_store](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L790-L791). |
| Why reading that flag through `row_to_json()` is the same predicate where the column exists, and a no-op where it does not | [pg_proc.dat#row_to_json](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8975-L8977), [json.c#composite_to_json](../../../../raw/postgres-17/src/backend/utils/adt/json.c#L546-L579), [json.c#datum_to_json-bool](../../../../raw/postgres-17/src/backend/utils/adt/json.c#L212-L221), [pg_operator.dat#json-arrow-text](../../../../raw/postgres-17/src/include/catalog/pg_operator.dat#L3160-L3162), [pg_proc.dat#json_object_field_text](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L9078-L9081), [jsonfuncs.c#json_object_field_text](../../../../raw/postgres-17/src/backend/utils/adt/jsonfuncs.c#L881-L895), [bool.c#parse_bool_with_len](../../../../raw/postgres-17/src/backend/utils/adt/bool.c#L36-L58), [bool.c#boolin](../../../../raw/postgres-17/src/backend/utils/adt/bool.c#L126-L150). |
| Why the base catalogs are not an alternative access path, and what the view's lateral costs to serialise | [system_views.sql#pg_statistic_ext_data-revoke](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L382-L383), [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L301-L307), [system_views.sql#pg_stats-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L194). |

## Open Questions

These are the limitations that survived the 2026-09-08 implementation run and
the 2026-09-09 re-verification, each with the number that measured it. The
2026-09-09 pass added
[Multicolumn key groups without extended statistics](#multicolumn-key-groups-without-extended-statistics)
and replaced the platform question with
[Fixture recipes that do not reproduce](#fixture-recipes-that-do-not-reproduce).
The measurement-script review of the same day added four questions about the
scripts rather than about the estimator; the full re-run of 2026-09-10 closed
two of them (the partial re-run and the 12 leg's unmeasured full runtime),
left
[Fixture statements are marked disposable, not tagged](#fixture-statements-are-marked-disposable-not-tagged)
and
[The 12 leg's settings have no citable apply scope here](#the-12-legs-settings-have-no-citable-apply-scope-here)
open, and added
[Attribution row counts differ between hosts](#attribution-row-counts-differ-between-hosts).

The 2026-09-10 pass closed the parse half of
[Cross-version execution of the revised statement](#cross-version-execution-of-the-revised-statement)
and the whole of the widening question that used to sit beside it, narrowed the
two script questions above to what is still untested, and left every estimator
limitation below untouched: none of them is about which `ANALYZE` pass the
`extstat` CTE reads.

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
locale were outside both runs. Every number on this page is a 17.11 build with
`--locale=C` at `max_data_alignment` 8 and `database_block_size` 8192, on
x86-64 Linux with gcc until 2026-09-10 and since then also on arm64 macOS with
Apple clang and ICU 78, where the whole suite reproduced; both passes recorded
those two values from `pg_control_init()` rather than assuming them. A second
architecture is not a second alignment or block size, so the geometry
constants remain measured at one setting of each. The nondeterministic-collation
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

The 2026-09-09 run measured what the probe does and does not reach. It answered
correctly on all five zero-count partial indexes it was emitted for — `false`
for the two drained queues and the VACUUM-only drain, `true` for the refilled
subset and for the index built on an analysed empty table — but it was not
emitted for fixture 120 at all, because the generator carries the same
`pg_relation_size(c.oid) > 1024 * 1024` filter as the report and that index is
eight blocks. A subset small enough to be missed by the ANALYZE sample is also
small enough to fall under the probe generator's own size filter, so the reading
most in need of validation is the one least likely to get a probe.
[Validation probes](#validation-probes),
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09).

### Fixture verdicts that depend on the ANALYZE sample

Fixture 120 does not give the same answer twice. Three runs of the same script
against freshly created databases read 87.5 %, `-50.0 %` and 87.5 % on it,
because the fixture sets `default_statistics_target` to 1, which samples 300
rows, and whether those 300 rows include any of the 2,000 rows in a 1,000,000-row
table decides whether the index's modelled row count is 0 or a few thousand.
Both outcomes are legitimate readings of the same physical index, and the
verdict flips between `CRITICAL FALSE POSITIVE` and `PASS` with them. The suite
therefore has at least one cell that cannot be compared run to run, and the
honest fix is either to seed the sample, to score the fixture over repetitions,
or to state its verdict as a distribution. Three more runs on Darwin arm64 on
2026-09-10 read 87.5 %, `-50.0 %` and `-50.0 %`. No other fixture was
observed to flip its verdict on either platform, but several moved within
their verdict with the sample: `b95` read 88.4 and then 89.9 against 89.1,
`p76` 50.8 and then 50.3 against 49.9, `f91` `-256.8` and then `-248.6`, the
12 leg's `p1004` 89.1 and then 90.2 against 89.1, and the moved-number half of
the attribution changed membership between runs. Nothing in the harness
proves that no other verdict can flip.
[analyze.c#std_typanalyze-minrows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1894),
[guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2074).

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
corrected, not reported. The whole suite has now been put to the current text —
tests 1-17 on 2026-09-09, and tests 18-91 with fixtures 92-121 by the script
under [The PostgreSQL 17 suite script](#the-postgresql-17-suite-script) — so
what is open is no longer coverage but three failures the run leaves standing
and one it cannot judge:

- `i103` at 84.1 % and `x109` at 62.5 %, both reported, both alertable, both
  against a measured 0.0. They were predicted, they reproduced, and neither has
  a fix in the statement.
- `f84`, a partial index whose `reltuples` was forged stale, at 94.2 % with an
  empty `caveats` string. The statement cannot see a catalog count that lies.
- Whether that is a pass depends on the scoring column, which is still the
  asker's decision under
  [Scoring column for the partial-index contract](#scoring-column-for-the-partial-index-contract).

The contract's remedy — correct the statement, do not merely report — is
therefore still owed on the first two, and this page has no candidate change
that removes them without hiding correct readings.
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09),
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
pass criteria above are written for whichever column is chosen. The 2026-09-09
run measured the cost of the choice on the numbered suite: on the floor the
112 fixtures come out 80 PASS / 20 critical false positives / 7 false
negatives, on the point estimate 76 / 26 / 5, and the sharpest single case is
control 99, a duplicate-heavy non-partial index whose point estimate of 93.6 %
is exact against a measured 93.6 % while its floor reads 81.3 %.
[Calibration by insertion pattern](#calibration-by-insertion-pattern),
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09),
[Reading the output](#reading-the-output).

### Cross-version execution of the revised statement

**Fixed on 2026-09-10, and the fix is measured on both servers.** The text
filed until then selected `pg_stats_ext` rows with `inherited = false`, naming a
column this version's view defines from `stxdinherit` and the pinned 12 server
does not have; it was refused with
`ERROR: column se.inherited does not exist` before any fixture was built. The
`extstat` CTE now reads that flag with
`coalesce((row_to_json(se) ->> 'inherited')::boolean, false) = false`, and the
exact filed text is accepted unmodified on both majors: `transform_edits=0` on
the 12 leg, and `EXCEPT` in both directions over 186 rows in six databases on
the 17 leg.
[The portable extended-statistics filter](#the-portable-extended-statistics-filter),
[What the 2026-09-10 targeted re-run measured](#what-the-2026-09-10-targeted-re-run-measured),
[system_views.sql#pg_stats_ext-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290),
[system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309).

The old transformer's widening is also settled rather than dropped: four
fixtures now carry extended-statistics objects, three of them on inheritance
parents, and the widened text reads 2.3, 60.1 and -33.7 where the filed text
reads 0.8, 59.7 and 0.8 against measured reclaims of 0.0, 59.7 and 0.0. On the
12 server the widened text and the filed text agree exactly, because that
server records one pass per statistics object.

Three things stay open, none of them the parse. **The 12 leg still scores 19
fixtures against the 17 leg's 112**, so the partial-index contract is enforced
on one major only, and the four new extended-statistics fixtures are outside
that contract. **The v12 companion page still documents its own Method A**, not
this text, so a reader following the cross-version link finds a different
statement. And **only two majors are tested**: nothing here says what the
`row_to_json` read does on 13, 14, 15 or 16, where the column arrived at some
release this page may not cite; the construct is designed to be indifferent to
that, but indifference is not a measurement.
[The v12 publication protocol](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md#the-v12-publication-protocol).

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

### Attribution row counts differ between hosts

The `EXCEPT` attribution between the current and the superseded text returned
26 rows in each direction on the 2026-09-09 Linux run, read as 23 rows with a
moved number and 3 that differ in the caveat string alone. The two Darwin arm64
runs of 2026-09-10 returned 33 and then 31 rows in each direction, read the
same way as 22 and 11, then 21 and 10. The moved-number half is expected to
vary: five of its rows differ between the texts by 0.1 to 0.4 points
(`f83` 48.7 against 48.6, `i101` `-1.0` against `-1.1`, `i102` `-1.1`
against `-1.2`, `p49b` 1.1 against 1.0, `p54` 0.0 against 0.4), which is the
size of a rounding step, so whether such a row lands in the set depends on the
`ANALYZE` sample. The caveat-only half should not vary, and here it does: on
Darwin the ten rows are the eight zero-row fixtures (`nz_k`, `p113b`,
`p113c`, `p115`, `p116`, `p117`, `p118`, and `x109` with
`statistics target zero on an index column`) plus the harness's own
`plan_pkey` and `res_pkey`, every one differing because the current text adds
a caveat the superseded text does not have. The Linux output that read 3 was
deleted with its sandbox on 2026-09-10, so the discrepancy cannot be
reconciled from this page; the next run on that host settles it, and until
then the 26-row figure under
[What the two scripts measured on 2026-09-09](#what-the-two-scripts-measured-on-2026-09-09)
is a recorded output whose split this page cannot re-derive.
[What the 2026-09-10 full re-run measured on Darwin arm64](#what-the-2026-09-10-full-re-run-measured-on-darwin-arm64).

### Fixture statements are marked disposable, not tagged

`MANDATORY Production SQL` asks for an inline `/* wiki_... */` tag after the
leading verb, and `MANDATORY Measurement Script` extends it to "the statements
the script sends"; the same rule separately asks that fixture statements be
marked disposable. The review took those as two requirements for two kinds of
statement: the statements whose output this page publishes now carry 47 tags,
31 in the 17 leg and 16 in the 12 leg, and the remaining statements — about 740
fixture `CREATE`, `INSERT`, `ANALYZE`, `VACUUM`, `UPDATE` and `DROP` lines,
almost all of them in the 17 leg's numbered-suite block — carry disposability
banners instead. That reading is not stated in the rule, and the alternative —
tagging every fixture statement — was not adopted, because it is a
seven-hundred-line diff to scripts this pass could not re-run. If the strict
reading is intended, the tagging pass and a full re-run belong together.

### The 12 leg's settings have no citable apply scope here

The 12 leg writes `listen_addresses`, `unix_socket_directories`, `port`,
`autovacuum`, `fsync`, `shared_buffers`, `maintenance_work_mem` and
`max_parallel_maintenance_workers` into its cluster's `postgresql.conf`, and
sets `client_min_messages`, `statement_timeout` and `lock_timeout` per session.
`MANDATORY GUC Changes` wants each one's context and apply scope from the
same-version definition, but `MANDATORY Citations` forbids a v17 page from
citing the pinned 12 checkout at all, and this page has no validated 12.2
`pg_settings` capture. The run itself is unaffected, because every file-level
setting is in place before the first start, but a reader who changes one of them
on a 12.2 server has to look up its scope elsewhere. The clean fix is a
`pg_settings` capture on the 12 leg, added to `stage_facts` and re-run, or the
same table on a v12 page.

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
- [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2829)
- [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3597)
- [guc_tables.c#log_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4873-L4877)
- [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4426-L4430)
- [pg_class.h#relkind](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173)
- [vacuum.c#expand_vacuum_rel-partitions](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L963-L982)
- [pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L41-L50)
- [pg_proc.dat#pg_control_init](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11997)
- [pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L204)
- [startup.c#single-query-action](../../../../raw/postgres-17/src/bin/psql/startup.c#L377-L386)
- [mainloop.c:376](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L376)
- [mainloop.c#die_on_error](../../../../raw/postgres-17/src/bin/psql/mainloop.c#L587-L594)
- [psql-ref.sgml#Exit-Status](../../../../raw/postgres-17/doc/src/sgml/ref/psql-ref.sgml#L627-L636)
- [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4437-L4441)
- [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2394-L2397)
- [guc_tables.c#logging_collector](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1641-L1644)
- [system_views.sql#pg_stats-inherited](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L194)
- [system_views.sql#pg_statistic_ext_data-revoke](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L382-L383)
- [pg_statistic_ext_data.h:35](../../../../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L35)
- [pg_statistic_ext_data.h:57](../../../../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L57)
- [analyze.c#BuildRelationExtStatistics-call](../../../../raw/postgres-17/src/backend/commands/analyze.c#L604-L606)
- [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L111-L114)
- [extended_stats.c#statext_store](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L790-L791)
- [pg_proc.dat#row_to_json](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L8975-L8977)
- [pg_proc.dat#json_object_field_text](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L9078-L9081)
- [pg_operator.dat#json-arrow-text](../../../../raw/postgres-17/src/include/catalog/pg_operator.dat#L3160-L3162)
- [json.c#composite_to_json](../../../../raw/postgres-17/src/backend/utils/adt/json.c#L546-L579)
- [json.c#datum_to_json-bool](../../../../raw/postgres-17/src/backend/utils/adt/json.c#L212-L221)
- [jsonfuncs.c#json_object_field_text](../../../../raw/postgres-17/src/backend/utils/adt/jsonfuncs.c#L881-L895)
- [bool.c#parse_bool_with_len](../../../../raw/postgres-17/src/backend/utils/adt/bool.c#L36-L58)
- [bool.c#boolin](../../../../raw/postgres-17/src/backend/utils/adt/bool.c#L126-L150)
- [postmaster.c#process_pm_shutdown_request-fast](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2266-L2305)
- [postmaster.c#process_pm_shutdown_request-immediate](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L2307-L2342)
- [checkpointer.c#ShutdownRequestPending](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L584-L600)
- [xlog.c#ShutdownXLOG](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6580-L6621)
- [xlogrecovery.c#not-properly-shut-down](../../../../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L922-L949)
- [pg_ctl.c#shutdown-modes-usage](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L2006-L2011)
- [pg_ctl-ref.sgml#shutdown-modes](../../../../raw/postgres-17/doc/src/sgml/ref/pg_ctl-ref.sgml#L186-L198)
- [pg_ctl.c#do_status](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L1336-L1388)
- [pg_ctl-ref.sgml#status-exit-status](../../../../raw/postgres-17/doc/src/sgml/ref/pg_ctl-ref.sgml#L221-L227)
- [pg_ctl-ref.sgml#-m](../../../../raw/postgres-17/doc/src/sgml/ref/pg_ctl-ref.sgml#L310-L320)
- [pg_ctl.c#do_stop](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L1015-L1065)
- [miscinit.c#UnlinkLockFiles](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L1170-L1194)
- [miscinit.c#DIRECTORY_LOCK_FILE](../../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L60)
- [pg_ctl.c#start_postmaster-command](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L489-L494)
- [pg_ctl.c#pgdata_opt](../../../../raw/postgres-17/src/bin/pg_ctl/pg_ctl.c#L2278)
- [pqcomm.c#RemoveSocketFiles](../../../../raw/postgres-17/src/backend/libpq/pqcomm.c#L846-L861)
- [installation.sgml#SIP](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L3611-L3618)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [versions](../../../versions.md)
