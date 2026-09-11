---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# A COMMENT-Stored Baseline B-Tree Index-Maintenance Heuristic for PostgreSQL 12 Through 17 (unverified)

## Contents

- [Question](#question)
  - [Prompt corrections](#prompt-corrections)
  - [The follow-up prompt](#the-follow-up-prompt)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What is stored, and where](#what-is-stored-and-where)
  - [Step 1: the plan](#step-1-the-plan)
  - [Step 2: carrying it out](#step-2-carrying-it-out)
  - [How to read the plan](#how-to-read-the-plan)
  - [The decision ladder, in order](#the-decision-ladder-in-order)
  - [Why every candidate filter is there](#why-every-candidate-filter-is-there)
  - [The three gates, measured at every boundary](#the-three-gates-measured-at-every-boundary)
  - [The index's own entry count, and who writes it](#the-indexs-own-entry-count-and-who-writes-it)
  - [The simulated auto-analyze the tests now run](#the-simulated-auto-analyze-the-tests-now-run)
  - [Wasted space, and where the threshold lands](#wasted-space-and-where-the-threshold-lands)
  - [What the gate still cannot see](#what-the-gate-still-cannot-see)
  - [The comment survives both REINDEX forms](#the-comment-survives-both-reindex-forms)
  - [Privileges: measure, rebuild, write the baseline](#privileges-measure-rebuild-write-the-baseline)
  - [Locks, transactions and the two commands a DO block cannot reach](#locks-transactions-and-the-two-commands-a-do-block-cannot-reach)
  - [What it costs to run](#what-it-costs-to-run)
  - [What this revision changed, and why](#what-this-revision-changed-and-why)
  - [Two defects the first revision's suite found](#two-defects-the-first-revisions-suite-found)
  - [What is version-local between 12 and 17](#what-is-version-local-between-12-and-17)
  - [The ported mandatory suite](#the-ported-mandatory-suite)
  - [Results on 17.11](#results-on-1711)
  - [Results on 12.2](#results-on-122)
  - [The four false negatives that remain](#the-four-false-negatives-that-remain)
  - [Filed predictions against measured verdicts](#filed-predictions-against-measured-verdicts)
  - [Comment parsing and preservation](#comment-parsing-and-preservation)
  - [Idempotence, dry runs and dumps](#idempotence-dry-runs-and-dumps)
  - [Mandatory test review](#mandatory-test-review)
  - [What still needs to be tested](#what-still-needs-to-be-tested)
- [Measurement Script](#measurement-script)
  - [How to use the two leg scripts](#how-to-use-the-two-leg-scripts)
  - [The stages, both legs](#the-stages-both-legs)
  - [What the scripts read from the environment](#what-the-scripts-read-from-the-environment)
  - [Prerequisites](#prerequisites)
  - [Last run](#last-run)
  - [The PostgreSQL 17 leg script](#the-postgresql-17-leg-script)
  - [The PostgreSQL 12 leg script](#the-postgresql-12-leg-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

### Prompt corrections

Filed after prompt-hygiene correction, at the asker's request. The original prompt
wrote `agents.md` for `AGENTS.md`, lowercase `postgresql` for `PostgreSQL`, `btree`
for `B-tree`, `question Create` without the colon, `add all mandatory test from
question` for `Add all mandatory tests from the question`, and put a space before
the closing full stop. The asker chose "correct and restate", and chose to port
the full numbered suite, to build and measure both version legs, and to keep the
page in v17 while linking the v12 pages rather than citing another version's
source. The corrected text is below.

The 2026-09-11 follow-up prompt was corrected the same way, again at the asker's
request: it wrote `pg_statindex` for `pgstatindex()`, `store also index tuples`
for `also store the index's own tuple count`, `change more 10%` for `change more
than **10%**`, lowercase `analyze` for the `ANALYZE` command, began two sentences
in lower case, and put a space before a comma. Three scoping answers were taken
before drafting: **re-run both legs** rather than one, **bump the payload format
version to 2** so an old baseline is re-initialized rather than half-read, and
**apply the `ANALYZE` rule everywhere**, writing the deliberate catalog forgeries
after it so they survive, and recording which fixtures lose their original point.

Follow `AGENTS.md`, in PostgreSQL 17. Question:

Create a PostgreSQL B-tree index-maintenance heuristic compatible with versions
**12 through 17**.

Store the index size and its table's estimated tuple count in the index comment.
Each time the heuristic runs, compare the current values with the stored values
and run `pgstatindex()` if either:

- The index size has increased by **20% or more**; or
- The table tuple count has increased or decreased by **20% or more**.

Use `pgstatindex()` to calculate the index's wasted-space percentage. If wasted
space is **greater than 40%**, trigger a reindex. If wasted space is **40% or
less**, do not reindex; instead, update the index comment with the current index
size and table tuple count.

After a successful reindex, update or initialize the index comment with the new
index size and current table tuple count.

The heuristic should safely handle missing or invalid comment metadata, preserve
any existing user-defined comment, and work with PostgreSQL versions 12 through 17.

Add all mandatory tests from the question [Testing the PostgreSQL 12 Core-SQL
B-Tree Bloat Method on PostgreSQL 17
(unverified)](btree-index-bloat-core-sql-only.md).

### The follow-up prompt

Filed 2026-09-11, after the corrections above:

Follow `AGENTS.md`, in PostgreSQL 17, for question: A COMMENT-Stored Baseline
B-Tree Index-Maintenance Heuristic for PostgreSQL 12 Through 17 (unverified).
Update the heuristic to also store the index's own tuple count, and have the
**20%** up-and-down trigger for `pgstatindex()` apply to it as well. Update the
mandatory tests so that a test changing more than **10%** of the heap tuples
calls `ANALYZE`, to simulate an auto-analyze.

## Answer

### Verdict

Two texts do it, and both run unchanged on 12.2 and 17.11: a read-only statement
that decides, and a `DO` block that carries the decision out. The baseline lives
in an 86-byte `@btmaint:` payload appended to the index's own comment, which
survives `REINDEX INDEX` because the comment is keyed by the index's OID, and
survives `REINDEX INDEX CONCURRENTLY` because `index_concurrently_swap` moves the
`pg_description` row to the new index
([index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784),
[pg_description.h#FormData](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L44-L57)).

**The follow-up asked for two changes, and together they close the blind spot the
first revision measured.** The payload now carries a third value, the index's own
`pg_class.reltuples`, and the same 20 % up-or-down test runs on it; and the ported
mandatory suite now simulates the auto-analyze that churn would have triggered on
a server with autovacuum on. Both legs were rebuilt from their pins and the whole
suite re-run:

| Result | 17.11 | 12.2 | Two-gate revision, 17.11 |
|---|---|---|---|
| Numbered fixtures scored | 140 | 122 | 140 |
| Gate decision equal to the same arithmetic done independently | 140 of 140 | 122 of 122 | 140 of 140 |
| `PASS` | **136** | **118** | 119 |
| `FALSE NEGATIVE` | **4** | **4** | 21 |
| `CRITICAL FALSE POSITIVE`, `FALSE POSITIVE` | 0 | 0 | 0 |
| Indexes rebuilt, and their mean measured reclaim | 103, 86.3 % | 85, 87.0 % | 84, 87.1 % |
| Fresh indexes wrongly rebuilt (the eight false-positive constructions) | 0 of 8 | 0 of 8 | 0 of 8 |
| Baselines readable after the run | 140 of 140 | 122 of 122 | 140 of 140 |
| Stored index count equal to the catalog's at baseline time | 140 of 140 | 122 of 122 | - |

**The third gate is what opened the previously lost fixtures, and it opened
nothing else.** Of 116 measured fixtures on 17.11, the index-count test fired on
115, the table-count test on 85 and the size test on 9; **26 were measured that
the two-gate form skipped**, and 21 of those 26 were genuinely bloated, with a
mean measured reclaim of 76.3 %. Not one measurement was lost: every fixture the
two-gate form measured, the three-gate form measures too. The five healthy indexes
it newly measured cost one `pgstatindex()` call each and were correctly left
alone or rebuilt above the 35 % bar; no fixture became a false positive. The 12.2
leg agrees fixture for fixture: 25 newly measured, 21 of them bloated.

**All three gates are needed, and each catches something the others cannot.** One
fixture is measured by the size test alone: `p76`, bloated by updating an indexed
key, whose file doubled (`size_ratio` 1.9964) while its entry count moved from
100,000 to 100,284. No fixture is caught by the table count alone.

**Four false negatives remain, and they are no longer gate losses.** On both
servers the index-count gate *did* open `p113a`, `p113c`, `p65` and `p67` — and
`pgstatindex` then reported 0.0 % to 0.1 % wasted on a file a rebuild emptied by
89.1 % to 100.0 %. All four deleted or moved rows out of the index without a
following `VACUUM`, so the dead entries are still physically present and the
density the decision reads is a density of dead tuples. That is a limit of the
measurement the brief specifies, not of the gate, and no catalog-only input can
fix it.

Where the gate fires, the decision is as good as before: over 116 measured
fixtures on 17.11, `wasted_pct` under-estimates the reclaim a rebuild gave back by
a mean of 10.7 points (min -5.9, max 100.0 — the max being those four unvacuumed
files), 108 of 116 within 15 points, and 8 over-estimates. The nine-point curve is
unchanged, so the 40 % threshold still trips at 45.0 % of entries deleted.

### What is stored, and where

`COMMENT ON INDEX` stores one string per index in `pg_description`, keyed by
`(objoid, classoid, objsubid)` with `objsubid = 0`, and a new comment replaces the
old one whole - there is no append
([pg_description.h#FormData](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L44-L57),
[comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L133-L171),
[comment.sgml#replaces](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L89-L93)).
So the heuristic has to rewrite the whole comment on every write, and preserving
the human text is its job, not the server's.

The payload is one line, appended after any human text:

```text
hand-written note kept by the DBA
@btmaint:{"v":2,"sz":4513792,"tup":200000,"itup":200000,"at":"2026-09-11T15:11:43-04"}
```

| Field | Meaning | Why it is there |
|---|---|---|
| `v` | payload format version, now `2` | a change of shape must not be read as a baseline; a mismatch is treated as unreadable |
| `sz` | `pg_relation_size(index)` at baseline time, bytes | the question's first input; main fork only ([system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)) |
| `tup` | the table's `pg_class.reltuples`, rounded | the question's second input |
| `itup` | the index's own `pg_class.reltuples`, rounded | the follow-up's input: a partial index's population is not its table's ([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)) |
| `at` | when the baseline was written | staleness is otherwise invisible; it takes no part in any decision |

Measured length: **86 bytes** for an index with no human comment, 14 more than the
two-value payload. The payload is plain text, not `jsonb`, and it is read with
`substring(... from '...')` rather than a cast, for a reason that is
version-local: casting a malformed payload to `jsonb` raises, one raised error
aborts the whole statement, and the guard that would prevent it,
`pg_input_is_valid()`, exists on 17 and not on 12 - measured, 1 matching `pg_proc`
row against 0. A regex `substring` returns NULL instead of raising, and each
field's pattern ends in `[,}]`, so a number longer than the pattern allows fails
to match rather than silently truncating.

**Version 1 baselines are re-initialized, not half-read.** The `baseline` test
treats a missing `itup` exactly as it treats a missing `sz`: unreadable. Both
shapes were measured — a well-formed `{"v":1,"sz":1,"tup":1}` and a
`{"v":2,"sz":1,"tup":1}` with the key simply absent — and both read `invalid`,
take the `initialize` branch, and end at 86 bytes. The cost of the upgrade is
therefore one comment write per index on the first run, with no rebuild and no
measurement, and the third gate is live from the second run on. The alternative,
accepting a v1 payload and running two gates for that index, was rejected: it
would leave a database in which two indexes with the same comment shape are gated
differently, and nothing in the comment would say so.

### Step 1: the plan

Read-only. It decides, prints the exact commands, and writes nothing.

```sql
-- B-tree index-maintenance heuristic, step 1: decide, write nothing.
-- One text, unchanged on PostgreSQL 12 through 17.
--
--   params   thresholds, the comment marker and the two page-layout constants
--   cand     every B-tree index this session may both measure and comment on
--   parsed   the @btmaint: payload pulled out of the index's own comment
--   gate     the three 20 % tests, and the states that bypass them
--   meas     one pgstatindex() call per gated index, and none for the rest
--   wasted   free bytes beyond a rebuild at this index's own fillfactor
--   plan     the action, and the exact commands that carry it out
--
-- Gate: measure when the index has grown by 20 % or more, when the table's
-- estimated tuple count has moved by 20 % or more in either direction, or when
-- the index's own estimated entry count has moved by 20 % or more in either
-- direction.
-- Decide: wasted_pct > 40 -> reindex; 40 or less -> just refresh the baseline.
-- Nothing here writes.  Step 2 applies the plan.

SET /* wiki_btmaint_statement_timeout */ statement_timeout = '15min';
SET /* wiki_btmaint_lock_timeout */ lock_timeout = '5s';

WITH params AS (
    SELECT current_setting('block_size')::numeric AS bs,
           24::numeric        AS page_header,     -- SizeOfPageHeaderData
           16::numeric        AS btree_special,   -- MAXALIGN(BTPageOpaqueData)
           1.20::numeric      AS grow_ratio,      -- index-size gate
           0.20::numeric      AS tuple_ratio,     -- tuple-count gate, table and index
           40::numeric        AS wasted_max,      -- reindex above this percent
           0::numeric         AS min_index_bytes, -- ignore anything smaller
           2::numeric         AS fmt              -- payload format version
),
cand AS MATERIALIZED (
    SELECT c.oid                  AS idx_oid,
           n.nspname              AS schema_name,
           c.relname              AS index_name,
           t.relname              AS table_name,
           pg_relation_size(c.oid) AS cur_bytes,
           t.reltuples::numeric   AS cur_tuples,
           c.reltuples::numeric   AS cur_idx_tuples,
           (SELECT o.option_value::int
              FROM pg_options_to_table(c.reloptions) o
             WHERE o.option_name = 'fillfactor')      AS fillfactor_opt,
           d.description          AS cmt,
           pg_has_role(c.relowner, 'USAGE')           AS owns_index,
           pg_has_role(t.relowner, 'USAGE')           AS owns_table
      FROM pg_class c
      JOIN pg_namespace n   ON n.oid = c.relnamespace
      JOIN pg_index x       ON x.indexrelid = c.oid
      JOIN pg_class t       ON t.oid = x.indrelid
      JOIN pg_am a          ON a.oid = c.relam
      LEFT JOIN pg_description d ON d.objoid = c.oid
                               AND d.classoid = 'pg_class'::regclass
                               AND d.objsubid = 0
     CROSS JOIN params p
     WHERE a.amname = 'btree'                    -- pgstatindex takes no other AM
       AND c.relkind = 'i'                       -- 'I' has no storage
       AND x.indisvalid AND x.indisready AND x.indislive
       AND NOT pg_is_other_temp_schema(c.relnamespace)
       AND (c.relpersistence <> 'u' OR NOT pg_is_in_recovery())
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
       AND pg_relation_size(c.oid) >= p.min_index_bytes
),
parsed AS MATERIALIZED (
    SELECT c.*,
           pay.payload,
           substring(pay.payload from '"v":([0-9]{1,6})[,}]')::numeric      AS pv,
           substring(pay.payload from '"sz":([0-9]{1,25})[,}]')::numeric    AS base_bytes,
           substring(pay.payload from
                     '"tup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric AS base_tuples,
           substring(pay.payload from
                     '"itup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric AS base_idx_tuples,
           substring(pay.payload from '"at":"([^"]{1,40})"')            AS base_at,
           -- Two passes, in this order: a well-formed payload goes first, so
           -- that human text on the same line survives, and only a leftover
           -- marker is then removed to the end of its line.
           regexp_replace(
             regexp_replace(coalesce(c.cmt, ''),
                            '[[:space:]]*@btmaint:\{[^}]*\}', '', 'g'),
                            '[[:space:]]*@btmaint:[^\n]*', '', 'g')      AS user_cmt
      FROM cand c
      CROSS JOIN LATERAL (
           SELECT substring(c.cmt from '@btmaint:(\{[^}]*\})') AS payload) pay
),
gate AS MATERIALIZED (
    SELECT s.*,
           p.bs, p.grow_ratio, p.tuple_ratio, p.wasted_max,
           p.bs - p.page_header - p.btree_special AS leaf_cap,
           COALESCE(s.fillfactor_opt, 90)         AS fillfactor,
           CASE WHEN s.cmt IS NULL OR s.cmt !~ '@btmaint:'   THEN 'absent'
                WHEN s.pv IS DISTINCT FROM p.fmt
                  OR s.base_bytes IS NULL
                  OR s.base_tuples IS NULL
                  OR s.base_idx_tuples IS NULL               THEN 'invalid'
                ELSE 'ok' END                                AS baseline,
           -- reltuples is -1 on a relation no ANALYZE, VACUUM or index build
           -- has counted since PostgreSQL 14; 12 and 13 leave 0 there instead
           (s.cur_tuples < 0)                                AS tuples_unknown,
           (s.cur_idx_tuples < 0)                            AS idx_tuples_unknown
      FROM parsed s CROSS JOIN params p
),
decided AS MATERIALIZED (
    SELECT g.*,
           CASE WHEN g.base_bytes > 0
                THEN round(g.cur_bytes / g.base_bytes, 4) END AS size_ratio,
           CASE WHEN g.base_tuples > 0 AND NOT g.tuples_unknown
                THEN round(g.cur_tuples / g.base_tuples, 4) END AS tuple_ratio_now,
           CASE WHEN g.base_idx_tuples > 0 AND NOT g.idx_tuples_unknown
                THEN round(g.cur_idx_tuples / g.base_idx_tuples, 4) END
                                                              AS idx_tuple_ratio_now,
           (g.baseline = 'ok' AND g.base_bytes > 0
            AND g.cur_bytes >= g.base_bytes * g.grow_ratio)     AS size_gate,
           (g.baseline = 'ok' AND NOT g.tuples_unknown
            AND g.base_tuples >= 0
            AND (CASE WHEN g.base_tuples = 0
                      THEN g.cur_tuples > 0
                      ELSE abs(g.cur_tuples - g.base_tuples)
                           >= g.base_tuples * g.tuple_ratio END))  AS tuple_gate,
           -- The index's own entry count, gated exactly like the table's.  A
           -- partial index's population is not its table's population, which
           -- is the whole reason this test is separate.
           (g.baseline = 'ok' AND NOT g.idx_tuples_unknown
            AND g.base_idx_tuples >= 0
            AND (CASE WHEN g.base_idx_tuples = 0
                      THEN g.cur_idx_tuples > 0
                      ELSE abs(g.cur_idx_tuples - g.base_idx_tuples)
                           >= g.base_idx_tuples * g.tuple_ratio END)) AS idx_tuple_gate,
           (g.baseline = 'ok' AND g.base_bytes > 0
            AND g.cur_bytes < g.base_bytes)                     AS shrank
      FROM gate g
),
staged AS MATERIALIZED (
    SELECT d.*,
           CASE WHEN NOT d.owns_index                    THEN 'blocked'
                WHEN d.baseline <> 'ok'                  THEN 'initialize'
                WHEN d.shrank                            THEN 'refresh'
                WHEN d.size_gate OR d.tuple_gate
                  OR d.idx_tuple_gate                    THEN 'measure'
                ELSE 'skip' END                          AS stage
      FROM decided d
),
gated AS MATERIALIZED (
    SELECT s.* FROM staged s WHERE s.stage = 'measure'
),
meas AS MATERIALIZED (
    -- One pgstatindex() call per gated index, and none at all for the rest.
    -- The function's input relation is the gated rows themselves, so an index
    -- that did not pass the gate is never opened.  A WHERE or an ON clause
    -- would not do: a set-returning function in FROM is executed first and
    -- filtered afterwards, which read every candidate index end to end.
    SELECT g.*, m.index_size, m.leaf_pages, m.empty_pages, m.deleted_pages,
           m.avg_leaf_density, m.leaf_fragmentation, m.version AS meta_version
      FROM gated g, LATERAL pgstatindex(g.idx_oid::regclass) m
),
merged AS (
    SELECT * FROM meas
    UNION ALL
    SELECT s.*, NULL::bigint, NULL::bigint, NULL::bigint, NULL::bigint,
           NULL::float8, NULL::float8, NULL::int
      FROM staged s WHERE s.stage <> 'measure'
),
wasted AS (
    SELECT m.*,
           (m.leaf_cap - (m.bs * (100 - m.fillfactor)) / 100) / m.leaf_cap AS target_density,
           CASE WHEN m.leaf_pages > 0 AND m.avg_leaf_density <> 'NaN'::float8
                THEN (m.avg_leaf_density / 100)::numeric
                ELSE 0::numeric END                                       AS density,
           COALESCE(m.empty_pages, 0) + COALESCE(m.deleted_pages, 0)       AS dead_pages
      FROM merged m
),
scored AS (
    SELECT w.*,
           CASE WHEN w.stage <> 'measure' THEN NULL
                ELSE GREATEST(round(w.leaf_pages * w.leaf_cap * w.target_density)
                              - round(w.leaf_pages * w.leaf_cap * w.density), 0)
                     + w.dead_pages * w.bs END AS wasted_bytes
      FROM wasted w
),
plan AS (
    SELECT s.*,
           CASE WHEN s.stage <> 'measure' OR s.index_size = 0 THEN NULL
                ELSE round(100 * s.wasted_bytes / s.index_size, 1) END AS wasted_pct,
           CASE WHEN s.stage <> 'measure' THEN s.stage
                WHEN s.index_size = 0     THEN 'skip'
                WHEN round(100 * s.wasted_bytes / s.index_size, 1) > s.wasted_max
                     THEN 'reindex'
                ELSE 'update' END                                      AS action
      FROM scored s
)
SELECT /* wiki_btmaint_plan_12_17 */
       p.schema_name,
       p.index_name,
       p.table_name,
       p.action,
       p.baseline,
       pg_size_pretty(p.cur_bytes) AS index_size,
       pg_size_pretty(p.base_bytes::bigint) AS baseline_size,
       p.size_ratio,
       p.cur_tuples AS table_tuples,
       p.base_tuples AS baseline_tuples,
       p.tuple_ratio_now,
       p.cur_idx_tuples AS index_tuples,
       p.base_idx_tuples AS baseline_index_tuples,
       p.idx_tuple_ratio_now,
       CASE WHEN p.stage = 'measure' THEN round(p.avg_leaf_density::numeric, 2) END
           AS avg_leaf_density,
       CASE WHEN p.stage = 'measure' THEN p.dead_pages END AS dead_pages,
       p.wasted_pct,
       array_to_string(array_remove(ARRAY[
           CASE WHEN NOT p.owns_index THEN 'not the index owner: no comment can be written' END,
           CASE WHEN p.owns_index AND NOT p.owns_table THEN 'not the table owner: REINDEX may be refused' END,
           CASE WHEN p.baseline = 'invalid' THEN 'unreadable @btmaint: payload, replaced' END,
           CASE WHEN p.tuples_unknown THEN 'table reltuples unknown: table tuple gate cannot fire' END,
           CASE WHEN p.idx_tuples_unknown THEN 'index reltuples unknown: index tuple gate cannot fire' END,
           CASE WHEN p.baseline = 'ok' AND p.base_tuples = 0 THEN 'baseline table tuple count was zero' END,
           CASE WHEN p.baseline = 'ok' AND p.base_idx_tuples = 0 THEN 'baseline index tuple count was zero' END,
           CASE WHEN p.shrank THEN 'index smaller than its baseline: rebuilt elsewhere' END,
           CASE WHEN p.size_gate THEN 'size gate fired' END,
           CASE WHEN p.tuple_gate THEN 'table tuple gate fired' END,
           CASE WHEN p.idx_tuple_gate THEN 'index tuple gate fired' END,
           CASE WHEN p.stage = 'measure' AND p.leaf_pages = 0 THEN 'no leaf pages' END,
           CASE WHEN p.stage = 'measure' AND p.leaf_fragmentation >= 30
                THEN 'fragmented, not wasted space' END,
           CASE WHEN p.fillfactor <> 90 THEN 'fillfactor ' || p.fillfactor END
       ], NULL), '; ') AS notes,
       -- The comment is written now for every action but 'reindex', whose new
       -- baseline is only known after the rebuild; step 2 writes that one.
       CASE WHEN p.action IN ('initialize', 'refresh', 'update')
            THEN format('COMMENT ON INDEX %I.%I IS %L', p.schema_name, p.index_name,
                   CASE WHEN length(btrim(p.user_cmt)) > 0
                        THEN btrim(p.user_cmt) || E'\n' ELSE '' END
                   || '@btmaint:{"v":2,"sz":' || p.cur_bytes
                   || ',"tup":' || round(GREATEST(p.cur_tuples, -1))
                   || ',"itup":' || round(GREATEST(p.cur_idx_tuples, -1))
                   || ',"at":"' || to_char(now(), 'YYYY-MM-DD"T"HH24:MI:SSOF') || '"}') END
           AS comment_command,
       CASE WHEN p.action = 'reindex'
            THEN format('REINDEX INDEX %I.%I', p.schema_name, p.index_name) END
           AS reindex_command
  FROM plan p
 ORDER BY CASE p.action WHEN 'reindex' THEN 0 WHEN 'update' THEN 1
                        WHEN 'initialize' THEN 2 WHEN 'refresh' THEN 3
                        WHEN 'blocked' THEN 4 ELSE 5 END,
          p.cur_bytes DESC, p.schema_name, p.index_name;
```

It needs `CREATE EXTENSION pgstattuple` in the database being examined; both
checkouts ship the same 1.5 control file and the same `pgstatindex(regclass)`
declaration, revoked from `PUBLIC` and granted to `pg_stat_scan_tables`
([pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
`statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so the two `SET`
lines apply at session scope and need neither reload nor restart
([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2612-L2620),
[guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2623-L2631)).

### Step 2: carrying it out

Run it at the top level, not inside `BEGIN`: it commits once per index, and a
`DO` block cannot commit inside an explicit transaction block - measured,
`ERROR: invalid transaction termination` on both servers.

```sql
-- B-tree index-maintenance heuristic, step 2: carry out the plan.
-- One text, unchanged on PostgreSQL 12 through 17.  Run it at the top level:
-- it commits once per index, which no DO block can do inside BEGIN ... END.
--
-- The CTE pipeline below, from "WITH params AS (" to the line before its final
-- SELECT, is byte-identical to the pipeline of step 1.  Only the final SELECT
-- differs: step 1 presents the plan to a reader, step 2 hands five parallel
-- arrays to the driver.  Everything after the snapshot is re-read per index,
-- because a snapshot older than one lock is not a safe thing to act on.
SET /* wiki_btmaint_apply_statement_timeout */ statement_timeout = '15min';
SET /* wiki_btmaint_apply_lock_timeout */ lock_timeout = '5s';

DO /* wiki_btmaint_apply_12_17 */ $btmaint$
DECLARE
    dry_run     boolean := false;  -- true: measure and report, write nothing
    wasted_max  numeric := 40;     -- reindex above this percent; matches step 1
    max_reindex integer := 1000;   -- cap on rebuilds per run; 1 is conservative
    v_oid     oid[];
    v_stage   text[];
    v_leafcap numeric[];
    v_ff      numeric[];
    v_bs      numeric[];
    i         integer;
    n_re      integer := 0;
    n_upd     integer := 0;
    n_init    integer := 0;
    n_ref     integer := 0;
    n_block   integer := 0;
    n_err     integer := 0;
    r_nsp     text;
    r_idx     text;
    r_bytes   numeric;
    r_tuples  numeric;
    r_itup    numeric;
    r_user    text;
    r_tdens   numeric;
    r_wasted  numeric;
    r_act     text;
    r_cmt     text;
BEGIN
WITH params AS (
    SELECT current_setting('block_size')::numeric AS bs,
           24::numeric        AS page_header,     -- SizeOfPageHeaderData
           16::numeric        AS btree_special,   -- MAXALIGN(BTPageOpaqueData)
           1.20::numeric      AS grow_ratio,      -- index-size gate
           0.20::numeric      AS tuple_ratio,     -- tuple-count gate, table and index
           40::numeric        AS wasted_max,      -- reindex above this percent
           0::numeric         AS min_index_bytes, -- ignore anything smaller
           2::numeric         AS fmt              -- payload format version
),
cand AS MATERIALIZED (
    SELECT c.oid                  AS idx_oid,
           n.nspname              AS schema_name,
           c.relname              AS index_name,
           t.relname              AS table_name,
           pg_relation_size(c.oid) AS cur_bytes,
           t.reltuples::numeric   AS cur_tuples,
           c.reltuples::numeric   AS cur_idx_tuples,
           (SELECT o.option_value::int
              FROM pg_options_to_table(c.reloptions) o
             WHERE o.option_name = 'fillfactor')      AS fillfactor_opt,
           d.description          AS cmt,
           pg_has_role(c.relowner, 'USAGE')           AS owns_index,
           pg_has_role(t.relowner, 'USAGE')           AS owns_table
      FROM pg_class c
      JOIN pg_namespace n   ON n.oid = c.relnamespace
      JOIN pg_index x       ON x.indexrelid = c.oid
      JOIN pg_class t       ON t.oid = x.indrelid
      JOIN pg_am a          ON a.oid = c.relam
      LEFT JOIN pg_description d ON d.objoid = c.oid
                               AND d.classoid = 'pg_class'::regclass
                               AND d.objsubid = 0
     CROSS JOIN params p
     WHERE a.amname = 'btree'                    -- pgstatindex takes no other AM
       AND c.relkind = 'i'                       -- 'I' has no storage
       AND x.indisvalid AND x.indisready AND x.indislive
       AND NOT pg_is_other_temp_schema(c.relnamespace)
       AND (c.relpersistence <> 'u' OR NOT pg_is_in_recovery())
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
       AND pg_relation_size(c.oid) >= p.min_index_bytes
),
parsed AS MATERIALIZED (
    SELECT c.*,
           pay.payload,
           substring(pay.payload from '"v":([0-9]{1,6})[,}]')::numeric      AS pv,
           substring(pay.payload from '"sz":([0-9]{1,25})[,}]')::numeric    AS base_bytes,
           substring(pay.payload from
                     '"tup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric AS base_tuples,
           substring(pay.payload from
                     '"itup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric AS base_idx_tuples,
           substring(pay.payload from '"at":"([^"]{1,40})"')            AS base_at,
           -- Two passes, in this order: a well-formed payload goes first, so
           -- that human text on the same line survives, and only a leftover
           -- marker is then removed to the end of its line.
           regexp_replace(
             regexp_replace(coalesce(c.cmt, ''),
                            '[[:space:]]*@btmaint:\{[^}]*\}', '', 'g'),
                            '[[:space:]]*@btmaint:[^\n]*', '', 'g')      AS user_cmt
      FROM cand c
      CROSS JOIN LATERAL (
           SELECT substring(c.cmt from '@btmaint:(\{[^}]*\})') AS payload) pay
),
gate AS MATERIALIZED (
    SELECT s.*,
           p.bs, p.grow_ratio, p.tuple_ratio, p.wasted_max,
           p.bs - p.page_header - p.btree_special AS leaf_cap,
           COALESCE(s.fillfactor_opt, 90)         AS fillfactor,
           CASE WHEN s.cmt IS NULL OR s.cmt !~ '@btmaint:'   THEN 'absent'
                WHEN s.pv IS DISTINCT FROM p.fmt
                  OR s.base_bytes IS NULL
                  OR s.base_tuples IS NULL
                  OR s.base_idx_tuples IS NULL               THEN 'invalid'
                ELSE 'ok' END                                AS baseline,
           -- reltuples is -1 on a relation no ANALYZE, VACUUM or index build
           -- has counted since PostgreSQL 14; 12 and 13 leave 0 there instead
           (s.cur_tuples < 0)                                AS tuples_unknown,
           (s.cur_idx_tuples < 0)                            AS idx_tuples_unknown
      FROM parsed s CROSS JOIN params p
),
decided AS MATERIALIZED (
    SELECT g.*,
           CASE WHEN g.base_bytes > 0
                THEN round(g.cur_bytes / g.base_bytes, 4) END AS size_ratio,
           CASE WHEN g.base_tuples > 0 AND NOT g.tuples_unknown
                THEN round(g.cur_tuples / g.base_tuples, 4) END AS tuple_ratio_now,
           CASE WHEN g.base_idx_tuples > 0 AND NOT g.idx_tuples_unknown
                THEN round(g.cur_idx_tuples / g.base_idx_tuples, 4) END
                                                              AS idx_tuple_ratio_now,
           (g.baseline = 'ok' AND g.base_bytes > 0
            AND g.cur_bytes >= g.base_bytes * g.grow_ratio)     AS size_gate,
           (g.baseline = 'ok' AND NOT g.tuples_unknown
            AND g.base_tuples >= 0
            AND (CASE WHEN g.base_tuples = 0
                      THEN g.cur_tuples > 0
                      ELSE abs(g.cur_tuples - g.base_tuples)
                           >= g.base_tuples * g.tuple_ratio END))  AS tuple_gate,
           -- The index's own entry count, gated exactly like the table's.  A
           -- partial index's population is not its table's population, which
           -- is the whole reason this test is separate.
           (g.baseline = 'ok' AND NOT g.idx_tuples_unknown
            AND g.base_idx_tuples >= 0
            AND (CASE WHEN g.base_idx_tuples = 0
                      THEN g.cur_idx_tuples > 0
                      ELSE abs(g.cur_idx_tuples - g.base_idx_tuples)
                           >= g.base_idx_tuples * g.tuple_ratio END)) AS idx_tuple_gate,
           (g.baseline = 'ok' AND g.base_bytes > 0
            AND g.cur_bytes < g.base_bytes)                     AS shrank
      FROM gate g
),
staged AS MATERIALIZED (
    SELECT d.*,
           CASE WHEN NOT d.owns_index                    THEN 'blocked'
                WHEN d.baseline <> 'ok'                  THEN 'initialize'
                WHEN d.shrank                            THEN 'refresh'
                WHEN d.size_gate OR d.tuple_gate
                  OR d.idx_tuple_gate                    THEN 'measure'
                ELSE 'skip' END                          AS stage
      FROM decided d
)
SELECT /* wiki_btmaint_snapshot_12_17 */
       array_agg(s.idx_oid  ORDER BY s.cur_bytes DESC, s.idx_oid),
       array_agg(s.stage    ORDER BY s.cur_bytes DESC, s.idx_oid),
       array_agg(s.leaf_cap ORDER BY s.cur_bytes DESC, s.idx_oid),
       array_agg(s.fillfactor::numeric ORDER BY s.cur_bytes DESC, s.idx_oid),
       array_agg(s.bs       ORDER BY s.cur_bytes DESC, s.idx_oid)
  INTO v_oid, v_stage, v_leafcap, v_ff, v_bs
  FROM staged s
 WHERE s.stage <> 'skip';

FOR i IN 1 .. COALESCE(array_length(v_oid, 1), 0) LOOP
    -- Re-read the index under its own lock-free catalog snapshot.  A row that
    -- has gone is a dropped index, and a dropped index is not an error here.
    SELECT n.nspname, c.relname, pg_relation_size(c.oid), t.reltuples::numeric,
           c.reltuples::numeric,
           regexp_replace(
             regexp_replace(COALESCE(d.description, ''),
                            '[[:space:]]*@btmaint:\{[^}]*\}', '', 'g'),
                            '[[:space:]]*@btmaint:[^\n]*', '', 'g')
      INTO r_nsp, r_idx, r_bytes, r_tuples, r_itup, r_user
      FROM pg_class c
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      LEFT JOIN pg_description d ON d.objoid = c.oid
                               AND d.classoid = 'pg_class'::regclass
                               AND d.objsubid = 0
     WHERE c.oid = v_oid[i];
    IF NOT FOUND OR r_bytes IS NULL THEN
        RAISE NOTICE 'btmaint: index % gone since the snapshot, skipped', v_oid[i];
        CONTINUE;
    END IF;

    r_act := v_stage[i];
    IF r_act = 'blocked' THEN
        n_block := n_block + 1;
        RAISE NOTICE 'btmaint: %.% not owned by %, no comment written',
                     r_nsp, r_idx, current_user;
        CONTINUE;
    END IF;

    IF r_act = 'measure' THEN
        r_tdens := (v_leafcap[i] - (v_bs[i] * (100 - v_ff[i])) / 100) / v_leafcap[i];
        BEGIN
            SELECT round(100 * (GREATEST(round(s.leaf_pages * v_leafcap[i] * r_tdens)
                                         - round(s.leaf_pages * v_leafcap[i] *
                                            CASE WHEN s.leaf_pages > 0
                                                  AND s.avg_leaf_density <> 'NaN'::float8
                                                 THEN (s.avg_leaf_density / 100)::numeric
                                                 ELSE 0::numeric END), 0)
                                + (s.empty_pages + s.deleted_pages) * v_bs[i])
                       / NULLIF(s.index_size, 0), 1)
              INTO r_wasted
              FROM pgstatindex(v_oid[i]::regclass) s;
        EXCEPTION WHEN OTHERS THEN
            n_err := n_err + 1;
            RAISE WARNING 'btmaint: pgstatindex(%.%) failed: %', r_nsp, r_idx, SQLERRM;
            CONTINUE;
        END;
        IF r_wasted IS NOT NULL AND r_wasted > wasted_max THEN
            r_act := 'reindex';
        ELSE
            r_act := 'update';
        END IF;
        RAISE NOTICE 'btmaint: %.% wasted % %% -> %', r_nsp, r_idx,
                     COALESCE(r_wasted::text, 'null'), r_act;
    END IF;

    IF r_act = 'reindex' THEN
        IF n_re >= max_reindex THEN
            RAISE NOTICE 'btmaint: %.% needs a rebuild, max_reindex % reached',
                         r_nsp, r_idx, max_reindex;
            CONTINUE;
        END IF;
        IF NOT dry_run THEN
            EXECUTE format('REINDEX /* wiki_btmaint_reindex */ INDEX %I.%I', r_nsp, r_idx);
            -- The rebuild resized the file and recounted both the heap and the
            -- index, so all three baseline values are re-read after it, not
            -- before.
            SELECT pg_relation_size(c.oid), t.reltuples::numeric, c.reltuples::numeric
              INTO r_bytes, r_tuples, r_itup
              FROM pg_class c
              JOIN pg_index x ON x.indexrelid = c.oid
              JOIN pg_class t ON t.oid = x.indrelid
             WHERE c.oid = v_oid[i];
        END IF;
        n_re := n_re + 1;
    ELSIF r_act = 'initialize' THEN n_init := n_init + 1;
    ELSIF r_act = 'refresh'    THEN n_ref  := n_ref  + 1;
    ELSE                            n_upd  := n_upd  + 1;
    END IF;

    r_cmt := CASE WHEN length(btrim(r_user)) > 0 THEN btrim(r_user) || E'\n' ELSE '' END
             || '@btmaint:{"v":2,"sz":' || r_bytes
             || ',"tup":' || round(GREATEST(r_tuples, -1))
             || ',"itup":' || round(GREATEST(r_itup, -1))
             || ',"at":"' || to_char(now(), 'YYYY-MM-DD"T"HH24:MI:SSOF') || '"}';
    IF NOT dry_run THEN
        EXECUTE format('COMMENT /* wiki_btmaint_comment */ ON INDEX %I.%I IS %L',
                       r_nsp, r_idx, r_cmt);
        COMMIT;
    END IF;
END LOOP;

RAISE NOTICE 'btmaint: reindex=% update=% initialize=% refresh=% blocked=% failed=% dry_run=%',
             n_re, n_upd, n_init, n_ref, n_block, n_err, dry_run;
END
$btmaint$;
```

The pipeline in step 2, from `WITH params AS (` through the end of the `staged`
CTE, is byte-identical to the same region of step 1 - 100 lines, SHA-256
`62225bce7d3e31b9…` in both texts, checked by the scripts on every run. Only the
final `SELECT` differs: step 1 presents the plan to a reader, step 2 hands five
parallel arrays to its driver. The gate is therefore defined once.

### How to read the plan

`action` first. Everything else is the evidence behind it.

| Column | Means | Watch for |
|---|---|---|
| `action` | `reindex`, `update`, `refresh`, `initialize`, `skip`, `blocked` | `blocked` means this session cannot write the comment at all |
| `baseline` | `ok`, `absent`, `invalid` | `invalid` is a marker that could not be parsed; it is replaced, not trusted |
| `size_ratio` | current bytes over stored bytes | `>= 1.20` is the first gate |
| `tuple_ratio_now` | current table `reltuples` over stored | a move of 20 % either way is the second gate |
| `index_tuples`, `baseline_index_tuples`, `idx_tuple_ratio_now` | the index's own `pg_class.reltuples`, now and at baseline, and their ratio | a move of 20 % either way is the third gate; this is the column that sees a partial index drain |
| `avg_leaf_density`, `dead_pages` | straight from `pgstatindex`, present only for measured rows | density is blind to pages holding nothing; `dead_pages` is 100 % waste |
| `wasted_pct` | free bytes beyond a rebuild at this index's own fillfactor, plus every empty and deleted page, over the file | `> 40` is the rebuild decision |
| `notes` | why a row looks odd, and which gates fired | fourteen strings, listed in the ladder below |
| `comment_command`, `reindex_command` | exactly what step 2 will run | `comment_command` is NULL for a `reindex` row, because its new baseline is only known after the rebuild |

### The decision ladder, in order

The first rule that matches decides. This is the order in the `staged` CTE, and
the suite scores it against the same arithmetic computed independently: 140 of
140 agreed on 17.11 and 122 of 122 on 12.2.

| # | Condition | Action | Measures? | Writes? |
|---|---|---|---|---|
| 1 | this session does not own the index | `blocked` | no | no |
| 2 | no readable `@btmaint:` payload, including any v1 payload | `initialize` | no | baseline |
| 3 | the index is *smaller* than its stored baseline | `refresh` | no | baseline |
| 4 | size ratio `>= 1.20`, or table tuples moved `>= 20 %`, or index tuples moved `>= 20 %` | `measure` | one `pgstatindex()` call | see below |
| 5 | none of the above | `skip` | no | no |
| 4a | measured `wasted_pct > 40` | `reindex` | - | `REINDEX`, then the post-rebuild baseline |
| 4b | measured `wasted_pct <= 40` | `update` | - | baseline at current values |

The three tests in rule 4 are an `OR`, so the order among them does not matter,
and each is disabled separately when its input is unusable: a `reltuples` of `-1`
disables that count's test and says so in `notes`, while a stored count of zero
makes any non-zero current count a fire, because an increase from zero has no
finite ratio.

Rule 3 is not in the brief and is needed: something else rebuilt or truncated the
index, the stored size is now too high, and a high baseline can hide real growth
for a long time. It is also how a rebuild gets its new baseline when a
`REINDEX INDEX CONCURRENTLY` was run by hand outside step 2 - fixture 121's
`nzb_k`, which is rebuilt while its table is empty and then reloaded, took the
`refresh` branch on both servers.

Rule 4b is the brief's own instruction, and it has a consequence worth saying out
loud: **the baseline ratchets upward**. An index measured at 35 % wasted is not
rebuilt, and its baseline is then reset to the larger current size, so the next
20 % of growth is counted from there. Over several runs a slowly bloating index
needs progressively more absolute growth to be looked at again. The nine-point
curve below shows exactly where that leaves it.

Fourteen `notes` strings exist, and each one is a fact the reader would otherwise
have to guess: `not the index owner: no comment can be written`, `not the table
owner: REINDEX may be refused`, `unreadable @btmaint: payload, replaced`, `table
reltuples unknown: table tuple gate cannot fire`, `index reltuples unknown: index
tuple gate cannot fire`, `baseline table tuple count was zero`, `baseline index
tuple count was zero`, `index smaller than its baseline: rebuilt elsewhere`, `size
gate fired`, `table tuple gate fired`, `index tuple gate fired`, `no leaf pages`,
`fragmented, not wasted space`, and `fillfactor N`. The three "gate fired" strings
replace the previous revision's single `both gates fired`: with three tests, which
one opened the measurement is the interesting part, and every measured row now
names its own reason.

### Why every candidate filter is there

`pgstatindex` raises on four shapes, and one raised error aborts the whole
statement, so each one is excluded before the function is ever reached. All four
refusals were measured on both servers by calling the function directly on the
excluded index:

| Excluded by | Because | Measured refusal |
|---|---|---|
| `a.amname = 'btree'` | the function opens the relation and demands a B-tree index | `relation "e_am_hash" is not a btree index`, identically for gist, spgist, gin and brin ([pgstatindex.c#IS_BTREE](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228)) |
| `c.relkind = 'i'` | a partitioned index has no storage, and fails the same test | `relation "e_part_k" is not a btree index` ([pgstatindex.c#IS_INDEX](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70-L71)) |
| `NOT pg_is_other_temp_schema(...)` | another session's local buffers are not visible | `cannot access temporary tables of other sessions` ([pgstatindex.c#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L230-L238)) |
| `x.indisvalid AND x.indisready AND x.indislive` | an index that is not ready can report a size too low for the table | `index "e_inv_k" is not valid` on 17.11; **on 12.2 the same call returns a row** ([pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250)) |

The leaf index of a partitioned table is `relkind = 'i'` and is a candidate,
which the suite confirms: `e_part_1_k_idx` was initialized while `e_part_k` was
not. Two further filters are not about refusals: `nspname NOT IN ('pg_catalog',
'information_schema', 'pg_toast')` keeps the sweep away from catalogs whose
comments are part of the installation, and `(relpersistence <> 'u' OR NOT
pg_is_in_recovery())` keeps it off unlogged indexes on a standby. `pg_relation_size`
itself is safe against a concurrent drop: it opens with `try_relation_open` and
returns NULL rather than raising, so a dropped index leaves the candidate set
quietly ([dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L368)).

### The three gates, measured at every boundary

Ten forged baselines, one index each, on a 200,000-row table, with one input moved
per fixture and the other two set to the current value so they cannot fire. Every
boundary is exact, and the measured `notes` column names the one gate that opened
each row. Identical on 12.2 and 17.11:

| Fixture | `size_ratio` | `tuple_ratio` | `idx_tuple_ratio` | Action | Gate named in `notes` |
|---|---|---|---|---|---|
| `e_g_size_on` | 1.200000 | 1.000000 | 1.000000 | `update` (measured) | size gate fired |
| `e_g_size_off` | 1.199999 | 1.000000 | 1.000000 | `skip` | - |
| `e_g_up_on` | 1.000000 | 1.200005 | 1.000000 | `update` (measured) | table tuple gate fired |
| `e_g_up_off` | 1.000000 | 1.199890 | 1.000000 | `skip` | - |
| `e_g_dn_on` | 1.000000 | 0.800000 | 1.000000 | `update` (measured) | table tuple gate fired |
| `e_g_dn_off` | 1.000000 | 0.800106 | 1.000000 | `skip` | - |
| `e_g_iup_on` | 1.000000 | 1.000000 | 1.200005 | `update` (measured) | index tuple gate fired |
| `e_g_iup_off` | 1.000000 | 1.000000 | 1.199890 | `skip` | - |
| `e_g_idn_on` | 1.000000 | 1.000000 | 0.800000 | `update` (measured) | index tuple gate fired |
| `e_g_idn_off` | 1.000000 | 1.000000 | 0.800106 | `skip` | - |

All three gates are `>=` comparisons on `numeric`, so 1.20 exactly fires, and the
brief's "20 % or more" is implemented as written on each input. Four edge states
are handled explicitly rather than left to arithmetic: a stored count of zero, on
either counter, makes any non-zero current count a fire (an increase from zero has
no finite ratio), and `reltuples < 0` - the `-1` that means "no ANALYZE, VACUUM or
index build has counted this relation yet" from PostgreSQL 14 on
([pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)) -
disables that counter's gate and says so in `notes`. Both unknown states were
measured with a forged `-1`, one on a table and one on an index, each with a
readable baseline so the gate it disables is a gate that would otherwise have an
opinion:

| Fixture | Forged | Candidate? | Action | `notes` |
|---|---|---|---|---|
| `e_unk_k` | the table's `reltuples = -1` | yes | `skip` | `table reltuples unknown: table tuple gate cannot fire` |
| `e_iunk_k` | the index's `reltuples = -1` | yes | `skip` | `index reltuples unknown: index tuple gate cannot fire` |

### The index's own entry count, and who writes it

The third gate is only as fresh as the last writer of `pg_class.reltuples` for the
index, and there are exactly three of them. All three were measured on both
servers, on one 10,000-row table carrying a plain index and a partial index whose
predicate selects one row in five:

| Step | table | plain index | partial index | Writer |
|---|---|---|---|---|
| after both `CREATE INDEX` | - | 10000 | **2000** | `index_update_stats`, an exact count from the build's own scan ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2788-L2812)) |
| after `ANALYZE` | 10000 | 10000 | **2000** | `do_analyze_rel`'s per-index update, `ceil(tupleFract * totalrows)` ([analyze.c#same-for-indexes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)) |
| after deleting 98 % of the subset, then `VACUUM` | 8200 | 8200 | **200** | `update_relstats_all_indexes`, from the AM's own `num_index_tuples` ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099)) |
| after a second `ANALYZE` | 8200 | 8200 | 200 | the sample agreed with the exact count |
| after a 10-row `UPDATE` and another `VACUUM` | 8200 | 8200 | 200 | nothing: see the estimated-count rule below |
| after `REINDEX INDEX` on the partial index | - | - | 200 | `index_update_stats` again |

Three properties of those writers decide what the gate can see:

- **ANALYZE estimates a partial index from its sample, and a plain index from the
  table.** `tupleFract` starts at 1.0 for every index and is only refined for an
  index `compute_index_stats` processes, which it does when the index has
  statistics columns *or* a predicate
  ([analyze.c#tupleFract-init](../../../../raw/postgres-17/src/backend/commands/analyze.c#L443-L449),
  [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L827-L863)).
  For a partial index the fraction is the share of sampled rows that pass the
  predicate, `numindexrows / numrows`
  ([analyze.c#tupleFract-from-sample](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)).
  So for a plain index the third gate carries no information the second one does
  not; for a partial index it is the only catalog number that moves at all.
- **VACUUM writes an exact count, but only when it is exact.**
  `update_relstats_all_indexes` skips any index whose result came back with
  `estimated_count` set, and `btvacuumcleanup` sets exactly that whenever it runs
  as a cleanup-only scan, because it then counts index items per page rather than
  live TIDs
  ([nbtree.c#btvacuumcleanup-estimated](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L874-L892),
  [nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1345-L1362)).
  `lazy_cleanup_all_indexes` also marks the count estimated when the heap scan
  skipped pages
  ([vacuumlazy.c#lazy_cleanup_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2349-L2360)).
  The measured 10-row `UPDATE` above is that case: a `VACUUM` ran, and no count
  moved.
- **A VACUUM can skip index vacuuming entirely.** When fewer than
  `BYPASS_THRESHOLD_PAGES`, 2 % of heap pages, hold dead items, `lazy_vacuum`
  bypasses index vacuuming and does cleanup only
  ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89),
  [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1940)),
  which by the rule above leaves the index's count where it was.

The suite checks the value the heuristic stores against the value the catalog
holds at baseline time, independently of the filed text: **140 of 140 agree on
17.11 and 122 of 122 on 12.2**, and every acted row wrote an `itup` back.

### The simulated auto-analyze the tests now run

The follow-up's second instruction is a property of the *tests*, not of the
heuristic: a fixture that changes more than 10 % of a table's heap tuples must
`ANALYZE` it, because on a server with autovacuum on the launcher would have. The
suite's cluster runs `autovacuum = off` so that nothing moves a fixture between
the baseline, the churn, the decision and the rebuild oracle, so the `ANALYZE`
has to be supplied.

It is not hand-written per fixture. The new step reads the engine's own counter
and applies the engine's own test:

```text
n_mod_since_analyze > autovacuum_analyze_threshold
                      + autovacuum_analyze_scale_factor * reltuples
```

That is `relation_needs_vacanalyze`'s `doanalyze` verbatim
([autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3060-L3096),
[autovacuum.c#anl-thresholds](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3005-L3018)),
and at the shipped defaults of 50 tuples and 0.1 it is "50 rows plus 10 % of the
table"
([guc_tables.c#autovacuum_analyze_threshold](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375),
[guc_tables.c#autovacuum_analyze_scale_factor](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914),
[config.sgml#autovacuum_analyze_scale_factor](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8832-L8850)).
Both defaults were read back from each running server: 50 and 0.1 on 12.2 and on
17.11. The counter itself is `pg_stat_all_tables.n_mod_since_analyze`
([system_views.sql#n_mod_since_analyze](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L689)),
which the flush adds to and `pgstat_report_analyze` resets
([pgstat_relation.c#report_analyze-reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L338),
[pgstat_relation.c#flush-mod_since_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L845-L860)).

What that decided, per leg:

| | 17.11 | 12.2 |
|---|---|---|
| Fixture tables considered | 99 | 97 |
| Analyzed, because the churn crossed the threshold | **77** | **57** |
| Left alone | 22 | 40 |
| Counter back to zero afterwards | 77 of 77 | 57 of 57 |
| Smallest modified share analyzed | **11.1 %** | **11.1 %** |
| Largest modified share left alone | **9.8 %** | **9.8 %** |

The two boundary rows are the rule itself, measured: `pb73` moved 50,000 of
450,000 rows, 11.1 %, and was analyzed; `f88t` moved 17,778 of 182,222 rows,
9.8 %, and was not. The step also records the counter after the fact, which is how
the run proves the `ANALYZE` really fired rather than reporting an intention.

Two honest caveats about the census, both visible in its own output:

- **It can analyze a table that was analyzed moments earlier.** A backend flushes
  pending table statistics at most once per `PGSTAT_MIN_INTERVAL`, 1000 ms
  ([pgstat.c#PGSTAT_MIN_INTERVAL](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L110-L122),
  [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L655)),
  so a `DELETE`, `VACUUM`, `ANALYZE` sequence in one session can reset
  `mod_since_analyze` to zero and *then* have the delete's own modification count
  added to it. In the edge database the curve fixtures show both outcomes side by
  side: `e_del80` reads 400,000 mods after its own `ANALYZE`, while `e_del90`,
  whose flush happened to land before its `ANALYZE`, reads 0. The census therefore
  over-triggers rather than under-triggers, which is the safe direction for a test
  that is meant to make counts fresh.
- **It repaired several fixtures whose point was a stale count.** That is what
  "apply the rule everywhere" means, and the affected fixtures are named in
  [The ported mandatory suite](#the-ported-mandatory-suite). The two deliberate
  catalog forgeries are written after the census, so they survive it.

### Wasted space, and where the threshold lands

`pgstatindex` reports one density number for the whole index: `100 - free_space /
max_avail * 100` over live leaf pages only, where `max_avail` is
`BLCKSZ - (BLCKSZ - pd_special + SizeOfPageHeaderData)` per leaf page, and
`NaN` when there are no leaf pages at all
([pgstatindex.c#avg_leaf_density](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372),
[pgstatindex.c#max_avail](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L324)).
Wasted space is therefore built from three measured quantities, and measured
against a rebuild at the index's *own* fillfactor, so a correctly built index
reports no waste whatever its fillfactor:

```text
leaf_cap       = block_size - 24 - 16                     -- header, special area
target_density = (leaf_cap - block_size * (100 - fillfactor) / 100) / leaf_cap
wasted_bytes   = GREATEST(leaf_pages * leaf_cap * target_density
                          - leaf_pages * leaf_cap * density, 0)
                 + (empty_pages + deleted_pages) * block_size
wasted_pct     = 100 * wasted_bytes / index_size
```

The two constants are the page header and the B-tree special area, and the target
is the same `BLCKSZ * (100 - fillfactor) / 100` the builder itself leaves on a leaf
page ([nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145),
[nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671)).
The leaf term is clamped at zero because an index denser than its target is not
holding negative waste, while empty and deleted pages count in full - they hold
nothing at any fillfactor. `index_size` is `pgstatindex`'s own count of every
page including the metapage
([pgstatindex.c#index_size](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L350-L357),
[pgstattuple.sgml#metapage](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L273)).
This is the definition the sibling page
[B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17
(unverified)](btree-bloat-with-pgstatindex.md#follow-up-wasted-space-measured-against-the-fillfactor)
settled on, reused here unchanged so the two pages measure the same thing.

Nine 500,000-row tables, one unique-key index each, a known scattered fraction of
rows deleted and vacuumed, and the baseline already stored. The curve is
**identical on 12.2 and 17.11**, because unique keys give deduplication nothing
to do:

| Rows deleted | `wasted_pct` | Action |
|---|---|---|
| 10 % | not measured | `skip` - no gate fired |
| 20 % | 17.7 | `update` |
| 30 % | 26.6 | `update` |
| 40 % | 35.5 | `update` |
| 50 % | 44.3 | `reindex` |
| 60 % | 53.2 | `reindex` |
| 70 % | 62.1 | `reindex` |
| 80 % | 71.0 | `reindex` |
| 90 % | 79.9 | `reindex` |

The reading is `0.888 × fraction deleted` to three digits across the whole range,
which is what the formula predicts: `target_density` (0.8995 at 8192 and
fillfactor 90) times the leaf share of the file. So **the 40 % threshold trips at
45.0 % of entries deleted**, and the 10 % row is the gate's own blind spot rather
than the threshold's. The third gate does not change that row: these are
non-partial unique indexes, so a 10 % drain moves the table's count and the
index's count by the same 10 %, both under the 20 % threshold, and the file has
not grown. The curve is unchanged from the two-gate revision, point for point.

### What the gate still cannot see

The previous revision lost 21 of 140 fixtures on 17.11, and the same 21 of 122 on
12.2, to the gate: all partial indexes, in three families. **The index-count gate
and the simulated auto-analyze between them close all 21**, and the two changes
are needed in combination, because two of the three families are a gate problem
and one is a statistics problem:

| Family that the two-gate form lost | Fixtures | Why it was lost | What fixed it | Now |
|---|---|---|---|---|
| The subset drained, the table did not | `p113b`, `p113c`, `p114`, `p117` | `tuple_ratio` 1.0000 against `idx_tuple_ratio` 0.0000 to 0.0103 | the index-count gate | all four measured; `p113b`, `p114`, `p117` rebuilt at 98.9 % to 100.0 % reclaim, `p113c` still lost at the threshold |
| The table moved 15-19 %, the index lost 75-95 % | `p77`, `f86`-`f91`, `i100`, `p69`, `p74`, `p75`, `b92`-`b95` | the table's count moved 18 %, two points under the gate | the index-count gate | all 14 measured and rebuilt, 40.5 % to 94.2 % reclaim |
| Nothing had counted anything yet | `p113a`, `p65`, `p67` | no `VACUUM` and no `ANALYZE`, so neither count had moved | the simulated auto-analyze moved the index's count | all three measured; all three still lost at the threshold, below |

The second family is the sharpest illustration of why the table's count is the
wrong input: deleting 90 % of a subset that is 20 % of the table moves the
table's count by exactly 18 %, and the gate needs 20 %. Four of those fixtures
(`b92`-`b95`) are the sibling page's own threshold-calibration controls, built to
sit just under a threshold, and the index's own count moves from 1.0000 to about
0.10 on exactly the same churn.

The third family is the one the first revision called invisible to any
catalog-only gate, and it was — with autovacuum off and no `ANALYZE` in the
recipe. With the 10 % rule in force, an `ANALYZE` runs, the partial index's
estimated entry count collapses, and the gate opens. What happens next is a
separate failure, described in
[The four false negatives that remain](#the-four-false-negatives-that-remain).

### The comment survives both REINDEX forms

Two indexes on one table, the same comment on both, md5 of the comment text
recorded before and after:

| Index | Rebuilt with | OID before | OID after | Comment md5 |
|---|---|---|---|---|
| `e_surv_a` | `REINDEX INDEX` | 18146 | 18146 | `f6e063ed…` unchanged |
| `e_surv_b` | `REINDEX INDEX CONCURRENTLY` | 18147 | **18149** | `f6e063ed…` unchanged |

On 12.2 the same fixture reads 17414 unchanged and 17415 -> 17417, with the same
comment md5 before and after.

The plain form keeps the same `pg_class` row and only swaps the relfilenode, so a
comment keyed by the index OID cannot move
([index.c#reindex_index-locks](../../../../raw/postgres-17/src/backend/catalog/index.c#L3600-L3612),
[index.c#reindex_index-index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L3786-L3790)).
The concurrent form builds a new index and swaps, and the comment survives only
because `index_concurrently_swap` explicitly rewrites the `pg_description` row's
`objoid` under `RowExclusiveLock`, taking the first match and stopping
([index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784)).
The measured OID change proves it was the second path.

One thing the rebuild does that the brief does not mention: it refreshes both row
counts. `index_build` calls `index_update_stats` twice, once for the heap with the
tuples its scan counted and once for the index
([index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3129-L3134),
[index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2788-L2812)).
Measured on both servers: the table's `reltuples` reads `-1` on 17.11 (`0` on
12.2) after `CREATE TABLE` and after 1,000 inserts, then **1000 after `CREATE
INDEX`**, 1000 after `ANALYZE`, `-1` again on 17.11 (`0` on 12.2) after
`TRUNCATE`, and **2000 after `REINDEX`** once 1,000 more rows were added. The
index's own count behaves the same way and is exact after either build: a partial
index reads 2000 of a 10,000-row table after `CREATE INDEX`, and 200 after a
`REINDEX` that followed a drain. That is why step 2 re-reads **all three** values
*after* the rebuild rather than before: the post-rebuild baseline it stores is
two counted numbers and a file size, not an estimate.

### Privileges: measure, rebuild, write the baseline

Three different permissions, and they do not line up. Measured with a role holding
only `SELECT` on the table plus `EXECUTE` on `pgstatindex`:

| Operation | Requirement | 17.11 | 12.2 |
|---|---|---|---|
| `pgstatindex()` | `EXECUTE`, revoked from `PUBLIC`, granted to `pg_stat_scan_tables` | granted, works | granted, works |
| `REINDEX INDEX` | `MAINTAIN` on the table from v16; ownership before | `permission denied for index e_none`, then **accepted after `GRANT MAINTAIN`** | `must be owner of index e_none`; `MAINTAIN` does not exist (`unrecognized privilege type`) |
| `COMMENT ON INDEX` | ownership of the index, always | `must be owner of index e_none`, **still refused with `MAINTAIN`** | `must be owner of index e_none` |
| The heuristic's own verdict | `owns_index` | `blocked / not the index owner: no comment can be written` | the same |

So on 17 a `MAINTAIN` grantee can rebuild an index it may not comment on, which
would rebuild the file and then lose the new baseline; the heuristic refuses the
whole row instead and says why. The asymmetry is in the source: `REINDEX` checks
`pg_class_aclcheck(table, ACL_MAINTAIN)`
([indexcmds.c#RangeVarCallbackForReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2905-L2912),
[reindex.sgml#MAINTAIN](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L300-L316)),
while `COMMENT` calls `check_object_ownership`, which for an index is
`object_ownercheck(RelationRelationId, ...)`
([comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L78),
[objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2387-L2400),
[comment.sgml#owner](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L100-L102)).

The text also carries an `owns_table` test, and the suite shows it can never
differ from `owns_index`: zero indexes in either database had an owner different
from their table's, and `ALTER INDEX ... OWNER TO` does not change it - it emits
`WARNING: cannot change owner of index "e_none"` with the hint to change the
table's ownership instead, and the owners were unchanged afterwards. The test
stays as a cheap guard, not because it is reachable.

### Locks, transactions and the two commands a DO block cannot reach

| Statement | Lock it takes | Measured |
|---|---|---|
| `COMMENT ON INDEX` | `ShareUpdateExclusiveLock` on the index, held to commit | `ShareUpdateExclusiveLock on e_none`, read from `pg_locks` inside the transaction |
| `REINDEX INDEX` | `AccessExclusiveLock` on the index, `ShareLock` on the table | from source ([index.c#reindex_index-locks](../../../../raw/postgres-17/src/backend/catalog/index.c#L3600-L3612), [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2822-L2829)) |

`ShareUpdateExclusiveLock` conflicts with itself and with everything above it, so
two concurrent runs of step 2 serialize on the comment rather than corrupting it,
and a run blocks behind `VACUUM` or `CREATE INDEX CONCURRENTLY` on the same index
([lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L76-L80)).
`lock_timeout = '5s'` is what turns that wait into a reported failure instead of a
stall.

Three refusals shape step 2's design, all measured identically on 12.2 and 17.11:

- `REINDEX INDEX CONCURRENTLY` inside a `DO` block: `REINDEX CONCURRENTLY cannot
  be executed from a function`. Plain `REINDEX INDEX` is accepted. The concurrent
  form is refused because `ExecReindex` calls `PreventInTransactionBlock`
  ([indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2736-L2738)),
  so an operator who needs the online form runs the `reindex_command` column of
  step 1 at the top level and lets the next run's rule 3 store the new baseline.
- `VACUUM` inside a `DO` block: `VACUUM cannot be executed from a function`.
  That is why the suite's own drains are generated and executed one statement at
  a time rather than looped inside a procedure.
- `COMMENT ON INDEX x IS 'a' || 'b'`: `syntax error at or near "||"`. The comment
  text is a string literal in the grammar, never an expression
  ([gram.y#comment_text](../../../../raw/postgres-17/src/backend/parser/gram.y#L7219-L7222)),
  which is the whole reason step 1 cannot write a post-rebuild baseline and step 2
  has to build the literal in PL/pgSQL and `EXECUTE` it.

### What it costs to run

Measured six times in each state on a database holding 143 B-tree indexes and
561 MB of index files on 17.11, 125 indexes and 569 MB on 12.2. The gated state
is forged by halving every stored `sz` while leaving both counts current, so it
isolates one gate and reads every gated index end to end:

| State | Gated indexes | Wall time, six runs | Buffers |
|---|---|---|---|
| Settled, 17.11 | 0 of 143 | 19.3 - 21.0 ms | 4,074 hit, 0 read |
| Every baseline halved, 17.11 | 140 of 143 | 108.7 - 130.4 ms | 17,915 hit, 57,932 read |
| Settled, 12.2 | 0 of 125 | 18.9 - 20.4 ms | 2,448 hit, 0 read |
| Every baseline halved, 12.2 | 122 of 125 | 112.3 - 130.2 ms | 15,454 hit, 59,861 read |

The third gate costs no data-page reads, which is the structural point: it reads
one more column from a `pg_class` row the statement already has. The settled
readings are **0 reads** on both servers, at 4,074 buffer hits on 17.11 and 2,448
on 12.2; the 17.11 figure is 532 hits above the two-gate revision's 3,542 while
12.2's is unchanged, so those 532 hits are not attributable to the new column on
this evidence. The fully gated state touches 75,847 buffers, about 592 MB, and the
hit/read split moves between runs with what shared buffers already held: 57,932
read on this run against 51,051 on an earlier one for the same 75,818 total.
`pgstatindex` reads every page of
every index it is called on, through a `BAS_BULKREAD` strategy ring
([pgstatindex.c#bstrategy](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L222),
[pgstatindex.c#scan-loop](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).
A scheduled run in a healthy database is therefore a catalog query, and the
expensive path is entered only for indexes that moved.

### What this revision changed, and why

| Change | Where | Why |
|---|---|---|
| A third payload field, `itup` | both texts | a partial index's population is not its table's, which cost the previous revision 21 fixtures |
| `fmt` 1 -> 2, and a missing `itup` reads `invalid` | both texts | a half-read baseline would gate two indexes differently for no visible reason |
| `idx_tuple_gate`, `OR`-ed into rule 4 | both texts | the follow-up's instruction, at the same 20 % up-or-down threshold |
| `idx_tuples_unknown`, and two new `notes` strings for the unknown and zero cases | both texts | `-1` must disable a gate, not read as a 100 % collapse |
| `both gates fired` replaced by three per-gate strings | step 1's `notes` | with three tests, which one fired is the part a reader needs |
| Three new output columns | step 1 | the gate's third input has to be visible to be checked |
| All three values re-read after a rebuild | step 2 | the post-rebuild baseline must be counted, not estimated |
| A simulated auto-analyze step, driven by the engine's own threshold | both leg scripts | the follow-up's second instruction, applied to every fixture table |
| Fixture 84's forgery moved out of the churn file | both leg scripts | an `ANALYZE` would have repaired the forgery the fixture exists to test |
| Four new gate-boundary fixtures and two new comment shapes | both leg scripts | a new gate and a new payload shape need their own boundaries |
| A forged index `reltuples = -1` fixture | both leg scripts | the unknown-index-count branch had no test |
| A per-gate attribution and a two-gate counterfactual in the scorer | both leg scripts | "the third gate helped" is a claim that needs its own arithmetic |

### Two defects the first revision's suite found

**The gate did not gate.** The first filed text measured with
`LEFT JOIN LATERAL (SELECT * FROM pgstatindex(...) WHERE stage = 'measure')`,
which reads correctly and is wrong: a set-returning function in `FROM` is
executed into a tuplestore and the qualification is applied to the rows it
produced, not before it runs
([nodeFunctionscan.c#FunctionNext](../../../../raw/postgres-17/src/backend/executor/nodeFunctionscan.c#L59-L112),
[execScan.c#ExecScan](../../../../raw/postgres-17/src/backend/executor/execScan.c#L164-L205)).
The cost stage caught it: a *settled* database, with nothing to do, read 57,964
buffers. The repair makes the gated rows the lateral's input relation - a
`MATERIALIZED` CTE of the rows whose stage is `measure`, joined laterally, with
the rest added back by `UNION ALL` - and the settled reading fell to 3,542 hits
and zero reads.

**A malformed marker was kept as if it were a human note.** The payload stripper
matched `@btmaint:\{[^}]*\}` only, so a payload with no closing brace, and
`@btmaint:not json at all`, survived the rewrite and were carried forward as the
user's own text - measured at 95 and 97 bytes where a clean initialize then wrote
72. The repair is two passes in one expression: a well-formed payload is removed
first, so human text on the same line survives, and only then is any leftover
marker removed to the end of its line. Both fixtures read `invalid` and end at the
clean length - now 86 bytes - while `e_mid`,
`@btmaint:{...} and text after it`, still keeps its trailing sentence.

Both repairs are in the filed texts, and every number on this page was measured
after them. The 2026-09-11 re-measurement found no new defect in the two texts;
it did find one in the new harness step, a `round(double precision, integer)`
error from multiplying a `numeric` scale factor by a `real` `reltuples`, which
aborted the first census run and is fixed by casting the count in both leg
scripts.

### What is version-local between 12 and 17

The two texts are identical on both servers; what differs is around them. Every
row was discovered by the scripts on the running server, not assumed:

| Fact | 17.11 | 12.2 | Consequence for the design |
|---|---|---|---|
| Filed texts execute unmodified | yes | **yes**, exit 0, two baselines written, second run a no-op | the compatibility claim |
| `WITH ... AS MATERIALIZED` | accepted | accepted | the gate's optimization fences |
| `pg_input_is_valid()` | 1 `pg_proc` row | **0** | payload is parsed by regex, not by a guarded cast |
| `MAINTAIN` privilege | exists | `unrecognized privilege type: "MAINTAIN"` | no privilege name may appear in the portable text |
| `deduplicate_items` reloption | accepted | `unrecognized parameter` | four fixtures skipped on 12 |
| B-tree support function 4 | accepted | `invalid function number 4, must be between 1 and 3` | nine fixtures skipped on 12 |
| ICU collations | accepted | `ICU is not supported in this build` | five fixtures skipped on 12 |
| `reltuples` on an uncounted table | `-1` | `0` | the `tuples_unknown` branch fires only from v14; on 12 a zero is ambiguous |
| `pgstatindex` on an invalid index | `index "…" is not valid` | **returns a row** | the `indisvalid` filter protects both, for different reasons |
| `REINDEX` refusal for a non-owner | `permission denied for index` | `must be owner of index` | same outcome, different message |
| `COMMIT` inside `DO` at top level | accepted | accepted | one transaction per index |
| `pgstattuple` extension version | 1.5 | 1.5 | same `pgstatindex(regclass)` signature |
| Index `reltuples` after `CREATE INDEX`, plain and partial | 10000, 2000 | 10000, 2000 | the third gate's baseline is an exact count on both |
| Index `reltuples` after `ANALYZE`, plain and partial | 10000, 2000 | 10000, 2000 | a partial index is estimated from the sample on both |
| Index `reltuples` after a drain and `VACUUM` | 8200, 200 | 8200, 200 | the third gate sees the drain on both |
| `autovacuum_analyze_threshold`, `autovacuum_analyze_scale_factor` | 50, 0.1 | 50, 0.1 | the 10 % test the mandatory tests simulate is the same on both |
| `pg_stat_force_next_flush()` | 1 `pg_proc` row | **0** | the census waits a second and reads from a new session instead |

The v12 side of each source claim is on the v12 pages rather than cited here, as
this page may only cite `raw/postgres-17/`: [How pgstatindex Calculates B-Tree
Index Statistics in PostgreSQL 12
(unverified)](../../../v12/questions/indexing/how-pgstatindex-calculates-information.md),
[All Outcomes That Leave an Invalid Index in PostgreSQL 12, Including a Failed
CREATE INDEX CONCURRENTLY
(unverified)](../../../v12/questions/indexing/invalid-index-outcomes.md),
[Physical Index Statistics, Tuple Counts, and Bytes per Tuple in PostgreSQL 12
(unverified)](../../../v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md),
and [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for
Every Non-B-Tree Index in PostgreSQL 12
(unverified)](../../../v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md).

### The ported mandatory suite

The sibling page's mandatory suite is its numbered fixtures: tests 1-17 (the
deduplication gate), 18-91 (partial indexes), and controls 92-121 (threshold
calibration, non-partial controls, variable-width `INCLUDE`, expression
statistics, the drained queue and the stale-zero shapes). All of them are ported
here, recipe by recipe, and scored against this heuristic instead of against an
estimator. Three things had to change, and all three are deliberate:

1. **Every recipe is split in two.** The originals built one final state and
   asked one question about it; a before-and-after heuristic needs a baseline.
   The build file carries each recipe up to and including the creation of the
   index that is scored, then the filed apply block stores an as-built baseline in
   every comment, and the churn file carries the rest of the recipe.
2. **Fixtures with no churn of their own get a uniform drain**, 90 % of their heap
   blocks followed by `VACUUM` and `ANALYZE`, so that a shape fixture designed for
   a one-shot estimator can also be asked an after question. The fixtures whose
   whole point is that a fresh index must *not* be touched are exempt and stay
   untouched: 70-71, 78-85, 96-97, 101-105, 108-112, 116 and 120.
3. **Every fixture table that moved more than 10 % of its heap tuples is
   analyzed**, between the churn and the decision, because a server with
   autovacuum on would have analyzed it. The trigger is the engine's own; see
   [The simulated auto-analyze the tests now run](#the-simulated-auto-analyze-the-tests-now-run).

The third change costs coverage, and the cost is named here rather than hidden.
Every ported fixture whose point was a stale count had its table analyzed, with
the measured modified share in brackets: 64 `stale statistics after inserts into
the subset` (140 %), 65 `stale statistics after deletes, no VACUUM` (118 %), 69
`stale reltuples, VACUUM but no ANALYZE` (143.9 %), 85 `stale table statistics`
(120 %), 98 `plain index, stale row counts after 300,000 inserts` (160 %), 106
`expression index, no statistics row, 90% deleted` (900 %), 113a and 113c (200 %
and 100 %), 115 `index built on an analysed empty table, then loaded`, 117
`drained, then VACUUM only`, and all three tables of 121 `three ways to leave a
stale row count behind`. Those fixtures now test the heuristic against *fresh*
statistics, which is what a production server with autovacuum on would give it.
The stale-count shapes they used to cover are covered instead by the three forged
fixtures — a table `reltuples = -1`, an index `reltuples = -1`, and fixture 84's
forged partial-index count — because a forgery written after the census is one no
`ANALYZE` can repair. Two fixtures keep a stale count without any forgery: `p119`
inserted 50,000 rows into a 1,050,000-row table (4.8 %) and `pb72` deleted 25,000
of 475,000 (5.3 %), both under the 10 % rule and both left alone.

`pg_stat_force_next_flush()` is still not called anywhere, unlike the original
recipes, because this heuristic reads `pg_class.reltuples` and never a cumulative
statistics view - and the same fixture text then runs on a 12 server, which has no
such function (measured, 0 `pg_proc` rows). The census reads
`n_mod_since_analyze` instead, one second after the churn sessions have exited,
which is what makes their pending statistics visible on both servers.

The scoring is a measured `REINDEX INDEX` on every fixture after the heuristic has
had its turn, which makes `actual_pct` - the reclaim a rebuild of the churned file
really gave back - the only oracle. Verdicts follow the sibling page's bands,
with the thresholds this brief specifies:

| Verdict | Rule |
|---|---|
| `CRITICAL FALSE POSITIVE` | rebuilt, and the rebuild gave back less than 10 % |
| `FALSE POSITIVE` | rebuilt, and the rebuild gave back less than 35 % |
| `FALSE NEGATIVE` | not rebuilt, and a rebuild would have given back 50 % or more |
| `PASS` | everything else |

Each fixture is also checked against the gate arithmetic recomputed
independently from the recorded baseline (`expected_stage`), and against a
prediction filed before the run (`want_stage`).

### Results on 17.11

140 fixtures, 143 B-tree indexes swept (the three extra are the harness's own
primary keys), `make check` 225 of 225 with `pgstattuple` 1, `pageinspect` 8 and
`amcheck` 3, and **0 unexpected server errors**.

| Group | Fixtures | Rebuilt | Updated | Skipped | Refreshed | False negatives | Mean `wasted_pct` | Mean actual |
|---|---|---|---|---|---|---|---|---|
| `gate` (1-17) | 28 | 28 | 0 | 0 | 0 | 0 | 78.6 | 88.4 |
| `partial` (18-77) | 64 | 57 | 5 | 2 | 0 | 2 | 71.9 | 80.2 |
| `falsepos` (78-85) | 8 | 0 | 1 | 7 | 0 | 0 | 0.1 | 0.0 |
| `falseneg` (86-91) | 6 | 6 | 0 | 0 | 0 | 0 | 73.5 | 73.0 |
| `control` (92-112) | 21 | 8 | 1 | 12 | 0 | 0 | 69.9 | 34.3 |
| `zero` (113-121) | 13 | 4 | 6 | 2 | 1 | 2 | 36.0 | 43.5 |

- All 28 deduplication-gate shapes were measured and rebuilt after the 90 % drain,
  and every one was right to rebuild: mean reclaim 88.4 %. The equal-image class
  of the key made no difference to the decision, which is the answer this group
  gives for a heuristic that reads physical density rather than modelling keys.
- **The six false-negative constructions are all caught now.** The previous
  revision skipped all six and lost five of them; the index-count gate opened
  every one, and all six were rebuilt at 40.5 % to 89.9 % measured reclaim.
- The eight false-positive constructions - predicate-conditioned width, NULL,
  `n_distinct` and MCV mismatches, a missing statistics row, a forged partial
  `reltuples`, stale table statistics - are still **not rebuilt**, and a rebuild
  of each would have returned 0.0 %. Seven are skipped outright. The eighth is
  fixture 84, whose forged index count of 5,000 against a real 100,000 now opens
  the gate: it is measured, reads 0.1 % wasted, and is left alone. One wasted
  `pgstatindex()` call is the whole price of a forged count.
- Accuracy of the decision input against the oracle, over the 116 measured
  fixtures: mean error `+10.7` points (the reading under-estimates), min `-5.9`,
  max `+100.0`, 108 of 116 within 15 points, 8 over-estimates. The `+100.0` is
  `p113a`, one of the four unvacuumed files below.
- Which gate opened each measurement: size 9, table count 85, index count 115, and
  26 measured by the index count alone. One fixture, `p76`, is opened by the size
  test alone. No fixture is opened by the table count alone, and no measured
  fixture had no gate fire.
- Payload health after the run: 140 of 140 readable, 0 unparseable, 0 without a
  marker, and 140 of 140 stored index counts equal to the catalog's.

### Results on 12.2

122 fixtures (18 skipped for the missing features listed above), 125 indexes
swept, `make check` 192 of 192 with `pgstattuple` 1, `pageinspect` 5 and `amcheck`
2, and **0 unexpected server errors**.

| Group | Fixtures | Rebuilt | Updated | Skipped | Refreshed | False negatives | Mean `wasted_pct` | Mean actual |
|---|---|---|---|---|---|---|---|---|
| `gate` | 13 | 13 | 0 | 0 | 0 | 0 | 80.1 | 89.9 |
| `partial` | 61 | 54 | 5 | 2 | 0 | 2 | 73.0 | 81.1 |
| `falsepos` | 8 | 0 | 1 | 7 | 0 | 0 | 0.1 | 0.0 |
| `falseneg` | 6 | 6 | 0 | 0 | 0 | 0 | 72.1 | 72.5 |
| `control` | 21 | 8 | 1 | 12 | 0 | 0 | 70.0 | 34.3 |
| `zero` | 13 | 4 | 5 | 3 | 1 | 2 | 39.4 | 43.5 |

The two servers agree on every structural result: the same four false negatives,
the same four fixtures named, 0 false positives, 0 gate disagreements, 25 newly
measured of which 21 bloated, 122 of 122 stored index counts equal to the
catalog's, and an accuracy profile inside a point of the 17 leg's (97 measured,
mean error `+11.0`, min `-0.3`, max `+100.0`, 89 of 97 within 15, 5
over-estimates). The gate attribution is the same shape too: size 9, table count
67, index count 95, 25 by the index count alone, and again exactly one fixture
(`p76`) opened by the size test alone. What differs is the physical size of the
same fixture: the
duplicate-heavy control index `e_am_btree` has 547 leaf pages on 12.2 against 173
on 17.11, because v13 deduplication compresses its 97 distinct keys over 200,000
rows. The decision is unmoved by that - it is a filter control, initialized and
then left alone on both servers - but a reader
comparing absolute sizes across a `pg_upgrade` boundary should expect the file to
be several times larger before the upgrade, and the first post-upgrade baseline to
be written against an index that cannot deduplicate until it is rebuilt; see
[Checking Whether an Index Needs a Rebuild to Enable Deduplication After
pg_upgrade From PostgreSQL 12 to 17
(unverified)](btree-deduplication-after-pg-upgrade.md).

### The four false negatives that remain

All four are on both servers, and all four are `lost_by = threshold`, not
`lost_by = gate`: the index-count test opened every one of them, `pgstatindex` was
called, and the number it returned was 0.0 % or 0.1 % on a file whose rebuild gave
back 89.1 % to 100.0 %. The 17.11 rows, verbatim from `$OUT/lost.txt`:

```text
 num | leg |  idx  | action | wasted_pct | actual_pct |  lost_by  | size_ratio | tuple_ratio | idx_tuple_ratio
 113 | a   | p113a | update |        0.0 |      100.0 | threshold |     1.0000 |      1.0000 |          0.0000
 113 | c   | p113c | update |        0.0 |      100.0 | threshold |     1.0000 |      1.0000 |          0.0000
  65 |     | p65   | update |        0.1 |       89.1 | threshold |     1.0000 |      0.8200 |          0.0995
  67 |     | p67   | update |        0.1 |       89.1 | threshold |     1.0000 |      1.0000 |          0.0948
```

One property is common to all four and absent from every fixture that passed:
**no `VACUUM` ran after the rows left the index**. `p113a` and `p113c` moved every
row of a 1,000,000-row queue from `pending` to `done`, one with an `ANALYZE` and
one without; `p65` deleted 98 % of a subset; `p67` updated rows out of the
predicate. The index entries are all still physically there, marked dead but not
removed, so `avg_leaf_density` is high, `empty_pages` and `deleted_pages` are
zero, and the wasted-space formula has nothing to add up. The rebuild then writes
an index holding almost nothing, which is where the 89-100 % comes from.

That is a known property of the measurement rather than of the gate, and the
sibling page measured the same thing from the other direction: its estimator
under-read a deleted-but-not-vacuumed index by 90 points
([B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17
(unverified)](btree-bloat-with-pgstatindex.md)). Nothing in `pgstatindex`'s output
distinguishes a dead entry from a live one, so no threshold on its numbers can
catch these four. What *would* catch them is a fourth input the brief does not
have: the index's entry count against its own leaf-page capacity, or
`pg_stat_all_tables.n_dead_tup` on the table. Both are filed as open questions.

### Filed predictions against measured verdicts

A `want_stage` was filed for every fixture before the first run, and it was left
untouched for this one, because a prediction rewritten after the fact proves
nothing. On 17.11 it is now right for **129 of 140** against 119 before, and on
12.2 for **112 of 122** against 100. Every previous miss of the form "predicted
`measure`, measured `skip`" is gone: all 20 of them now measure.

The direction of the remaining misses has flipped. All 11 on 17.11, and all 10 on
12.2, predicted `skip` and got `measure`, which is the third gate doing its job on
fixtures whose predictions were written when only two gates existed:

| Prediction | Fixtures | What the measurement then decided |
|---|---|---|
| `skip`, measured `reindex` | `p29`, `p113b`, `p114`, `p117` | 77.6 % to 99.9 % wasted, 87.4 % to 100.0 % actually reclaimed - correct rebuilds |
| `skip`, measured `update` | `p65`, `f84`, `p113a`, `p113c`, `p118`, `p119`, `p120` | 0.0 % to 5.9 % wasted; three are unvacuumed false negatives, the other four are healthy indexes correctly left alone |

The 12.2 leg is the same list without `p120`, and the difference is instructive:
`p120` is the fixture whose 2,000-row subset a 300-row sample can miss, and on
17.11 the census re-analyzed its table and the partial index's *estimated* count
came back at 0.1941 of the baseline, which fires the gate on sampling noise
alone. On 12.2 the fixture's own `ANALYZE` was the last writer, the count stayed
at 1.0000, and the row was skipped. The consequence is one `pgstatindex()` call
on a healthy index, which read 5.9 % wasted and was left alone, and it is the
clearest single example of what the third gate costs. No prediction is now wrong
in the expensive direction: nothing predicted `measure` was skipped.

### Comment parsing and preservation

Fifteen comment shapes, each read and then rewritten by the filed texts. The two
new rows are the migration cases the payload change creates:

| Case | Comment before | `baseline` | Action | Comment after |
|---|---|---|---|---|
| absent | none | `absent` | `initialize` | 86-byte payload |
| human text only | `human note, kept verbatim` | `absent` | `initialize` | note, newline, payload; 112 bytes |
| empty `sz` | `@btmaint:{"v":2,"sz":,"tup":1,"itup":1}` | `invalid` | `initialize` | 86 bytes, nothing left over |
| wrong version | `…{"v":9,…}` | `invalid` | `initialize` | 86 bytes |
| non-numeric `sz` | `…"sz":"big"…` | `invalid` | `initialize` | 86 bytes |
| truncated payload | `@btmaint:{"v":2,"sz":1` | `invalid` | `initialize` | 86 bytes |
| marker without JSON | `@btmaint:not json at all` | `invalid` | `initialize` | 86 bytes |
| missing `sz` key | `…{"v":2,"tup":1,"itup":1}` | `invalid` | `initialize` | 86 bytes |
| 32-digit `sz` | `…"sz":999…999,…` | `invalid` | `initialize` | 86 bytes |
| **a v1 payload** | `@btmaint:{"v":1,"sz":1,"tup":1}` | `invalid` | `initialize` | 86 bytes |
| **missing `itup` key** | `@btmaint:{"v":2,"sz":1,"tup":1}` | `invalid` | `initialize` | 86 bytes |
| marker mid-string | `@btmaint:{…} and text after it` | `ok` | `update` | ` and text after it`, newline, payload; 104 bytes |
| two markers | two payload lines | `ok` | `update` | one payload; 86 bytes |
| quoting | quote, backslash, per cent, newline | `ok` | `update` | human text byte-identical, verified by equality on the exact original |
| 8,000-character note | 8,000 `L` | `ok` | `update` | 8,087 bytes, the 8,000 `L` intact |

The `invalid` reading is deliberately generous: a comment that contains the marker
at all but yields no parseable payload is `invalid`, not `absent`, so the reader
sees `unreadable @btmaint: payload, replaced` rather than a silent
re-initialization. All fifteen behave identically on 12.2, including both
migration cases.

### Idempotence, dry runs and dumps

- **Three consecutive runs** of step 2 on the same database: the first acted on 35
  indexes (`reindex=5 update=14 initialize=16` on 17.11, `reindex=5 update=14
  initialize=14` on 12.2), the second and third reported
  `reindex=0 update=0 initialize=0 refresh=0 blocked=0 failed=0`. A settled
  database is not written to at all - no comment churn, no WAL, no locks beyond
  the read. That the second run writes nothing is also the proof that the payload
  it wrote on the first run is one it can read back: an unreadable `itup` would
  re-initialize every index on every run.
- **`dry_run := true`** reports the same 35 decisions, including the five rebuilds
  it would run, and the md5 of every `pg_class` comment in the database is
  unchanged afterwards.
- **`pg_dump`** of one fixture table emitted 15 `COMMENT ON INDEX` statements, all
  15 carrying the payload. The baseline therefore survives dump and restore, which
  cuts both ways: restoring an old dump restores an old baseline, and the first run
  after the restore will compare today's index against it. Rule 3 catches the
  shrink direction; a stale-low baseline is caught only by the three growth tests.

### Mandatory test review

| Group | Tests | Fixtures and oracle | 17.11 | 12.2 |
|---|---|---|---|---|
| Deduplication gate | 1-17 | 28 indexes on two 500,000-row tables, drained 90 %; measured `REINDEX INDEX` | run, 28 of 28 `PASS` | run, 13 of 13 `PASS`, 15 skipped for missing features |
| Partial indexes | 18-77 | 64 indexes over 58 tables, each with its own churn or the uniform drain | run, 62 `PASS`, 2 `FALSE NEGATIVE` | run, 59 `PASS`, 2 `FALSE NEGATIVE` |
| False-positive constructions | 78-85 | 8 freshly built indexes that must not be touched | run, 8 of 8 left un-rebuilt (7 skipped, 1 measured and updated) | the same |
| False-negative constructions | 86-91 | 6 genuinely bloated, vacuumed and analysed | run, 6 of 6 `PASS`, all rebuilt | the same |
| Change A-D controls | 92-112 | threshold calibration 92-95, non-partial 96-99, `INCLUDE` 100-105, expression statistics 106-112 | run, 21 of 21 `PASS` | the same |
| Drained queue and change E | 113a-c, 114-121 | one 1,000,000-row table per state, `reltuples = 0` shapes | run, 11 `PASS`, 2 `FALSE NEGATIVE` | the same |
| Simulated auto-analyze | the new rule | every fixture table, gated on the engine's own threshold | run, 77 of 99 analyzed, boundary at 9.8 % / 11.1 % | run, 57 of 97 analyzed, same boundary |
| This heuristic's acceptance fixtures | - | 15 comment shapes, 11 filter shapes, 10 gate boundaries, 9 curve points, survival, privileges, locks, dry run, idempotence, dump | run | run |
| Engine regression | `make check` plus `pgstattuple`, `pageinspect`, `amcheck` | temporary installation in the build tree | 225, 1, 8, 3 - all passed | 192, 1, 5, 2 - all passed |
| Repository checks | `scripts/wiki_lint`, block hashes, pipeline identity, Contents anchors | this repository | pass | pass |

The contract the sibling page set is that a statement failing a mandatory test is
corrected, not merely reported. Two defects were found and corrected in the filed
texts of the first revision; see
[Two defects the first revision's suite found](#two-defects-the-first-revisions-suite-found).
This revision corrects the largest reported failure of that revision rather than
re-reporting it: 21 of the 21 gate losses are closed. The four that remain are
**not** corrected, because correcting them needs an input the brief does not
specify and `pgstatindex` does not expose; they are reported, quantified, and
filed as the first open question.

### What still needs to be tested

1. **A fourth input for the unvacuumed case.** The four remaining false negatives
   are files full of dead index entries, which `pgstatindex` reports as dense.
   Neither a stored entry count against leaf-page capacity nor
   `pg_stat_all_tables.n_dead_tup` has been tried as a further gate or as a
   correction to `wasted_pct`.
2. **Concurrent rebuilds end to end.** `REINDEX INDEX CONCURRENTLY` is measured
   only for comment survival. A run where step 1 emits the concurrent command, an
   operator executes it at the top level, and the next run stores the baseline
   through rule 3, has not been scored as a loop.
3. **A second block size.** Every geometry constant is derived from
   `block_size`, and both legs ran at 8192 with `max_data_alignment` 8. A
   `--with-blocksize=16` build has not been tried.
4. **13, 14, 15 and 16.** The claim is "12 through 17" and the measured legs are
   12.2 and 17.11. The intermediate majors are covered only by the feature facts
   the two legs discovered, not by a run.
5. **A non-C locale and a non-UTF8 encoding**, and an ICU-enabled 12 build, which
   would let the five ICU fixtures run on both legs instead of one.
6. **Two sessions racing on one index.** The lock mode is measured; the behaviour
   of two overlapping step-2 runs, and of a step-2 run against a concurrent
   `DROP INDEX`, is not.
7. **A partitioned table end to end.** Leaf indexes are candidates and the
   partitioned index is excluded, both measured, but no fixture drains a partition
   and watches the parent.
8. **Standby behaviour.** The `relpersistence <> 'u' OR NOT pg_is_in_recovery()`
   filter has never been exercised on a real standby, and no part of the heuristic
   can write a comment there.
9. **How often sampling noise opens the third gate.** `p120` is one measured
   example, at `idx_tuple_ratio` 0.1941 on an index whose entry count had not
   really moved. The suite has no fixture family that sweeps subset selectivity
   against `default_statistics_target` to find where a sample-based count becomes
   noisy enough to cost regular measurements.
10. **A run with autovacuum actually on.** The mandatory tests simulate the
    auto-analyze; they do not let the launcher do it. A cluster with
    `autovacuum = on` would also vacuum, which would change what `pgstatindex`
    sees on exactly the four fixtures that still fail, and is the obvious next
    experiment.

## Measurement Script

Every number on this page comes from the two scripts below: one per version leg,
Bash and SQL only, each self-contained apart from the two texts it takes out of
this page.

| Item | This suite |
|---|---|
| Purpose | Build the pinned checkout, run the engine regression suites, start an isolated cluster, take this page's two texts out of this page, port every numbered fixture of the sibling page's mandatory suite, store an as-built baseline in every index comment, churn each fixture, simulate the auto-analyze the churn would have triggered, apply the catalog forgeries an `ANALYZE` would have repaired, run the heuristic, and score every decision against a measured `REINDEX INDEX`. It also measures this heuristic's own surface: comment parsing and preservation, the candidate filters, all three gate boundaries, the 40 % decision curve, privileges, locks, dry runs, idempotence, dumps, and cost. Every figure in [Verdict](#verdict) through [Idempotence, dry runs and dumps](#idempotence-dry-runs-and-dumps) is one of its outputs. |
| Invocation | From the repository root: `bash btmaint_suite_v17.sh` and `bash btmaint_suite_v12.sh`. Selected stages: `bash btmaint_suite_v17.sh suite score`. |
| Stages | 17 leg: `build check cluster sql texts facts suite edge cost score criteria report`, in that default order, plus `stop` and `clean`. The 12 leg inserts `exact` between `texts` and `facts`. The simulated auto-analyze and the forgeries are steps inside `suite` and `edge`, not stages of their own, because a decision taken without them would be a decision on statistics no production server would have had. |
| Environment | `WIKI_ROOT` (default `$PWD`), `PAGE` (default this page), `SRC` (default `raw/postgres-17` or `raw/postgres-12`), `SANDBOX` (default `.wiki-runtime/tmp/btmaint`), `PORT` (55417 / 55412), `JOBS` (8). |
| Prerequisites | A C toolchain, `make`, and the two pinned checkouts. The 17 leg configures `--with-icu`, the 12 leg `--without-icu`. `pgstattuple` is installed from the same build. No installed PostgreSQL is used. |
| Output | Everything under `$SANDBOX/out` (17) and `$SANDBOX/out12` (12). Read `criteria.txt` first; then `counters.txt`, `verdicts.txt`, `lost.txt`, `newly_measured.txt`, `autoanalyze.txt`, `cost.txt`, `facts.txt`, and the `edge_*.txt` files. |
| Runtime | Measured from each invocation's own log file, first write to last, on 22 cores at `JOBS=16` with `fsync = off` in the sandbox cluster: the 17 leg's `build check cluster sql` took 1 min 10 s and its `cluster` through `report` re-run 1 min 43 s; the 12 leg's `build` through `facts` took 1 min 10 s, `suite` 1 min 17 s, and `edge` through `report` 16 s. A re-run from a built tree therefore needs about two minutes per leg. |
| Cleanup | `bash btmaint_suite_v17.sh clean` and `bash btmaint_suite_v12.sh clean`: each stops its own server with `pg_ctl -m fast -w stop`, confirms no `postmaster.pid`, no matching process and an empty socket directory, and only then deletes the sandbox. Both were run before this page was filed. |

### How to use the two leg scripts

Save each block to a file of the name in its first comment line, in the
repository root, and run it from there. Nothing outside `$SANDBOX` is written, and
the pinned checkouts are read only: both builds are VPATH builds in directories
under `.wiki-runtime/tmp/`.

The scripts take the two texts under test out of this page, by fence order:
block 1 is [Step 1: the plan](#step-1-the-plan) and block 2 is
[Step 2: carrying it out](#step-2-carrying-it-out). Each is hashed against a
baseline recorded in the script, so editing this page's SQL without re-measuring
is visible in the output:

```text
report 93b64e2dd33d411951ceffd9d665a3e3e8ce3a71fde60bbca9e61c219febd0e9 match
apply  7427d62d2ca3bb43dd2f7c5a8aa97c27fc8111d65d3066594660c2808d47992d match
report lines=243 bytes=13167
apply  lines=267 bytes=13028
pipeline report 39a2e57379dde00ba6eb64edb38b1d81d68f594aa349e2bc3cdb0aafbdc4ef02
pipeline apply  39a2e57379dde00ba6eb64edb38b1d81d68f594aa349e2bc3cdb0aafbdc4ef02
pipeline lines  118
pipeline identical yes
```

Both legs printed those eight lines. The shared pipeline grew from 100 lines to
118 with the third gate, and is still byte-identical in the two texts.

`stage_texts` also builds the one harness object the suite needs from step 1: a
view over the filed statement, with exactly one documented edit - the two `SET`
lines dropped, because a view cannot carry them. Nothing else is touched, so the
view decides exactly what the filed statement decides, and the scorer reads
decisions without re-typing them.

### The stages, both legs

| Stage | What it does |
|---|---|
| `build` | Configures and builds the pinned checkout out of tree, installs it plus `pgstattuple`, `pageinspect` and `amcheck` under `$SANDBOX/inst/NN`. Skipped if the binary is already there. |
| `check` | `make check` and the three contrib suites, with the pass counts and any `regression.diffs` copied where `clean` will not delete them. |
| `cluster` | `initdb --locale=C --encoding=UTF8`, writes the cluster settings, starts the server, records `version()` and `pg_control_init()`, creates the `suite` and `edge` databases with `pgstattuple`. Idempotent. |
| `sql` | Writes every SQL file the suite uses: the harness, the fixture build and churn files, the PostgreSQL 13 and ICU fixture files, the simulated auto-analyze census, the forgery files, and the edge fixtures. |
| `texts` | Extracts, hashes and line-counts the two texts, checks that their shared pipeline region is byte-identical, and builds the one-edit view. |
| `exact` (12 leg) | Executes both filed texts verbatim on this server before any fixture exists, and records the outcome, the comment written, and what a second run does. |
| `facts` | Discovers every version-local fact the texts depend on, on the running server, including who writes an index's `reltuples` and the two auto-analyze defaults. |
| `suite` | Builds the numbered fixtures, runs the filed apply block to store as-built baselines, churns, **runs the simulated auto-analyze and then the catalog forgeries**, records the filed report's decisions, runs the filed apply block again to act, then rebuilds every fixture as the oracle. It dies rather than continue if no table crossed the auto-analyze threshold, because that would mean the churn counters were invisible. |
| `edge` | Builds and scores this heuristic's own fixtures: 15 comment shapes, 11 filter shapes, 10 gate boundaries, the 9-point curve, comment survival across both `REINDEX` forms, privileges including `MAINTAIN` where it exists, the lock `COMMENT` takes, a dry run, three consecutive runs, and a `pg_dump`. It applies the same auto-analyze rule and then re-applies its three forgeries. |
| `cost` | Settles the database with one apply run, times the filed report six times and reads its buffer counts, then halves every stored `sz`, leaving both counts current, so the size gate alone fires everywhere, and repeats. |
| `score` | Writes the verdict rows, the verdict and action counts, the per-group table, the accuracy row, the gate-agreement row, the per-gate attribution, the two-gate counterfactual, the newly-measured detail, the stored-index-count check, the lost-fixture detail, and the payload-health row. |
| `criteria` | Collects the pass criteria into one file and audits the server log for any error the suite did not deliberately provoke. |
| `report` | Lists the output directory. |
| `stop` | `pg_ctl -m fast -w stop`, then confirms the teardown and dies rather than report a stop that did not happen. |
| `clean` | `stop`, then refuses to delete anything outside `.wiki-runtime/tmp/` before removing the sandbox. |

### What the scripts read from the environment

| Variable | Default | Effect |
|---|---|---|
| `WIKI_ROOT` | `$PWD` | Repository root; also the containment boundary `clean` checks. |
| `PAGE` | `wiki/v17/questions/indexing/btree-comment-baseline-maintenance-heuristic.md` | Where the two texts come from. |
| `SRC` | `$WIKI_ROOT/raw/postgres-17` or `raw/postgres-12` | The pinned checkout to build. |
| `SANDBOX` | `$WIKI_ROOT/.wiki-runtime/tmp/btmaint` | Everything written lives here. |
| `PORT` | 55417 (17), 55412 (12) | Non-default port; the socket directory is inside the sandbox. |
| `JOBS` | 8 | `make -j`. |

Cluster settings and their apply scope, all written before the first start:
`listen_addresses`, `port`, `unix_socket_directories`, `shared_buffers` and
`logging_collector` are `PGC_POSTMASTER`, so they need a restart and are set
before one; `fsync` and `autovacuum` are `PGC_SIGHUP`, so a reload;
`maintenance_work_mem` and `max_parallel_maintenance_workers` are `PGC_USERSET`,
session scope. `autovacuum` is off so that no background vacuum moves a fixture
between the baseline, the churn, the decision and the oracle.

Both scripts mark their fixture statements as disposable. They create, forge and
drop tables, indexes, operator classes, collations, roles and catalog rows in the
sandbox cluster's own databases, including four deliberate `pg_class` and
`pg_index` forgeries, and are not meant for a database anyone cares about. The
forgeries are applied *after* the simulated auto-analyze, in `forge.sql` and
`edge_forge.sql`, because an `ANALYZE` of the table rewrites `reltuples` for the
table and for every index on it, which would repair exactly the state those
fixtures exist to create.

### Prerequisites

- A C toolchain and `make`. Both legs build their own server.
- Development headers for ICU for the 17 leg; the 12 leg is configured
  `--without-icu` and records its five ICU fixtures as skipped.
- The two pinned checkouts at `raw/postgres-17` and `raw/postgres-12`.
- `sha256sum`, `cmp`, `diff`, `grep`, `sed`, `cut`, `tr`, `wc` and `pgrep`.

### Last run

| Item | 17 leg | 12 leg |
|---|---|---|
| Date | 2026-09-11 | 2026-09-11 |
| Server version | 17.11 | 12.2 |
| Pin | `786db8dcf168bd9df8f55047337525ac19118b1c` | `45b88269a353ad93744772791feb6d01bc7e1e42` |
| `database_block_size` | 8192 | 8192 |
| `max_data_alignment` | 8 | 8 |
| Platform | Linux x86_64, gcc 13.3.0 | Linux x86_64, gcc 13.3.0 |
| Locale, encoding | C, UTF8 | C, UTF8 |
| `autovacuum_analyze_threshold`, `autovacuum_analyze_scale_factor` | 50, 0.1 | 50, 0.1 |
| Engine tests | 225 core, `pgstattuple` 1, `pageinspect` 8, `amcheck` 3 | 192 core, `pgstattuple` 1, `pageinspect` 5, `amcheck` 2 |
| Unexpected server errors | 0 | 0 |
| Text hashes | `93b64e2d…` report, `7427d62d…` apply | the same two |
| Script SHA-256 | `c2f99506a492395a17b7cbb66209974c34b23e38c9ca0dc214867ef6fd8f0889` | `7318192e8a8bb95b0ba8f2f354fd708bf017807cfe647208b1f9fbdee6272606` |

Every figure on this page comes from that pair of runs, taken with the script text
published below: the 12 leg in one pass from `build` to `report`, and the 17 leg
re-run in one pass from `cluster` to `report` after its `build` and `check` so
that no number predates the final script. Both servers were then stopped by the
scripts' own `stop` stage and the 12 GB sandbox deleted by `clean`, with no
`postmaster.pid`, no matching process and an empty socket directory confirmed for
each.

### The PostgreSQL 17 leg script

```bash
#!/usr/bin/env bash
#
# btmaint_suite_v17.sh - the whole test suite of the PostgreSQL 17 wiki page
# "A COMMENT-Stored Baseline B-Tree Index-Maintenance Heuristic for PostgreSQL
# 12 Through 17", in bash and SQL only.
#
# It builds 17.11 out of tree from the pinned checkout, runs the engine
# regression suites, starts an isolated cluster, takes the page's two texts out
# of the page itself, ports every numbered fixture of the sibling page's
# mandatory suite (tests 1-17, 18-91 and controls 92-121), stores an as-built
# baseline in each index comment, churns each fixture, simulates the auto-analyze
# that churn would have triggered on a server with autovacuum on, runs the
# heuristic, and scores every decision against a measured REINDEX INDEX.  A
# second group of fixtures covers this heuristic's own surface: comment parsing,
# comment preservation, the candidate filters, all three gate boundaries and the
# 40 % decision curve.
#
# The simulated auto-analyze is the mandatory-test rule that any fixture moving
# more than 10 % of a table's heap tuples must ANALYZE it.  It is not
# hand-annotated per fixture: stage_autoanalyze reads the engine's own
# n_mod_since_analyze counter and applies the engine's own threshold,
# autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * reltuples
# (50 + 0.1 * reltuples at the defaults), so the tables it analyzes are exactly
# the tables an autovacuum launcher would have analyzed.  Every table it
# considered, with its counter, its threshold and the verdict, is recorded.
#
# The pinned checkout is read only: everything this script writes lives under
# $SANDBOX (default .wiki-runtime/tmp/btmaint).
#
# Usage, from the repository root:
#   bash btmaint_suite_v17.sh                  # every stage, in order
#   bash btmaint_suite_v17.sh suite score      # selected stages
#   bash btmaint_suite_v17.sh clean            # stop and delete the sandbox
#
# Stages: build check cluster sql texts facts suite edge cost score criteria
#         report stop clean
#
# Environment: WIKI_ROOT PAGE SRC SANDBOX PORT JOBS
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/indexing/btree-comment-baseline-maintenance-heuristic.md}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-17}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/btmaint}"
PORT="${PORT:-55417}"
JOBS="${JOBS:-8}"

BUILD="$SANDBOX/build/17"; INST="$SANDBOX/inst/17"; DATA="$SANDBOX/data17"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql"; SOCK="$SANDBOX/sock"; BIN="$INST/bin"
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE=postgres

# SHA-256 baselines of the two fenced SQL blocks of the page, in page order:
# the report statement and the apply block.  A changed text must be re-measured
# and the hash refiled; that is the point of recording them here.
BASE_REPORT=93b64e2dd33d411951ceffd9d665a3e3e8ce3a71fde60bbca9e61c219febd0e9
BASE_APPLY=7427d62d2ca3bb43dd2f7c5a8aa97c27fc8111d65d3066594660c2808d47992d

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
# err() runs a statement that is expected to fail and prints the message only.
err() { "$BIN/psql" -X -q -v ON_ERROR_STOP=0 -d "$1" -c "$2" 2>&1 \
        | grep -E '^(ERROR|psql:.*ERROR)' | head -1; }
# errf() sends a statement through a file, so that a body full of quotes and
# dollar signs reaches the server exactly as written, and reports the first
# error line or "accepted".
errf() {
  local db=$1 out
  out=$(printf '%s\n' "$2" | "$BIN/psql" -X -q -v ON_ERROR_STOP=0 -d "$db" -f - 2>&1 \
        | grep -E 'ERROR' | head -1)
  printf '%s' "${out:-accepted}"
}

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

# plan_view <report-text> : the one documented edit.  Drop the two SET lines,
# which a view cannot carry, and wrap the rest in CREATE VIEW plan_v.  Nothing
# else is touched, so the view computes exactly what the filed statement does.
plan_view() {
  local file=$1 line
  printf 'DROP VIEW IF EXISTS plan_v;\nCREATE VIEW plan_v AS\n'
  while IFS= read -r line; do
    case $line in
      "SET /* wiki_btmaint_statement_timeout"*) continue ;;
      "SET /* wiki_btmaint_lock_timeout"*)      continue ;;
    esac
    printf '%s\n' "$line"
  done < "$file"
}

# ---------------------------------------------------------------- build ------
stage_build() {
  say "build 17.11 out of tree from $SRC"
  if [ -x "$BIN/postgres" ]; then
    note "already built: $("$BIN/postgres" --version)"; return 0
  fi
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC"
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --with-icu \
      > configure.log 2>&1 ) || { cp "$BUILD/configure.log" "$OUT/" 2>/dev/null
                                  die "configure failed, see $OUT/configure.log"; }
  ( cd "$BUILD" && make -j"$JOBS" -s > make.log 2>&1 \
      && make -s install > install.log 2>&1 ) \
    || { cp "$BUILD"/*.log "$OUT/" 2>/dev/null; die "make failed"; }
  local m
  for m in pgstattuple pageinspect amcheck; do
    ( cd "$BUILD" && make -C "contrib/$m" -s install >> install.log 2>&1 ) \
      || die "contrib/$m install failed"
  done
  cp "$BUILD"/configure.log "$BUILD"/make.log "$BUILD"/install.log "$OUT/" 2>/dev/null
  note "$("$BIN/postgres" --version)"
}

# ---------------------------------------------------------------- check ------
stage_check() {
  say "engine regression suites"
  : > "$OUT/checks.txt"
  ( cd "$BUILD" && make -s check > check_core.log 2>&1 )
  printf 'core=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' \
        "$BUILD/check_core.log" | tail -1)" >> "$OUT/checks.txt"
  local m
  for m in pgstattuple pageinspect amcheck; do
    ( cd "$BUILD" && make -s -C "contrib/$m" check > "check_$m.log" 2>&1 )
    printf '%s=%s %s\n' "$m" "$?" \
      "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' \
          "$BUILD/check_$m.log" | tail -1)" >> "$OUT/checks.txt"
  done
  cp "$BUILD"/check_*.log "$OUT/" 2>/dev/null
  local d
  for d in "$BUILD/src/test/regress" "$BUILD"/contrib/*; do
    [ -f "$d/regression.diffs" ] \
      && cp "$d/regression.diffs" "$OUT/diffs_$(basename "$d").txt"
  done
  cat "$OUT/checks.txt" >&2
}

# ---------------------------------------------------------------- cluster ----
# Cluster settings and their apply scope, all written before the first start:
#   listen_addresses, port, unix_socket_directories, shared_buffers,
#   logging_collector                                  -> PGC_POSTMASTER, restart
#   fsync, autovacuum                                  -> PGC_SIGHUP, reload
#   maintenance_work_mem, max_parallel_maintenance_workers
#                                                      -> PGC_USERSET, session
# autovacuum is off so that no background vacuum moves a fixture between the
# baseline, the churn, the decision and the rebuild oracle.
stage_cluster() {
  say "isolated cluster on port $PORT"
  mkdir -p "$OUT" "$SQLD" "$SOCK"
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    note "already running"
  else
    if [ ! -d "$DATA" ]; then
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
logging_collector = off
CONF
    fi
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w start > /dev/null \
      || die "server start failed"
  fi
  note "$(s postgres 'SELECT /* wiki_btmaint_version */ version()')"
  s postgres "SELECT /* wiki_btmaint_platform */
                     'max_data_alignment=' || max_data_alignment ||
              ' database_block_size=' || database_block_size FROM pg_control_init()" \
    | tee "$OUT/platform.txt" >&2
  printf 'uname: %s\n' "$(uname -sm)" >> "$OUT/platform.txt"
  local db
  for db in suite edge; do
    s postgres "SELECT /* wiki_btmaint_database_exists */ 1
                  FROM pg_database WHERE datname = '$db'" | grep -q 1 \
      || "$BIN/createdb" -T template0 -E UTF8 --locale=C "$db"
    q "$db" 'CREATE EXTENSION IF NOT EXISTS pgstattuple' || die "pgstattuple failed"
  done
}

# ------------------------------------------------------------------ sql ------
# Every SQL file this suite uses, written out from here so that the script is
# self-contained.  The two texts under test are NOT here: they come out of the
# page itself, in stage_texts.
stage_sql() {
  say "write the suite's SQL files to $SQLD"
  mkdir -p "$SQLD"
  cat > "$SQLD/harness.sql" <<'HARNESS'
-- Harness for the ported numbered suite.  Disposable: every object below is
-- created in the sandbox cluster's suite database and is not meant for a
-- database anyone cares about.  The harness never touches the heuristic's own
-- two texts; it only records, snapshots and scores.
SET /* wiki_btmaint_harness_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_harness_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_harness_lock_timeout */ lock_timeout = '5s';

DROP TABLE IF EXISTS plan CASCADE;
DROP TABLE IF EXISTS snap CASCADE;
DROP TABLE IF EXISTS truth CASCADE;
DROP VIEW IF EXISTS verdicts CASCADE;

-- One row per numbered fixture index.
CREATE TABLE plan(num int, leg text DEFAULT '', grp text, req text, idx text,
                  want_stage text, note text,
                  PRIMARY KEY (num, leg));

-- One row per index per phase.  phase is 'built', 'init', 'churned', 'applied'.
-- idx_tuples is the index's own pg_class.reltuples, which the heuristic now
-- reads as its third gate input; base_idx_tuples is the itup field of the
-- stored payload, parsed here independently of the filed text.
CREATE TABLE snap(phase text, idx text, idx_oid oid, bytes bigint,
                  tbl_tuples numeric, idx_tuples numeric, cmt text, payload text,
                  base_bytes numeric, base_tuples numeric, base_idx_tuples numeric,
                  PRIMARY KEY (phase, idx));

-- What the heuristic decided, and what a rebuild actually gave back.
CREATE TABLE truth(idx text PRIMARY KEY, action text, baseline text,
                   wasted_pct numeric, notes text,
                   bytes_churned bigint, bytes_applied bigint, bytes_fresh bigint,
                   reindexed_by_heuristic bool, cmd_report text, cmd_written text);

CREATE OR REPLACE FUNCTION plan_add(n int, g text, r text, i text,
                                    w text DEFAULT NULL, lg text DEFAULT '',
                                    nt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS
$$ INSERT INTO plan(num, leg, grp, req, idx, want_stage, note)
   VALUES (n, lg, g, r, i, w, nt) $$;

-- take_snap records, for every planned index, the facts the heuristic reads:
-- the file size, the table's estimated tuple count, and the baseline stored in
-- the index's own comment.  It parses the payload the same way the filed text
-- does, independently, so a disagreement is visible.
CREATE OR REPLACE PROCEDURE take_snap(ph text) LANGUAGE plpgsql AS $sn$
BEGIN
  DELETE FROM snap WHERE phase = ph;
  INSERT INTO snap
  SELECT ph, p.idx, c.oid, pg_relation_size(c.oid), t.reltuples::numeric,
         c.reltuples::numeric, d.description,
         substring(d.description from '@btmaint:(\{[^}]*\})'),
         substring(d.description from '"sz":([0-9]{1,25})[,}]')::numeric,
         substring(d.description from
                   '"tup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric,
         substring(d.description from
                   '"itup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric
    FROM plan p
    JOIN pg_class c ON c.relname = p.idx AND c.relkind = 'i'
    JOIN pg_index x ON x.indexrelid = c.oid
    JOIN pg_class t ON t.oid = x.indrelid
    LEFT JOIN pg_description d ON d.objoid = c.oid
                              AND d.classoid = 'pg_class'::regclass
                              AND d.objsubid = 0;
END $sn$;

-- ground_truth rebuilds every planned index and records the fresh size.  It is
-- the oracle: what REINDEX INDEX actually gives back on the churned file.
CREATE OR REPLACE PROCEDURE ground_truth() LANGUAGE plpgsql AS $gt$
DECLARE p record;
BEGIN
  FOR p IN SELECT idx FROM plan ORDER BY num, leg LOOP
    EXECUTE format('REINDEX /* wiki_btmaint_oracle */ INDEX %I', p.idx);
    UPDATE truth SET bytes_fresh = pg_relation_size(p.idx::regclass)
     WHERE idx = p.idx;
  END LOOP;
END $gt$;

-- The verdict view. actual_pct is what the rebuild of the churned file gave
-- back, measured, and is the only oracle. expected_stage recomputes the gate
-- from the recorded baseline, independently of the filed text.
CREATE OR REPLACE VIEW verdicts AS
SELECT p.num, p.leg, p.grp, p.idx, p.req, p.want_stage,
       t.action, t.baseline, t.wasted_pct, t.reindexed_by_heuristic,
       bu.bytes AS bytes_built, ic.bytes AS bytes_init,
       t.bytes_churned, t.bytes_applied, t.bytes_fresh,
       round(100.0 * (t.bytes_churned - t.bytes_fresh)
             / GREATEST(t.bytes_churned, 1), 1)                  AS actual_pct,
       round(100.0 * (t.bytes_churned - t.bytes_applied)
             / GREATEST(t.bytes_churned, 1), 1)                  AS applied_pct,
       ic.base_bytes AS base_bytes, ic.base_tuples AS base_tuples,
       ch.tbl_tuples AS churned_tuples,
       CASE WHEN ic.base_bytes > 0
            THEN round(ch.bytes / ic.base_bytes, 4) END          AS size_ratio,
       CASE WHEN ic.base_tuples > 0
            THEN round(ch.tbl_tuples / ic.base_tuples, 4) END    AS tuple_ratio,
       -- The index's own entry count: the baseline the payload stored, the
       -- count the catalog holds after the churn, and their ratio.  The
       -- payload value is what the heuristic reads; ic.idx_tuples is the live
       -- catalog value at baseline time, so the two must agree.
       ic.base_idx_tuples, ch.idx_tuples AS churned_idx_tuples,
       (ic.base_idx_tuples = round(ic.idx_tuples))               AS itup_stored_matches,
       CASE WHEN ic.base_idx_tuples > 0
            THEN round(ch.idx_tuples / ic.base_idx_tuples, 4) END AS idx_tuple_ratio,
       -- Which of the three gates fires, each recomputed here from the
       -- recorded baseline, independently of the filed text.
       (ic.base_bytes > 0 AND ch.bytes >= ic.base_bytes * 1.20)  AS size_gate_fires,
       (ch.tbl_tuples >= 0 AND ic.base_tuples >= 0
        AND (CASE WHEN ic.base_tuples = 0 THEN ch.tbl_tuples > 0
                  ELSE abs(ch.tbl_tuples - ic.base_tuples)
                       >= ic.base_tuples * 0.20 END))            AS tbl_gate_fires,
       (ch.idx_tuples >= 0 AND ic.base_idx_tuples >= 0
        AND (CASE WHEN ic.base_idx_tuples = 0 THEN ch.idx_tuples > 0
                  ELSE abs(ch.idx_tuples - ic.base_idx_tuples)
                       >= ic.base_idx_tuples * 0.20 END))        AS idx_gate_fires,
       -- The sz the report proposed to store, against the sz that was stored.
       substring(t.cmd_report from '"sz":([0-9]+)')::numeric      AS sz_reported,
       substring(t.cmd_written from '"sz":([0-9]+)')::numeric     AS sz_written,
       substring(t.cmd_written from '"itup":(-?[0-9]+)')::numeric AS itup_written,
       CASE WHEN ic.payload IS NULL                              THEN 'initialize'
            WHEN ic.base_bytes IS NULL OR ic.base_tuples IS NULL
              OR ic.base_idx_tuples IS NULL                      THEN 'initialize'
            WHEN ic.base_bytes > 0 AND ch.bytes < ic.base_bytes  THEN 'refresh'
            WHEN ic.base_bytes > 0 AND ch.bytes >= ic.base_bytes * 1.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples > 0
             AND abs(ch.tbl_tuples - ic.base_tuples) >= ic.base_tuples * 0.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples = 0 AND ch.tbl_tuples > 0
                                                                 THEN 'measure'
            WHEN ch.idx_tuples >= 0 AND ic.base_idx_tuples > 0
             AND abs(ch.idx_tuples - ic.base_idx_tuples) >= ic.base_idx_tuples * 0.20
                                                                 THEN 'measure'
            WHEN ch.idx_tuples >= 0 AND ic.base_idx_tuples = 0 AND ch.idx_tuples > 0
                                                                 THEN 'measure'
            ELSE 'skip' END                                      AS expected_stage,
       -- The two-gate form this page filed before the index count was stored,
       -- recomputed on the same fixtures so the change can be quantified.
       CASE WHEN ic.payload IS NULL                              THEN 'initialize'
            WHEN ic.base_bytes IS NULL OR ic.base_tuples IS NULL THEN 'initialize'
            WHEN ic.base_bytes > 0 AND ch.bytes < ic.base_bytes  THEN 'refresh'
            WHEN ic.base_bytes > 0 AND ch.bytes >= ic.base_bytes * 1.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples > 0
             AND abs(ch.tbl_tuples - ic.base_tuples) >= ic.base_tuples * 0.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples = 0 AND ch.tbl_tuples > 0
                                                                 THEN 'measure'
            ELSE 'skip' END                                      AS two_gate_stage,
       CASE WHEN t.action IN ('reindex', 'update') THEN 'measure'
            ELSE t.action END                                    AS taken_stage,
       -- The rebuild oracle against the 40 % decision.
       CASE WHEN t.action IS NULL                                THEN 'ABSENT'
            WHEN t.action = 'reindex'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) < 10    THEN 'CRITICAL FALSE POSITIVE'
            WHEN t.action = 'reindex'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) < 35    THEN 'FALSE POSITIVE'
            WHEN t.action <> 'reindex'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) >= 50   THEN 'FALSE NEGATIVE'
            ELSE 'PASS' END                                      AS verdict,
       -- Where a false negative was lost: the gate never measured, or the
       -- measurement read 40 % or less on a file a rebuild did shrink.
       CASE WHEN t.action IN ('skip', 'initialize', 'refresh')
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) >= 50   THEN 'gate'
            WHEN t.action = 'update'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) >= 50   THEN 'threshold'
            ELSE NULL END                                        AS lost_by,
       t.notes, p.note
  FROM plan p
  LEFT JOIN truth t ON t.idx = p.idx
  LEFT JOIN snap bu ON bu.phase = 'built'   AND bu.idx = p.idx
  LEFT JOIN snap ic ON ic.phase = 'init'    AND ic.idx = p.idx
  LEFT JOIN snap ch ON ch.phase = 'churned' AND ch.idx = p.idx;
HARNESS
  cat > "$SQLD/fixtures_build.sql" <<'FIXTURES_BUILD'
-- The numbered suite, build phase: every fixture up to and including the
-- creation of the index that is scored.  The churn file carries the rest of
-- each recipe, so that the heuristic can store a baseline for a freshly built
-- index and then be asked about the same index after it has been disturbed.
--
-- Disposable fixtures.  Everything below creates, forges and drops objects in
-- the suite database of the sandbox cluster.  It is not meant for a database
-- anyone cares about.
--
-- The recipes are the numbered fixtures of the PostgreSQL 17 page "Testing the
-- PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17": tests 1-17
-- (deduplication gate), 18-91 (partial indexes) and controls 92-121.  Two
-- deviations, both deliberate: pg_stat_force_next_flush() is not called,
-- because this heuristic reads pg_class.reltuples and never a cumulative
-- statistics view, and the four deduplicate_items fixtures and the four ICU
-- fixtures live in their own files, because those features do not exist in
-- every server this text has to run on.
SET /* wiki_btmaint_fixtures_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_fixtures_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_fixtures_lock_timeout */ lock_timeout = '5s';
SET /* wiki_btmaint_fixtures_maintenance_work_mem */ maintenance_work_mem = '256MB';

-- ======================================================== tests 1-17 ========
-- The deduplication gate's two 500,000-row tables and its custom operator
-- classes.  For this heuristic the group asks a different question than it did
-- for the estimator: every equal-image class must survive the same 90 % drain
-- and be measured, and the 40 % decision must agree with a rebuild.
--
-- Every fixture that needs a B-tree support function 4 lives in the
-- PostgreSQL 13 file, because support function 4 is the deduplication
-- equal-image callback: a 12 server answers "invalid function number 4, must
-- be between 1 and 3" and the whole file would abort here.  What stays is the
-- one custom operator class that declares no support function 4 at all.
CREATE OPERATOR CLASS int4_ei_none FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4);

CREATE TABLE t AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s, ((i % 5000)::numeric) AS n,
       (i % 5000)::float4 AS f4, (i % 5000)::float8 AS f8, (i % 7)::int4 AS d
  FROM generate_series(1, 500000) i;
CREATE TABLE t2 AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE t;
ANALYZE t2;

CREATE INDEX i_int4          ON t (a);
CREATE INDEX i_int8          ON t (b);
CREATE INDEX i_text_det      ON t (s);
CREATE INDEX i_numeric       ON t (n);
CREATE INDEX i_float4        ON t (f4);
CREATE INDEX i_float8        ON t (f8);
CREATE INDEX i_multi_ok      ON t (a, b);
CREATE INDEX i_multi_bad     ON t (a, n);
CREATE INDEX i_expr_num      ON t ((a::numeric));
CREATE INDEX i_inc           ON t (a) INCLUDE (d);
CREATE INDEX i_ei_none       ON t (a int4_ei_none);
CREATE UNIQUE INDEX i_uniq   ON t (u);
CREATE INDEX i2_ok           ON t2 (a, b);

SELECT plan_add(1,  'gate', 'int4 key, 100 rows per key', 'i_int4', 'measure', 'i_int4');
SELECT plan_add(2,  'gate', 'int8 key, 100 rows per key', 'i_int8', 'measure', 'i_int8');
SELECT plan_add(3,  'gate', 'text key, deterministic default collation', 'i_text_det', 'measure', 'i_text_det');
SELECT plan_add(5,  'gate', 'numeric key, no equal-image support', 'i_numeric', 'measure', 'i_numeric');
SELECT plan_add(6,  'gate', 'float4 key', 'i_float4', 'measure', 'i_float4');
SELECT plan_add(6,  'gate', 'float8 key', 'i_float8', 'measure', 'i_float8');
SELECT plan_add(7,  'gate', 'two equal-image key columns', 'i_multi_ok', 'measure', 'i_multi_ok');
SELECT plan_add(8,  'gate', 'one non-equal-image key column', 'i_multi_bad', 'measure', 'i_multi_bad');
SELECT plan_add(9,  'gate', 'expression key, numeric', 'i_expr_num', 'measure', 'i_expr_num');
SELECT plan_add(10, 'gate', 'INCLUDE column refuses deduplication', 'i_inc', 'measure', 'i_inc');
SELECT plan_add(12, 'gate', 'opclass with no support function 4', 'i_ei_none', 'measure', 'i_ei_none');
SELECT plan_add(17, 'gate', 'unique key control; carried to the 12 leg as test 17', 'i_uniq', 'measure', 'i_uniq');
SELECT plan_add(7,  'gate', 'second table, two equal-image columns', 'i2_ok', 'measure', 'i2_ok');

-- ======================================================= tests 18-21 ========
CREATE TABLE pt1 AS
SELECT i::bigint AS k, (i % 100)::int AS sel FROM generate_series(1, 1000000) i;
ANALYZE pt1;
CREATE INDEX p18 ON pt1 (k) WHERE sel < 20;
CREATE INDEX p19 ON pt1 (k) WHERE sel < 1;
CREATE INDEX p20 ON pt1 (k) WHERE sel < 10;
CREATE INDEX p21 ON pt1 (k) WHERE sel < 80;
SELECT plan_add(18, 'partial', 'baseline, subset distribution = table (20%)', 'p18', 'measure');
SELECT plan_add(19, 'partial', 'very selective, ~1%', 'p19', 'measure');
SELECT plan_add(20, 'partial', 'moderately selective, ~10%', 'p20', 'measure');
SELECT plan_add(21, 'partial', 'large subset, ~80%', 'p21', 'measure');

-- ======================================================= tests 22-33 ========
CREATE TABLE pd22 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd22;
CREATE INDEX p22 ON pd22 (k) WHERE hot;
SELECT plan_add(22, 'partial', 'highly duplicated subset, unique outside', 'p22', 'measure');

CREATE TABLE pd23 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE ((i / 5) % 100)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd23;
CREATE INDEX p23 ON pd23 (k) WHERE hot;
SELECT plan_add(23, 'partial', 'highly unique subset, duplicated outside', 'p23', 'measure');

CREATE TABLE pd24 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50000)::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd24;
CREATE INDEX p24 ON pd24 (k) WHERE hot;
SELECT plan_add(24, 'partial', 'n_distinct radically different in the subset', 'p24', 'measure');

CREATE TABLE pd25 AS SELECT (i % 100 = 0) AS hot,
       CASE WHEN i % 100 = 0 THEN ((i / 100) % 997)::int ELSE (i % 5)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd25;
CREATE INDEX p25 ON pd25 (k) WHERE hot;
SELECT plan_add(25, 'partial', 'MCV distribution differs inside the subset', 'p25', 'measure');

CREATE TABLE pd26 AS SELECT (i % 50 = 0) AS hot,
       CASE WHEN i % 50 = 0 THEN (1000000 + i)::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd26;
CREATE INDEX p26 ON pd26 (k) WHERE hot;
SELECT plan_add(26, 'partial', 'table-wide MCVs absent inside the subset', 'p26', 'measure');

CREATE TABLE pd27 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 AND i % 100 <> 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd27;
CREATE INDEX p27 ON pd27 (k) WHERE hot;
SELECT plan_add(27, 'partial', 'NULL-heavy subset, non-NULL outside', 'p27', 'measure');

CREATE TABLE pd28 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN i::bigint ELSE NULL END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd28;
CREATE INDEX p28 ON pd28 (k) WHERE hot;
SELECT plan_add(28, 'partial', 'NULL-free subset, NULL-heavy table (bigint)', 'p28', 'measure');

CREATE TABLE pd29 AS
SELECT CASE WHEN i % 5 = 0 THEN NULL ELSE lpad(i::text, 20, '0') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pd29;
CREATE INDEX p29 ON pd29 (s) WHERE s IS NULL;
SELECT plan_add(29, 'partial', 'all-NULL partial index, WHERE s IS NULL', 'p29', 'skip');

CREATE TABLE pd30 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pd30;
CREATE INDEX p30 ON pd30 (s) WHERE hot;
SELECT plan_add(30, 'partial', 'subset values wider than outside (13 against 204 bytes)', 'p30', 'measure');

CREATE TABLE pd31 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN lpad((i % 9)::text, 12, 'n')
            ELSE repeat('W', 190) || lpad(i::text, 10, '0') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pd31;
CREATE INDEX p31 ON pd31 (s) WHERE hot;
SELECT plan_add(31, 'partial', 'subset values narrower than outside', 'p31', 'measure');

CREATE TABLE pw32 AS
SELECT (i % 50 = 0) AS hot,
       CASE WHEN i % 50 = 0 THEN repeat('W', 390) || lpad(i::text, 10, '0')
            ELSE repeat('n', 18) || (i % 9)::text END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pw32;
CREATE INDEX p32 ON pw32 (s) WHERE hot;
SELECT plan_add(32, 'partial', 'extreme width mismatch (27 against 404 bytes)', 'p32', 'measure');

CREATE TABLE pd33 AS SELECT (i % 5 = 0) AS hot,
       lpad(i::text, 10 + (i % 40), 'x') AS s FROM generate_series(1, 500000) i;
ANALYZE pd33;
CREATE INDEX p33 ON pd33 (s) WHERE hot;
SELECT plan_add(33, 'partial', 'variable-width values, same range inside and out', 'p33', 'measure');

-- ======================================================= tests 34-39 ========
CREATE TABLE pd34 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd34;
CREATE INDEX p34 ON pd34 (k) WHERE hot;
SELECT plan_add(34, 'partial', 'dedup-heavy subset, 1000 rows per key', 'p34', 'measure');

CREATE TABLE pd35 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd35;
CREATE INDEX p35 ON pd35 (k) WHERE hot;
SELECT plan_add(35, 'partial', 'duplicate-heavy table, unique subset', 'p35', 'measure');

CREATE TABLE pd36 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN 42 ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd36;
CREATE INDEX p36 ON pd36 (k) WHERE hot;
SELECT plan_add(36, 'partial', 'one key group, 100,000 TIDs against a 132 cap', 'p36', 'measure');

CREATE TABLE pd37 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd37;
CREATE INDEX p37 ON pd37 (k) WHERE hot;
SELECT plan_add(37, 'partial', 'NULL deduplication, every subset key NULL', 'p37', 'measure');

CREATE TABLE pd39 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd39;
CREATE UNIQUE INDEX p39 ON pd39 (k) WHERE hot;
SELECT plan_add(39, 'partial', 'partial UNIQUE index', 'p39', 'measure');

-- ======================================================= tests 40-47 ========
CREATE TABLE pd40 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 97)::int  END AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd40;
CREATE INDEX p40 ON pd40 (a, b) WHERE hot;
SELECT plan_add(40, 'partial', 'two-column key correlated only in the subset', 'p40', 'measure');

CREATE TABLE pd41 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 97)::int  ELSE (i % 100)::int END AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd41;
CREATE INDEX p41 ON pd41 (a, b) WHERE hot;
SELECT plan_add(41, 'partial', 'two-column key independent only in the subset', 'p41', 'measure');

CREATE TABLE pd42 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50)::int ELSE i::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50)::int ELSE i::int END AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd42;
CREATE INDEX p42 ON pd42 (a, b) WHERE hot;
SELECT plan_add(42, 'partial', 'multi-column duplicate keys in the subset', 'p42', 'measure');

CREATE TABLE pd43 AS SELECT (i % 5 = 0) AS hot, i::int AS a, (i * 2)::int AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd43;
CREATE INDEX p43 ON pd43 (a, b) WHERE hot;
SELECT plan_add(43, 'partial', 'multi-column unique keys in the subset', 'p43', 'measure');

CREATE TABLE pd44a AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd44a;
CREATE INDEX p44a ON pd44a (a, b) WHERE hot;
SELECT plan_add(44, 'partial', 'multicolumn key, no ndistinct object', 'p44a', 'measure', 'a');
CREATE TABLE pd44b AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS pd44b_nd (ndistinct) ON a, b FROM pd44b;
ANALYZE pd44b;
CREATE INDEX p44b ON pd44b (a, b) WHERE hot;
SELECT plan_add(44, 'partial', 'multicolumn key, with CREATE STATISTICS (ndistinct)', 'p44b', 'measure', 'b');

CREATE TABLE pd45 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 97)::int  ELSE (i % 100)::int END AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS pd45_nd (ndistinct) ON a, b FROM pd45;
ANALYZE pd45;
CREATE INDEX p45 ON pd45 (a, b) WHERE hot;
SELECT plan_add(45, 'partial', 'extended statistics wrong for the subset', 'p45', 'measure');

CREATE TABLE pd46 AS SELECT (i % 5 = 0) AS hot, i::int AS k, (i % 7)::int AS pay
  FROM generate_series(1, 500000) i;
ANALYZE pd46;
CREATE INDEX p46 ON pd46 (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(46, 'partial', 'partial index with INCLUDE columns', 'p46', 'measure');

CREATE TABLE pi47 AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS payload
  FROM generate_series(1, 500000) i;
ANALYZE pi47;
CREATE INDEX p47 ON pi47 (k) INCLUDE (payload) WHERE hot;
SELECT plan_add(47, 'partial', 'wide INCLUDE values inside the subset', 'p47', 'measure');

-- ======================================================= tests 48-55 ========
CREATE TABLE pe48 AS SELECT (i % 5 = 0) AS active,
       CASE WHEN i % 5 = 0 THEN 'NAME' || lpad(((i / 5) % 20)::text, 6, '0')
            ELSE 'name' || lpad((i % 100)::text, 6, '0') END AS name
  FROM generate_series(1, 500000) i;
ANALYZE pe48;
CREATE INDEX p48 ON pe48 (lower(name)) WHERE active;
SELECT plan_add(48, 'partial', 'partial expression index, lower(name) WHERE active', 'p48', 'measure');
CREATE TABLE pe48b AS SELECT (i % 5 = 0) AS active,
       CASE WHEN i % 5 = 0 THEN 'NAME' || lpad(((i / 5) % 20)::text, 6, '0')
            ELSE 'name' || lpad((i % 100)::text, 6, '0') END AS name
  FROM generate_series(1, 500000) i;
CREATE INDEX p48b ON pe48b (lower(name)) WHERE active;
ANALYZE pe48b;
SELECT plan_add(48, 'partial', 'the same after one ANALYZE with the index in place', 'p48b', 'measure', 'b');

CREATE TABLE pe49 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pe49;
CREATE INDEX p49 ON pe49 (upper(s)) WHERE hot;
SELECT plan_add(49, 'partial', 'expression width mismatch in the subset', 'p49', 'measure');
CREATE TABLE pe49b AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX p49b ON pe49b (upper(s)) WHERE hot;
ANALYZE pe49b;
SELECT plan_add(49, 'partial', 'the same after one ANALYZE with the index in place', 'p49b', 'measure', 'b');

CREATE TABLE pe50 AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE pe50;
CREATE INDEX p50 ON pe50 (upper(s)) WHERE hot;
SELECT plan_add(50, 'partial', 'missing expression statistics, 32-byte fallback', 'p50', 'measure');
CREATE TABLE pe50b AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX p50b ON pe50b (upper(s)) WHERE hot;
ANALYZE pe50b;
SELECT plan_add(50, 'partial', 'the same after one ANALYZE with the index in place', 'p50b', 'measure', 'b');

CREATE TABLE pf AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pf;
CREATE INDEX p53 ON pf (k) WHERE hot;
CREATE INDEX p54 ON pf (k) WITH (fillfactor = 100) WHERE hot;
CREATE INDEX p55 ON pf (k) WITH (fillfactor = 70)  WHERE hot;
SELECT plan_add(53, 'partial', 'default fillfactor 90', 'p53', 'measure');
SELECT plan_add(54, 'partial', 'fillfactor = 100', 'p54', 'measure');
SELECT plan_add(55, 'partial', 'fillfactor = 70', 'p55', 'measure');

-- ======================================================= tests 56-63 ========
CREATE TABLE ps AS
SELECT (i % 5 = 0) AS flag,
       CASE WHEN i % 5 = 0 THEN 'OPEN' ELSE 'CLOSED' END AS status,
       timestamptz '2020-01-01' + (i * interval '1 minute') AS created,
       CASE WHEN i % 5 = 0 THEN NULL ELSE i::int END AS nk,
       i::int AS k, (i % 1000)::int AS k2
  FROM generate_series(1, 500000) i;
ANALYZE ps;
CREATE INDEX p56 ON ps (k) WHERE flag;
CREATE INDEX p57 ON ps (k) WHERE status = 'OPEN';
CREATE INDEX p58 ON ps (k) WHERE created >= timestamptz '2020-09-01';
CREATE INDEX p59 ON ps (k) WHERE nk IS NULL;
CREATE INDEX p60 ON ps (k) WHERE nk IS NOT NULL;
CREATE INDEX p61 ON ps (k) WHERE flag AND status = 'OPEN';
CREATE INDEX p62 ON ps (k) WHERE k < 100000;
CREATE INDEX p63 ON ps (k2) WHERE k >= 400000;
SELECT plan_add(56, 'partial', 'boolean predicate, WHERE flag', 'p56', 'measure');
SELECT plan_add(57, 'partial', 'equality predicate, status = OPEN', 'p57', 'measure');
SELECT plan_add(58, 'partial', 'range predicate, created >= ...', 'p58', 'measure');
SELECT plan_add(59, 'partial', 'IS NULL predicate on a non-key column', 'p59', 'measure');
SELECT plan_add(60, 'partial', 'IS NOT NULL predicate', 'p60', 'measure');
SELECT plan_add(61, 'partial', 'multi-column predicate', 'p61', 'measure');
SELECT plan_add(62, 'partial', 'predicate correlated with the indexed value', 'p62', 'measure');
SELECT plan_add(63, 'partial', 'predicate negatively correlated with the value', 'p63', 'measure');

-- ======================================================= tests 64-69 ========
CREATE TABLE pc64 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc64;
CREATE INDEX p64 ON pc64 (k) WHERE hot;
SELECT plan_add(64, 'partial', 'stale statistics after inserts into the subset', 'p64', 'measure');

CREATE TABLE pc65 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc65;
CREATE INDEX p65 ON pc65 (k) WHERE hot;
SELECT plan_add(65, 'partial', 'stale statistics after deletes, no VACUUM', 'p65', 'skip');

CREATE TABLE pc66 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc66;
CREATE INDEX p66 ON pc66 (k) WHERE hot;
SELECT plan_add(66, 'partial', 'rows entering the index (false -> true)', 'p66', 'measure');

CREATE TABLE pc67 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc67;
CREATE INDEX p67 ON pc67 (k) WHERE hot;
SELECT plan_add(67, 'partial', 'rows leaving the index (true -> false)', 'p67', 'measure');

CREATE TABLE pc68 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc68;
CREATE INDEX p68 ON pc68 (k) WHERE hot;
SELECT plan_add(68, 'partial', 'heavy predicate churn, then VACUUM + ANALYZE', 'p68', 'measure');

CREATE TABLE pc69 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc69;
CREATE INDEX p69 ON pc69 (k) WHERE hot;
SELECT plan_add(69, 'partial', 'stale reltuples, VACUUM but no ANALYZE', 'p69', 'measure');

-- ======================================================= tests 70-77 ========
CREATE TABLE pb AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pb;
CREATE INDEX p70 ON pb (k) WHERE hot;
SELECT plan_add(70, 'partial', 'freshly created partial index', 'p70', 'skip');
CREATE INDEX p71 ON pb (k) WHERE hot;
SELECT plan_add(71, 'partial', 'freshly REINDEXed partial index', 'p71', 'skip');

CREATE TABLE pb72 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb72;
CREATE INDEX p72 ON pb72 (k) WHERE hot;
SELECT plan_add(72, 'partial', '25% of the subset deleted', 'p72', 'measure');

CREATE TABLE pb73 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb73;
CREATE INDEX p73 ON pb73 (k) WHERE hot;
SELECT plan_add(73, 'partial', '50% of the subset deleted', 'p73', 'measure');

CREATE TABLE pb74 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb74;
CREATE INDEX p74 ON pb74 (k) WHERE hot;
SELECT plan_add(74, 'partial', '75% of the subset deleted', 'p74', 'measure');

CREATE TABLE pb75 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb75;
CREATE INDEX p75 ON pb75 (k) WHERE hot;
SELECT plan_add(75, 'partial', '90% of the subset deleted (corrected recipe)', 'p75', 'measure');

CREATE TABLE pb76 AS SELECT (i % 5 = 0) AS hot, i::int AS k, 'x'::text AS pad
  FROM generate_series(1, 500000) i;
ANALYZE pb76;
CREATE INDEX p76 ON pb76 (k) WHERE hot;
SELECT plan_add(76, 'partial', 'bloated through indexed-key UPDATEs', 'p76', 'measure');

CREATE TABLE pb77 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb77;
CREATE INDEX p77 ON pb77 (k) WHERE hot;
SELECT plan_add(77, 'partial', 'many empty and deleted B-tree pages', 'p77', 'measure');

-- ======================================================= tests 78-85 ========
CREATE TABLE f78t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 290) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE f78t;
CREATE INDEX f78 ON f78t (s) WHERE hot;
SELECT plan_add(78, 'falsepos', 'predicate-conditioned width mismatch', 'f78', 'skip');

CREATE TABLE f79t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('t', 300) || lpad(i::text, 4, '0')
            ELSE NULL END AS s
  FROM generate_series(1, 500000) i;
ANALYZE f79t;
CREATE INDEX f79 ON f79t (s) WHERE hot;
SELECT plan_add(79, 'falsepos', 'predicate-conditioned NULL mismatch', 'f79', 'skip');

CREATE TABLE f80t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE f80t;
CREATE INDEX f80 ON f80t (k) WHERE hot;
SELECT plan_add(80, 'falsepos', 'predicate-conditioned n_distinct mismatch', 'f80', 'skip');

CREATE TABLE f81t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 20)::int ELSE 7 END AS k
  FROM generate_series(1, 500000) i;
ANALYZE f81t;
CREATE INDEX f81 ON f81t (k) WHERE hot;
SELECT plan_add(81, 'falsepos', 'predicate-conditioned MCV mismatch', 'f81', 'skip');

CREATE TABLE f82t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 89)::int  END AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS f82_nd (ndistinct) ON a, b FROM f82t;
ANALYZE f82t;
CREATE INDEX f82 ON f82t (a, b) WHERE hot;
SELECT plan_add(82, 'falsepos', 'predicate-conditioned multi-column correlation', 'f82', 'skip');

CREATE TABLE f83t AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE f83t;
CREATE INDEX f83 ON f83t (md5(s), lower(s)) WHERE hot;
SELECT plan_add(83, 'falsepos', 'missing index/expression statistics', 'f83', 'skip');

CREATE TABLE f84t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE f84t;
CREATE INDEX f84 ON f84t (k) WHERE hot;
SELECT plan_add(84, 'falsepos', 'stale partial-index reltuples', 'f84', 'skip');

CREATE TABLE f85t AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE f85t;
UPDATE f85t SET s = repeat('W', 200) || s WHERE hot;
VACUUM f85t;
CREATE INDEX f85 ON f85t (s) WHERE hot;
SELECT plan_add(85, 'falsepos', 'stale table statistics', 'f85', 'skip');

-- ======================================================= tests 86-91 ========
CREATE TABLE f86t AS SELECT (i % 5 = 0) AS hot, ((i / 5) % 100)::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE f86t;
CREATE INDEX f86 ON f86t (k) WHERE hot;
SELECT plan_add(86, 'falseneg', 'duplicate concentration inside the subset', 'f86', 'measure');

CREATE TABLE f87t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE f87t;
CREATE INDEX f87 ON f87t (k) WHERE hot;
SELECT plan_add(87, 'falseneg', 'NULL concentration inside the subset', 'f87', 'measure');

CREATE TABLE f88t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN lpad((i % 9)::text, 9, '0')
            ELSE repeat('W', 390) || lpad(i::text, 10, '0') END AS s
  FROM generate_series(1, 200000) i;
ANALYZE f88t;
CREATE INDEX f88 ON f88t (s) WHERE hot;
SELECT plan_add(88, 'falseneg', 'subset narrower than table statistics', 'f88', 'measure');

CREATE TABLE f89t AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
ANALYZE f89t;
CREATE INDEX f89 ON f89t (a, b) WHERE hot;
SELECT plan_add(89, 'falseneg', 'conditional multi-column correlation', 'f89', 'measure');

CREATE TABLE f90t AS SELECT (i % 5 = 0) AS hot, ((i / 5) % 1000)::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE f90t;
CREATE INDEX f90 ON f90t (k) WHERE hot;
SELECT plan_add(90, 'falseneg', 'real deduplication stronger than predicted', 'f90', 'measure');

CREATE TABLE f91t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN lpad((i % 9)::text, 9, '0')
            ELSE repeat('W', 390) || lpad(i::text, 10, '0') END AS s,
       i::int AS ord
  FROM generate_series(1, 200000) i;
ANALYZE f91t;
CREATE INDEX f91 ON f91t (s) WHERE hot;
SELECT plan_add(91, 'falseneg', 'many deleted pages plus an over-predicting model', 'f91', 'measure');

-- ====================================================== tests 92-95 =========
CREATE TABLE b92t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE b92t;
CREATE INDEX b92 ON b92t (k) WHERE hot;
SELECT plan_add(92, 'control', '1,000 rows updated under the old GUC threshold', 'b92', 'measure');

CREATE TABLE b93t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE b93t;
CREATE INDEX b93 ON b93t (k) WHERE hot;
SELECT plan_add(93, 'control', 'rows updated above the old GUC threshold', 'b93', 'measure');

CREATE TABLE b94t (hot bool, k int)
  WITH (autovacuum_analyze_threshold = 100, autovacuum_analyze_scale_factor = 0);
INSERT INTO b94t SELECT (i % 5 = 0), i FROM generate_series(1, 500000) i;
ANALYZE b94t;
CREATE INDEX b94 ON b94t (k) WHERE hot;
SELECT plan_add(94, 'control', '1,000 rows updated, table reloption threshold 100', 'b94', 'measure');

CREATE TABLE b95t (hot bool, k int)
  WITH (autovacuum_analyze_threshold = 200000, autovacuum_analyze_scale_factor = 1);
INSERT INTO b95t SELECT (i % 5 = 0), i FROM generate_series(1, 500000) i;
ANALYZE b95t;
CREATE INDEX b95 ON b95t (k) WHERE hot;
SELECT plan_add(95, 'control', 'many rows updated, table reloption threshold 200,000', 'b95', 'measure');

-- ====================================================== tests 96-99 =========
CREATE TABLE np AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE np;
CREATE INDEX np96 ON np (k);
CREATE INDEX np97 ON np (upper(s));
SELECT plan_add(96, 'control', 'plain index, fresh statistics', 'np96', 'skip');
SELECT plan_add(97, 'control', 'expression index, no statistics row', 'np97', 'skip');

CREATE TABLE np98t AS SELECT i::int AS k FROM generate_series(1, 500000) i;
ANALYZE np98t;
CREATE INDEX np98 ON np98t (k);
SELECT plan_add(98, 'control', 'plain index, stale row counts after 300,000 inserts', 'np98', 'measure');

CREATE TABLE np99t AS SELECT (i % 1000)::int AS k FROM generate_series(1, 500000) i;
ANALYZE np99t;
CREATE INDEX np99 ON np99t (k);
SELECT plan_add(99, 'control', 'duplicate-heavy index, genuinely reclaimable', 'np99', 'measure');

-- ===================================================== tests 100-105 ========
CREATE TABLE i100t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i100t;
CREATE INDEX i100 ON i100t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(100, 'control', 'partial + INCLUDE (text), 90% of the subset deleted', 'i100', 'measure');

CREATE TABLE i101t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i101t;
CREATE INDEX i101 ON i101t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(101, 'control', 'partial + INCLUDE (text), same width inside and outside', 'i101', 'skip');

CREATE TABLE i102t AS SELECT i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i102t;
CREATE INDEX i102 ON i102t (k) INCLUDE (pay);
SELECT plan_add(102, 'control', 'non-partial + wide INCLUDE (text), freshly built', 'i102', 'skip');

CREATE TABLE i103t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad(i::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE i103t;
CREATE INDEX i103 ON i103t (s) WHERE hot;
SELECT plan_add(103, 'control', 'partial + wide key column, unique values', 'i103', 'skip');

CREATE TABLE i104t AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN lpad((i % 9)::text, 12, 'n')
            ELSE repeat('W', 190) || lpad(i::text, 10, '0') END AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i104t;
CREATE INDEX i104 ON i104t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(104, 'control', 'partial + INCLUDE (text) narrower inside the subset', 'i104', 'skip');

CREATE TABLE i105t AS SELECT (i % 20 = 0) AS hot, i::int AS k, (i % 7)::int AS n,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i105t;
CREATE INDEX i105 ON i105t (k) INCLUDE (n, pay) WHERE hot;
SELECT plan_add(105, 'control', 'partial + INCLUDE (int, text), mixed non-key widths', 'i105', 'skip');

-- ===================================================== tests 106-112 ========
CREATE TABLE x106t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x106t;
CREATE INDEX x106 ON x106t (upper(s));
SELECT plan_add(106, 'control', 'expression index, no statistics row, 90% deleted', 'x106', 'measure');

CREATE TABLE x107t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x107t;
CREATE INDEX x107 ON x107t (upper(s));
SELECT plan_add(107, 'control', 'the same, with one ANALYZE after the build', 'x107', 'measure');

CREATE TABLE x108t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX x108 ON x108t (upper(s));
SELECT plan_add(108, 'control', 'expression index on a never-analysed table', 'x108', 'skip');

CREATE TABLE x109t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ALTER TABLE x109t ALTER COLUMN s SET STATISTICS 0;
ANALYZE x109t;
CREATE INDEX x109 ON x109t (s);
SELECT plan_add(109, 'control', 'plain index, key column with SET STATISTICS 0', 'x109', 'skip');

CREATE TABLE x110t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x110t;
CREATE INDEX x110 ON x110t (k, upper(s));
SELECT plan_add(110, 'control', 'mixed key (k, upper(s)), no statistics row', 'x110', 'skip');

CREATE TABLE x111t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x111t;
CREATE INDEX x111 ON x111t (left(s, 3));
SELECT plan_add(111, 'control', 'narrow expression left(s, 3), no statistics row', 'x111', 'skip');

CREATE TABLE x112t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x112t;
CREATE INDEX x112 ON x112t (upper(s)) WHERE hot;
SELECT plan_add(112, 'control', 'partial expression index, no statistics row', 'x112', 'skip');

-- ===================================================== tests 113-121 ========
CREATE TABLE q113a AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q113a;
CREATE INDEX p113a ON q113a (id) WHERE state = 'pending';
SELECT plan_add(113, 'zero', 'drained queue, nothing run', 'p113a', 'skip', 'a');

CREATE TABLE q113b AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q113b;
CREATE INDEX p113b ON q113b (id) WHERE state = 'pending';
SELECT plan_add(113, 'zero', 'drained queue, VACUUM + ANALYZE', 'p113b', 'skip', 'b');

CREATE TABLE q113c AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q113c;
CREATE INDEX p113c ON q113c (id) WHERE state = 'pending';
SELECT plan_add(113, 'zero', 'drained queue, ANALYZE only', 'p113c', 'skip', 'c');

CREATE TABLE q114 AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q114;
CREATE INDEX p114 ON q114 (id) WHERE state = 'pending';
SELECT plan_add(114, 'zero', 'queue drained to 1%, VACUUM + ANALYZE', 'p114', 'skip');

CREATE TABLE q115(id int, state text);
ANALYZE q115;
CREATE INDEX p115 ON q115 (id) WHERE state = 'pending';
SELECT plan_add(115, 'zero', 'index built on an analysed empty table, then loaded', 'p115', 'measure');

CREATE TABLE q116 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p116 ON q116 (id) WHERE state = 'pending';
ANALYZE q116;
SELECT plan_add(116, 'zero', 'subset empty from the start and measured empty', 'p116', 'skip');

CREATE TABLE q117 AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q117;
CREATE INDEX p117 ON q117 (id) WHERE state = 'pending';
SELECT plan_add(117, 'zero', 'drained, then VACUUM only', 'p117', 'skip');

CREATE TABLE q118 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p118 ON q118 (id) WHERE state = 'pending';
ANALYZE q118;
SELECT plan_add(118, 'zero', 'subset measured empty, then 50,000 rows arrive', 'p118', 'skip');

CREATE TABLE q119 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p119 ON q119 (id) WHERE state = 'pending';
ANALYZE q119;
SELECT plan_add(119, 'zero', 'fixture 118 after one ANALYZE', 'p119', 'skip');

CREATE TABLE q120 AS SELECT i::int AS id,
       CASE WHEN i <= 2000 THEN 'pending' ELSE 'done' END::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p120 ON q120 (id) WHERE state = 'pending';
SET default_statistics_target = 1;
ANALYZE q120;
RESET default_statistics_target;
SELECT plan_add(120, 'zero', 'a 300-row sample missed a 2,000-row subset', 'p120', 'skip');

CREATE TABLE nz AS SELECT i::int AS k FROM generate_series(1, 1000000) i;
ANALYZE nz;
CREATE INDEX nz_k ON nz (k);
SELECT plan_add(121, 'zero', 'emptied, vacuumed, reloaded without ANALYZE', 'nz_k', 'measure', 'nz_k');

CREATE TABLE nzb AS SELECT i::int AS k FROM generate_series(1, 1000000) i;
ANALYZE nzb;
CREATE INDEX nzb_k ON nzb (k);
SELECT plan_add(121, 'zero', 'the same, plus a REINDEX while the table is empty', 'nzb_k', 'refresh', 'nzb_k');

CREATE TABLE trunc_t AS SELECT i::int AS k FROM generate_series(1, 300000) i;
ANALYZE trunc_t;
CREATE INDEX i_trunc ON trunc_t (k);
SELECT plan_add(121, 'zero', 'TRUNCATE then reload without ANALYZE', 'i_trunc', 'skip', 'i_trunc');

SELECT count(*) AS planned_fixtures FROM plan;
FIXTURES_BUILD
  cat > "$SQLD/fixtures_v13.sql" <<'FIXTURES_V13'
-- Build phase, every fixture that needs a feature PostgreSQL 12 does not have.
-- Two families:
--
--   1. B-tree support function 4, the deduplication equal-image callback.  A
--      12 server refuses "FUNCTION 4" in CREATE OPERATOR CLASS outright, and
--      has no btequalimage or btvarstrequalimage to alias, so tests 13, 14, 15
--      and 16 are skipped there.
--   2. The deduplicate_items reloption, which 12 calls an unrecognized
--      parameter, so test 11 and test 38 are skipped there.
--
-- This file runs only where server_version_num >= 130000; otherwise the
-- fixtures are recorded as skipped and the suite is scored without them.
-- Disposable fixtures, in the sandbox cluster's suite database only.
SET /* wiki_btmaint_v13_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_v13_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_v13_lock_timeout */ lock_timeout = '5s';
SET /* wiki_btmaint_v13_maintenance_work_mem */ maintenance_work_mem = '256MB';

CREATE OR REPLACE FUNCTION ei_true(oid)  RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;
CREATE OR REPLACE FUNCTION ei_false(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT false $$;
CREATE OR REPLACE FUNCTION ei_alias(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btequalimage';
CREATE OR REPLACE FUNCTION ei_renamed(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btvarstrequalimage';
-- test 16, the impostor: a SQL function wearing the built-in's name.  It must
-- be schema-qualified in the operator class or pg_catalog wins the lookup.
CREATE OR REPLACE FUNCTION public.btequalimage(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;

CREATE OPERATOR CLASS int4_ei_true FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_true(oid);
CREATE OPERATOR CLASS int4_ei_false FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_false(oid);
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

CREATE INDEX i_ei_false  ON t (a int4_ei_false);
CREATE INDEX i_ei_true   ON t (a int4_ei_true);
CREATE INDEX i_ei_alias  ON t (a int4_ei_alias);
CREATE INDEX i_mixed_tf  ON t (a int4_ei_true, b int8_ei_false);
CREATE INDEX i_mixed_ft  ON t (a int4_ei_false, b int8_ei_true);
CREATE INDEX i_squat     ON t (s text_squat);
CREATE INDEX i_text_det2 ON t (s text_renamed);
CREATE INDEX i2_tf       ON t2 (a int4_ei_true, b int8_ei_false);
CREATE INDEX i2_ft       ON t2 (a int4_ei_false, b int8_ei_true);
SELECT plan_add(13, 'gate', 'support function 4 returns false', 'i_ei_false', 'measure', 'i_ei_false');
SELECT plan_add(14, 'gate', 'support function 4 returns true by design', 'i_ei_true', 'measure', 'i_ei_true');
SELECT plan_add(14, 'gate', 'internal alias of btequalimage', 'i_ei_alias', 'measure', 'i_ei_alias');
SELECT plan_add(15, 'gate', 'mixed true/false support functions', 'i_mixed_tf', 'measure', 'i_mixed_tf');
SELECT plan_add(15, 'gate', 'mixed false/true support functions', 'i_mixed_ft', 'measure', 'i_mixed_ft');
SELECT plan_add(16, 'gate', 'SQL impostor named btequalimage', 'i_squat', 'measure', 'i_squat');
SELECT plan_add(16, 'gate', 'renamed internal support function 4', 'i_text_det2', 'measure', 'i_text_det2');
SELECT plan_add(15, 'gate', 'second table, mixed true/false', 'i2_tf', 'measure', 'i2_tf');
SELECT plan_add(15, 'gate', 'second table, mixed false/true', 'i2_ft', 'measure', 'i2_ft');

CREATE INDEX i_dupoff   ON t (a) WITH (deduplicate_items = off);
CREATE INDEX i_text_off ON t (s) WITH (deduplicate_items = off);
CREATE INDEX i2_off     ON t2 (s) WITH (deduplicate_items = off);
SELECT plan_add(11, 'gate', 'deduplicate_items = off, int4 key', 'i_dupoff', 'measure', 'i_dupoff');
SELECT plan_add(11, 'gate', 'deduplicate_items = off, text key', 'i_text_off', 'measure', 'i_text_off');
SELECT plan_add(11, 'gate', 'deduplicate_items = off, second table', 'i2_off', 'measure', 'i2_off');

CREATE TABLE pd38 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd38;
CREATE INDEX p38 ON pd38 (k) WITH (deduplicate_items = off) WHERE hot;
SELECT plan_add(38, 'partial', 'deduplicate_items = off', 'p38', 'measure');
FIXTURES_V13
  cat > "$SQLD/fixtures_icu.sql" <<'FIXTURES_ICU'
-- Build phase, the four ICU fixtures: tests 3, 4, 9 and 51-52.  A server built
-- --without-icu cannot create these collations, so this file runs only where
-- CREATE COLLATION (provider = icu) succeeds and the fixtures are otherwise
-- recorded as skipped.
-- Disposable fixtures, in the sandbox cluster's suite database only.
SET /* wiki_btmaint_icu_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_icu_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_icu_lock_timeout */ lock_timeout = '5s';

CREATE COLLATION ci   (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
CREATE COLLATION cdet (provider = icu, locale = 'und');

CREATE INDEX i_text_icu_det  ON t (s COLLATE cdet);
CREATE INDEX i_text_nondet   ON t (s COLLATE ci);
CREATE INDEX i_expr_lower_ci ON t ((lower(s)) COLLATE ci);
SELECT plan_add(3, 'gate', 'text key, deterministic ICU collation', 'i_text_icu_det', 'measure', 'i_text_icu_det');
SELECT plan_add(4, 'gate', 'text key, nondeterministic ICU collation', 'i_text_nondet', 'measure', 'i_text_nondet');
SELECT plan_add(9, 'gate', 'expression key under a nondeterministic collation', 'i_expr_lower_ci', 'measure', 'i_expr_lower_ci');

CREATE COLLATION suite_det    (provider = icu, locale = 'und');
CREATE COLLATION suite_nondet (provider = icu, locale = 'und-u-ks-level2',
                               deterministic = false);
CREATE TABLE pc51 AS SELECT (i % 5 = 0) AS hot,
       'key' || lpad(((i / 5) % 100)::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE pc51;
CREATE INDEX p51 ON pc51 (s COLLATE suite_det) WHERE hot;
CREATE INDEX p52 ON pc51 (s COLLATE suite_nondet) WHERE hot;
SELECT plan_add(51, 'partial', 'deterministic ICU collation', 'p51', 'measure');
SELECT plan_add(52, 'partial', 'nondeterministic ICU collation', 'p52', 'measure');
FIXTURES_ICU
  cat > "$SQLD/fixtures_churn.sql" <<'FIXTURES_CHURN'
-- The numbered suite, churn phase: everything each recipe does after its index
-- exists, run after the heuristic has stored an as-built baseline in every
-- index comment.  Two kinds of churn:
--
--   1. the recipe's own, transcribed verbatim from the numbered fixtures;
--   2. a uniform 90 % block drain for the shape fixtures that have no churn of
--      their own, because a fixture designed for a one-shot estimator never
--      needed a "before" and an "after".
--
-- The fixtures whose whole point is that a fresh index must not be touched are
-- deliberately left alone: 70-71, 78-85, 96-97, 101-105, 108-112, 116 and 120.
--
-- Disposable fixtures, in the sandbox cluster's suite database only.
SET /* wiki_btmaint_churn_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_churn_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_churn_lock_timeout */ lock_timeout = '5s';

-- ---------------------------------------------------------- recipe churn ----
-- 64: stale statistics after inserts into the subset.
INSERT INTO pc64 SELECT true, 500000 + i FROM generate_series(1, 200000) i;

-- 65: stale statistics after deletes, no VACUUM.
DELETE FROM pc65 WHERE hot AND k % 50 <> 0;

-- 66: rows entering the index (false -> true).
UPDATE pc66 SET hot = true WHERE NOT hot AND k % 5 = 1;

-- 67: rows leaving the index (true -> false).
UPDATE pc67 SET hot = false WHERE hot AND k % 50 <> 0;

-- 68: heavy predicate churn, then VACUUM + ANALYZE.
UPDATE pc68 SET hot = true  WHERE k % 3 = 0;
UPDATE pc68 SET hot = false WHERE k % 3 = 0;
UPDATE pc68 SET hot = true  WHERE k % 3 = 1;
UPDATE pc68 SET hot = false WHERE k % 3 = 1;
UPDATE pc68 SET hot = (k % 10 = 0);
VACUUM pc68;
ANALYZE pc68;

-- 69: stale reltuples, VACUUM but no ANALYZE.
DELETE FROM pc69 WHERE hot AND k % 50 <> 0;
VACUUM pc69;

-- 71: a REINDEX nobody told the heuristic about.
REINDEX INDEX p71;

-- 72-75: a quarter, a half, three quarters and nine tenths of the subset.
DELETE FROM pb72 WHERE hot AND (k / 5) % 4 = 0;
VACUUM pb72;
ANALYZE pb72;
DELETE FROM pb73 WHERE hot AND (k / 5) % 2 = 0;
VACUUM pb73;
ANALYZE pb73;
DELETE FROM pb74 WHERE hot AND (k / 5) % 4 <> 0;
VACUUM pb74;
ANALYZE pb74;
DELETE FROM pb75 WHERE hot AND (k / 5) % 10 <> 0;
VACUUM pb75;
ANALYZE pb75;

-- 76: bloated through indexed-key UPDATEs.
UPDATE pb76 SET k = k + 1000000 WHERE hot;
VACUUM pb76;
ANALYZE pb76;

-- 77: many empty and deleted B-tree pages, contiguous 95 %.
DELETE FROM pb77 WHERE hot AND k < 475000;
VACUUM pb77;
ANALYZE pb77;

-- 84's forged partial-index reltuples has moved to forge.sql, which runs after
-- the simulated auto-analyze, because an ANALYZE of its table would overwrite
-- the forgery and the fixture would stop testing anything.

-- 86-91: genuinely bloated, VACUUMed and ANALYZEd.
DELETE FROM f86t WHERE hot AND k >= 25;
VACUUM f86t;
ANALYZE f86t;
DELETE FROM f87t WHERE hot AND k IS NOT NULL;
VACUUM f87t;
ANALYZE f87t;
DELETE FROM f88t WHERE hot AND s > lpad('4', 9, '0');
VACUUM f88t;
ANALYZE f88t;
DELETE FROM f89t WHERE hot AND a >= 25;
VACUUM f89t;
ANALYZE f89t;
DELETE FROM f90t WHERE hot AND k >= 250;
VACUUM f90t;
ANALYZE f90t;
DELETE FROM f91t WHERE hot AND ord < 190000;
VACUUM f91t;
ANALYZE f91t;

-- 92-95: a reclaimable partial index, then a known number of row changes.
DELETE FROM b92t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b92t;
ANALYZE b92t;
UPDATE b92t SET k = k WHERE k % 500 = 0;
DELETE FROM b93t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b93t;
ANALYZE b93t;
UPDATE b93t SET k = k WHERE k % 2 = 0;
DELETE FROM b94t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b94t;
ANALYZE b94t;
UPDATE b94t SET k = k WHERE k % 500 = 0;
DELETE FROM b95t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b95t;
ANALYZE b95t;
UPDATE b95t SET k = k WHERE k % 2 = 0;

-- 98: 300,000 inserts and no ANALYZE.
INSERT INTO np98t SELECT 500000 + i FROM generate_series(1, 300000) i;

-- 99: duplicate-heavy, genuinely reclaimable.
DELETE FROM np99t WHERE k >= 60;
VACUUM np99t;
ANALYZE np99t;

-- 100: partial + INCLUDE, 90 % of the subset deleted.
DELETE FROM i100t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM i100t;
ANALYZE i100t;

-- 106: expression index, 90 % deleted, VACUUM but no ANALYZE.
DELETE FROM x106t WHERE k % 10 <> 0;
VACUUM x106t;

-- 107: the same with one ANALYZE.
DELETE FROM x107t WHERE k % 10 <> 0;
VACUUM x107t;
ANALYZE x107t;

-- 113a-c: the drained queue in three states.
UPDATE q113a SET state = 'done';
UPDATE q113b SET state = 'done';
VACUUM q113b;
ANALYZE q113b;
UPDATE q113c SET state = 'done';
ANALYZE q113c;

-- 114: drained to 1 %.
UPDATE q114 SET state = 'done' WHERE id % 100 <> 0;
VACUUM q114;
ANALYZE q114;

-- 115: an index built on an analysed empty table, then loaded.
INSERT INTO q115 SELECT i, 'pending' FROM generate_series(1, 1000000) i;

-- 117: drained, then VACUUM only.
UPDATE q117 SET state = 'done';
VACUUM q117;

-- 118, 119: the subset was empty at the last ANALYZE, then rows arrived.
INSERT INTO q118 SELECT 1000000 + i, 'pending' FROM generate_series(1, 50000) i;
INSERT INTO q119 SELECT 1000000 + i, 'pending' FROM generate_series(1, 50000) i;
ANALYZE q119;

-- 121: three ways to leave a stale row count behind.
DELETE FROM nz;
VACUUM nz;
INSERT INTO nz SELECT i FROM generate_series(1, 500000) i;
DELETE FROM nzb;
VACUUM nzb;
REINDEX INDEX nzb_k;
INSERT INTO nzb SELECT i FROM generate_series(1, 500000) i;
TRUNCATE trunc_t;
INSERT INTO trunc_t SELECT i FROM generate_series(1, 300000) i;

-- ------------------------------------------------------- the uniform drain ---
-- 90 % of the heap blocks of every shape fixture that has no churn of its own,
-- then VACUUM and ANALYZE, so that the index has empty and deleted pages the
-- gate can see and a rebuild can give back.  The commands are generated and
-- executed one at a time because VACUUM cannot run inside a transaction block.
SELECT /* wiki_btmaint_drain_generator */ format(st.tmpl, tb.name)
  FROM (VALUES (1, 't'), (2, 't2'), (3, 'pt1'), (4, 'pd22'), (5, 'pd23'),
               (6, 'pd24'), (7, 'pd25'), (8, 'pd26'), (9, 'pd27'), (10, 'pd28'),
               (11, 'pd29'), (12, 'pd30'), (13, 'pd31'), (14, 'pw32'),
               (15, 'pd33'), (16, 'pd34'), (17, 'pd35'), (18, 'pd36'),
               (19, 'pd37'), (20, 'pd39'), (21, 'pd40'), (22, 'pd41'),
               (23, 'pd42'), (24, 'pd43'), (25, 'pd44a'), (26, 'pd44b'),
               (27, 'pd45'), (28, 'pd46'), (29, 'pi47'), (30, 'pe48'),
               (31, 'pe48b'), (32, 'pe49'), (33, 'pe49b'), (34, 'pe50'),
               (35, 'pe50b'), (36, 'pf'), (37, 'ps')) tb(n, name)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_drain */ FROM %I WHERE ((ctid::text::point)[0])::int %% 10 <> 0'),
        (2, 'VACUUM /* wiki_btmaint_drain */ %I'),
        (3, 'ANALYZE /* wiki_btmaint_drain */ %I')) st(k, tmpl)
 ORDER BY tb.n, st.k
\gexec
FIXTURES_CHURN
  cat > "$SQLD/churn_v13.sql" <<'CHURN_V13'
-- Churn phase for the deduplicate_items fixtures, drained like the other
-- shape fixtures.  Disposable, suite database of the sandbox cluster only.
SET /* wiki_btmaint_churn13_client_min_messages */ client_min_messages = warning;
SELECT /* wiki_btmaint_drain13_generator */ format(st.tmpl, tb.name)
  FROM (VALUES (1, 'pd38')) tb(n, name)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_drain */ FROM %I WHERE ((ctid::text::point)[0])::int %% 10 <> 0'),
        (2, 'VACUUM /* wiki_btmaint_drain */ %I'),
        (3, 'ANALYZE /* wiki_btmaint_drain */ %I')) st(k, tmpl)
 ORDER BY tb.n, st.k
\gexec
CHURN_V13
  cat > "$SQLD/churn_icu.sql" <<'CHURN_ICU'
-- Churn phase for the ICU fixtures, drained like the other shape fixtures.
-- Disposable, suite database of the sandbox cluster only.
SET /* wiki_btmaint_churnicu_client_min_messages */ client_min_messages = warning;
SELECT /* wiki_btmaint_drainicu_generator */ format(st.tmpl, tb.name)
  FROM (VALUES (1, 'pc51')) tb(n, name)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_drain */ FROM %I WHERE ((ctid::text::point)[0])::int %% 10 <> 0'),
        (2, 'VACUUM /* wiki_btmaint_drain */ %I'),
        (3, 'ANALYZE /* wiki_btmaint_drain */ %I')) st(k, tmpl)
 ORDER BY tb.n, st.k
\gexec
CHURN_ICU
  cat > "$SQLD/autoanalyze.sql" <<'AUTOANALYZE'
-- The simulated auto-analyze, and the mandatory-test rule behind it: a fixture
-- that changes more than 10 % of a table's heap tuples must ANALYZE it, because
-- on a server with autovacuum on the launcher would have.
--
-- The trigger is not hand-written per fixture.  It is the engine's own test,
-- n_mod_since_analyze > autovacuum_analyze_threshold +
-- autovacuum_analyze_scale_factor * reltuples, which is exactly what
-- relation_needs_vacanalyze() compares; at the defaults that is 50 + 10 % of
-- the table's estimated row count.  autovacuum is off in this cluster, so no
-- background worker can have analyzed anything, and every ANALYZE below is one
-- this fixture set asked for.
--
-- Both counter states are recorded: autoanl holds what the churn left behind,
-- autoanl_after holds the same counters once the ANALYZEs have run, which is
-- how the run proves the simulation actually fired.
-- Disposable fixtures, suite database of the sandbox cluster only.
SET /* wiki_btmaint_autoanl_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_autoanl_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_autoanl_lock_timeout */ lock_timeout = '5s';

DROP TABLE IF EXISTS autoanl;
DROP TABLE IF EXISTS autoanl_after;
CREATE TABLE autoanl AS
SELECT /* wiki_btmaint_autoanalyze_census */
       c.relname                                   AS tbl,
       GREATEST(c.reltuples, 0)::numeric           AS reltuples,
       st.n_mod_since_analyze::numeric             AS mods,
       round(current_setting('autovacuum_analyze_threshold')::numeric
             + current_setting('autovacuum_analyze_scale_factor')::numeric
               * GREATEST(c.reltuples, 0)::numeric, 1) AS threshold,
       round(100 * st.n_mod_since_analyze
             / GREATEST(c.reltuples, 1)::numeric, 1)   AS mod_pct,
       (st.n_mod_since_analyze
        > current_setting('autovacuum_analyze_threshold')::numeric
          + current_setting('autovacuum_analyze_scale_factor')::numeric
            * GREATEST(c.reltuples, 0)::numeric)       AS would_autoanalyze
  FROM pg_stat_all_tables st
  JOIN pg_class c ON c.oid = st.relid
 WHERE st.schemaname = 'public'
   AND c.relkind = 'r'
   AND c.relname NOT IN ('plan', 'snap', 'truth', 'autoanl', 'autoanl_after');

SELECT /* wiki_btmaint_autoanalyze_generator */
       format('ANALYZE /* wiki_btmaint_autoanalyze */ %I', tbl)
  FROM autoanl WHERE would_autoanalyze ORDER BY tbl
\gexec

CREATE TABLE autoanl_after AS
SELECT /* wiki_btmaint_autoanalyze_recheck */
       c.relname AS tbl, st.n_mod_since_analyze::numeric AS mods,
       GREATEST(c.reltuples, 0)::numeric AS reltuples
  FROM pg_stat_all_tables st
  JOIN pg_class c ON c.oid = st.relid
 WHERE st.schemaname = 'public' AND c.relkind = 'r'
   AND c.relname IN (SELECT tbl FROM autoanl WHERE would_autoanalyze);
AUTOANALYZE
  cat > "$SQLD/forge.sql" <<'FORGE'
-- The numbered suite's catalog forgeries, applied after the simulated
-- auto-analyze so that an ANALYZE cannot overwrite them.  Fixture 84 is a
-- partial index whose recorded entry count is deliberately wrong, which is now
-- an input the heuristic reads rather than one it ignores.
-- Disposable catalog forgery, suite database of the sandbox cluster only.
SET /* wiki_btmaint_forge_client_min_messages */ client_min_messages = warning;
UPDATE /* wiki_btmaint_forge_84 */ pg_class SET reltuples = 5000
 WHERE relname = 'f84';
SELECT /* wiki_btmaint_forge_check */ 'f84 reltuples now ' || reltuples
  FROM pg_class WHERE relname = 'f84';
FORGE
  cat > "$SQLD/edge_autoanalyze.sql" <<'EDGE_AUTOANALYZE'
-- The same simulated auto-analyze rule, applied to the edge database so that
-- no fixture group is exempt from it.  Its curve fixtures analyze themselves
-- as part of their recipe, so this census is expected to find little or
-- nothing left to do; that is a result, and it is recorded rather than assumed.
-- Disposable fixtures, edge database of the sandbox cluster only.
SET /* wiki_btmaint_eautoanl_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_eautoanl_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_eautoanl_lock_timeout */ lock_timeout = '5s';

DROP TABLE IF EXISTS eautoanl;
CREATE TABLE eautoanl AS
SELECT /* wiki_btmaint_eautoanalyze_census */
       c.relname                                   AS tbl,
       GREATEST(c.reltuples, 0)::numeric           AS reltuples,
       st.n_mod_since_analyze::numeric             AS mods,
       round(current_setting('autovacuum_analyze_threshold')::numeric
             + current_setting('autovacuum_analyze_scale_factor')::numeric
               * GREATEST(c.reltuples, 0)::numeric, 1) AS threshold,
       round(100 * st.n_mod_since_analyze
             / GREATEST(c.reltuples, 1)::numeric, 1)   AS mod_pct,
       (st.n_mod_since_analyze
        > current_setting('autovacuum_analyze_threshold')::numeric
          + current_setting('autovacuum_analyze_scale_factor')::numeric
            * GREATEST(c.reltuples, 0)::numeric)       AS would_autoanalyze
  FROM pg_stat_all_tables st
  JOIN pg_class c ON c.oid = st.relid
 WHERE st.schemaname = 'public'
   AND c.relkind = 'r'
   AND c.relname NOT IN ('ecase', 'eplan', 'eautoanl');

SELECT /* wiki_btmaint_eautoanalyze_generator */
       format('ANALYZE /* wiki_btmaint_autoanalyze */ %I', tbl)
  FROM eautoanl WHERE would_autoanalyze ORDER BY tbl
\gexec
EDGE_AUTOANALYZE
  cat > "$SQLD/edge_forge.sql" <<'EDGE_FORGE'
-- The edge database's three catalog forgeries, re-applied after the simulated
-- auto-analyze for the same reason fixture 84's is: an ANALYZE of the table
-- rewrites reltuples for the table and for every index on it, which would
-- quietly repair the two "nothing has counted this" fixtures.
-- Disposable catalog forgeries, edge database of the sandbox cluster only.
SET /* wiki_btmaint_eforge_client_min_messages */ client_min_messages = warning;
UPDATE /* wiki_btmaint_eforge_invalid */ pg_index SET indisvalid = false
 WHERE indexrelid = 'e_inv_k'::regclass;
UPDATE /* wiki_btmaint_eforge_tbl_unknown */ pg_class SET reltuples = -1
 WHERE relname = 'e_unk';
UPDATE /* wiki_btmaint_eforge_idx_unknown */ pg_class SET reltuples = -1
 WHERE relname = 'e_iunk_k';
SELECT /* wiki_btmaint_eforge_check */ relname || ' reltuples ' || reltuples
  FROM pg_class WHERE relname IN ('e_unk', 'e_iunk_k') ORDER BY relname;
EDGE_FORGE
  cat > "$SQLD/edge_build.sql" <<'EDGE_BUILD'
-- The heuristic's own acceptance fixtures: comment parsing, comment
-- preservation, the candidate filters, the three gate boundaries and the 40 %
-- decision curve.  Everything here is disposable and lives in the edge
-- database of the sandbox cluster.
SET /* wiki_btmaint_edge_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_edge_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_edge_lock_timeout */ lock_timeout = '5s';
SET /* wiki_btmaint_edge_maintenance_work_mem */ maintenance_work_mem = '256MB';

DROP TABLE IF EXISTS ecase CASCADE;
CREATE TABLE ecase(name text PRIMARY KEY, idx text, kind text, note text);
CREATE OR REPLACE FUNCTION ecase_add(n text, i text, k text, nt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS
$$ INSERT INTO ecase VALUES (n, i, k, nt) $$;

-- set_payload writes a forged baseline, so that a gate boundary can be hit
-- exactly without waiting for a table to grow.  Harness object, not part of
-- the heuristic.  itup is the index's own entry count, the third gate input.
CREATE OR REPLACE PROCEDURE set_payload(idx text, sz numeric, tup numeric,
                                        itup numeric DEFAULT 0,
                                        usr text DEFAULT NULL,
                                        raw text DEFAULT NULL)
LANGUAGE plpgsql AS $sp$
DECLARE body text;
BEGIN
  body := COALESCE(raw, '@btmaint:{"v":2,"sz":' || sz || ',"tup":' || tup
                        || ',"itup":' || itup
                        || ',"at":"2026-01-01T00:00:00+00"}');
  EXECUTE format('COMMENT ON INDEX %I IS %L', idx,
                 CASE WHEN usr IS NULL THEN body ELSE usr || E'\n' || body END);
END $sp$;

-- ---------------------------------------------------- comment parse cases ---
CREATE TABLE e_t AS SELECT i::int AS k, (i % 977)::int AS d
  FROM generate_series(1, 200000) i;
ANALYZE e_t;
CREATE INDEX e_none  ON e_t (k);
CREATE INDEX e_user  ON e_t (d);
CREATE INDEX e_bad1  ON e_t ((k + 1));
CREATE INDEX e_bad2  ON e_t ((k + 2));
CREATE INDEX e_bad3  ON e_t ((k + 3));
CREATE INDEX e_bad4  ON e_t ((k + 4));
CREATE INDEX e_bad5  ON e_t ((k + 5));
CREATE INDEX e_bad6  ON e_t ((k + 6));
CREATE INDEX e_bad7  ON e_t ((k + 7));
CREATE INDEX e_bad8  ON e_t ((k + 12));
CREATE INDEX e_bad9  ON e_t ((k + 13));
CREATE INDEX e_quote ON e_t ((k + 8));
CREATE INDEX e_long  ON e_t ((k + 9));
CREATE INDEX e_mid   ON e_t ((k + 10));
CREATE INDEX e_two   ON e_t ((k + 11));

COMMENT ON INDEX e_user IS 'human note, kept verbatim';
CALL set_payload('e_bad1', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":,"tup":1,"itup":1}');
CALL set_payload('e_bad2', 0, 0, 0, NULL, '@btmaint:{"v":9,"sz":1,"tup":1,"itup":1}');
CALL set_payload('e_bad3', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":"big","tup":1,"itup":1}');
CALL set_payload('e_bad4', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":1');
CALL set_payload('e_bad5', 0, 0, 0, NULL, '@btmaint:not json at all');
CALL set_payload('e_bad6', 0, 0, 0, NULL, '@btmaint:{"v":2,"tup":1,"itup":1}');
CALL set_payload('e_bad7', 0, 0, 0, NULL,
                 '@btmaint:{"v":2,"sz":99999999999999999999999999999999,"tup":1,"itup":1}');
-- The migration case: a well-formed payload in the format this page filed
-- before the index entry count was stored.  It must read invalid, because a
-- v1 baseline cannot answer the index tuple gate.
CALL set_payload('e_bad8', 0, 0, 0, NULL, '@btmaint:{"v":1,"sz":1,"tup":1}');
-- A v2 payload with the itup key missing, which is the same defect arriving
-- without the version number moving.
CALL set_payload('e_bad9', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":1,"tup":1}');
CALL set_payload('e_quote', 0, 0, 0,
                 'quote '' backslash \ percent %s newline'
                 || E'\n' || 'second line of the human note');
CALL set_payload('e_long', 0, 0, 0, repeat('L', 8000));
COMMENT ON INDEX e_mid IS '@btmaint:{"v":2,"sz":1,"tup":1,"itup":1} and text after it';
-- COMMENT takes a string literal, never an expression, so the two-marker case
-- is written as one literal with an embedded newline.
COMMENT ON INDEX e_two IS E'@btmaint:{"v":2,"sz":1,"tup":1,"itup":1}\n@btmaint:{"v":2,"sz":2,"tup":2,"itup":2}';
SELECT ecase_add('absent payload', 'e_none', 'parse', 'no comment at all');
SELECT ecase_add('user comment only', 'e_user', 'parse', 'human text, no marker');
SELECT ecase_add('empty sz', 'e_bad1', 'parse', 'malformed JSON');
SELECT ecase_add('wrong format version', 'e_bad2', 'parse', 'v = 9');
SELECT ecase_add('non-numeric sz', 'e_bad3', 'parse', 'quoted string');
SELECT ecase_add('truncated payload', 'e_bad4', 'parse', 'no closing brace');
SELECT ecase_add('marker without JSON', 'e_bad5', 'parse', 'free text');
SELECT ecase_add('missing sz key', 'e_bad6', 'parse', 'tup and itup only');
SELECT ecase_add('32-digit sz', 'e_bad7', 'parse', 'longer than the regex bound');
SELECT ecase_add('v1 payload, no index count', 'e_bad8', 'parse', 'the migration case');
SELECT ecase_add('missing itup key', 'e_bad9', 'parse', 'v2 shape, key absent');
SELECT ecase_add('quoting round trip', 'e_quote', 'parse', 'quote, backslash, percent, newline');
SELECT ecase_add('8000-character note', 'e_long', 'parse', 'toasted description');
SELECT ecase_add('marker mid-string', 'e_mid', 'parse', 'text after the payload');
SELECT ecase_add('two markers', 'e_two', 'parse', 'both must go');

-- ------------------------------------------------------ candidate filters ---
CREATE TABLE e_am AS SELECT i::int AS k, point(i, i) AS p, (i % 97)::int AS d,
       to_tsvector('simple', 'w' || i) AS tv
  FROM generate_series(1, 200000) i;
ANALYZE e_am;
CREATE INDEX e_am_hash   ON e_am USING hash (k);
CREATE INDEX e_am_gist   ON e_am USING gist (p);
CREATE INDEX e_am_spgist ON e_am USING spgist (p);
CREATE INDEX e_am_gin    ON e_am USING gin (tv);
CREATE INDEX e_am_brin   ON e_am USING brin (k);
CREATE INDEX e_am_btree  ON e_am (d);
SELECT ecase_add('hash index', 'e_am_hash', 'filter', 'pgstatindex refuses');
SELECT ecase_add('gist index', 'e_am_gist', 'filter', 'pgstatindex refuses');
SELECT ecase_add('spgist index', 'e_am_spgist', 'filter', 'pgstatindex refuses');
SELECT ecase_add('gin index', 'e_am_gin', 'filter', 'pgstatindex refuses');
SELECT ecase_add('brin index', 'e_am_brin', 'filter', 'pgstatindex refuses');
SELECT ecase_add('btree control on the same table', 'e_am_btree', 'filter', 'must be a candidate');

CREATE TABLE e_part(k int, v text) PARTITION BY RANGE (k);
CREATE TABLE e_part_1 PARTITION OF e_part FOR VALUES FROM (1) TO (100001);
INSERT INTO e_part SELECT i, lpad(i::text, 20, '0') FROM generate_series(1, 100000) i;
ANALYZE e_part;
CREATE INDEX e_part_k ON e_part (k);
SELECT ecase_add('partitioned index', 'e_part_k', 'filter', 'relkind I, no storage');
SELECT ecase_add('leaf partition index', 'e_part_1_k_idx', 'filter', 'relkind i, a candidate');

CREATE TABLE e_inv AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_inv;
CREATE INDEX e_inv_k ON e_inv (k);
UPDATE pg_index SET indisvalid = false
 WHERE indexrelid = 'e_inv_k'::regclass;    -- disposable catalog forgery
SELECT ecase_add('invalid index', 'e_inv_k', 'filter', 'forged indisvalid = false');

-- Both "nothing has counted this" fixtures get a readable baseline first, so
-- that the gate they disable is a gate that would otherwise have an opinion,
-- and the reason appears in notes instead of the row reading 'initialize'.
CREATE TABLE e_unk AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_unk;
CREATE INDEX e_unk_k ON e_unk (k);
CREATE TABLE e_iunk AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_iunk;
CREATE INDEX e_iunk_k ON e_iunk (k);
DO $eu$
DECLARE sz numeric; tup numeric; itup numeric;
BEGIN
  sz  := pg_relation_size('e_unk_k'::regclass);
  SELECT reltuples::numeric INTO tup  FROM pg_class WHERE relname = 'e_unk';
  SELECT reltuples::numeric INTO itup FROM pg_class WHERE relname = 'e_unk_k';
  CALL set_payload('e_unk_k', sz, tup, itup);
  sz  := pg_relation_size('e_iunk_k'::regclass);
  SELECT reltuples::numeric INTO tup  FROM pg_class WHERE relname = 'e_iunk';
  SELECT reltuples::numeric INTO itup FROM pg_class WHERE relname = 'e_iunk_k';
  CALL set_payload('e_iunk_k', sz, tup, itup);
END $eu$;
UPDATE pg_class SET reltuples = -1 WHERE relname = 'e_unk';     -- disposable forgery
UPDATE pg_class SET reltuples = -1 WHERE relname = 'e_iunk_k';  -- disposable forgery
SELECT ecase_add('table reltuples unknown', 'e_unk_k', 'filter', 'forged reltuples = -1');
SELECT ecase_add('index reltuples unknown', 'e_iunk_k', 'filter', 'forged index reltuples = -1');

-- --------------------------------------------------------- gate boundaries ---
-- One 200,000-row table, ten indexes, ten forged baselines.  The index size
-- and both entry counts are read back and the baseline is set so that the
-- ratio is exactly on, or just under, each of the three thresholds.  Only one
-- input is moved per fixture; the other two are set to the current value so
-- they cannot fire.
CREATE TABLE e_gate AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_gate;
CREATE INDEX e_g_size_on  ON e_gate (k);
CREATE INDEX e_g_size_off ON e_gate ((k + 1));
CREATE INDEX e_g_up_on    ON e_gate ((k + 2));
CREATE INDEX e_g_up_off   ON e_gate ((k + 3));
CREATE INDEX e_g_dn_on    ON e_gate ((k + 4));
CREATE INDEX e_g_dn_off   ON e_gate ((k + 5));
CREATE INDEX e_g_iup_on   ON e_gate ((k + 6));
CREATE INDEX e_g_iup_off  ON e_gate ((k + 7));
CREATE INDEX e_g_idn_on   ON e_gate ((k + 8));
CREATE INDEX e_g_idn_off  ON e_gate ((k + 9));
DO $eg$
DECLARE sz numeric; tup numeric; itup numeric;
BEGIN
  sz  := pg_relation_size('e_g_size_on'::regclass);
  tup := (SELECT reltuples::numeric FROM pg_class WHERE relname = 'e_gate');
  -- every index here is non-partial on the same table, so CREATE INDEX wrote
  -- the same entry count into all ten of them
  itup := (SELECT reltuples::numeric FROM pg_class WHERE relname = 'e_g_size_on');
  -- size gate: current / baseline >= 1.20 fires
  CALL set_payload('e_g_size_on',  floor(sz / 1.20),    tup, itup);
  CALL set_payload('e_g_size_off', ceil(sz / 1.20) + 1, tup, itup);
  -- table tuple gate: |current - baseline| >= 0.20 * baseline fires
  CALL set_payload('e_g_up_on',  sz, floor(tup / 1.20),        itup);
  CALL set_payload('e_g_up_off', sz, ceil(tup / 1.1999) + 1,   itup);
  CALL set_payload('e_g_dn_on',  sz, ceil(tup / 0.80),         itup);
  CALL set_payload('e_g_dn_off', sz, floor(tup / 0.8001) - 1,  itup);
  -- index tuple gate: the same arithmetic on the index's own entry count
  CALL set_payload('e_g_iup_on',  sz, tup, floor(itup / 1.20));
  CALL set_payload('e_g_iup_off', sz, tup, ceil(itup / 1.1999) + 1);
  CALL set_payload('e_g_idn_on',  sz, tup, ceil(itup / 0.80));
  CALL set_payload('e_g_idn_off', sz, tup, floor(itup / 0.8001) - 1);
END $eg$;
SELECT ecase_add('size ratio exactly 1.20', 'e_g_size_on', 'gate', 'must measure');
SELECT ecase_add('size ratio just under 1.20', 'e_g_size_off', 'gate', 'must skip');
SELECT ecase_add('table tuples up by 20%', 'e_g_up_on', 'gate', 'must measure');
SELECT ecase_add('table tuples up by just under 20%', 'e_g_up_off', 'gate', 'must skip');
SELECT ecase_add('table tuples down by 20%', 'e_g_dn_on', 'gate', 'must measure');
SELECT ecase_add('table tuples down by just under 20%', 'e_g_dn_off', 'gate', 'must skip');
SELECT ecase_add('index tuples up by 20%', 'e_g_iup_on', 'gate', 'must measure');
SELECT ecase_add('index tuples up by just under 20%', 'e_g_iup_off', 'gate', 'must skip');
SELECT ecase_add('index tuples down by 20%', 'e_g_idn_on', 'gate', 'must measure');
SELECT ecase_add('index tuples down by just under 20%', 'e_g_idn_off', 'gate', 'must skip');

-- ------------------------------------------------- the 40 % decision curve ---
-- Nine 500,000-row tables, one index each, drained by a known fraction and
-- vacuumed, with a baseline that makes the tuple gate fire everywhere.  The
-- curve is wasted_pct against the fraction deleted, and the action it produces.
DO $dc$
DECLARE f int; sz numeric; tup numeric; itup numeric;
BEGIN
  FOR f IN SELECT unnest(ARRAY[10, 20, 30, 40, 50, 60, 70, 80, 90]) LOOP
    EXECUTE format('CREATE TABLE e_del%s AS SELECT i::int AS k
                      FROM generate_series(1, 500000) i', f);
    EXECUTE format('ANALYZE e_del%s', f);
    EXECUTE format('CREATE INDEX e_del%s_k ON e_del%s (k)', f, f);
    -- the as-built baseline, stored the way a first run would store it.  A
    -- CALL argument cannot be a subquery, so all three values are read first.
    sz := pg_relation_size(format('e_del%s_k', f)::regclass);
    SELECT reltuples::numeric INTO tup FROM pg_class
      WHERE relname = format('e_del%s', f);
    SELECT reltuples::numeric INTO itup FROM pg_class
      WHERE relname = format('e_del%s_k', f);
    CALL set_payload(format('e_del%s_k', f), sz, tup, itup);
    EXECUTE format('SELECT ecase_add(''%s%% of the rows deleted'', ''e_del%s_k'',
                                     ''curve'', ''vacuumed, not reindexed'')', f, f);
  END LOOP;
END $dc$;
EDGE_BUILD
  cat > "$SQLD/edge_churn.sql" <<'EDGE_CHURN'
-- The 40 % decision curve: delete a known, scattered fraction of every
-- e_delN table, then VACUUM and ANALYZE so the entries are really gone.
-- Scattered deletion is the low-density case the wasted-space formula is
-- about; contiguous deletion, which empties whole pages instead, is fixtures
-- 77 and 91 of the numbered suite.  Generated one command at a time because
-- VACUUM cannot run inside a transaction block.
-- Disposable, edge database of the sandbox cluster only.
SET /* wiki_btmaint_edgechurn_client_min_messages */ client_min_messages = warning;
SELECT /* wiki_btmaint_curve_generator */ format(st.tmpl, tb.f)
  FROM (VALUES (10), (20), (30), (40), (50), (60), (70), (80), (90)) tb(f)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_curve */ FROM e_del%1$s WHERE k %% 100 < %1$s'),
        (2, 'VACUUM /* wiki_btmaint_curve */ e_del%1$s'),
        (3, 'ANALYZE /* wiki_btmaint_curve */ e_del%1$s')) st(k, tmpl)
 ORDER BY tb.f, st.k
\gexec
EDGE_CHURN
  note "$(ls -1 "$SQLD" | tr "\n" " ")"
}

# ---------------------------------------------------------------- texts ------
stage_texts() {
  say "the page's two texts, hashed, and the one-edit harness view"
  [ -f "$PAGE" ] || die "no page at $PAGE; set PAGE or run from the repository root"
  md_block sql 1 "$PAGE" > "$SQLD/report.sql"
  md_block sql 2 "$PAGE" > "$SQLD/apply.sql"
  [ -s "$SQLD/report.sql" ] || die "block 1 of $PAGE is empty"
  [ -s "$SQLD/apply.sql" ]  || die "block 2 of $PAGE is empty"
  : > "$OUT/hashes.txt"
  local h1 h2
  h1=$(sha256sum < "$SQLD/report.sql" | cut -d' ' -f1)
  h2=$(sha256sum < "$SQLD/apply.sql"  | cut -d' ' -f1)
  printf 'report %s %s\n' "$h1" \
    "$([ "$h1" = "$BASE_REPORT" ] && echo match || echo DIFFERS)" >> "$OUT/hashes.txt"
  printf 'apply  %s %s\n' "$h2" \
    "$([ "$h2" = "$BASE_APPLY" ] && echo match || echo DIFFERS)" >> "$OUT/hashes.txt"
  printf 'report lines=%s bytes=%s\n' "$(grep -c '' "$SQLD/report.sql")" \
    "$(wc -c < "$SQLD/report.sql")" >> "$OUT/hashes.txt"
  printf 'apply  lines=%s bytes=%s\n' "$(grep -c '' "$SQLD/apply.sql")" \
    "$(wc -c < "$SQLD/apply.sql")" >> "$OUT/hashes.txt"
  # The shared pipeline: from "WITH params AS (" through the end of the staged
  # CTE, which is where the two texts part company - step 1 goes on to measure
  # in SQL, step 2 hands the snapshot to its driver.  The page claims this
  # region is byte-identical in both; this is the check that makes the claim
  # auditable.
  sed -n '/^WITH params AS ($/,/^      FROM decided d$/p' \
      "$SQLD/report.sql" > "$OUT/pipeline_report.sql"
  sed -n '/^WITH params AS ($/,/^      FROM decided d$/p' \
      "$SQLD/apply.sql" > "$OUT/pipeline_apply.sql"
  printf 'pipeline report %s\npipeline apply  %s\npipeline lines  %s\n' \
    "$(sha256sum < "$OUT/pipeline_report.sql" | cut -d' ' -f1)" \
    "$(sha256sum < "$OUT/pipeline_apply.sql" | cut -d' ' -f1)" \
    "$(grep -c '' "$OUT/pipeline_report.sql")" >> "$OUT/hashes.txt"
  if cmp -s "$OUT/pipeline_report.sql" "$OUT/pipeline_apply.sql"; then
    printf 'pipeline identical yes\n' >> "$OUT/hashes.txt"
  else
    printf 'pipeline identical NO\n' >> "$OUT/hashes.txt"
    diff "$OUT/pipeline_report.sql" "$OUT/pipeline_apply.sql" > "$OUT/pipeline_diff.txt"
  fi
  plan_view "$SQLD/report.sql" > "$SQLD/plan_view.sql"
  cat "$OUT/hashes.txt" >&2
}

# ---------------------------------------------------------------- facts ------
# Every version-local fact the two texts depend on, discovered on the running
# server rather than assumed.  The 12 leg runs the same stage, which is how the
# two servers are compared.
stage_facts() {
  say "version-local facts"
  : > "$OUT/facts.txt"
  local fact
  fact() { printf '%-34s %s\n' "$1" "$2" >> "$OUT/facts.txt"; }
  fact server_version_num "$(s suite 'SHOW server_version_num')"
  fact block_size "$(s suite 'SHOW block_size')"
  fact materialized_cte_accepted \
    "$(s suite 'WITH x AS MATERIALIZED (SELECT 1) SELECT count(*) FROM x' 2>&1 | tail -1)"
  fact pgstatindex_version \
    "$(s suite "SELECT extversion FROM pg_extension WHERE extname = 'pgstattuple'")"
  fact has_pg_input_is_valid \
    "$(s suite "SELECT count(*) FROM pg_proc WHERE proname = 'pg_input_is_valid'")"
  fact has_force_next_flush \
    "$(s suite "SELECT count(*) FROM pg_proc WHERE proname = 'pg_stat_force_next_flush'")"
  q suite 'DROP TABLE IF EXISTS zz_d' > /dev/null 2>&1
  q suite 'CREATE TABLE zz_d(k int)' > /dev/null
  fact deduplicate_items_reloption \
    "$(errf suite 'CREATE INDEX zz_di ON zz_d (k) WITH (deduplicate_items = off)')"
  q suite 'DROP TABLE IF EXISTS zz_d' > /dev/null 2>&1
  fact maintain_privilege \
    "$(errf suite "SELECT has_table_privilege('pg_class', 'MAINTAIN')")"
  fact icu_collation \
    "$(errf suite "CREATE COLLATION zz_icu (provider = icu, locale = 'und')")"
  q suite 'DROP COLLATION IF EXISTS zz_icu' > /dev/null 2>&1
  # reltuples on a table nothing has counted, and after each writer.
  q suite 'DROP TABLE IF EXISTS zz_rt' > /dev/null 2>&1
  q suite 'CREATE TABLE zz_rt(k int)' > /dev/null
  fact reltuples_after_create "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'INSERT INTO zz_rt SELECT i FROM generate_series(1,1000) i' > /dev/null
  fact reltuples_after_insert "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'CREATE INDEX zz_rt_k ON zz_rt (k)' > /dev/null
  fact reltuples_after_create_index "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'ANALYZE zz_rt' > /dev/null
  fact reltuples_after_analyze "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'TRUNCATE zz_rt' > /dev/null
  fact reltuples_after_truncate "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'INSERT INTO zz_rt SELECT i FROM generate_series(1,2000) i' > /dev/null
  q suite 'REINDEX INDEX zz_rt_k' > /dev/null
  fact reltuples_after_reindex "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  # The index's own reltuples, which is the third gate input: who writes it,
  # what each writer writes, and what a partial index gets.  A plain index and
  # a partial index on the same 10,000-row table, where the predicate selects
  # exactly one row in five.
  fact autovacuum_analyze_threshold "$(s suite 'SHOW autovacuum_analyze_threshold')"
  fact autovacuum_analyze_scale_factor "$(s suite 'SHOW autovacuum_analyze_scale_factor')"
  q suite 'DROP TABLE IF EXISTS zz_it' > /dev/null 2>&1
  q suite 'CREATE TABLE zz_it AS SELECT i::int AS k, (i % 5 = 0) AS hot
             FROM generate_series(1,10000) i' > /dev/null
  q suite 'CREATE INDEX zz_it_all ON zz_it (k)' > /dev/null
  q suite 'CREATE INDEX zz_it_part ON zz_it (k) WHERE hot' > /dev/null
  local ir
  ir() { s suite "SELECT reltuples FROM pg_class WHERE relname='$1'"; }
  fact idx_reltuples_after_build "plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'ANALYZE zz_it' > /dev/null
  fact idx_reltuples_after_analyze \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'DELETE FROM zz_it WHERE hot AND k % 50 <> 0' > /dev/null
  q suite 'VACUUM zz_it' > /dev/null
  fact idx_reltuples_after_vacuum \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'ANALYZE zz_it' > /dev/null
  fact idx_reltuples_after_analyze2 \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  # A VACUUM with nothing to delete is cleanup-only, and an index AM that
  # reports an estimated count leaves pg_class alone: the count can therefore
  # be older than the last VACUUM.
  q suite 'UPDATE zz_it SET k = k + 100000 WHERE k % 1000 = 0' > /dev/null
  q suite 'VACUUM zz_it' > /dev/null
  fact idx_reltuples_after_small_vacuum \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'REINDEX INDEX zz_it_part' > /dev/null
  fact idx_reltuples_after_reindex "partial=$(ir zz_it_part)"
  q suite 'DROP TABLE IF EXISTS zz_it' > /dev/null 2>&1
  # Transaction control inside DO, which the apply block needs for its COMMIT,
  # and the two commands that cannot be reached from inside one.
  fact commit_inside_do \
    "$(errf suite 'DO $x$ BEGIN PERFORM 1; COMMIT; END $x$;')"
  fact commit_inside_do_in_xact \
    "$(errf suite 'BEGIN; DO $x$ BEGIN PERFORM 1; COMMIT; END $x$; COMMIT;')"
  fact reindex_plain_inside_do \
    "$(errf suite 'DO $x$ BEGIN EXECUTE $q$REINDEX INDEX zz_rt_k$q$; END $x$;')"
  fact reindex_conc_inside_do \
    "$(errf suite 'DO $x$ BEGIN EXECUTE $q$REINDEX INDEX CONCURRENTLY zz_rt_k$q$; END $x$;')"
  fact reindex_conc_top_level "$(errf suite 'REINDEX INDEX CONCURRENTLY zz_rt_k;')"
  fact vacuum_inside_do \
    "$(errf suite 'DO $x$ BEGIN EXECUTE $q$VACUUM zz_rt$q$; END $x$;')"
  fact comment_expression \
    "$(errf suite "COMMENT ON INDEX zz_rt_k IS 'a' || 'b';")"
  fact comment_literal "$(errf suite "COMMENT ON INDEX zz_rt_k IS 'ab';")"
  q suite 'DROP TABLE IF EXISTS zz_rt' > /dev/null 2>&1
  cat "$OUT/facts.txt" >&2
}

# ---------------------------------------------------------------- suite ------
# The ported numbered suite, in eight steps: build, baseline, churn,
# auto-analyze, forge, decide, act, oracle.  Steps 2 and 7 run the page's apply
# block exactly as filed.
stage_suite() {
  say "the ported numbered suite: tests 1-17, 18-91 and controls 92-121"
  q suite 'DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public' > /dev/null
  q suite 'CREATE EXTENSION IF NOT EXISTS pgstattuple' > /dev/null
  f suite "$SQLD/harness.sql"        || die "harness failed"
  f suite "$SQLD/fixtures_build.sql" > "$OUT/suite_build.log" 2>&1 \
    || { tail -5 "$OUT/suite_build.log" >&2; die "build fixtures failed"; }
  local have13=no haveicu=no
  if [ "$(s suite 'SHOW server_version_num')" -ge 130000 ]; then
    f suite "$SQLD/fixtures_v13.sql" >> "$OUT/suite_build.log" 2>&1 \
      && have13=yes || note "v13 fixtures failed, skipped"
  else
    note "server is older than 13: the four deduplicate_items fixtures are skipped"
  fi
  if f suite "$SQLD/fixtures_icu.sql" >> "$OUT/suite_build.log" 2>&1; then
    haveicu=yes
  else
    note "no ICU in this build: the five ICU fixtures are skipped"
  fi
  printf 'v13_fixtures=%s icu_fixtures=%s\n' "$have13" "$haveicu" > "$OUT/suite_groups.txt"
  note "$(s suite 'SELECT count(*) || $$ planned fixtures$$ FROM plan')"

  q suite 'CALL take_snap($$built$$)' || die "snapshot built failed"
  say "baseline: the filed apply block, first run"
  f suite "$SQLD/apply.sql" > "$OUT/apply_init.log" 2>&1 \
    || { tail -5 "$OUT/apply_init.log" >&2; die "apply (init) failed"; }
  grep -c 'initialize=' "$OUT/apply_init.log" > /dev/null
  tail -1 "$OUT/apply_init.log" >&2
  q suite 'CALL take_snap($$init$$)' || die "snapshot init failed"

  say "churn"
  f suite "$SQLD/fixtures_churn.sql" > "$OUT/suite_churn.log" 2>&1 \
    || { tail -5 "$OUT/suite_churn.log" >&2; die "churn failed"; }
  [ "$have13" = yes ] && f suite "$SQLD/churn_v13.sql" >> "$OUT/suite_churn.log" 2>&1
  [ "$haveicu" = yes ] && f suite "$SQLD/churn_icu.sql" >> "$OUT/suite_churn.log" 2>&1

  # The mandatory-test rule: a fixture that moved more than 10 % of a table's
  # heap tuples gets an ANALYZE, because autovacuum would have run one.  The
  # churn ran in psql processes that have now exited, so their statistics are
  # flushed; one second of grace covers the collector interval of the older
  # server, whose reader also waits for a fresh file.
  say "simulated auto-analyze: the engine's own threshold on its own counters"
  sleep 1
  f suite "$SQLD/autoanalyze.sql" > "$OUT/autoanalyze.log" 2>&1 \
    || { tail -5 "$OUT/autoanalyze.log" >&2; die "simulated auto-analyze failed"; }
  t suite "SELECT /* wiki_btmaint_autoanalyze_report */ tbl, reltuples, mods,
                  threshold, mod_pct, would_autoanalyze
             FROM autoanl ORDER BY would_autoanalyze DESC, mods DESC, tbl" \
    > "$OUT/autoanalyze.txt" 2>&1
  { printf 'tables considered  %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl')"
    printf 'analyzed           %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl WHERE would_autoanalyze')"
    printf 'left alone         %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl WHERE NOT would_autoanalyze')"
    printf 'counters reset     %s of %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl_after WHERE mods = 0')" \
      "$(s suite 'SELECT count(*) FROM autoanl_after')"
    printf 'mod_pct of those analyzed, min/median/max %s\n' \
      "$(s suite "SELECT min(mod_pct) || ' / ' ||
                         percentile_disc(0.5) WITHIN GROUP (ORDER BY mod_pct) || ' / ' ||
                         max(mod_pct) FROM autoanl WHERE would_autoanalyze")"
    printf 'largest mod_pct left alone %s\n' \
      "$(s suite "SELECT COALESCE(max(mod_pct)::text, 'none')
                    FROM autoanl WHERE NOT would_autoanalyze")"; } \
    >> "$OUT/autoanalyze.txt"
  [ "$(s suite 'SELECT count(*) FROM autoanl WHERE would_autoanalyze')" -gt 0 ] \
    || die "no table crossed the auto-analyze threshold: the churn counters are not visible"
  tail -6 "$OUT/autoanalyze.txt" >&2

  say "forge: the catalog forgeries an ANALYZE would have overwritten"
  f suite "$SQLD/forge.sql" > "$OUT/forge.log" 2>&1 || die "forge failed"
  cat "$OUT/forge.log" >&2
  q suite 'CALL take_snap($$churned$$)' || die "snapshot churned failed"

  say "decide: the filed report statement, as filed and through the one-edit view"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d suite -f "$SQLD/report.sql" \
    > "$OUT/report_churned.txt" 2>&1 || die "filed report failed"
  f suite "$SQLD/plan_view.sql" || die "plan_v failed"
  f suite /dev/stdin <<'SQL' || die "recording the plan failed"
DELETE FROM truth;
INSERT INTO truth(idx, action, baseline, wasted_pct, notes, bytes_churned, cmd_report)
SELECT v.index_name, v.action, v.baseline, v.wasted_pct, v.notes, s.bytes,
       v.comment_command
  FROM plan_v v
  JOIN snap s ON s.phase = 'churned' AND s.idx = v.index_name;
SQL
  note "$(s suite 'SELECT count(*) || $$ decisions recorded$$ FROM truth')"

  say "act: the filed apply block, second run"
  f suite "$SQLD/apply.sql" > "$OUT/apply_act.log" 2>&1 \
    || { tail -20 "$OUT/apply_act.log" >&2; die "apply (act) failed"; }
  tail -1 "$OUT/apply_act.log" >&2
  q suite 'CALL take_snap($$applied$$)' || die "snapshot applied failed"
  f suite /dev/stdin <<'SQL' || die "recording the applied state failed"
UPDATE truth t
   SET bytes_applied = a.bytes,
       cmd_written = a.cmt,
       reindexed_by_heuristic = (a.bytes < t.bytes_churned)
  FROM snap a
 WHERE a.phase = 'applied' AND a.idx = t.idx;
SQL

  say "oracle: REINDEX INDEX on every fixture"
  f suite /dev/stdin <<'SQL' || die "ground truth failed"
SET /* wiki_btmaint_oracle_statement_timeout */ statement_timeout = '900s';
CALL ground_truth();
SQL
  note "$(s suite 'SELECT count(*) || $$ fixtures with an oracle$$ FROM truth WHERE bytes_fresh IS NOT NULL')"
}

# ---------------------------------------------------------------- edge -------
stage_edge() {
  say "the heuristic's own acceptance fixtures"
  q edge 'DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public' > /dev/null
  q edge 'CREATE EXTENSION IF NOT EXISTS pgstattuple' > /dev/null
  f edge "$SQLD/edge_build.sql" > "$OUT/edge_build.log" 2>&1 \
    || { tail -5 "$OUT/edge_build.log" >&2; die "edge fixtures failed"; }
  f edge "$SQLD/edge_churn.sql" > "$OUT/edge_churn.log" 2>&1 \
    || { tail -5 "$OUT/edge_churn.log" >&2; die "edge churn failed"; }
  # The same >10 % rule, and then the forgeries an ANALYZE would have repaired.
  sleep 1
  f edge "$SQLD/edge_autoanalyze.sql" > "$OUT/edge_autoanalyze.log" 2>&1 \
    || { tail -5 "$OUT/edge_autoanalyze.log" >&2; die "edge auto-analyze failed"; }
  f edge "$SQLD/edge_forge.sql" > "$OUT/edge_forge.log" 2>&1 \
    || die "edge forge failed"
  t edge "SELECT /* wiki_btmaint_eautoanalyze_report */ tbl, reltuples, mods,
                 threshold, mod_pct, would_autoanalyze
            FROM eautoanl ORDER BY would_autoanalyze DESC, mods DESC, tbl" \
    > "$OUT/edge_autoanalyze.txt" 2>&1
  cat "$OUT/edge_forge.log" >> "$OUT/edge_autoanalyze.txt"
  f edge "$SQLD/plan_view.sql" || die "plan_v in edge failed"

  # What the filed statement decides, and what the raw pgstatindex call does to
  # the shapes the candidate filter is there to keep out.
  q edge 'DROP TABLE IF EXISTS eplan' > /dev/null 2>&1
  f edge /dev/stdin <<'SQL' || die "edge plan failed"
CREATE TABLE eplan AS
SELECT e.name, e.idx, e.kind, e.note, v.action, v.baseline, v.wasted_pct,
       v.size_ratio, v.tuple_ratio_now, v.idx_tuple_ratio_now, v.notes,
       v.comment_command, (v.index_name IS NOT NULL) AS is_candidate
  FROM ecase e LEFT JOIN plan_v v ON v.index_name = e.idx;
SQL
  t edge "SELECT /* wiki_btmaint_edge_parse */ name, idx, action, baseline, notes
            FROM eplan WHERE kind = 'parse' ORDER BY idx" > "$OUT/edge_parse.txt" 2>&1
  t edge "SELECT /* wiki_btmaint_edge_filter */ name, idx, is_candidate, action, notes
            FROM eplan WHERE kind = 'filter' ORDER BY idx" > "$OUT/edge_filter.txt" 2>&1
  t edge "SELECT /* wiki_btmaint_edge_gate */ e.name, e.idx, p.action,
                 round(pg_relation_size(e.idx::regclass)
                       / substring(obj_description(e.idx::regclass, 'pg_class')
                                   from '\"sz\":([0-9]+)')::numeric, 6) AS size_ratio,
                 round((SELECT t.reltuples::numeric FROM pg_class c
                          JOIN pg_index x ON x.indexrelid = c.oid
                          JOIN pg_class t ON t.oid = x.indrelid
                         WHERE c.oid = e.idx::regclass)
                       / substring(obj_description(e.idx::regclass, 'pg_class')
                                   from '\"tup\":([0-9]+)')::numeric, 6) AS tuple_ratio,
                 round((SELECT c.reltuples::numeric FROM pg_class c
                         WHERE c.oid = e.idx::regclass)
                       / substring(obj_description(e.idx::regclass, 'pg_class')
                                   from '\"itup\":([0-9]+)')::numeric, 6)
                     AS idx_tuple_ratio,
                 p.notes
            FROM ecase e JOIN eplan p ON p.idx = e.idx
           WHERE e.kind = 'gate' ORDER BY e.idx" > "$OUT/edge_gate.txt" 2>&1
  t edge "SELECT /* wiki_btmaint_edge_curve */ name, idx, action, wasted_pct
            FROM eplan WHERE kind = 'curve'
           ORDER BY length(name), name" > "$OUT/edge_curve.txt" 2>&1

  say "what pgstatindex does to the shapes the filter excludes"
  : > "$OUT/edge_refusals.txt"
  local ix
  for ix in e_am_hash e_am_gist e_am_spgist e_am_gin e_am_brin e_part_k e_inv_k; do
    printf '%-12s %s\n' "$ix" \
      "$(err edge "SELECT * FROM pgstatindex('$ix')" | head -1)" >> "$OUT/edge_refusals.txt"
  done
  printf '%-12s %s\n' e_am_btree \
    "$(s edge "SELECT 'row returned, leaf_pages=' || leaf_pages FROM pgstatindex('e_am_btree')")" \
    >> "$OUT/edge_refusals.txt"
  # A temporary index belonging to another session: the filter must exclude it,
  # and a direct call must refuse it.  The owning session is this psql process,
  # which stays alive while the second one asks by OID, because the asking
  # session cannot resolve another session's pg_temp schema by name.
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=0 -d edge > "$OUT/edge_temp.txt" 2>&1 <<SQL
CREATE TEMP TABLE e_tmp AS SELECT i::int AS k FROM generate_series(1, 50000) i;
CREATE INDEX e_tmp_k ON e_tmp (k);
ANALYZE e_tmp;
SELECT 'own session candidate rows: ' || count(*) FROM plan_v WHERE index_name = 'e_tmp_k';
SELECT 'own session pgstatindex leaf_pages: ' || leaf_pages FROM pgstatindex('e_tmp_k');
\o | tr -d '\n' > $OUT/edge_temp_oid.txt
SELECT 'e_tmp_k'::regclass::oid;
\o
\! $BIN/psql -X -At -q -d edge -c "SELECT 'other session candidate rows: ' || count(*) FROM plan_v WHERE index_name = 'e_tmp_k'"
\! $BIN/psql -X -At -q -d edge -c "SELECT * FROM pgstatindex(\$(cat $OUT/edge_temp_oid.txt)::oid::regclass)" 2>&1 | tail -1
SQL

  say "comment survival across both REINDEX forms"
  : > "$OUT/edge_survival.txt"
  f edge /dev/stdin <<'SQL'
DROP TABLE IF EXISTS e_surv CASCADE;
CREATE TABLE e_surv AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_surv;
CREATE INDEX e_surv_a ON e_surv (k);
CREATE INDEX e_surv_b ON e_surv ((k + 1));
COMMENT ON INDEX e_surv_a IS E'human note\n@btmaint:{"v":2,"sz":1,"tup":2,"itup":3}';
COMMENT ON INDEX e_surv_b IS E'human note\n@btmaint:{"v":2,"sz":1,"tup":2,"itup":3}';
SQL
  printf 'before      a=%s b=%s\n' \
    "$(s edge "SELECT md5(obj_description('e_surv_a'::regclass,'pg_class'))")" \
    "$(s edge "SELECT md5(obj_description('e_surv_b'::regclass,'pg_class'))")" \
    >> "$OUT/edge_survival.txt"
  printf 'oid before  a=%s b=%s\n' \
    "$(s edge "SELECT 'e_surv_a'::regclass::oid")" \
    "$(s edge "SELECT 'e_surv_b'::regclass::oid")" >> "$OUT/edge_survival.txt"
  q edge 'REINDEX INDEX e_surv_a' > /dev/null || die "plain reindex failed"
  q edge 'REINDEX INDEX CONCURRENTLY e_surv_b' > /dev/null \
    || printf 'concurrent reindex failed\n' >> "$OUT/edge_survival.txt"
  printf 'after       a=%s b=%s\n' \
    "$(s edge "SELECT md5(obj_description('e_surv_a'::regclass,'pg_class'))")" \
    "$(s edge "SELECT md5(obj_description('e_surv_b'::regclass,'pg_class'))")" \
    >> "$OUT/edge_survival.txt"
  printf 'oid after   a=%s b=%s\n' \
    "$(s edge "SELECT 'e_surv_a'::regclass::oid")" \
    "$(s edge "SELECT 'e_surv_b'::regclass::oid")" >> "$OUT/edge_survival.txt"
  printf 'text        %s\n' \
    "$(s edge "SELECT replace(obj_description('e_surv_b'::regclass,'pg_class'), chr(10), ' | ')")" \
    >> "$OUT/edge_survival.txt"

  say "privileges: a reader, and a MAINTAIN grantee where the privilege exists"
  : > "$OUT/edge_priv.txt"
  q edge 'DROP ROLE IF EXISTS btm_reader' > /dev/null 2>&1
  q edge 'CREATE ROLE btm_reader LOGIN' > /dev/null
  # stage_edge recreated the public schema, so its owner-only default ACL has
  # to be opened before the reader can even see a relation in it.
  q edge 'GRANT USAGE ON SCHEMA public TO btm_reader' > /dev/null
  q edge 'GRANT SELECT ON e_t TO btm_reader' > /dev/null
  q edge 'GRANT EXECUTE ON FUNCTION pgstatindex(regclass) TO btm_reader' > /dev/null
  q edge 'GRANT SELECT ON plan_v TO btm_reader' > /dev/null
  printf 'owner candidate rows      %s\n' \
    "$(s edge "SELECT count(*) FROM plan_v WHERE index_name = 'e_none'")" >> "$OUT/edge_priv.txt"
  printf 'reader rows for e_none    %s\n' \
    "$("$BIN/psql" -X -At -q -U btm_reader -d edge \
        -c "SELECT count(*) FROM plan_v WHERE index_name = 'e_none'" 2>&1 | tail -1)" \
    >> "$OUT/edge_priv.txt"
  printf 'reader action on e_none   %s\n' \
    "$("$BIN/psql" -X -At -q -F ' / ' -U btm_reader -d edge \
        -c "SELECT action, notes FROM plan_v WHERE index_name = 'e_none'" 2>&1 | tail -1)" \
    >> "$OUT/edge_priv.txt"
  printf 'reader COMMENT refusal    %s\n' \
    "$("$BIN/psql" -X -q -U btm_reader -d edge \
        -c "COMMENT ON INDEX e_none IS 'x'" 2>&1 | grep -E 'ERROR' | head -1)" \
    >> "$OUT/edge_priv.txt"
  printf 'reader REINDEX refusal    %s\n' \
    "$("$BIN/psql" -X -q -U btm_reader -d edge -c 'REINDEX INDEX e_none' 2>&1 \
        | grep -E 'ERROR' | head -1)" >> "$OUT/edge_priv.txt"
  if [ "$(s edge 'SHOW server_version_num')" -ge 160000 ]; then
    q edge 'GRANT MAINTAIN ON e_t TO btm_reader' > /dev/null \
      && printf 'MAINTAIN granted          yes\n' >> "$OUT/edge_priv.txt"
    printf 'MAINTAIN REINDEX          %s\n' \
      "$("$BIN/psql" -X -q -U btm_reader -d edge -c 'REINDEX INDEX e_none' 2>&1 \
          | grep -E 'ERROR' | head -1 || echo 'accepted')" >> "$OUT/edge_priv.txt"
    printf 'MAINTAIN COMMENT          %s\n' \
      "$("$BIN/psql" -X -q -U btm_reader -d edge -c "COMMENT ON INDEX e_none IS 'x'" 2>&1 \
          | grep -E 'ERROR' | head -1)" >> "$OUT/edge_priv.txt"
    printf 'MAINTAIN action on e_none %s\n' \
      "$("$BIN/psql" -X -At -q -U btm_reader -d edge \
          -c "SELECT action FROM plan_v WHERE index_name = 'e_none'" 2>&1 | tail -1)" \
      >> "$OUT/edge_priv.txt"
  else
    printf 'MAINTAIN privilege        %s\n' \
      "$(err edge "SELECT has_table_privilege('e_t','MAINTAIN')" | head -1)" >> "$OUT/edge_priv.txt"
  fi
  printf 'index owner = table owner %s\n' \
    "$(s edge "SELECT count(*) FROM pg_class c JOIN pg_index x ON x.indexrelid = c.oid
                JOIN pg_class t ON t.oid = x.indrelid
               WHERE c.relkind = 'i' AND c.relowner <> t.relowner")" >> "$OUT/edge_priv.txt"
  # ALTER INDEX ... OWNER TO does not raise: it warns and does nothing, so the
  # probe has to read WARNING as well as ERROR, and then re-read the owners.
  printf 'ALTER INDEX OWNER answer   %s\n' \
    "$("$BIN/psql" -X -q -d edge -c 'ALTER INDEX e_none OWNER TO btm_reader' 2>&1 \
        | grep -E 'WARNING|ERROR|HINT' | tr '\n' ' ')" >> "$OUT/edge_priv.txt"
  printf 'owners after that attempt  %s\n' \
    "$(s edge "SELECT c.relowner::regrole::text || ' / ' || t.relowner::regrole::text
                 FROM pg_class c JOIN pg_index x ON x.indexrelid = c.oid
                 JOIN pg_class t ON t.oid = x.indrelid WHERE c.relname = 'e_none'")" \
    >> "$OUT/edge_priv.txt"

  say "the lock COMMENT ON INDEX takes, and what dry_run writes"
  s edge "BEGIN; COMMENT /* wiki_btmaint_lockprobe */ ON INDEX e_none IS 'lock probe';
          SELECT l.mode || ' on ' || l.relation::regclass
            FROM pg_locks l WHERE l.relation = 'e_none'::regclass; ROLLBACK" \
    > "$OUT/edge_lock.txt" 2>&1
  local before after
  before=$(s edge "SELECT md5(string_agg(objoid || ':' || description, ',' ORDER BY objoid))
                     FROM pg_description WHERE classoid = 'pg_class'::regclass")
  sed 's/^    dry_run     boolean := false;/    dry_run     boolean := true; --EDIT/' \
    "$SQLD/apply.sql" > "$SQLD/apply_dry.sql"
  cmp -s "$SQLD/apply.sql" "$SQLD/apply_dry.sql" && die "the dry_run edit matched nothing"
  f edge "$SQLD/apply_dry.sql" > "$OUT/edge_dryrun.log" 2>&1 || die "dry run failed"
  after=$(s edge "SELECT md5(string_agg(objoid || ':' || description, ',' ORDER BY objoid))
                    FROM pg_description WHERE classoid = 'pg_class'::regclass")
  printf 'pg_description md5 before %s\npg_description md5 after  %s\nunchanged %s\n' \
    "$before" "$after" "$([ "$before" = "$after" ] && echo yes || echo NO)" \
    > "$OUT/edge_dryrun.txt"
  tail -1 "$OUT/edge_dryrun.log" >> "$OUT/edge_dryrun.txt"

  say "apply for real, then twice more, to see what settles"
  f edge "$SQLD/apply.sql" > "$OUT/edge_apply1.log" 2>&1 || die "edge apply 1 failed"
  f edge "$SQLD/apply.sql" > "$OUT/edge_apply2.log" 2>&1 || die "edge apply 2 failed"
  f edge "$SQLD/apply.sql" > "$OUT/edge_apply3.log" 2>&1 || die "edge apply 3 failed"
  { printf 'run 1 %s\n' "$(tail -1 "$OUT/edge_apply1.log")"
    printf 'run 2 %s\n' "$(tail -1 "$OUT/edge_apply2.log")"
    printf 'run 3 %s\n' "$(tail -1 "$OUT/edge_apply3.log")"; } > "$OUT/edge_idempotence.txt"

  # The comment text that was actually written, against the command the report
  # printed, and the human text that had to survive both.
  t edge "SELECT /* wiki_btmaint_edge_written */ e.idx,
                 length(obj_description(e.idx::regclass, 'pg_class')) AS len,
                 replace(left(obj_description(e.idx::regclass, 'pg_class'), 120),
                         chr(10), ' | ') AS comment_now
            FROM ecase e WHERE e.kind = 'parse' ORDER BY e.idx" \
    > "$OUT/edge_written.txt" 2>&1
  # No LIKE here: the human text contains a per cent sign, which LIKE would
  # read as a wildcard.  left() on the exact original is an equality test.
  s edge "WITH orig(txt) AS (VALUES ('quote '' backslash \ percent %s newline'
                                     || chr(10) || 'second line of the human note'))
          SELECT 'quote case survived: ' ||
                 (left(obj_description('e_quote'::regclass, 'pg_class'),
                       length(o.txt)) = o.txt)
            FROM orig o" >> "$OUT/edge_written.txt" 2>&1
  s edge "SELECT 'long note kept: ' ||
                 (left(obj_description('e_long'::regclass,'pg_class'), 8000)
                  = repeat('L', 8000))" >> "$OUT/edge_written.txt" 2>&1
  s edge "SELECT 'payload bytes: ' ||
                 length(substring(obj_description('e_none'::regclass,'pg_class')
                                  from '@btmaint:\{[^}]*\}'))" >> "$OUT/edge_written.txt" 2>&1

  say "does a dump carry the baseline"
  "$BIN/pg_dump" -d edge -t e_t --schema-only > "$OUT/edge_dump.sql" 2>&1
  printf 'COMMENT ON INDEX lines in the dump: %s\nwith a payload: %s\n' \
    "$(grep -c '^COMMENT ON INDEX' "$OUT/edge_dump.sql")" \
    "$(grep -c '@btmaint:' "$OUT/edge_dump.sql")" > "$OUT/edge_dump.txt"
  cat "$OUT/edge_dump.txt" >&2
}

# ---------------------------------------------------------------- cost -------
# What a run costs, in the two states an operator actually meets: a settled
# database where every index is skipped, and one where the size gate fires
# everywhere, which is the worst case because pgstatindex reads every page of
# every gated index.  The gated state is forged by halving each stored sz; the
# suite has already been scored by the time this stage runs, and the forgery is
# recorded so no later stage reads those comments as real baselines.
stage_cost() {
  say "cost: the filed report in a settled and in a fully gated database"
  : > "$OUT/cost.txt"
  local i
  # Settle first: the oracle's rebuilds left many indexes smaller than their
  # stored baseline, which is a refresh, not a skip.  One apply run takes those
  # back to a state where the report has nothing to do, which is the state an
  # operator's scheduled run meets almost every time.
  f suite "$SQLD/apply.sql" > "$OUT/cost_settle.log" 2>&1
  printf 'settled state\n' >> "$OUT/cost.txt"
  s suite "SELECT '  gated indexes: ' || count(*) FILTER (WHERE action <> 'skip') ||
                  ' of ' || count(*) FROM plan_v" >> "$OUT/cost.txt"
  for i in 1 2 3 4 5 6; do
    printf '  run %s %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite -c '\timing on' \
           -f "$SQLD/report.sql" 2>&1 | grep -E '^Time:' | tail -1)" >> "$OUT/cost.txt"
  done
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
    -c 'EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM plan_v' 2>&1 \
    | grep -m 1 -E 'Buffers: shared' | sed 's/^ */  total /' >> "$OUT/cost.txt"
  printf 'fully gated state (every stored sz halved)\n' >> "$OUT/cost.txt"
  # Only the size input is forged: both tuple counts are written at their
  # current values, so the fully gated reading isolates one gate rather than
  # three, and the worst case it measures is still every gated index read end
  # to end.
  f suite /dev/stdin > /dev/null 2>&1 <<'SQL'
DO $fg$
DECLARE r record;
BEGIN
  FOR r IN SELECT p.idx, pg_relation_size(p.idx::regclass) AS b,
                  c.reltuples::numeric AS itup, t.reltuples::numeric AS tup
             FROM plan p
             JOIN pg_class c ON c.relname = p.idx AND c.relkind = 'i'
             JOIN pg_index x ON x.indexrelid = c.oid
             JOIN pg_class t ON t.oid = x.indrelid LOOP
    EXECUTE format('COMMENT /* wiki_btmaint_cost_forgery */ ON INDEX %I IS %L',
                   r.idx, '@btmaint:{"v":2,"sz":' || (r.b / 2)::bigint
                          || ',"tup":' || round(GREATEST(r.tup, -1))
                          || ',"itup":' || round(GREATEST(r.itup, -1))
                          || ',"at":"2026-01-01T00:00:00+00"}');
  END LOOP;
END $fg$;
SQL
  s suite "SELECT '  gated indexes: ' || count(*) FILTER (WHERE action <> 'skip') ||
                  ' of ' || count(*) FROM plan_v" >> "$OUT/cost.txt"
  for i in 1 2 3 4 5 6; do
    printf '  run %s %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite -c '\timing on' \
           -f "$SQLD/report.sql" 2>&1 | grep -E '^Time:' | tail -1)" >> "$OUT/cost.txt"
  done
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
    -c 'EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM plan_v' 2>&1 \
    | grep -m 1 -E 'Buffers: shared' | sed 's/^ */  total /' >> "$OUT/cost.txt"
  s suite "SELECT '  total index bytes: ' ||
                  pg_size_pretty(sum(pg_relation_size(idx::regclass))) FROM plan" \
    >> "$OUT/cost.txt"
  cat "$OUT/cost.txt" >&2
}

# ---------------------------------------------------------------- score ------
stage_score() {
  say "score"
  t suite "SELECT /* wiki_btmaint_verdict_rows */ num, leg, grp, idx, action,
                  wasted_pct, actual_pct, applied_pct, verdict, lost_by,
                  expected_stage, taken_stage, two_gate_stage, want_stage,
                  size_ratio, tuple_ratio, idx_tuple_ratio
             FROM verdicts ORDER BY num, leg" > "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_verdict_counts */ verdict, count(*)
             FROM verdicts GROUP BY 1 ORDER BY 2 DESC" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_group_counts */ grp,
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE action = 'reindex') AS reindexed,
                  count(*) FILTER (WHERE action = 'update')  AS updated,
                  count(*) FILTER (WHERE action = 'skip')    AS skipped,
                  count(*) FILTER (WHERE action = 'refresh') AS refreshed,
                  count(*) FILTER (WHERE verdict = 'FALSE NEGATIVE') AS false_neg,
                  round(avg(wasted_pct), 1) AS avg_wasted,
                  round(avg(actual_pct), 1) AS avg_actual
             FROM verdicts GROUP BY grp ORDER BY grp" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_accuracy */
                  count(*) AS measured,
                  round(avg(actual_pct - wasted_pct), 1)  AS mean_error,
                  round(min(actual_pct - wasted_pct), 1)  AS min_error,
                  round(max(actual_pct - wasted_pct), 1)  AS max_error,
                  count(*) FILTER (WHERE abs(actual_pct - wasted_pct) <= 15) AS within_15,
                  count(*) FILTER (WHERE wasted_pct > actual_pct) AS over_estimates
             FROM verdicts WHERE wasted_pct IS NOT NULL" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_action_counts */ action, count(*),
                  round(avg(actual_pct), 1) AS avg_actual
             FROM verdicts GROUP BY 1 ORDER BY 2 DESC" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_gate_agreement */
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE expected_stage = taken_stage) AS gate_agrees,
                  count(*) FILTER (WHERE want_stage IS NOT NULL
                                     AND want_stage = taken_stage) AS want_hit,
                  count(*) FILTER (WHERE want_stage IS NOT NULL
                                     AND want_stage <> taken_stage) AS want_miss
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_lost */ num, leg, idx, action, wasted_pct,
                  actual_pct, lost_by, size_ratio, tuple_ratio
             FROM verdicts WHERE lost_by IS NOT NULL ORDER BY actual_pct DESC" \
    > "$OUT/lost.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_disagree */ num, leg, idx, expected_stage,
                  taken_stage, baseline, size_ratio, tuple_ratio, notes
             FROM verdicts WHERE expected_stage <> taken_stage ORDER BY num, leg" \
    > "$OUT/disagree.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_want_miss */ num, leg, idx, want_stage,
                  taken_stage, action, wasted_pct, actual_pct, req
             FROM verdicts WHERE want_stage IS NOT NULL AND want_stage <> taken_stage
            ORDER BY num, leg" > "$OUT/want_miss.txt" 2>&1
  # Did step 2 store the baseline step 1 said it would?  For every action but
  # reindex the two sz values are the same state and must agree; for reindex
  # step 1 prints no command and the stored size must be the post-rebuild one.
  t suite "SELECT /* wiki_btmaint_cmd_match */ action,
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE sz_reported IS NOT NULL) AS reported,
                  count(*) FILTER (WHERE sz_reported = sz_written) AS sz_agrees,
                  count(*) FILTER (WHERE sz_written = bytes_applied) AS sz_is_current
             FROM verdicts GROUP BY action ORDER BY action" >> "$OUT/verdicts.txt" 2>&1
  # Which gate opened each measured fixture, and what the third one changed.
  t suite "SELECT /* wiki_btmaint_gate_attribution */
                  count(*) FILTER (WHERE taken_stage = 'measure') AS measured,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND size_gate_fires) AS by_size,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND tbl_gate_fires) AS by_table_tuples,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND idx_gate_fires) AS by_index_tuples,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND idx_gate_fires AND NOT size_gate_fires
                                     AND NOT tbl_gate_fires) AS index_gate_only,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND NOT size_gate_fires AND NOT tbl_gate_fires
                                     AND NOT idx_gate_fires) AS no_gate
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_two_gate_delta */
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE two_gate_stage = taken_stage) AS same_as_two_gate,
                  count(*) FILTER (WHERE two_gate_stage = 'skip'
                                     AND taken_stage = 'measure') AS newly_measured,
                  count(*) FILTER (WHERE two_gate_stage = 'skip'
                                     AND taken_stage = 'measure'
                                     AND actual_pct >= 50) AS newly_measured_bloated,
                  count(*) FILTER (WHERE two_gate_stage = 'skip'
                                     AND taken_stage = 'measure'
                                     AND actual_pct < 50) AS newly_measured_healthy,
                  count(*) FILTER (WHERE two_gate_stage = 'measure'
                                     AND taken_stage <> 'measure') AS lost_measurements
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_newly_measured */ num, leg, idx, action,
                  wasted_pct, actual_pct, verdict, idx_tuple_ratio, req
             FROM verdicts WHERE two_gate_stage = 'skip' AND taken_stage = 'measure'
            ORDER BY actual_pct DESC" > "$OUT/newly_measured.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_itup_stored */
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE itup_stored_matches) AS itup_matches_catalog,
                  count(*) FILTER (WHERE base_idx_tuples IS NULL) AS itup_missing,
                  count(*) FILTER (WHERE itup_written IS NOT NULL) AS itup_written_back
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_lost_detail */ num, leg, idx, actual_pct,
                  size_ratio, tuple_ratio, idx_tuple_ratio,
                  size_gate_fires, tbl_gate_fires, idx_gate_fires, req
             FROM verdicts WHERE lost_by IS NOT NULL
            ORDER BY actual_pct DESC" >> "$OUT/lost.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_payload_health */
                  count(*) AS indexes,
                  count(*) FILTER (WHERE payload IS NOT NULL) AS with_payload,
                  count(*) FILTER (WHERE base_bytes IS NULL) AS unparseable,
                  count(*) FILTER (WHERE cmt !~ '@btmaint:') AS no_marker
             FROM snap WHERE phase = 'applied'" >> "$OUT/verdicts.txt" 2>&1
  tail -40 "$OUT/verdicts.txt" >&2
}

# ------------------------------------------------- expected server errors ---
# Every server-side error this suite provokes is deliberate: the refusals the
# candidate filter exists to prevent, the privilege refusals, and the ALTER
# INDEX OWNER refusal.  An allowed error is a pair, the message and a fragment
# of the statement that raised it, so the same message from another statement is
# not allowed.  Only the lines after the mark this run wrote are read.
UNEXPECTED_ERRORS=0
check_server_errors() {
  local log=$1
  [ -f "$log" ] || { note "no server log"; return 0; }
  grep -E '^[0-9-]+ [0-9:.]+ [A-Z]+ (ERROR|FATAL|PANIC)' "$log" \
    | grep -Ev 'is not a btree index|cannot access temporary tables of other sessions|index "[^"]+" is not valid|must be owner of index|permission denied for (table|index)|must be owner of table|cannot change owner of index|relation "[^"]+" does not exist|REINDEX CONCURRENTLY cannot run inside a transaction block|cannot be executed from a function|deduplicate_items|ICU is not supported|unrecognized privilege type|column "inherited" does not exist' \
    > "$OUT/server_errors.txt"
  UNEXPECTED_ERRORS=$(grep -c '' "$OUT/server_errors.txt")
  printf 'unexpected server errors: %s\n' "$UNEXPECTED_ERRORS"
  [ "$UNEXPECTED_ERRORS" -gt 0 ] && head -5 "$OUT/server_errors.txt"
  return 0
}

# ---------------------------------------------------------------- criteria ---
stage_criteria() {
  say "pass criteria"
  # Every counter comes through a file, not through -c: a dollar-quoted string
  # inside a double-quoted shell argument would have $$ replaced by the shell's
  # own process id before psql ever saw it.
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d suite -f - > "$OUT/counters.txt" 2>&1 <<'SQL'
SELECT 'planned=' || count(*) FROM plan;
SELECT '  ' || verdict || '=' || count(*) FROM verdicts GROUP BY verdict ORDER BY verdict;
SELECT '  gate agree=' || count(*) FILTER (WHERE expected_stage = taken_stage) ||
       ' disagree='    || count(*) FILTER (WHERE expected_stage <> taken_stage) ||
       ' want_hit='    || count(*) FILTER (WHERE want_stage = taken_stage) ||
       ' want_miss='   || count(*) FILTER (WHERE want_stage <> taken_stage)
  FROM verdicts;
SELECT '  action ' || action || '=' || count(*) FROM verdicts GROUP BY action ORDER BY action;
SELECT '  payload with=' || count(*) FILTER (WHERE payload IS NOT NULL) ||
       ' unparseable='   || count(*) FILTER (WHERE base_bytes IS NULL) ||
       ' no_marker='     || count(*) FILTER (WHERE cmt !~ '@btmaint:')
  FROM snap WHERE phase = 'applied';
SELECT '  false negatives=' || count(*) FILTER (WHERE verdict = 'FALSE NEGATIVE') ||
       ' of which the index tuple gate opened=' ||
       count(*) FILTER (WHERE verdict = 'FALSE NEGATIVE' AND idx_gate_fires)
  FROM verdicts;
SELECT '  measured=' || count(*) FILTER (WHERE taken_stage = 'measure') ||
       ' by size='   || count(*) FILTER (WHERE taken_stage = 'measure' AND size_gate_fires) ||
       ' by table='  || count(*) FILTER (WHERE taken_stage = 'measure' AND tbl_gate_fires) ||
       ' by index='  || count(*) FILTER (WHERE taken_stage = 'measure' AND idx_gate_fires) ||
       ' index only='|| count(*) FILTER (WHERE taken_stage = 'measure' AND idx_gate_fires
                                          AND NOT size_gate_fires AND NOT tbl_gate_fires)
  FROM verdicts;
SELECT '  versus the two-gate form: same=' ||
       count(*) FILTER (WHERE two_gate_stage = taken_stage) ||
       ' newly measured=' || count(*) FILTER (WHERE two_gate_stage = 'skip'
                                                AND taken_stage = 'measure') ||
       ' of those bloated=' || count(*) FILTER (WHERE two_gate_stage = 'skip'
                                                 AND taken_stage = 'measure'
                                                 AND actual_pct >= 50) ||
       ' healthy=' || count(*) FILTER (WHERE two_gate_stage = 'skip'
                                        AND taken_stage = 'measure'
                                        AND actual_pct < 50)
  FROM verdicts;
SELECT '  itup stored = catalog count: ' ||
       count(*) FILTER (WHERE itup_stored_matches) || ' of ' || count(*)
  FROM verdicts;
SELECT '  simulated auto-analyze: ' ||
       count(*) FILTER (WHERE would_autoanalyze) || ' of ' || count(*) ||
       ' tables analyzed, largest left alone ' ||
       COALESCE(max(mod_pct) FILTER (WHERE NOT would_autoanalyze)::text, 'none') || ' %'
  FROM autoanl;
SQL
  { printf '1. texts\n'; cat "$OUT/hashes.txt" 2>/dev/null
    printf '2. engine checks\n'; cat "$OUT/checks.txt" 2>/dev/null
    printf '3. fixture groups\n'; cat "$OUT/suite_groups.txt" 2>/dev/null
    printf '4. counters\n'; sed 's/^/   /' "$OUT/counters.txt" 2>/dev/null
    printf '5. cost\n'; sed 's/^/   /' "$OUT/cost.txt" 2>/dev/null
    printf '5a. simulated auto-analyze\n'
    sed 's/^/   /' "$OUT/autoanalyze.txt" 2>/dev/null
    printf '6. edge groups\n'
    for fl in edge_parse edge_filter edge_gate edge_curve edge_refusals \
              edge_survival edge_priv edge_lock edge_dryrun edge_idempotence \
              edge_written edge_dump edge_autoanalyze; do
      printf '   --- %s\n' "$fl"; sed 's/^/   /' "$OUT/$fl.txt" 2>/dev/null
    done
    printf '7. facts\n'; sed 's/^/   /' "$OUT/facts.txt" 2>/dev/null
  } > "$OUT/criteria.txt" 2>&1
  { printf '8. server errors\n'; check_server_errors "$OUT/server.log"; } >> "$OUT/criteria.txt" 2>&1
  tail -60 "$OUT/criteria.txt" >&2
  note "full criteria in $OUT/criteria.txt"
}

# ---------------------------------------------------------------- report -----
stage_report() {
  say "report written to $OUT"
  ls -1 "$OUT" >&2
}

# ---------------------------------------------------------------- stop -------
# -m fast disconnects clients and lets the checkpointer write a shutdown
# checkpoint, so the next start needs no recovery.  The stop is then confirmed
# the way the teardown rule asks, and the stage dies rather than report a stop
# that did not happen, so clean never deletes a live cluster.
stage_stop() {
  say "stop the server cleanly"
  [ -x "$BIN/pg_ctl" ] || { note "no server binary under $BIN"; return 0; }
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null 2>&1 \
      || die "pg_ctl -m fast stop failed"
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
  note "confirmed: no postmaster.pid, no postgres process on $DATA, socket empty"
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
  inside_tmp "$SANDBOX" || die "refusing to delete $SANDBOX outside .wiki-runtime/tmp/"
  rm -rf "$SANDBOX"; say "sandbox deleted"
}

main() {
  local stages=("$@")
  [ ${#stages[@]} -eq 0 ] && stages=(build check cluster sql texts facts suite \
                                     edge cost score criteria report)
  local st
  for st in "${stages[@]}"; do
    case $st in
      build|check|cluster|sql|texts|facts|suite|edge|cost|score|criteria|report|stop|clean)
        "stage_$st" ;;
      *) die "unknown stage: $st" ;;
    esac
  done
}

main "$@"
```

### The PostgreSQL 12 leg script

```bash
#!/usr/bin/env bash
#
# btmaint_suite_v12.sh - the PostgreSQL 12 leg of the suite behind the wiki page
# "A COMMENT-Stored Baseline B-Tree Index-Maintenance Heuristic for PostgreSQL
# 12 Through 17", in bash and SQL only.  It is the compatibility half of the
# page's claim: the two texts the page files must run on a 12 server without
# one character changed.
#
# The leg starts with stage_exact, which executes both filed texts verbatim and
# records the outcome as a result in its own right, before any fixture exists.
# Everything else version-local is discovered on the running server rather than
# assumed: stage_facts asks the questions the 17 leg asks, and the fixture
# groups needing a feature this server lacks - B-tree support function 4 and
# the deduplicate_items reloption of PostgreSQL 13, and ICU collations in a
# build configured --without-icu - are skipped and recorded as skipped.
#
# It builds 12.2 out of tree from the pinned checkout, runs the engine
# regression suites, starts an isolated cluster, takes the page's two texts out
# of the page itself, ports every numbered fixture of the sibling page's
# mandatory suite (tests 1-17, 18-91 and controls 92-121), stores an as-built
# baseline in each index comment, churns each fixture, simulates the auto-analyze
# that churn would have triggered on a server with autovacuum on, runs the
# heuristic, and scores every decision against a measured REINDEX INDEX.  A
# second group of fixtures covers this heuristic's own surface: comment parsing,
# comment preservation, the candidate filters, all three gate boundaries and the
# 40 % decision curve.
#
# The simulated auto-analyze is the mandatory-test rule that any fixture moving
# more than 10 % of a table's heap tuples must ANALYZE it.  It is not
# hand-annotated per fixture: stage_autoanalyze reads the engine's own
# n_mod_since_analyze counter and applies the engine's own threshold,
# autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * reltuples
# (50 + 0.1 * reltuples at the defaults), so the tables it analyzes are exactly
# the tables an autovacuum launcher would have analyzed.  Every table it
# considered, with its counter, its threshold and the verdict, is recorded.
#
# The pinned checkout is read only: everything this script writes lives under
# $SANDBOX (default .wiki-runtime/tmp/btmaint).
#
# Usage, from the repository root:
#   bash btmaint_suite_v12.sh                  # every stage, in order
#   bash btmaint_suite_v12.sh suite score      # selected stages
#   bash btmaint_suite_v12.sh clean            # stop and delete the sandbox
#
# Stages: build check cluster sql texts exact facts suite edge cost score
#         criteria report stop clean
#
# Environment: WIKI_ROOT PAGE SRC SANDBOX PORT JOBS
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/indexing/btree-comment-baseline-maintenance-heuristic.md}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-12}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/btmaint}"
PORT="${PORT:-55412}"
JOBS="${JOBS:-8}"

BUILD="$SANDBOX/build/12"; INST="$SANDBOX/inst/12"; DATA="$SANDBOX/data12"
OUT="$SANDBOX/out12"; SQLD="$SANDBOX/sql12"; SOCK="$SANDBOX/sock12"; BIN="$INST/bin"
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE=postgres

# SHA-256 baselines of the two fenced SQL blocks of the page, in page order:
# the report statement and the apply block.  A changed text must be re-measured
# and the hash refiled; that is the point of recording them here.
BASE_REPORT=93b64e2dd33d411951ceffd9d665a3e3e8ce3a71fde60bbca9e61c219febd0e9
BASE_APPLY=7427d62d2ca3bb43dd2f7c5a8aa97c27fc8111d65d3066594660c2808d47992d

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
# err() runs a statement that is expected to fail and prints the message only.
err() { "$BIN/psql" -X -q -v ON_ERROR_STOP=0 -d "$1" -c "$2" 2>&1 \
        | grep -E '^(ERROR|psql:.*ERROR)' | head -1; }
# errf() sends a statement through a file, so that a body full of quotes and
# dollar signs reaches the server exactly as written, and reports the first
# error line or "accepted".
errf() {
  local db=$1 out
  out=$(printf '%s\n' "$2" | "$BIN/psql" -X -q -v ON_ERROR_STOP=0 -d "$db" -f - 2>&1 \
        | grep -E 'ERROR' | head -1)
  printf '%s' "${out:-accepted}"
}

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

# plan_view <report-text> : the one documented edit.  Drop the two SET lines,
# which a view cannot carry, and wrap the rest in CREATE VIEW plan_v.  Nothing
# else is touched, so the view computes exactly what the filed statement does.
plan_view() {
  local file=$1 line
  printf 'DROP VIEW IF EXISTS plan_v;\nCREATE VIEW plan_v AS\n'
  while IFS= read -r line; do
    case $line in
      "SET /* wiki_btmaint_statement_timeout"*) continue ;;
      "SET /* wiki_btmaint_lock_timeout"*)      continue ;;
    esac
    printf '%s\n' "$line"
  done < "$file"
}

# ---------------------------------------------------------------- build ------
stage_build() {
  say "build 12.2 out of tree from $SRC"
  if [ -x "$BIN/postgres" ]; then
    note "already built: $("$BIN/postgres" --version)"; return 0
  fi
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC"
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --without-icu \
      > configure.log 2>&1 ) || { cp "$BUILD/configure.log" "$OUT/" 2>/dev/null
                                  die "configure failed, see $OUT/configure.log"; }
  ( cd "$BUILD" && make -j"$JOBS" -s > make.log 2>&1 \
      && make -s install > install.log 2>&1 ) \
    || { cp "$BUILD"/*.log "$OUT/" 2>/dev/null; die "make failed"; }
  local m
  for m in pgstattuple pageinspect amcheck; do
    ( cd "$BUILD" && make -C "contrib/$m" -s install >> install.log 2>&1 ) \
      || die "contrib/$m install failed"
  done
  cp "$BUILD"/configure.log "$BUILD"/make.log "$BUILD"/install.log "$OUT/" 2>/dev/null
  note "$("$BIN/postgres" --version)"
}

# ---------------------------------------------------------------- check ------
stage_check() {
  say "engine regression suites"
  : > "$OUT/checks.txt"
  ( cd "$BUILD" && make -s check > check_core.log 2>&1 )
  printf 'core=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' \
        "$BUILD/check_core.log" | tail -1)" >> "$OUT/checks.txt"
  local m
  for m in pgstattuple pageinspect amcheck; do
    ( cd "$BUILD" && make -s -C "contrib/$m" check > "check_$m.log" 2>&1 )
    printf '%s=%s %s\n' "$m" "$?" \
      "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' \
          "$BUILD/check_$m.log" | tail -1)" >> "$OUT/checks.txt"
  done
  cp "$BUILD"/check_*.log "$OUT/" 2>/dev/null
  local d
  for d in "$BUILD/src/test/regress" "$BUILD"/contrib/*; do
    [ -f "$d/regression.diffs" ] \
      && cp "$d/regression.diffs" "$OUT/diffs_$(basename "$d").txt"
  done
  cat "$OUT/checks.txt" >&2
}

# ---------------------------------------------------------------- cluster ----
# Cluster settings and their apply scope, all written before the first start:
#   listen_addresses, port, unix_socket_directories, shared_buffers,
#   logging_collector                                  -> PGC_POSTMASTER, restart
#   fsync, autovacuum                                  -> PGC_SIGHUP, reload
#   maintenance_work_mem, max_parallel_maintenance_workers
#                                                      -> PGC_USERSET, session
# autovacuum is off so that no background vacuum moves a fixture between the
# baseline, the churn, the decision and the rebuild oracle.
stage_cluster() {
  say "isolated cluster on port $PORT"
  mkdir -p "$OUT" "$SQLD" "$SOCK"
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    note "already running"
  else
    if [ ! -d "$DATA" ]; then
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
logging_collector = off
CONF
    fi
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w start > /dev/null \
      || die "server start failed"
  fi
  note "$(s postgres 'SELECT /* wiki_btmaint_version */ version()')"
  s postgres "SELECT /* wiki_btmaint_platform */
                     'max_data_alignment=' || max_data_alignment ||
              ' database_block_size=' || database_block_size FROM pg_control_init()" \
    | tee "$OUT/platform.txt" >&2
  printf 'uname: %s\n' "$(uname -sm)" >> "$OUT/platform.txt"
  local db
  for db in suite edge; do
    s postgres "SELECT /* wiki_btmaint_database_exists */ 1
                  FROM pg_database WHERE datname = '$db'" | grep -q 1 \
      || "$BIN/createdb" -T template0 -E UTF8 --locale=C "$db"
    q "$db" 'CREATE EXTENSION IF NOT EXISTS pgstattuple' || die "pgstattuple failed"
  done
}

# ------------------------------------------------------------------ sql ------
# Every SQL file this suite uses, written out from here so that the script is
# self-contained.  The two texts under test are NOT here: they come out of the
# page itself, in stage_texts.
stage_sql() {
  say "write the suite's SQL files to $SQLD"
  mkdir -p "$SQLD"
  cat > "$SQLD/harness.sql" <<'HARNESS'
-- Harness for the ported numbered suite.  Disposable: every object below is
-- created in the sandbox cluster's suite database and is not meant for a
-- database anyone cares about.  The harness never touches the heuristic's own
-- two texts; it only records, snapshots and scores.
SET /* wiki_btmaint_harness_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_harness_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_harness_lock_timeout */ lock_timeout = '5s';

DROP TABLE IF EXISTS plan CASCADE;
DROP TABLE IF EXISTS snap CASCADE;
DROP TABLE IF EXISTS truth CASCADE;
DROP VIEW IF EXISTS verdicts CASCADE;

-- One row per numbered fixture index.
CREATE TABLE plan(num int, leg text DEFAULT '', grp text, req text, idx text,
                  want_stage text, note text,
                  PRIMARY KEY (num, leg));

-- One row per index per phase.  phase is 'built', 'init', 'churned', 'applied'.
-- idx_tuples is the index's own pg_class.reltuples, which the heuristic now
-- reads as its third gate input; base_idx_tuples is the itup field of the
-- stored payload, parsed here independently of the filed text.
CREATE TABLE snap(phase text, idx text, idx_oid oid, bytes bigint,
                  tbl_tuples numeric, idx_tuples numeric, cmt text, payload text,
                  base_bytes numeric, base_tuples numeric, base_idx_tuples numeric,
                  PRIMARY KEY (phase, idx));

-- What the heuristic decided, and what a rebuild actually gave back.
CREATE TABLE truth(idx text PRIMARY KEY, action text, baseline text,
                   wasted_pct numeric, notes text,
                   bytes_churned bigint, bytes_applied bigint, bytes_fresh bigint,
                   reindexed_by_heuristic bool, cmd_report text, cmd_written text);

CREATE OR REPLACE FUNCTION plan_add(n int, g text, r text, i text,
                                    w text DEFAULT NULL, lg text DEFAULT '',
                                    nt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS
$$ INSERT INTO plan(num, leg, grp, req, idx, want_stage, note)
   VALUES (n, lg, g, r, i, w, nt) $$;

-- take_snap records, for every planned index, the facts the heuristic reads:
-- the file size, the table's estimated tuple count, and the baseline stored in
-- the index's own comment.  It parses the payload the same way the filed text
-- does, independently, so a disagreement is visible.
CREATE OR REPLACE PROCEDURE take_snap(ph text) LANGUAGE plpgsql AS $sn$
BEGIN
  DELETE FROM snap WHERE phase = ph;
  INSERT INTO snap
  SELECT ph, p.idx, c.oid, pg_relation_size(c.oid), t.reltuples::numeric,
         c.reltuples::numeric, d.description,
         substring(d.description from '@btmaint:(\{[^}]*\})'),
         substring(d.description from '"sz":([0-9]{1,25})[,}]')::numeric,
         substring(d.description from
                   '"tup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric,
         substring(d.description from
                   '"itup":(-?[0-9]{1,25}(?:[.][0-9]{1,10})?)[,}]')::numeric
    FROM plan p
    JOIN pg_class c ON c.relname = p.idx AND c.relkind = 'i'
    JOIN pg_index x ON x.indexrelid = c.oid
    JOIN pg_class t ON t.oid = x.indrelid
    LEFT JOIN pg_description d ON d.objoid = c.oid
                              AND d.classoid = 'pg_class'::regclass
                              AND d.objsubid = 0;
END $sn$;

-- ground_truth rebuilds every planned index and records the fresh size.  It is
-- the oracle: what REINDEX INDEX actually gives back on the churned file.
CREATE OR REPLACE PROCEDURE ground_truth() LANGUAGE plpgsql AS $gt$
DECLARE p record;
BEGIN
  FOR p IN SELECT idx FROM plan ORDER BY num, leg LOOP
    EXECUTE format('REINDEX /* wiki_btmaint_oracle */ INDEX %I', p.idx);
    UPDATE truth SET bytes_fresh = pg_relation_size(p.idx::regclass)
     WHERE idx = p.idx;
  END LOOP;
END $gt$;

-- The verdict view. actual_pct is what the rebuild of the churned file gave
-- back, measured, and is the only oracle. expected_stage recomputes the gate
-- from the recorded baseline, independently of the filed text.
CREATE OR REPLACE VIEW verdicts AS
SELECT p.num, p.leg, p.grp, p.idx, p.req, p.want_stage,
       t.action, t.baseline, t.wasted_pct, t.reindexed_by_heuristic,
       bu.bytes AS bytes_built, ic.bytes AS bytes_init,
       t.bytes_churned, t.bytes_applied, t.bytes_fresh,
       round(100.0 * (t.bytes_churned - t.bytes_fresh)
             / GREATEST(t.bytes_churned, 1), 1)                  AS actual_pct,
       round(100.0 * (t.bytes_churned - t.bytes_applied)
             / GREATEST(t.bytes_churned, 1), 1)                  AS applied_pct,
       ic.base_bytes AS base_bytes, ic.base_tuples AS base_tuples,
       ch.tbl_tuples AS churned_tuples,
       CASE WHEN ic.base_bytes > 0
            THEN round(ch.bytes / ic.base_bytes, 4) END          AS size_ratio,
       CASE WHEN ic.base_tuples > 0
            THEN round(ch.tbl_tuples / ic.base_tuples, 4) END    AS tuple_ratio,
       -- The index's own entry count: the baseline the payload stored, the
       -- count the catalog holds after the churn, and their ratio.  The
       -- payload value is what the heuristic reads; ic.idx_tuples is the live
       -- catalog value at baseline time, so the two must agree.
       ic.base_idx_tuples, ch.idx_tuples AS churned_idx_tuples,
       (ic.base_idx_tuples = round(ic.idx_tuples))               AS itup_stored_matches,
       CASE WHEN ic.base_idx_tuples > 0
            THEN round(ch.idx_tuples / ic.base_idx_tuples, 4) END AS idx_tuple_ratio,
       -- Which of the three gates fires, each recomputed here from the
       -- recorded baseline, independently of the filed text.
       (ic.base_bytes > 0 AND ch.bytes >= ic.base_bytes * 1.20)  AS size_gate_fires,
       (ch.tbl_tuples >= 0 AND ic.base_tuples >= 0
        AND (CASE WHEN ic.base_tuples = 0 THEN ch.tbl_tuples > 0
                  ELSE abs(ch.tbl_tuples - ic.base_tuples)
                       >= ic.base_tuples * 0.20 END))            AS tbl_gate_fires,
       (ch.idx_tuples >= 0 AND ic.base_idx_tuples >= 0
        AND (CASE WHEN ic.base_idx_tuples = 0 THEN ch.idx_tuples > 0
                  ELSE abs(ch.idx_tuples - ic.base_idx_tuples)
                       >= ic.base_idx_tuples * 0.20 END))        AS idx_gate_fires,
       -- The sz the report proposed to store, against the sz that was stored.
       substring(t.cmd_report from '"sz":([0-9]+)')::numeric      AS sz_reported,
       substring(t.cmd_written from '"sz":([0-9]+)')::numeric     AS sz_written,
       substring(t.cmd_written from '"itup":(-?[0-9]+)')::numeric AS itup_written,
       CASE WHEN ic.payload IS NULL                              THEN 'initialize'
            WHEN ic.base_bytes IS NULL OR ic.base_tuples IS NULL
              OR ic.base_idx_tuples IS NULL                      THEN 'initialize'
            WHEN ic.base_bytes > 0 AND ch.bytes < ic.base_bytes  THEN 'refresh'
            WHEN ic.base_bytes > 0 AND ch.bytes >= ic.base_bytes * 1.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples > 0
             AND abs(ch.tbl_tuples - ic.base_tuples) >= ic.base_tuples * 0.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples = 0 AND ch.tbl_tuples > 0
                                                                 THEN 'measure'
            WHEN ch.idx_tuples >= 0 AND ic.base_idx_tuples > 0
             AND abs(ch.idx_tuples - ic.base_idx_tuples) >= ic.base_idx_tuples * 0.20
                                                                 THEN 'measure'
            WHEN ch.idx_tuples >= 0 AND ic.base_idx_tuples = 0 AND ch.idx_tuples > 0
                                                                 THEN 'measure'
            ELSE 'skip' END                                      AS expected_stage,
       -- The two-gate form this page filed before the index count was stored,
       -- recomputed on the same fixtures so the change can be quantified.
       CASE WHEN ic.payload IS NULL                              THEN 'initialize'
            WHEN ic.base_bytes IS NULL OR ic.base_tuples IS NULL THEN 'initialize'
            WHEN ic.base_bytes > 0 AND ch.bytes < ic.base_bytes  THEN 'refresh'
            WHEN ic.base_bytes > 0 AND ch.bytes >= ic.base_bytes * 1.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples > 0
             AND abs(ch.tbl_tuples - ic.base_tuples) >= ic.base_tuples * 0.20
                                                                 THEN 'measure'
            WHEN ch.tbl_tuples >= 0 AND ic.base_tuples = 0 AND ch.tbl_tuples > 0
                                                                 THEN 'measure'
            ELSE 'skip' END                                      AS two_gate_stage,
       CASE WHEN t.action IN ('reindex', 'update') THEN 'measure'
            ELSE t.action END                                    AS taken_stage,
       -- The rebuild oracle against the 40 % decision.
       CASE WHEN t.action IS NULL                                THEN 'ABSENT'
            WHEN t.action = 'reindex'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) < 10    THEN 'CRITICAL FALSE POSITIVE'
            WHEN t.action = 'reindex'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) < 35    THEN 'FALSE POSITIVE'
            WHEN t.action <> 'reindex'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) >= 50   THEN 'FALSE NEGATIVE'
            ELSE 'PASS' END                                      AS verdict,
       -- Where a false negative was lost: the gate never measured, or the
       -- measurement read 40 % or less on a file a rebuild did shrink.
       CASE WHEN t.action IN ('skip', 'initialize', 'refresh')
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) >= 50   THEN 'gate'
            WHEN t.action = 'update'
             AND round(100.0 * (t.bytes_churned - t.bytes_fresh)
                       / GREATEST(t.bytes_churned, 1), 1) >= 50   THEN 'threshold'
            ELSE NULL END                                        AS lost_by,
       t.notes, p.note
  FROM plan p
  LEFT JOIN truth t ON t.idx = p.idx
  LEFT JOIN snap bu ON bu.phase = 'built'   AND bu.idx = p.idx
  LEFT JOIN snap ic ON ic.phase = 'init'    AND ic.idx = p.idx
  LEFT JOIN snap ch ON ch.phase = 'churned' AND ch.idx = p.idx;
HARNESS
  cat > "$SQLD/fixtures_build.sql" <<'FIXTURES_BUILD'
-- The numbered suite, build phase: every fixture up to and including the
-- creation of the index that is scored.  The churn file carries the rest of
-- each recipe, so that the heuristic can store a baseline for a freshly built
-- index and then be asked about the same index after it has been disturbed.
--
-- Disposable fixtures.  Everything below creates, forges and drops objects in
-- the suite database of the sandbox cluster.  It is not meant for a database
-- anyone cares about.
--
-- The recipes are the numbered fixtures of the PostgreSQL 17 page "Testing the
-- PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17": tests 1-17
-- (deduplication gate), 18-91 (partial indexes) and controls 92-121.  Two
-- deviations, both deliberate: pg_stat_force_next_flush() is not called,
-- because this heuristic reads pg_class.reltuples and never a cumulative
-- statistics view, and the four deduplicate_items fixtures and the four ICU
-- fixtures live in their own files, because those features do not exist in
-- every server this text has to run on.
SET /* wiki_btmaint_fixtures_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_fixtures_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_fixtures_lock_timeout */ lock_timeout = '5s';
SET /* wiki_btmaint_fixtures_maintenance_work_mem */ maintenance_work_mem = '256MB';

-- ======================================================== tests 1-17 ========
-- The deduplication gate's two 500,000-row tables and its custom operator
-- classes.  For this heuristic the group asks a different question than it did
-- for the estimator: every equal-image class must survive the same 90 % drain
-- and be measured, and the 40 % decision must agree with a rebuild.
--
-- Every fixture that needs a B-tree support function 4 lives in the
-- PostgreSQL 13 file, because support function 4 is the deduplication
-- equal-image callback: a 12 server answers "invalid function number 4, must
-- be between 1 and 3" and the whole file would abort here.  What stays is the
-- one custom operator class that declares no support function 4 at all.
CREATE OPERATOR CLASS int4_ei_none FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4);

CREATE TABLE t AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s, ((i % 5000)::numeric) AS n,
       (i % 5000)::float4 AS f4, (i % 5000)::float8 AS f8, (i % 7)::int4 AS d
  FROM generate_series(1, 500000) i;
CREATE TABLE t2 AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE t;
ANALYZE t2;

CREATE INDEX i_int4          ON t (a);
CREATE INDEX i_int8          ON t (b);
CREATE INDEX i_text_det      ON t (s);
CREATE INDEX i_numeric       ON t (n);
CREATE INDEX i_float4        ON t (f4);
CREATE INDEX i_float8        ON t (f8);
CREATE INDEX i_multi_ok      ON t (a, b);
CREATE INDEX i_multi_bad     ON t (a, n);
CREATE INDEX i_expr_num      ON t ((a::numeric));
CREATE INDEX i_inc           ON t (a) INCLUDE (d);
CREATE INDEX i_ei_none       ON t (a int4_ei_none);
CREATE UNIQUE INDEX i_uniq   ON t (u);
CREATE INDEX i2_ok           ON t2 (a, b);

SELECT plan_add(1,  'gate', 'int4 key, 100 rows per key', 'i_int4', 'measure', 'i_int4');
SELECT plan_add(2,  'gate', 'int8 key, 100 rows per key', 'i_int8', 'measure', 'i_int8');
SELECT plan_add(3,  'gate', 'text key, deterministic default collation', 'i_text_det', 'measure', 'i_text_det');
SELECT plan_add(5,  'gate', 'numeric key, no equal-image support', 'i_numeric', 'measure', 'i_numeric');
SELECT plan_add(6,  'gate', 'float4 key', 'i_float4', 'measure', 'i_float4');
SELECT plan_add(6,  'gate', 'float8 key', 'i_float8', 'measure', 'i_float8');
SELECT plan_add(7,  'gate', 'two equal-image key columns', 'i_multi_ok', 'measure', 'i_multi_ok');
SELECT plan_add(8,  'gate', 'one non-equal-image key column', 'i_multi_bad', 'measure', 'i_multi_bad');
SELECT plan_add(9,  'gate', 'expression key, numeric', 'i_expr_num', 'measure', 'i_expr_num');
SELECT plan_add(10, 'gate', 'INCLUDE column refuses deduplication', 'i_inc', 'measure', 'i_inc');
SELECT plan_add(12, 'gate', 'opclass with no support function 4', 'i_ei_none', 'measure', 'i_ei_none');
SELECT plan_add(17, 'gate', 'unique key control; carried to the 12 leg as test 17', 'i_uniq', 'measure', 'i_uniq');
SELECT plan_add(7,  'gate', 'second table, two equal-image columns', 'i2_ok', 'measure', 'i2_ok');

-- ======================================================= tests 18-21 ========
CREATE TABLE pt1 AS
SELECT i::bigint AS k, (i % 100)::int AS sel FROM generate_series(1, 1000000) i;
ANALYZE pt1;
CREATE INDEX p18 ON pt1 (k) WHERE sel < 20;
CREATE INDEX p19 ON pt1 (k) WHERE sel < 1;
CREATE INDEX p20 ON pt1 (k) WHERE sel < 10;
CREATE INDEX p21 ON pt1 (k) WHERE sel < 80;
SELECT plan_add(18, 'partial', 'baseline, subset distribution = table (20%)', 'p18', 'measure');
SELECT plan_add(19, 'partial', 'very selective, ~1%', 'p19', 'measure');
SELECT plan_add(20, 'partial', 'moderately selective, ~10%', 'p20', 'measure');
SELECT plan_add(21, 'partial', 'large subset, ~80%', 'p21', 'measure');

-- ======================================================= tests 22-33 ========
CREATE TABLE pd22 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd22;
CREATE INDEX p22 ON pd22 (k) WHERE hot;
SELECT plan_add(22, 'partial', 'highly duplicated subset, unique outside', 'p22', 'measure');

CREATE TABLE pd23 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE ((i / 5) % 100)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd23;
CREATE INDEX p23 ON pd23 (k) WHERE hot;
SELECT plan_add(23, 'partial', 'highly unique subset, duplicated outside', 'p23', 'measure');

CREATE TABLE pd24 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50000)::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd24;
CREATE INDEX p24 ON pd24 (k) WHERE hot;
SELECT plan_add(24, 'partial', 'n_distinct radically different in the subset', 'p24', 'measure');

CREATE TABLE pd25 AS SELECT (i % 100 = 0) AS hot,
       CASE WHEN i % 100 = 0 THEN ((i / 100) % 997)::int ELSE (i % 5)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd25;
CREATE INDEX p25 ON pd25 (k) WHERE hot;
SELECT plan_add(25, 'partial', 'MCV distribution differs inside the subset', 'p25', 'measure');

CREATE TABLE pd26 AS SELECT (i % 50 = 0) AS hot,
       CASE WHEN i % 50 = 0 THEN (1000000 + i)::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd26;
CREATE INDEX p26 ON pd26 (k) WHERE hot;
SELECT plan_add(26, 'partial', 'table-wide MCVs absent inside the subset', 'p26', 'measure');

CREATE TABLE pd27 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 AND i % 100 <> 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd27;
CREATE INDEX p27 ON pd27 (k) WHERE hot;
SELECT plan_add(27, 'partial', 'NULL-heavy subset, non-NULL outside', 'p27', 'measure');

CREATE TABLE pd28 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN i::bigint ELSE NULL END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd28;
CREATE INDEX p28 ON pd28 (k) WHERE hot;
SELECT plan_add(28, 'partial', 'NULL-free subset, NULL-heavy table (bigint)', 'p28', 'measure');

CREATE TABLE pd29 AS
SELECT CASE WHEN i % 5 = 0 THEN NULL ELSE lpad(i::text, 20, '0') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pd29;
CREATE INDEX p29 ON pd29 (s) WHERE s IS NULL;
SELECT plan_add(29, 'partial', 'all-NULL partial index, WHERE s IS NULL', 'p29', 'skip');

CREATE TABLE pd30 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pd30;
CREATE INDEX p30 ON pd30 (s) WHERE hot;
SELECT plan_add(30, 'partial', 'subset values wider than outside (13 against 204 bytes)', 'p30', 'measure');

CREATE TABLE pd31 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN lpad((i % 9)::text, 12, 'n')
            ELSE repeat('W', 190) || lpad(i::text, 10, '0') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pd31;
CREATE INDEX p31 ON pd31 (s) WHERE hot;
SELECT plan_add(31, 'partial', 'subset values narrower than outside', 'p31', 'measure');

CREATE TABLE pw32 AS
SELECT (i % 50 = 0) AS hot,
       CASE WHEN i % 50 = 0 THEN repeat('W', 390) || lpad(i::text, 10, '0')
            ELSE repeat('n', 18) || (i % 9)::text END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pw32;
CREATE INDEX p32 ON pw32 (s) WHERE hot;
SELECT plan_add(32, 'partial', 'extreme width mismatch (27 against 404 bytes)', 'p32', 'measure');

CREATE TABLE pd33 AS SELECT (i % 5 = 0) AS hot,
       lpad(i::text, 10 + (i % 40), 'x') AS s FROM generate_series(1, 500000) i;
ANALYZE pd33;
CREATE INDEX p33 ON pd33 (s) WHERE hot;
SELECT plan_add(33, 'partial', 'variable-width values, same range inside and out', 'p33', 'measure');

-- ======================================================= tests 34-39 ========
CREATE TABLE pd34 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd34;
CREATE INDEX p34 ON pd34 (k) WHERE hot;
SELECT plan_add(34, 'partial', 'dedup-heavy subset, 1000 rows per key', 'p34', 'measure');

CREATE TABLE pd35 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd35;
CREATE INDEX p35 ON pd35 (k) WHERE hot;
SELECT plan_add(35, 'partial', 'duplicate-heavy table, unique subset', 'p35', 'measure');

CREATE TABLE pd36 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN 42 ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd36;
CREATE INDEX p36 ON pd36 (k) WHERE hot;
SELECT plan_add(36, 'partial', 'one key group, 100,000 TIDs against a 132 cap', 'p36', 'measure');

CREATE TABLE pd37 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd37;
CREATE INDEX p37 ON pd37 (k) WHERE hot;
SELECT plan_add(37, 'partial', 'NULL deduplication, every subset key NULL', 'p37', 'measure');

CREATE TABLE pd39 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd39;
CREATE UNIQUE INDEX p39 ON pd39 (k) WHERE hot;
SELECT plan_add(39, 'partial', 'partial UNIQUE index', 'p39', 'measure');

-- ======================================================= tests 40-47 ========
CREATE TABLE pd40 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 97)::int  END AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd40;
CREATE INDEX p40 ON pd40 (a, b) WHERE hot;
SELECT plan_add(40, 'partial', 'two-column key correlated only in the subset', 'p40', 'measure');

CREATE TABLE pd41 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 97)::int  ELSE (i % 100)::int END AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd41;
CREATE INDEX p41 ON pd41 (a, b) WHERE hot;
SELECT plan_add(41, 'partial', 'two-column key independent only in the subset', 'p41', 'measure');

CREATE TABLE pd42 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50)::int ELSE i::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 50)::int ELSE i::int END AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd42;
CREATE INDEX p42 ON pd42 (a, b) WHERE hot;
SELECT plan_add(42, 'partial', 'multi-column duplicate keys in the subset', 'p42', 'measure');

CREATE TABLE pd43 AS SELECT (i % 5 = 0) AS hot, i::int AS a, (i * 2)::int AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd43;
CREATE INDEX p43 ON pd43 (a, b) WHERE hot;
SELECT plan_add(43, 'partial', 'multi-column unique keys in the subset', 'p43', 'measure');

CREATE TABLE pd44a AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
ANALYZE pd44a;
CREATE INDEX p44a ON pd44a (a, b) WHERE hot;
SELECT plan_add(44, 'partial', 'multicolumn key, no ndistinct object', 'p44a', 'measure', 'a');
CREATE TABLE pd44b AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS pd44b_nd (ndistinct) ON a, b FROM pd44b;
ANALYZE pd44b;
CREATE INDEX p44b ON pd44b (a, b) WHERE hot;
SELECT plan_add(44, 'partial', 'multicolumn key, with CREATE STATISTICS (ndistinct)', 'p44b', 'measure', 'b');

CREATE TABLE pd45 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 97)::int  ELSE (i % 100)::int END AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS pd45_nd (ndistinct) ON a, b FROM pd45;
ANALYZE pd45;
CREATE INDEX p45 ON pd45 (a, b) WHERE hot;
SELECT plan_add(45, 'partial', 'extended statistics wrong for the subset', 'p45', 'measure');

CREATE TABLE pd46 AS SELECT (i % 5 = 0) AS hot, i::int AS k, (i % 7)::int AS pay
  FROM generate_series(1, 500000) i;
ANALYZE pd46;
CREATE INDEX p46 ON pd46 (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(46, 'partial', 'partial index with INCLUDE columns', 'p46', 'measure');

CREATE TABLE pi47 AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS payload
  FROM generate_series(1, 500000) i;
ANALYZE pi47;
CREATE INDEX p47 ON pi47 (k) INCLUDE (payload) WHERE hot;
SELECT plan_add(47, 'partial', 'wide INCLUDE values inside the subset', 'p47', 'measure');

-- ======================================================= tests 48-55 ========
CREATE TABLE pe48 AS SELECT (i % 5 = 0) AS active,
       CASE WHEN i % 5 = 0 THEN 'NAME' || lpad(((i / 5) % 20)::text, 6, '0')
            ELSE 'name' || lpad((i % 100)::text, 6, '0') END AS name
  FROM generate_series(1, 500000) i;
ANALYZE pe48;
CREATE INDEX p48 ON pe48 (lower(name)) WHERE active;
SELECT plan_add(48, 'partial', 'partial expression index, lower(name) WHERE active', 'p48', 'measure');
CREATE TABLE pe48b AS SELECT (i % 5 = 0) AS active,
       CASE WHEN i % 5 = 0 THEN 'NAME' || lpad(((i / 5) % 20)::text, 6, '0')
            ELSE 'name' || lpad((i % 100)::text, 6, '0') END AS name
  FROM generate_series(1, 500000) i;
CREATE INDEX p48b ON pe48b (lower(name)) WHERE active;
ANALYZE pe48b;
SELECT plan_add(48, 'partial', 'the same after one ANALYZE with the index in place', 'p48b', 'measure', 'b');

CREATE TABLE pe49 AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE pe49;
CREATE INDEX p49 ON pe49 (upper(s)) WHERE hot;
SELECT plan_add(49, 'partial', 'expression width mismatch in the subset', 'p49', 'measure');
CREATE TABLE pe49b AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX p49b ON pe49b (upper(s)) WHERE hot;
ANALYZE pe49b;
SELECT plan_add(49, 'partial', 'the same after one ANALYZE with the index in place', 'p49b', 'measure', 'b');

CREATE TABLE pe50 AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE pe50;
CREATE INDEX p50 ON pe50 (upper(s)) WHERE hot;
SELECT plan_add(50, 'partial', 'missing expression statistics, 32-byte fallback', 'p50', 'measure');
CREATE TABLE pe50b AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX p50b ON pe50b (upper(s)) WHERE hot;
ANALYZE pe50b;
SELECT plan_add(50, 'partial', 'the same after one ANALYZE with the index in place', 'p50b', 'measure', 'b');

CREATE TABLE pf AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pf;
CREATE INDEX p53 ON pf (k) WHERE hot;
CREATE INDEX p54 ON pf (k) WITH (fillfactor = 100) WHERE hot;
CREATE INDEX p55 ON pf (k) WITH (fillfactor = 70)  WHERE hot;
SELECT plan_add(53, 'partial', 'default fillfactor 90', 'p53', 'measure');
SELECT plan_add(54, 'partial', 'fillfactor = 100', 'p54', 'measure');
SELECT plan_add(55, 'partial', 'fillfactor = 70', 'p55', 'measure');

-- ======================================================= tests 56-63 ========
CREATE TABLE ps AS
SELECT (i % 5 = 0) AS flag,
       CASE WHEN i % 5 = 0 THEN 'OPEN' ELSE 'CLOSED' END AS status,
       timestamptz '2020-01-01' + (i * interval '1 minute') AS created,
       CASE WHEN i % 5 = 0 THEN NULL ELSE i::int END AS nk,
       i::int AS k, (i % 1000)::int AS k2
  FROM generate_series(1, 500000) i;
ANALYZE ps;
CREATE INDEX p56 ON ps (k) WHERE flag;
CREATE INDEX p57 ON ps (k) WHERE status = 'OPEN';
CREATE INDEX p58 ON ps (k) WHERE created >= timestamptz '2020-09-01';
CREATE INDEX p59 ON ps (k) WHERE nk IS NULL;
CREATE INDEX p60 ON ps (k) WHERE nk IS NOT NULL;
CREATE INDEX p61 ON ps (k) WHERE flag AND status = 'OPEN';
CREATE INDEX p62 ON ps (k) WHERE k < 100000;
CREATE INDEX p63 ON ps (k2) WHERE k >= 400000;
SELECT plan_add(56, 'partial', 'boolean predicate, WHERE flag', 'p56', 'measure');
SELECT plan_add(57, 'partial', 'equality predicate, status = OPEN', 'p57', 'measure');
SELECT plan_add(58, 'partial', 'range predicate, created >= ...', 'p58', 'measure');
SELECT plan_add(59, 'partial', 'IS NULL predicate on a non-key column', 'p59', 'measure');
SELECT plan_add(60, 'partial', 'IS NOT NULL predicate', 'p60', 'measure');
SELECT plan_add(61, 'partial', 'multi-column predicate', 'p61', 'measure');
SELECT plan_add(62, 'partial', 'predicate correlated with the indexed value', 'p62', 'measure');
SELECT plan_add(63, 'partial', 'predicate negatively correlated with the value', 'p63', 'measure');

-- ======================================================= tests 64-69 ========
CREATE TABLE pc64 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc64;
CREATE INDEX p64 ON pc64 (k) WHERE hot;
SELECT plan_add(64, 'partial', 'stale statistics after inserts into the subset', 'p64', 'measure');

CREATE TABLE pc65 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc65;
CREATE INDEX p65 ON pc65 (k) WHERE hot;
SELECT plan_add(65, 'partial', 'stale statistics after deletes, no VACUUM', 'p65', 'skip');

CREATE TABLE pc66 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc66;
CREATE INDEX p66 ON pc66 (k) WHERE hot;
SELECT plan_add(66, 'partial', 'rows entering the index (false -> true)', 'p66', 'measure');

CREATE TABLE pc67 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc67;
CREATE INDEX p67 ON pc67 (k) WHERE hot;
SELECT plan_add(67, 'partial', 'rows leaving the index (true -> false)', 'p67', 'measure');

CREATE TABLE pc68 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc68;
CREATE INDEX p68 ON pc68 (k) WHERE hot;
SELECT plan_add(68, 'partial', 'heavy predicate churn, then VACUUM + ANALYZE', 'p68', 'measure');

CREATE TABLE pc69 AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pc69;
CREATE INDEX p69 ON pc69 (k) WHERE hot;
SELECT plan_add(69, 'partial', 'stale reltuples, VACUUM but no ANALYZE', 'p69', 'measure');

-- ======================================================= tests 70-77 ========
CREATE TABLE pb AS SELECT (i % 5 = 0) AS hot, i::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE pb;
CREATE INDEX p70 ON pb (k) WHERE hot;
SELECT plan_add(70, 'partial', 'freshly created partial index', 'p70', 'skip');
CREATE INDEX p71 ON pb (k) WHERE hot;
SELECT plan_add(71, 'partial', 'freshly REINDEXed partial index', 'p71', 'skip');

CREATE TABLE pb72 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb72;
CREATE INDEX p72 ON pb72 (k) WHERE hot;
SELECT plan_add(72, 'partial', '25% of the subset deleted', 'p72', 'measure');

CREATE TABLE pb73 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb73;
CREATE INDEX p73 ON pb73 (k) WHERE hot;
SELECT plan_add(73, 'partial', '50% of the subset deleted', 'p73', 'measure');

CREATE TABLE pb74 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb74;
CREATE INDEX p74 ON pb74 (k) WHERE hot;
SELECT plan_add(74, 'partial', '75% of the subset deleted', 'p74', 'measure');

CREATE TABLE pb75 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb75;
CREATE INDEX p75 ON pb75 (k) WHERE hot;
SELECT plan_add(75, 'partial', '90% of the subset deleted (corrected recipe)', 'p75', 'measure');

CREATE TABLE pb76 AS SELECT (i % 5 = 0) AS hot, i::int AS k, 'x'::text AS pad
  FROM generate_series(1, 500000) i;
ANALYZE pb76;
CREATE INDEX p76 ON pb76 (k) WHERE hot;
SELECT plan_add(76, 'partial', 'bloated through indexed-key UPDATEs', 'p76', 'measure');

CREATE TABLE pb77 AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE pb77;
CREATE INDEX p77 ON pb77 (k) WHERE hot;
SELECT plan_add(77, 'partial', 'many empty and deleted B-tree pages', 'p77', 'measure');

-- ======================================================= tests 78-85 ========
CREATE TABLE f78t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 290) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE f78t;
CREATE INDEX f78 ON f78t (s) WHERE hot;
SELECT plan_add(78, 'falsepos', 'predicate-conditioned width mismatch', 'f78', 'skip');

CREATE TABLE f79t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('t', 300) || lpad(i::text, 4, '0')
            ELSE NULL END AS s
  FROM generate_series(1, 500000) i;
ANALYZE f79t;
CREATE INDEX f79 ON f79t (s) WHERE hot;
SELECT plan_add(79, 'falsepos', 'predicate-conditioned NULL mismatch', 'f79', 'skip');

CREATE TABLE f80t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN i::int ELSE (i % 3)::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE f80t;
CREATE INDEX f80 ON f80t (k) WHERE hot;
SELECT plan_add(80, 'falsepos', 'predicate-conditioned n_distinct mismatch', 'f80', 'skip');

CREATE TABLE f81t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 20)::int ELSE 7 END AS k
  FROM generate_series(1, 500000) i;
ANALYZE f81t;
CREATE INDEX f81 ON f81t (k) WHERE hot;
SELECT plan_add(81, 'falsepos', 'predicate-conditioned MCV mismatch', 'f81', 'skip');

CREATE TABLE f82t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 100)::int END AS a,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE (i % 89)::int  END AS b
  FROM generate_series(1, 500000) i;
CREATE STATISTICS f82_nd (ndistinct) ON a, b FROM f82t;
ANALYZE f82t;
CREATE INDEX f82 ON f82t (a, b) WHERE hot;
SELECT plan_add(82, 'falsepos', 'predicate-conditioned multi-column correlation', 'f82', 'skip');

CREATE TABLE f83t AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE f83t;
CREATE INDEX f83 ON f83t (md5(s), lower(s)) WHERE hot;
SELECT plan_add(83, 'falsepos', 'missing index/expression statistics', 'f83', 'skip');

CREATE TABLE f84t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE f84t;
CREATE INDEX f84 ON f84t (k) WHERE hot;
SELECT plan_add(84, 'falsepos', 'stale partial-index reltuples', 'f84', 'skip');

CREATE TABLE f85t AS SELECT (i % 5 = 0) AS hot, lpad(i::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE f85t;
UPDATE f85t SET s = repeat('W', 200) || s WHERE hot;
VACUUM f85t;
CREATE INDEX f85 ON f85t (s) WHERE hot;
SELECT plan_add(85, 'falsepos', 'stale table statistics', 'f85', 'skip');

-- ======================================================= tests 86-91 ========
CREATE TABLE f86t AS SELECT (i % 5 = 0) AS hot, ((i / 5) % 100)::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE f86t;
CREATE INDEX f86 ON f86t (k) WHERE hot;
SELECT plan_add(86, 'falseneg', 'duplicate concentration inside the subset', 'f86', 'measure');

CREATE TABLE f87t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN NULL ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE f87t;
CREATE INDEX f87 ON f87t (k) WHERE hot;
SELECT plan_add(87, 'falseneg', 'NULL concentration inside the subset', 'f87', 'measure');

CREATE TABLE f88t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN lpad((i % 9)::text, 9, '0')
            ELSE repeat('W', 390) || lpad(i::text, 10, '0') END AS s
  FROM generate_series(1, 200000) i;
ANALYZE f88t;
CREATE INDEX f88 ON f88t (s) WHERE hot;
SELECT plan_add(88, 'falseneg', 'subset narrower than table statistics', 'f88', 'measure');

CREATE TABLE f89t AS SELECT (i % 5 = 0) AS hot,
       ((i / 5) % 100)::int AS a, ((i / 5) % 100)::int AS b
  FROM generate_series(1, 500000) i;
ANALYZE f89t;
CREATE INDEX f89 ON f89t (a, b) WHERE hot;
SELECT plan_add(89, 'falseneg', 'conditional multi-column correlation', 'f89', 'measure');

CREATE TABLE f90t AS SELECT (i % 5 = 0) AS hot, ((i / 5) % 1000)::int AS k
  FROM generate_series(1, 500000) i;
ANALYZE f90t;
CREATE INDEX f90 ON f90t (k) WHERE hot;
SELECT plan_add(90, 'falseneg', 'real deduplication stronger than predicted', 'f90', 'measure');

CREATE TABLE f91t AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN lpad((i % 9)::text, 9, '0')
            ELSE repeat('W', 390) || lpad(i::text, 10, '0') END AS s,
       i::int AS ord
  FROM generate_series(1, 200000) i;
ANALYZE f91t;
CREATE INDEX f91 ON f91t (s) WHERE hot;
SELECT plan_add(91, 'falseneg', 'many deleted pages plus an over-predicting model', 'f91', 'measure');

-- ====================================================== tests 92-95 =========
CREATE TABLE b92t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE b92t;
CREATE INDEX b92 ON b92t (k) WHERE hot;
SELECT plan_add(92, 'control', '1,000 rows updated under the old GUC threshold', 'b92', 'measure');

CREATE TABLE b93t AS SELECT (i % 5 = 0) AS hot, i::int AS k FROM generate_series(1, 500000) i;
ANALYZE b93t;
CREATE INDEX b93 ON b93t (k) WHERE hot;
SELECT plan_add(93, 'control', 'rows updated above the old GUC threshold', 'b93', 'measure');

CREATE TABLE b94t (hot bool, k int)
  WITH (autovacuum_analyze_threshold = 100, autovacuum_analyze_scale_factor = 0);
INSERT INTO b94t SELECT (i % 5 = 0), i FROM generate_series(1, 500000) i;
ANALYZE b94t;
CREATE INDEX b94 ON b94t (k) WHERE hot;
SELECT plan_add(94, 'control', '1,000 rows updated, table reloption threshold 100', 'b94', 'measure');

CREATE TABLE b95t (hot bool, k int)
  WITH (autovacuum_analyze_threshold = 200000, autovacuum_analyze_scale_factor = 1);
INSERT INTO b95t SELECT (i % 5 = 0), i FROM generate_series(1, 500000) i;
ANALYZE b95t;
CREATE INDEX b95 ON b95t (k) WHERE hot;
SELECT plan_add(95, 'control', 'many rows updated, table reloption threshold 200,000', 'b95', 'measure');

-- ====================================================== tests 96-99 =========
CREATE TABLE np AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE np;
CREATE INDEX np96 ON np (k);
CREATE INDEX np97 ON np (upper(s));
SELECT plan_add(96, 'control', 'plain index, fresh statistics', 'np96', 'skip');
SELECT plan_add(97, 'control', 'expression index, no statistics row', 'np97', 'skip');

CREATE TABLE np98t AS SELECT i::int AS k FROM generate_series(1, 500000) i;
ANALYZE np98t;
CREATE INDEX np98 ON np98t (k);
SELECT plan_add(98, 'control', 'plain index, stale row counts after 300,000 inserts', 'np98', 'measure');

CREATE TABLE np99t AS SELECT (i % 1000)::int AS k FROM generate_series(1, 500000) i;
ANALYZE np99t;
CREATE INDEX np99 ON np99t (k);
SELECT plan_add(99, 'control', 'duplicate-heavy index, genuinely reclaimable', 'np99', 'measure');

-- ===================================================== tests 100-105 ========
CREATE TABLE i100t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i100t;
CREATE INDEX i100 ON i100t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(100, 'control', 'partial + INCLUDE (text), 90% of the subset deleted', 'i100', 'measure');

CREATE TABLE i101t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i101t;
CREATE INDEX i101 ON i101t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(101, 'control', 'partial + INCLUDE (text), same width inside and outside', 'i101', 'skip');

CREATE TABLE i102t AS SELECT i::int AS k, lpad(i::text, 60, '0') AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i102t;
CREATE INDEX i102 ON i102t (k) INCLUDE (pay);
SELECT plan_add(102, 'control', 'non-partial + wide INCLUDE (text), freshly built', 'i102', 'skip');

CREATE TABLE i103t AS SELECT (i % 20 = 0) AS hot,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad(i::text, 12, 'n') END AS s
  FROM generate_series(1, 500000) i;
ANALYZE i103t;
CREATE INDEX i103 ON i103t (s) WHERE hot;
SELECT plan_add(103, 'control', 'partial + wide key column, unique values', 'i103', 'skip');

CREATE TABLE i104t AS SELECT (i % 20 = 0) AS hot, i::int AS k,
       CASE WHEN i % 20 = 0 THEN lpad((i % 9)::text, 12, 'n')
            ELSE repeat('W', 190) || lpad(i::text, 10, '0') END AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i104t;
CREATE INDEX i104 ON i104t (k) INCLUDE (pay) WHERE hot;
SELECT plan_add(104, 'control', 'partial + INCLUDE (text) narrower inside the subset', 'i104', 'skip');

CREATE TABLE i105t AS SELECT (i % 20 = 0) AS hot, i::int AS k, (i % 7)::int AS n,
       CASE WHEN i % 20 = 0 THEN repeat('W', 190) || lpad(i::text, 10, '0')
            ELSE lpad((i % 9)::text, 12, 'n') END AS pay
  FROM generate_series(1, 500000) i;
ANALYZE i105t;
CREATE INDEX i105 ON i105t (k) INCLUDE (n, pay) WHERE hot;
SELECT plan_add(105, 'control', 'partial + INCLUDE (int, text), mixed non-key widths', 'i105', 'skip');

-- ===================================================== tests 106-112 ========
CREATE TABLE x106t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x106t;
CREATE INDEX x106 ON x106t (upper(s));
SELECT plan_add(106, 'control', 'expression index, no statistics row, 90% deleted', 'x106', 'measure');

CREATE TABLE x107t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x107t;
CREATE INDEX x107 ON x107t (upper(s));
SELECT plan_add(107, 'control', 'the same, with one ANALYZE after the build', 'x107', 'measure');

CREATE TABLE x108t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
CREATE INDEX x108 ON x108t (upper(s));
SELECT plan_add(108, 'control', 'expression index on a never-analysed table', 'x108', 'skip');

CREATE TABLE x109t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ALTER TABLE x109t ALTER COLUMN s SET STATISTICS 0;
ANALYZE x109t;
CREATE INDEX x109 ON x109t (s);
SELECT plan_add(109, 'control', 'plain index, key column with SET STATISTICS 0', 'x109', 'skip');

CREATE TABLE x110t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x110t;
CREATE INDEX x110 ON x110t (k, upper(s));
SELECT plan_add(110, 'control', 'mixed key (k, upper(s)), no statistics row', 'x110', 'skip');

CREATE TABLE x111t AS SELECT i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x111t;
CREATE INDEX x111 ON x111t (left(s, 3));
SELECT plan_add(111, 'control', 'narrow expression left(s, 3), no statistics row', 'x111', 'skip');

CREATE TABLE x112t AS SELECT (i % 5 = 0) AS hot, i::int AS k, lpad(i::text, 100, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE x112t;
CREATE INDEX x112 ON x112t (upper(s)) WHERE hot;
SELECT plan_add(112, 'control', 'partial expression index, no statistics row', 'x112', 'skip');

-- ===================================================== tests 113-121 ========
CREATE TABLE q113a AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q113a;
CREATE INDEX p113a ON q113a (id) WHERE state = 'pending';
SELECT plan_add(113, 'zero', 'drained queue, nothing run', 'p113a', 'skip', 'a');

CREATE TABLE q113b AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q113b;
CREATE INDEX p113b ON q113b (id) WHERE state = 'pending';
SELECT plan_add(113, 'zero', 'drained queue, VACUUM + ANALYZE', 'p113b', 'skip', 'b');

CREATE TABLE q113c AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q113c;
CREATE INDEX p113c ON q113c (id) WHERE state = 'pending';
SELECT plan_add(113, 'zero', 'drained queue, ANALYZE only', 'p113c', 'skip', 'c');

CREATE TABLE q114 AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q114;
CREATE INDEX p114 ON q114 (id) WHERE state = 'pending';
SELECT plan_add(114, 'zero', 'queue drained to 1%, VACUUM + ANALYZE', 'p114', 'skip');

CREATE TABLE q115(id int, state text);
ANALYZE q115;
CREATE INDEX p115 ON q115 (id) WHERE state = 'pending';
SELECT plan_add(115, 'zero', 'index built on an analysed empty table, then loaded', 'p115', 'measure');

CREATE TABLE q116 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p116 ON q116 (id) WHERE state = 'pending';
ANALYZE q116;
SELECT plan_add(116, 'zero', 'subset empty from the start and measured empty', 'p116', 'skip');

CREATE TABLE q117 AS SELECT i::int AS id, 'pending'::text AS state
  FROM generate_series(1, 1000000) i;
ANALYZE q117;
CREATE INDEX p117 ON q117 (id) WHERE state = 'pending';
SELECT plan_add(117, 'zero', 'drained, then VACUUM only', 'p117', 'skip');

CREATE TABLE q118 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p118 ON q118 (id) WHERE state = 'pending';
ANALYZE q118;
SELECT plan_add(118, 'zero', 'subset measured empty, then 50,000 rows arrive', 'p118', 'skip');

CREATE TABLE q119 AS SELECT i::int AS id, 'done'::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p119 ON q119 (id) WHERE state = 'pending';
ANALYZE q119;
SELECT plan_add(119, 'zero', 'fixture 118 after one ANALYZE', 'p119', 'skip');

CREATE TABLE q120 AS SELECT i::int AS id,
       CASE WHEN i <= 2000 THEN 'pending' ELSE 'done' END::text AS state
  FROM generate_series(1, 1000000) i;
CREATE INDEX p120 ON q120 (id) WHERE state = 'pending';
SET default_statistics_target = 1;
ANALYZE q120;
RESET default_statistics_target;
SELECT plan_add(120, 'zero', 'a 300-row sample missed a 2,000-row subset', 'p120', 'skip');

CREATE TABLE nz AS SELECT i::int AS k FROM generate_series(1, 1000000) i;
ANALYZE nz;
CREATE INDEX nz_k ON nz (k);
SELECT plan_add(121, 'zero', 'emptied, vacuumed, reloaded without ANALYZE', 'nz_k', 'measure', 'nz_k');

CREATE TABLE nzb AS SELECT i::int AS k FROM generate_series(1, 1000000) i;
ANALYZE nzb;
CREATE INDEX nzb_k ON nzb (k);
SELECT plan_add(121, 'zero', 'the same, plus a REINDEX while the table is empty', 'nzb_k', 'refresh', 'nzb_k');

CREATE TABLE trunc_t AS SELECT i::int AS k FROM generate_series(1, 300000) i;
ANALYZE trunc_t;
CREATE INDEX i_trunc ON trunc_t (k);
SELECT plan_add(121, 'zero', 'TRUNCATE then reload without ANALYZE', 'i_trunc', 'skip', 'i_trunc');

SELECT count(*) AS planned_fixtures FROM plan;
FIXTURES_BUILD
  cat > "$SQLD/fixtures_v13.sql" <<'FIXTURES_V13'
-- Build phase, every fixture that needs a feature PostgreSQL 12 does not have.
-- Two families:
--
--   1. B-tree support function 4, the deduplication equal-image callback.  A
--      12 server refuses "FUNCTION 4" in CREATE OPERATOR CLASS outright, and
--      has no btequalimage or btvarstrequalimage to alias, so tests 13, 14, 15
--      and 16 are skipped there.
--   2. The deduplicate_items reloption, which 12 calls an unrecognized
--      parameter, so test 11 and test 38 are skipped there.
--
-- This file runs only where server_version_num >= 130000; otherwise the
-- fixtures are recorded as skipped and the suite is scored without them.
-- Disposable fixtures, in the sandbox cluster's suite database only.
SET /* wiki_btmaint_v13_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_v13_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_v13_lock_timeout */ lock_timeout = '5s';
SET /* wiki_btmaint_v13_maintenance_work_mem */ maintenance_work_mem = '256MB';

CREATE OR REPLACE FUNCTION ei_true(oid)  RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;
CREATE OR REPLACE FUNCTION ei_false(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT false $$;
CREATE OR REPLACE FUNCTION ei_alias(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btequalimage';
CREATE OR REPLACE FUNCTION ei_renamed(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btvarstrequalimage';
-- test 16, the impostor: a SQL function wearing the built-in's name.  It must
-- be schema-qualified in the operator class or pg_catalog wins the lookup.
CREATE OR REPLACE FUNCTION public.btequalimage(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;

CREATE OPERATOR CLASS int4_ei_true FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_true(oid);
CREATE OPERATOR CLASS int4_ei_false FOR TYPE int4 USING btree AS
  OPERATOR 1 <(int4,int4), OPERATOR 2 <=(int4,int4), OPERATOR 3 =(int4,int4),
  OPERATOR 4 >=(int4,int4), OPERATOR 5 >(int4,int4),
  FUNCTION 1 btint4cmp(int4,int4), FUNCTION 4 ei_false(oid);
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

CREATE INDEX i_ei_false  ON t (a int4_ei_false);
CREATE INDEX i_ei_true   ON t (a int4_ei_true);
CREATE INDEX i_ei_alias  ON t (a int4_ei_alias);
CREATE INDEX i_mixed_tf  ON t (a int4_ei_true, b int8_ei_false);
CREATE INDEX i_mixed_ft  ON t (a int4_ei_false, b int8_ei_true);
CREATE INDEX i_squat     ON t (s text_squat);
CREATE INDEX i_text_det2 ON t (s text_renamed);
CREATE INDEX i2_tf       ON t2 (a int4_ei_true, b int8_ei_false);
CREATE INDEX i2_ft       ON t2 (a int4_ei_false, b int8_ei_true);
SELECT plan_add(13, 'gate', 'support function 4 returns false', 'i_ei_false', 'measure', 'i_ei_false');
SELECT plan_add(14, 'gate', 'support function 4 returns true by design', 'i_ei_true', 'measure', 'i_ei_true');
SELECT plan_add(14, 'gate', 'internal alias of btequalimage', 'i_ei_alias', 'measure', 'i_ei_alias');
SELECT plan_add(15, 'gate', 'mixed true/false support functions', 'i_mixed_tf', 'measure', 'i_mixed_tf');
SELECT plan_add(15, 'gate', 'mixed false/true support functions', 'i_mixed_ft', 'measure', 'i_mixed_ft');
SELECT plan_add(16, 'gate', 'SQL impostor named btequalimage', 'i_squat', 'measure', 'i_squat');
SELECT plan_add(16, 'gate', 'renamed internal support function 4', 'i_text_det2', 'measure', 'i_text_det2');
SELECT plan_add(15, 'gate', 'second table, mixed true/false', 'i2_tf', 'measure', 'i2_tf');
SELECT plan_add(15, 'gate', 'second table, mixed false/true', 'i2_ft', 'measure', 'i2_ft');

CREATE INDEX i_dupoff   ON t (a) WITH (deduplicate_items = off);
CREATE INDEX i_text_off ON t (s) WITH (deduplicate_items = off);
CREATE INDEX i2_off     ON t2 (s) WITH (deduplicate_items = off);
SELECT plan_add(11, 'gate', 'deduplicate_items = off, int4 key', 'i_dupoff', 'measure', 'i_dupoff');
SELECT plan_add(11, 'gate', 'deduplicate_items = off, text key', 'i_text_off', 'measure', 'i_text_off');
SELECT plan_add(11, 'gate', 'deduplicate_items = off, second table', 'i2_off', 'measure', 'i2_off');

CREATE TABLE pd38 AS SELECT (i % 5 = 0) AS hot,
       CASE WHEN i % 5 = 0 THEN ((i / 5) % 100)::int ELSE i::int END AS k
  FROM generate_series(1, 500000) i;
ANALYZE pd38;
CREATE INDEX p38 ON pd38 (k) WITH (deduplicate_items = off) WHERE hot;
SELECT plan_add(38, 'partial', 'deduplicate_items = off', 'p38', 'measure');
FIXTURES_V13
  cat > "$SQLD/fixtures_icu.sql" <<'FIXTURES_ICU'
-- Build phase, the four ICU fixtures: tests 3, 4, 9 and 51-52.  A server built
-- --without-icu cannot create these collations, so this file runs only where
-- CREATE COLLATION (provider = icu) succeeds and the fixtures are otherwise
-- recorded as skipped.
-- Disposable fixtures, in the sandbox cluster's suite database only.
SET /* wiki_btmaint_icu_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_icu_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_icu_lock_timeout */ lock_timeout = '5s';

CREATE COLLATION ci   (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
CREATE COLLATION cdet (provider = icu, locale = 'und');

CREATE INDEX i_text_icu_det  ON t (s COLLATE cdet);
CREATE INDEX i_text_nondet   ON t (s COLLATE ci);
CREATE INDEX i_expr_lower_ci ON t ((lower(s)) COLLATE ci);
SELECT plan_add(3, 'gate', 'text key, deterministic ICU collation', 'i_text_icu_det', 'measure', 'i_text_icu_det');
SELECT plan_add(4, 'gate', 'text key, nondeterministic ICU collation', 'i_text_nondet', 'measure', 'i_text_nondet');
SELECT plan_add(9, 'gate', 'expression key under a nondeterministic collation', 'i_expr_lower_ci', 'measure', 'i_expr_lower_ci');

CREATE COLLATION suite_det    (provider = icu, locale = 'und');
CREATE COLLATION suite_nondet (provider = icu, locale = 'und-u-ks-level2',
                               deterministic = false);
CREATE TABLE pc51 AS SELECT (i % 5 = 0) AS hot,
       'key' || lpad(((i / 5) % 100)::text, 8, '0') AS s
  FROM generate_series(1, 500000) i;
ANALYZE pc51;
CREATE INDEX p51 ON pc51 (s COLLATE suite_det) WHERE hot;
CREATE INDEX p52 ON pc51 (s COLLATE suite_nondet) WHERE hot;
SELECT plan_add(51, 'partial', 'deterministic ICU collation', 'p51', 'measure');
SELECT plan_add(52, 'partial', 'nondeterministic ICU collation', 'p52', 'measure');
FIXTURES_ICU
  cat > "$SQLD/fixtures_churn.sql" <<'FIXTURES_CHURN'
-- The numbered suite, churn phase: everything each recipe does after its index
-- exists, run after the heuristic has stored an as-built baseline in every
-- index comment.  Two kinds of churn:
--
--   1. the recipe's own, transcribed verbatim from the numbered fixtures;
--   2. a uniform 90 % block drain for the shape fixtures that have no churn of
--      their own, because a fixture designed for a one-shot estimator never
--      needed a "before" and an "after".
--
-- The fixtures whose whole point is that a fresh index must not be touched are
-- deliberately left alone: 70-71, 78-85, 96-97, 101-105, 108-112, 116 and 120.
--
-- Disposable fixtures, in the sandbox cluster's suite database only.
SET /* wiki_btmaint_churn_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_churn_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_churn_lock_timeout */ lock_timeout = '5s';

-- ---------------------------------------------------------- recipe churn ----
-- 64: stale statistics after inserts into the subset.
INSERT INTO pc64 SELECT true, 500000 + i FROM generate_series(1, 200000) i;

-- 65: stale statistics after deletes, no VACUUM.
DELETE FROM pc65 WHERE hot AND k % 50 <> 0;

-- 66: rows entering the index (false -> true).
UPDATE pc66 SET hot = true WHERE NOT hot AND k % 5 = 1;

-- 67: rows leaving the index (true -> false).
UPDATE pc67 SET hot = false WHERE hot AND k % 50 <> 0;

-- 68: heavy predicate churn, then VACUUM + ANALYZE.
UPDATE pc68 SET hot = true  WHERE k % 3 = 0;
UPDATE pc68 SET hot = false WHERE k % 3 = 0;
UPDATE pc68 SET hot = true  WHERE k % 3 = 1;
UPDATE pc68 SET hot = false WHERE k % 3 = 1;
UPDATE pc68 SET hot = (k % 10 = 0);
VACUUM pc68;
ANALYZE pc68;

-- 69: stale reltuples, VACUUM but no ANALYZE.
DELETE FROM pc69 WHERE hot AND k % 50 <> 0;
VACUUM pc69;

-- 71: a REINDEX nobody told the heuristic about.
REINDEX INDEX p71;

-- 72-75: a quarter, a half, three quarters and nine tenths of the subset.
DELETE FROM pb72 WHERE hot AND (k / 5) % 4 = 0;
VACUUM pb72;
ANALYZE pb72;
DELETE FROM pb73 WHERE hot AND (k / 5) % 2 = 0;
VACUUM pb73;
ANALYZE pb73;
DELETE FROM pb74 WHERE hot AND (k / 5) % 4 <> 0;
VACUUM pb74;
ANALYZE pb74;
DELETE FROM pb75 WHERE hot AND (k / 5) % 10 <> 0;
VACUUM pb75;
ANALYZE pb75;

-- 76: bloated through indexed-key UPDATEs.
UPDATE pb76 SET k = k + 1000000 WHERE hot;
VACUUM pb76;
ANALYZE pb76;

-- 77: many empty and deleted B-tree pages, contiguous 95 %.
DELETE FROM pb77 WHERE hot AND k < 475000;
VACUUM pb77;
ANALYZE pb77;

-- 84's forged partial-index reltuples has moved to forge.sql, which runs after
-- the simulated auto-analyze, because an ANALYZE of its table would overwrite
-- the forgery and the fixture would stop testing anything.

-- 86-91: genuinely bloated, VACUUMed and ANALYZEd.
DELETE FROM f86t WHERE hot AND k >= 25;
VACUUM f86t;
ANALYZE f86t;
DELETE FROM f87t WHERE hot AND k IS NOT NULL;
VACUUM f87t;
ANALYZE f87t;
DELETE FROM f88t WHERE hot AND s > lpad('4', 9, '0');
VACUUM f88t;
ANALYZE f88t;
DELETE FROM f89t WHERE hot AND a >= 25;
VACUUM f89t;
ANALYZE f89t;
DELETE FROM f90t WHERE hot AND k >= 250;
VACUUM f90t;
ANALYZE f90t;
DELETE FROM f91t WHERE hot AND ord < 190000;
VACUUM f91t;
ANALYZE f91t;

-- 92-95: a reclaimable partial index, then a known number of row changes.
DELETE FROM b92t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b92t;
ANALYZE b92t;
UPDATE b92t SET k = k WHERE k % 500 = 0;
DELETE FROM b93t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b93t;
ANALYZE b93t;
UPDATE b93t SET k = k WHERE k % 2 = 0;
DELETE FROM b94t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b94t;
ANALYZE b94t;
UPDATE b94t SET k = k WHERE k % 500 = 0;
DELETE FROM b95t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM b95t;
ANALYZE b95t;
UPDATE b95t SET k = k WHERE k % 2 = 0;

-- 98: 300,000 inserts and no ANALYZE.
INSERT INTO np98t SELECT 500000 + i FROM generate_series(1, 300000) i;

-- 99: duplicate-heavy, genuinely reclaimable.
DELETE FROM np99t WHERE k >= 60;
VACUUM np99t;
ANALYZE np99t;

-- 100: partial + INCLUDE, 90 % of the subset deleted.
DELETE FROM i100t WHERE hot AND (k / 5) % 10 <> 0;
VACUUM i100t;
ANALYZE i100t;

-- 106: expression index, 90 % deleted, VACUUM but no ANALYZE.
DELETE FROM x106t WHERE k % 10 <> 0;
VACUUM x106t;

-- 107: the same with one ANALYZE.
DELETE FROM x107t WHERE k % 10 <> 0;
VACUUM x107t;
ANALYZE x107t;

-- 113a-c: the drained queue in three states.
UPDATE q113a SET state = 'done';
UPDATE q113b SET state = 'done';
VACUUM q113b;
ANALYZE q113b;
UPDATE q113c SET state = 'done';
ANALYZE q113c;

-- 114: drained to 1 %.
UPDATE q114 SET state = 'done' WHERE id % 100 <> 0;
VACUUM q114;
ANALYZE q114;

-- 115: an index built on an analysed empty table, then loaded.
INSERT INTO q115 SELECT i, 'pending' FROM generate_series(1, 1000000) i;

-- 117: drained, then VACUUM only.
UPDATE q117 SET state = 'done';
VACUUM q117;

-- 118, 119: the subset was empty at the last ANALYZE, then rows arrived.
INSERT INTO q118 SELECT 1000000 + i, 'pending' FROM generate_series(1, 50000) i;
INSERT INTO q119 SELECT 1000000 + i, 'pending' FROM generate_series(1, 50000) i;
ANALYZE q119;

-- 121: three ways to leave a stale row count behind.
DELETE FROM nz;
VACUUM nz;
INSERT INTO nz SELECT i FROM generate_series(1, 500000) i;
DELETE FROM nzb;
VACUUM nzb;
REINDEX INDEX nzb_k;
INSERT INTO nzb SELECT i FROM generate_series(1, 500000) i;
TRUNCATE trunc_t;
INSERT INTO trunc_t SELECT i FROM generate_series(1, 300000) i;

-- ------------------------------------------------------- the uniform drain ---
-- 90 % of the heap blocks of every shape fixture that has no churn of its own,
-- then VACUUM and ANALYZE, so that the index has empty and deleted pages the
-- gate can see and a rebuild can give back.  The commands are generated and
-- executed one at a time because VACUUM cannot run inside a transaction block.
SELECT /* wiki_btmaint_drain_generator */ format(st.tmpl, tb.name)
  FROM (VALUES (1, 't'), (2, 't2'), (3, 'pt1'), (4, 'pd22'), (5, 'pd23'),
               (6, 'pd24'), (7, 'pd25'), (8, 'pd26'), (9, 'pd27'), (10, 'pd28'),
               (11, 'pd29'), (12, 'pd30'), (13, 'pd31'), (14, 'pw32'),
               (15, 'pd33'), (16, 'pd34'), (17, 'pd35'), (18, 'pd36'),
               (19, 'pd37'), (20, 'pd39'), (21, 'pd40'), (22, 'pd41'),
               (23, 'pd42'), (24, 'pd43'), (25, 'pd44a'), (26, 'pd44b'),
               (27, 'pd45'), (28, 'pd46'), (29, 'pi47'), (30, 'pe48'),
               (31, 'pe48b'), (32, 'pe49'), (33, 'pe49b'), (34, 'pe50'),
               (35, 'pe50b'), (36, 'pf'), (37, 'ps')) tb(n, name)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_drain */ FROM %I WHERE ((ctid::text::point)[0])::int %% 10 <> 0'),
        (2, 'VACUUM /* wiki_btmaint_drain */ %I'),
        (3, 'ANALYZE /* wiki_btmaint_drain */ %I')) st(k, tmpl)
 ORDER BY tb.n, st.k
\gexec
FIXTURES_CHURN
  cat > "$SQLD/churn_v13.sql" <<'CHURN_V13'
-- Churn phase for the deduplicate_items fixtures, drained like the other
-- shape fixtures.  Disposable, suite database of the sandbox cluster only.
SET /* wiki_btmaint_churn13_client_min_messages */ client_min_messages = warning;
SELECT /* wiki_btmaint_drain13_generator */ format(st.tmpl, tb.name)
  FROM (VALUES (1, 'pd38')) tb(n, name)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_drain */ FROM %I WHERE ((ctid::text::point)[0])::int %% 10 <> 0'),
        (2, 'VACUUM /* wiki_btmaint_drain */ %I'),
        (3, 'ANALYZE /* wiki_btmaint_drain */ %I')) st(k, tmpl)
 ORDER BY tb.n, st.k
\gexec
CHURN_V13
  cat > "$SQLD/churn_icu.sql" <<'CHURN_ICU'
-- Churn phase for the ICU fixtures, drained like the other shape fixtures.
-- Disposable, suite database of the sandbox cluster only.
SET /* wiki_btmaint_churnicu_client_min_messages */ client_min_messages = warning;
SELECT /* wiki_btmaint_drainicu_generator */ format(st.tmpl, tb.name)
  FROM (VALUES (1, 'pc51')) tb(n, name)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_drain */ FROM %I WHERE ((ctid::text::point)[0])::int %% 10 <> 0'),
        (2, 'VACUUM /* wiki_btmaint_drain */ %I'),
        (3, 'ANALYZE /* wiki_btmaint_drain */ %I')) st(k, tmpl)
 ORDER BY tb.n, st.k
\gexec
CHURN_ICU
  cat > "$SQLD/autoanalyze.sql" <<'AUTOANALYZE'
-- The simulated auto-analyze, and the mandatory-test rule behind it: a fixture
-- that changes more than 10 % of a table's heap tuples must ANALYZE it, because
-- on a server with autovacuum on the launcher would have.
--
-- The trigger is not hand-written per fixture.  It is the engine's own test,
-- n_mod_since_analyze > autovacuum_analyze_threshold +
-- autovacuum_analyze_scale_factor * reltuples, which is exactly what
-- relation_needs_vacanalyze() compares; at the defaults that is 50 + 10 % of
-- the table's estimated row count.  autovacuum is off in this cluster, so no
-- background worker can have analyzed anything, and every ANALYZE below is one
-- this fixture set asked for.
--
-- Both counter states are recorded: autoanl holds what the churn left behind,
-- autoanl_after holds the same counters once the ANALYZEs have run, which is
-- how the run proves the simulation actually fired.
-- Disposable fixtures, suite database of the sandbox cluster only.
SET /* wiki_btmaint_autoanl_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_autoanl_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_autoanl_lock_timeout */ lock_timeout = '5s';

DROP TABLE IF EXISTS autoanl;
DROP TABLE IF EXISTS autoanl_after;
CREATE TABLE autoanl AS
SELECT /* wiki_btmaint_autoanalyze_census */
       c.relname                                   AS tbl,
       GREATEST(c.reltuples, 0)::numeric           AS reltuples,
       st.n_mod_since_analyze::numeric             AS mods,
       round(current_setting('autovacuum_analyze_threshold')::numeric
             + current_setting('autovacuum_analyze_scale_factor')::numeric
               * GREATEST(c.reltuples, 0)::numeric, 1) AS threshold,
       round(100 * st.n_mod_since_analyze
             / GREATEST(c.reltuples, 1)::numeric, 1)   AS mod_pct,
       (st.n_mod_since_analyze
        > current_setting('autovacuum_analyze_threshold')::numeric
          + current_setting('autovacuum_analyze_scale_factor')::numeric
            * GREATEST(c.reltuples, 0)::numeric)       AS would_autoanalyze
  FROM pg_stat_all_tables st
  JOIN pg_class c ON c.oid = st.relid
 WHERE st.schemaname = 'public'
   AND c.relkind = 'r'
   AND c.relname NOT IN ('plan', 'snap', 'truth', 'autoanl', 'autoanl_after');

SELECT /* wiki_btmaint_autoanalyze_generator */
       format('ANALYZE /* wiki_btmaint_autoanalyze */ %I', tbl)
  FROM autoanl WHERE would_autoanalyze ORDER BY tbl
\gexec

CREATE TABLE autoanl_after AS
SELECT /* wiki_btmaint_autoanalyze_recheck */
       c.relname AS tbl, st.n_mod_since_analyze::numeric AS mods,
       GREATEST(c.reltuples, 0)::numeric AS reltuples
  FROM pg_stat_all_tables st
  JOIN pg_class c ON c.oid = st.relid
 WHERE st.schemaname = 'public' AND c.relkind = 'r'
   AND c.relname IN (SELECT tbl FROM autoanl WHERE would_autoanalyze);
AUTOANALYZE
  cat > "$SQLD/forge.sql" <<'FORGE'
-- The numbered suite's catalog forgeries, applied after the simulated
-- auto-analyze so that an ANALYZE cannot overwrite them.  Fixture 84 is a
-- partial index whose recorded entry count is deliberately wrong, which is now
-- an input the heuristic reads rather than one it ignores.
-- Disposable catalog forgery, suite database of the sandbox cluster only.
SET /* wiki_btmaint_forge_client_min_messages */ client_min_messages = warning;
UPDATE /* wiki_btmaint_forge_84 */ pg_class SET reltuples = 5000
 WHERE relname = 'f84';
SELECT /* wiki_btmaint_forge_check */ 'f84 reltuples now ' || reltuples
  FROM pg_class WHERE relname = 'f84';
FORGE
  cat > "$SQLD/edge_autoanalyze.sql" <<'EDGE_AUTOANALYZE'
-- The same simulated auto-analyze rule, applied to the edge database so that
-- no fixture group is exempt from it.  Its curve fixtures analyze themselves
-- as part of their recipe, so this census is expected to find little or
-- nothing left to do; that is a result, and it is recorded rather than assumed.
-- Disposable fixtures, edge database of the sandbox cluster only.
SET /* wiki_btmaint_eautoanl_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_eautoanl_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_eautoanl_lock_timeout */ lock_timeout = '5s';

DROP TABLE IF EXISTS eautoanl;
CREATE TABLE eautoanl AS
SELECT /* wiki_btmaint_eautoanalyze_census */
       c.relname                                   AS tbl,
       GREATEST(c.reltuples, 0)::numeric           AS reltuples,
       st.n_mod_since_analyze::numeric             AS mods,
       round(current_setting('autovacuum_analyze_threshold')::numeric
             + current_setting('autovacuum_analyze_scale_factor')::numeric
               * GREATEST(c.reltuples, 0)::numeric, 1) AS threshold,
       round(100 * st.n_mod_since_analyze
             / GREATEST(c.reltuples, 1)::numeric, 1)   AS mod_pct,
       (st.n_mod_since_analyze
        > current_setting('autovacuum_analyze_threshold')::numeric
          + current_setting('autovacuum_analyze_scale_factor')::numeric
            * GREATEST(c.reltuples, 0)::numeric)       AS would_autoanalyze
  FROM pg_stat_all_tables st
  JOIN pg_class c ON c.oid = st.relid
 WHERE st.schemaname = 'public'
   AND c.relkind = 'r'
   AND c.relname NOT IN ('ecase', 'eplan', 'eautoanl');

SELECT /* wiki_btmaint_eautoanalyze_generator */
       format('ANALYZE /* wiki_btmaint_autoanalyze */ %I', tbl)
  FROM eautoanl WHERE would_autoanalyze ORDER BY tbl
\gexec
EDGE_AUTOANALYZE
  cat > "$SQLD/edge_forge.sql" <<'EDGE_FORGE'
-- The edge database's three catalog forgeries, re-applied after the simulated
-- auto-analyze for the same reason fixture 84's is: an ANALYZE of the table
-- rewrites reltuples for the table and for every index on it, which would
-- quietly repair the two "nothing has counted this" fixtures.
-- Disposable catalog forgeries, edge database of the sandbox cluster only.
SET /* wiki_btmaint_eforge_client_min_messages */ client_min_messages = warning;
UPDATE /* wiki_btmaint_eforge_invalid */ pg_index SET indisvalid = false
 WHERE indexrelid = 'e_inv_k'::regclass;
UPDATE /* wiki_btmaint_eforge_tbl_unknown */ pg_class SET reltuples = -1
 WHERE relname = 'e_unk';
UPDATE /* wiki_btmaint_eforge_idx_unknown */ pg_class SET reltuples = -1
 WHERE relname = 'e_iunk_k';
SELECT /* wiki_btmaint_eforge_check */ relname || ' reltuples ' || reltuples
  FROM pg_class WHERE relname IN ('e_unk', 'e_iunk_k') ORDER BY relname;
EDGE_FORGE
  cat > "$SQLD/edge_build.sql" <<'EDGE_BUILD'
-- The heuristic's own acceptance fixtures: comment parsing, comment
-- preservation, the candidate filters, the three gate boundaries and the 40 %
-- decision curve.  Everything here is disposable and lives in the edge
-- database of the sandbox cluster.
SET /* wiki_btmaint_edge_client_min_messages */ client_min_messages = warning;
SET /* wiki_btmaint_edge_statement_timeout */ statement_timeout = '900s';
SET /* wiki_btmaint_edge_lock_timeout */ lock_timeout = '5s';
SET /* wiki_btmaint_edge_maintenance_work_mem */ maintenance_work_mem = '256MB';

DROP TABLE IF EXISTS ecase CASCADE;
CREATE TABLE ecase(name text PRIMARY KEY, idx text, kind text, note text);
CREATE OR REPLACE FUNCTION ecase_add(n text, i text, k text, nt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS
$$ INSERT INTO ecase VALUES (n, i, k, nt) $$;

-- set_payload writes a forged baseline, so that a gate boundary can be hit
-- exactly without waiting for a table to grow.  Harness object, not part of
-- the heuristic.  itup is the index's own entry count, the third gate input.
CREATE OR REPLACE PROCEDURE set_payload(idx text, sz numeric, tup numeric,
                                        itup numeric DEFAULT 0,
                                        usr text DEFAULT NULL,
                                        raw text DEFAULT NULL)
LANGUAGE plpgsql AS $sp$
DECLARE body text;
BEGIN
  body := COALESCE(raw, '@btmaint:{"v":2,"sz":' || sz || ',"tup":' || tup
                        || ',"itup":' || itup
                        || ',"at":"2026-01-01T00:00:00+00"}');
  EXECUTE format('COMMENT ON INDEX %I IS %L', idx,
                 CASE WHEN usr IS NULL THEN body ELSE usr || E'\n' || body END);
END $sp$;

-- ---------------------------------------------------- comment parse cases ---
CREATE TABLE e_t AS SELECT i::int AS k, (i % 977)::int AS d
  FROM generate_series(1, 200000) i;
ANALYZE e_t;
CREATE INDEX e_none  ON e_t (k);
CREATE INDEX e_user  ON e_t (d);
CREATE INDEX e_bad1  ON e_t ((k + 1));
CREATE INDEX e_bad2  ON e_t ((k + 2));
CREATE INDEX e_bad3  ON e_t ((k + 3));
CREATE INDEX e_bad4  ON e_t ((k + 4));
CREATE INDEX e_bad5  ON e_t ((k + 5));
CREATE INDEX e_bad6  ON e_t ((k + 6));
CREATE INDEX e_bad7  ON e_t ((k + 7));
CREATE INDEX e_bad8  ON e_t ((k + 12));
CREATE INDEX e_bad9  ON e_t ((k + 13));
CREATE INDEX e_quote ON e_t ((k + 8));
CREATE INDEX e_long  ON e_t ((k + 9));
CREATE INDEX e_mid   ON e_t ((k + 10));
CREATE INDEX e_two   ON e_t ((k + 11));

COMMENT ON INDEX e_user IS 'human note, kept verbatim';
CALL set_payload('e_bad1', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":,"tup":1,"itup":1}');
CALL set_payload('e_bad2', 0, 0, 0, NULL, '@btmaint:{"v":9,"sz":1,"tup":1,"itup":1}');
CALL set_payload('e_bad3', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":"big","tup":1,"itup":1}');
CALL set_payload('e_bad4', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":1');
CALL set_payload('e_bad5', 0, 0, 0, NULL, '@btmaint:not json at all');
CALL set_payload('e_bad6', 0, 0, 0, NULL, '@btmaint:{"v":2,"tup":1,"itup":1}');
CALL set_payload('e_bad7', 0, 0, 0, NULL,
                 '@btmaint:{"v":2,"sz":99999999999999999999999999999999,"tup":1,"itup":1}');
-- The migration case: a well-formed payload in the format this page filed
-- before the index entry count was stored.  It must read invalid, because a
-- v1 baseline cannot answer the index tuple gate.
CALL set_payload('e_bad8', 0, 0, 0, NULL, '@btmaint:{"v":1,"sz":1,"tup":1}');
-- A v2 payload with the itup key missing, which is the same defect arriving
-- without the version number moving.
CALL set_payload('e_bad9', 0, 0, 0, NULL, '@btmaint:{"v":2,"sz":1,"tup":1}');
CALL set_payload('e_quote', 0, 0, 0,
                 'quote '' backslash \ percent %s newline'
                 || E'\n' || 'second line of the human note');
CALL set_payload('e_long', 0, 0, 0, repeat('L', 8000));
COMMENT ON INDEX e_mid IS '@btmaint:{"v":2,"sz":1,"tup":1,"itup":1} and text after it';
-- COMMENT takes a string literal, never an expression, so the two-marker case
-- is written as one literal with an embedded newline.
COMMENT ON INDEX e_two IS E'@btmaint:{"v":2,"sz":1,"tup":1,"itup":1}\n@btmaint:{"v":2,"sz":2,"tup":2,"itup":2}';
SELECT ecase_add('absent payload', 'e_none', 'parse', 'no comment at all');
SELECT ecase_add('user comment only', 'e_user', 'parse', 'human text, no marker');
SELECT ecase_add('empty sz', 'e_bad1', 'parse', 'malformed JSON');
SELECT ecase_add('wrong format version', 'e_bad2', 'parse', 'v = 9');
SELECT ecase_add('non-numeric sz', 'e_bad3', 'parse', 'quoted string');
SELECT ecase_add('truncated payload', 'e_bad4', 'parse', 'no closing brace');
SELECT ecase_add('marker without JSON', 'e_bad5', 'parse', 'free text');
SELECT ecase_add('missing sz key', 'e_bad6', 'parse', 'tup and itup only');
SELECT ecase_add('32-digit sz', 'e_bad7', 'parse', 'longer than the regex bound');
SELECT ecase_add('v1 payload, no index count', 'e_bad8', 'parse', 'the migration case');
SELECT ecase_add('missing itup key', 'e_bad9', 'parse', 'v2 shape, key absent');
SELECT ecase_add('quoting round trip', 'e_quote', 'parse', 'quote, backslash, percent, newline');
SELECT ecase_add('8000-character note', 'e_long', 'parse', 'toasted description');
SELECT ecase_add('marker mid-string', 'e_mid', 'parse', 'text after the payload');
SELECT ecase_add('two markers', 'e_two', 'parse', 'both must go');

-- ------------------------------------------------------ candidate filters ---
CREATE TABLE e_am AS SELECT i::int AS k, point(i, i) AS p, (i % 97)::int AS d,
       to_tsvector('simple', 'w' || i) AS tv
  FROM generate_series(1, 200000) i;
ANALYZE e_am;
CREATE INDEX e_am_hash   ON e_am USING hash (k);
CREATE INDEX e_am_gist   ON e_am USING gist (p);
CREATE INDEX e_am_spgist ON e_am USING spgist (p);
CREATE INDEX e_am_gin    ON e_am USING gin (tv);
CREATE INDEX e_am_brin   ON e_am USING brin (k);
CREATE INDEX e_am_btree  ON e_am (d);
SELECT ecase_add('hash index', 'e_am_hash', 'filter', 'pgstatindex refuses');
SELECT ecase_add('gist index', 'e_am_gist', 'filter', 'pgstatindex refuses');
SELECT ecase_add('spgist index', 'e_am_spgist', 'filter', 'pgstatindex refuses');
SELECT ecase_add('gin index', 'e_am_gin', 'filter', 'pgstatindex refuses');
SELECT ecase_add('brin index', 'e_am_brin', 'filter', 'pgstatindex refuses');
SELECT ecase_add('btree control on the same table', 'e_am_btree', 'filter', 'must be a candidate');

CREATE TABLE e_part(k int, v text) PARTITION BY RANGE (k);
CREATE TABLE e_part_1 PARTITION OF e_part FOR VALUES FROM (1) TO (100001);
INSERT INTO e_part SELECT i, lpad(i::text, 20, '0') FROM generate_series(1, 100000) i;
ANALYZE e_part;
CREATE INDEX e_part_k ON e_part (k);
SELECT ecase_add('partitioned index', 'e_part_k', 'filter', 'relkind I, no storage');
SELECT ecase_add('leaf partition index', 'e_part_1_k_idx', 'filter', 'relkind i, a candidate');

CREATE TABLE e_inv AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_inv;
CREATE INDEX e_inv_k ON e_inv (k);
UPDATE pg_index SET indisvalid = false
 WHERE indexrelid = 'e_inv_k'::regclass;    -- disposable catalog forgery
SELECT ecase_add('invalid index', 'e_inv_k', 'filter', 'forged indisvalid = false');

-- Both "nothing has counted this" fixtures get a readable baseline first, so
-- that the gate they disable is a gate that would otherwise have an opinion,
-- and the reason appears in notes instead of the row reading 'initialize'.
CREATE TABLE e_unk AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_unk;
CREATE INDEX e_unk_k ON e_unk (k);
CREATE TABLE e_iunk AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_iunk;
CREATE INDEX e_iunk_k ON e_iunk (k);
DO $eu$
DECLARE sz numeric; tup numeric; itup numeric;
BEGIN
  sz  := pg_relation_size('e_unk_k'::regclass);
  SELECT reltuples::numeric INTO tup  FROM pg_class WHERE relname = 'e_unk';
  SELECT reltuples::numeric INTO itup FROM pg_class WHERE relname = 'e_unk_k';
  CALL set_payload('e_unk_k', sz, tup, itup);
  sz  := pg_relation_size('e_iunk_k'::regclass);
  SELECT reltuples::numeric INTO tup  FROM pg_class WHERE relname = 'e_iunk';
  SELECT reltuples::numeric INTO itup FROM pg_class WHERE relname = 'e_iunk_k';
  CALL set_payload('e_iunk_k', sz, tup, itup);
END $eu$;
UPDATE pg_class SET reltuples = -1 WHERE relname = 'e_unk';     -- disposable forgery
UPDATE pg_class SET reltuples = -1 WHERE relname = 'e_iunk_k';  -- disposable forgery
SELECT ecase_add('table reltuples unknown', 'e_unk_k', 'filter', 'forged reltuples = -1');
SELECT ecase_add('index reltuples unknown', 'e_iunk_k', 'filter', 'forged index reltuples = -1');

-- --------------------------------------------------------- gate boundaries ---
-- One 200,000-row table, ten indexes, ten forged baselines.  The index size
-- and both entry counts are read back and the baseline is set so that the
-- ratio is exactly on, or just under, each of the three thresholds.  Only one
-- input is moved per fixture; the other two are set to the current value so
-- they cannot fire.
CREATE TABLE e_gate AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_gate;
CREATE INDEX e_g_size_on  ON e_gate (k);
CREATE INDEX e_g_size_off ON e_gate ((k + 1));
CREATE INDEX e_g_up_on    ON e_gate ((k + 2));
CREATE INDEX e_g_up_off   ON e_gate ((k + 3));
CREATE INDEX e_g_dn_on    ON e_gate ((k + 4));
CREATE INDEX e_g_dn_off   ON e_gate ((k + 5));
CREATE INDEX e_g_iup_on   ON e_gate ((k + 6));
CREATE INDEX e_g_iup_off  ON e_gate ((k + 7));
CREATE INDEX e_g_idn_on   ON e_gate ((k + 8));
CREATE INDEX e_g_idn_off  ON e_gate ((k + 9));
DO $eg$
DECLARE sz numeric; tup numeric; itup numeric;
BEGIN
  sz  := pg_relation_size('e_g_size_on'::regclass);
  tup := (SELECT reltuples::numeric FROM pg_class WHERE relname = 'e_gate');
  -- every index here is non-partial on the same table, so CREATE INDEX wrote
  -- the same entry count into all ten of them
  itup := (SELECT reltuples::numeric FROM pg_class WHERE relname = 'e_g_size_on');
  -- size gate: current / baseline >= 1.20 fires
  CALL set_payload('e_g_size_on',  floor(sz / 1.20),    tup, itup);
  CALL set_payload('e_g_size_off', ceil(sz / 1.20) + 1, tup, itup);
  -- table tuple gate: |current - baseline| >= 0.20 * baseline fires
  CALL set_payload('e_g_up_on',  sz, floor(tup / 1.20),        itup);
  CALL set_payload('e_g_up_off', sz, ceil(tup / 1.1999) + 1,   itup);
  CALL set_payload('e_g_dn_on',  sz, ceil(tup / 0.80),         itup);
  CALL set_payload('e_g_dn_off', sz, floor(tup / 0.8001) - 1,  itup);
  -- index tuple gate: the same arithmetic on the index's own entry count
  CALL set_payload('e_g_iup_on',  sz, tup, floor(itup / 1.20));
  CALL set_payload('e_g_iup_off', sz, tup, ceil(itup / 1.1999) + 1);
  CALL set_payload('e_g_idn_on',  sz, tup, ceil(itup / 0.80));
  CALL set_payload('e_g_idn_off', sz, tup, floor(itup / 0.8001) - 1);
END $eg$;
SELECT ecase_add('size ratio exactly 1.20', 'e_g_size_on', 'gate', 'must measure');
SELECT ecase_add('size ratio just under 1.20', 'e_g_size_off', 'gate', 'must skip');
SELECT ecase_add('table tuples up by 20%', 'e_g_up_on', 'gate', 'must measure');
SELECT ecase_add('table tuples up by just under 20%', 'e_g_up_off', 'gate', 'must skip');
SELECT ecase_add('table tuples down by 20%', 'e_g_dn_on', 'gate', 'must measure');
SELECT ecase_add('table tuples down by just under 20%', 'e_g_dn_off', 'gate', 'must skip');
SELECT ecase_add('index tuples up by 20%', 'e_g_iup_on', 'gate', 'must measure');
SELECT ecase_add('index tuples up by just under 20%', 'e_g_iup_off', 'gate', 'must skip');
SELECT ecase_add('index tuples down by 20%', 'e_g_idn_on', 'gate', 'must measure');
SELECT ecase_add('index tuples down by just under 20%', 'e_g_idn_off', 'gate', 'must skip');

-- ------------------------------------------------- the 40 % decision curve ---
-- Nine 500,000-row tables, one index each, drained by a known fraction and
-- vacuumed, with a baseline that makes the tuple gate fire everywhere.  The
-- curve is wasted_pct against the fraction deleted, and the action it produces.
DO $dc$
DECLARE f int; sz numeric; tup numeric; itup numeric;
BEGIN
  FOR f IN SELECT unnest(ARRAY[10, 20, 30, 40, 50, 60, 70, 80, 90]) LOOP
    EXECUTE format('CREATE TABLE e_del%s AS SELECT i::int AS k
                      FROM generate_series(1, 500000) i', f);
    EXECUTE format('ANALYZE e_del%s', f);
    EXECUTE format('CREATE INDEX e_del%s_k ON e_del%s (k)', f, f);
    -- the as-built baseline, stored the way a first run would store it.  A
    -- CALL argument cannot be a subquery, so all three values are read first.
    sz := pg_relation_size(format('e_del%s_k', f)::regclass);
    SELECT reltuples::numeric INTO tup FROM pg_class
      WHERE relname = format('e_del%s', f);
    SELECT reltuples::numeric INTO itup FROM pg_class
      WHERE relname = format('e_del%s_k', f);
    CALL set_payload(format('e_del%s_k', f), sz, tup, itup);
    EXECUTE format('SELECT ecase_add(''%s%% of the rows deleted'', ''e_del%s_k'',
                                     ''curve'', ''vacuumed, not reindexed'')', f, f);
  END LOOP;
END $dc$;
EDGE_BUILD
  cat > "$SQLD/edge_churn.sql" <<'EDGE_CHURN'
-- The 40 % decision curve: delete a known, scattered fraction of every
-- e_delN table, then VACUUM and ANALYZE so the entries are really gone.
-- Scattered deletion is the low-density case the wasted-space formula is
-- about; contiguous deletion, which empties whole pages instead, is fixtures
-- 77 and 91 of the numbered suite.  Generated one command at a time because
-- VACUUM cannot run inside a transaction block.
-- Disposable, edge database of the sandbox cluster only.
SET /* wiki_btmaint_edgechurn_client_min_messages */ client_min_messages = warning;
SELECT /* wiki_btmaint_curve_generator */ format(st.tmpl, tb.f)
  FROM (VALUES (10), (20), (30), (40), (50), (60), (70), (80), (90)) tb(f)
 CROSS JOIN (VALUES
        (1, 'DELETE /* wiki_btmaint_curve */ FROM e_del%1$s WHERE k %% 100 < %1$s'),
        (2, 'VACUUM /* wiki_btmaint_curve */ e_del%1$s'),
        (3, 'ANALYZE /* wiki_btmaint_curve */ e_del%1$s')) st(k, tmpl)
 ORDER BY tb.f, st.k
\gexec
EDGE_CHURN
  note "$(ls -1 "$SQLD" | tr "\n" " ")"
}

# ---------------------------------------------------------------- texts ------
stage_texts() {
  say "the page's two texts, hashed, and the one-edit harness view"
  [ -f "$PAGE" ] || die "no page at $PAGE; set PAGE or run from the repository root"
  md_block sql 1 "$PAGE" > "$SQLD/report.sql"
  md_block sql 2 "$PAGE" > "$SQLD/apply.sql"
  [ -s "$SQLD/report.sql" ] || die "block 1 of $PAGE is empty"
  [ -s "$SQLD/apply.sql" ]  || die "block 2 of $PAGE is empty"
  : > "$OUT/hashes.txt"
  local h1 h2
  h1=$(sha256sum < "$SQLD/report.sql" | cut -d' ' -f1)
  h2=$(sha256sum < "$SQLD/apply.sql"  | cut -d' ' -f1)
  printf 'report %s %s\n' "$h1" \
    "$([ "$h1" = "$BASE_REPORT" ] && echo match || echo DIFFERS)" >> "$OUT/hashes.txt"
  printf 'apply  %s %s\n' "$h2" \
    "$([ "$h2" = "$BASE_APPLY" ] && echo match || echo DIFFERS)" >> "$OUT/hashes.txt"
  printf 'report lines=%s bytes=%s\n' "$(grep -c '' "$SQLD/report.sql")" \
    "$(wc -c < "$SQLD/report.sql")" >> "$OUT/hashes.txt"
  printf 'apply  lines=%s bytes=%s\n' "$(grep -c '' "$SQLD/apply.sql")" \
    "$(wc -c < "$SQLD/apply.sql")" >> "$OUT/hashes.txt"
  # The shared pipeline: from "WITH params AS (" through the end of the staged
  # CTE, which is where the two texts part company - step 1 goes on to measure
  # in SQL, step 2 hands the snapshot to its driver.  The page claims this
  # region is byte-identical in both; this is the check that makes the claim
  # auditable.
  sed -n '/^WITH params AS ($/,/^      FROM decided d$/p' \
      "$SQLD/report.sql" > "$OUT/pipeline_report.sql"
  sed -n '/^WITH params AS ($/,/^      FROM decided d$/p' \
      "$SQLD/apply.sql" > "$OUT/pipeline_apply.sql"
  printf 'pipeline report %s\npipeline apply  %s\npipeline lines  %s\n' \
    "$(sha256sum < "$OUT/pipeline_report.sql" | cut -d' ' -f1)" \
    "$(sha256sum < "$OUT/pipeline_apply.sql" | cut -d' ' -f1)" \
    "$(grep -c '' "$OUT/pipeline_report.sql")" >> "$OUT/hashes.txt"
  if cmp -s "$OUT/pipeline_report.sql" "$OUT/pipeline_apply.sql"; then
    printf 'pipeline identical yes\n' >> "$OUT/hashes.txt"
  else
    printf 'pipeline identical NO\n' >> "$OUT/hashes.txt"
    diff "$OUT/pipeline_report.sql" "$OUT/pipeline_apply.sql" > "$OUT/pipeline_diff.txt"
  fi
  plan_view "$SQLD/report.sql" > "$SQLD/plan_view.sql"
  cat "$OUT/hashes.txt" >&2
}

# ---------------------------------------------------------------- exact ------
# Step 8 of the sibling page's protocol, applied here: execute the exact filed
# texts on this server and record what happens, before any fixture exists and
# before anything is transformed.  Two small fixtures are created first, because
# a text that returns no rows proves nothing about whether its expressions ran.
stage_exact() {
  say "the exact filed texts, on this server, unmodified"
  : > "$OUT/exact.txt"
  q suite 'DROP TABLE IF EXISTS zz_exact CASCADE' > /dev/null 2>&1
  f suite /dev/stdin > /dev/null 2>&1 <<'SQL'
CREATE TABLE zz_exact AS SELECT i::int AS k, (i % 997)::int AS d
  FROM generate_series(1, 200000) i;
ANALYZE zz_exact;
CREATE INDEX zz_exact_k ON zz_exact (k);
CREATE INDEX zz_exact_d ON zz_exact (d);
COMMENT ON INDEX zz_exact_d IS 'a human note that must survive';
SQL
  local rc
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d suite -f "$SQLD/report.sql" \
    > "$OUT/exact_report.txt" 2>&1; rc=$?
  printf 'report exit=%s rows=%s\n' "$rc" \
    "$(grep -c '^ public' "$OUT/exact_report.txt")" >> "$OUT/exact.txt"
  grep -E 'ERROR|LINE' "$OUT/exact_report.txt" | head -3 >> "$OUT/exact.txt"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite -f "$SQLD/apply.sql" \
    > "$OUT/exact_apply.txt" 2>&1; rc=$?
  printf 'apply  exit=%s %s\n' "$rc" "$(tail -1 "$OUT/exact_apply.txt")" >> "$OUT/exact.txt"
  grep -E 'ERROR|LINE' "$OUT/exact_apply.txt" | head -3 >> "$OUT/exact.txt"
  printf 'comment written: %s\n' \
    "$(s suite "SELECT replace(obj_description('zz_exact_d'::regclass, 'pg_class'),
                               chr(10), ' | ')")" >> "$OUT/exact.txt"
  printf 'second run: %s\n' \
    "$(f suite "$SQLD/apply.sql" 2>&1 | tail -1)" >> "$OUT/exact.txt"
  q suite 'DROP TABLE IF EXISTS zz_exact CASCADE' > /dev/null 2>&1
  cat "$OUT/exact.txt" >&2
}

# ---------------------------------------------------------------- facts ------
# Every version-local fact the two texts depend on, discovered on the running
# server rather than assumed.  The 12 leg runs the same stage, which is how the
# two servers are compared.
stage_facts() {
  say "version-local facts"
  : > "$OUT/facts.txt"
  local fact
  fact() { printf '%-34s %s\n' "$1" "$2" >> "$OUT/facts.txt"; }
  fact server_version_num "$(s suite 'SHOW server_version_num')"
  fact block_size "$(s suite 'SHOW block_size')"
  fact materialized_cte_accepted \
    "$(s suite 'WITH x AS MATERIALIZED (SELECT 1) SELECT count(*) FROM x' 2>&1 | tail -1)"
  fact pgstatindex_version \
    "$(s suite "SELECT extversion FROM pg_extension WHERE extname = 'pgstattuple'")"
  fact has_pg_input_is_valid \
    "$(s suite "SELECT count(*) FROM pg_proc WHERE proname = 'pg_input_is_valid'")"
  fact has_force_next_flush \
    "$(s suite "SELECT count(*) FROM pg_proc WHERE proname = 'pg_stat_force_next_flush'")"
  q suite 'DROP TABLE IF EXISTS zz_d' > /dev/null 2>&1
  q suite 'CREATE TABLE zz_d(k int)' > /dev/null
  fact deduplicate_items_reloption \
    "$(errf suite 'CREATE INDEX zz_di ON zz_d (k) WITH (deduplicate_items = off)')"
  q suite 'DROP TABLE IF EXISTS zz_d' > /dev/null 2>&1
  fact maintain_privilege \
    "$(errf suite "SELECT has_table_privilege('pg_class', 'MAINTAIN')")"
  fact icu_collation \
    "$(errf suite "CREATE COLLATION zz_icu (provider = icu, locale = 'und')")"
  q suite 'DROP COLLATION IF EXISTS zz_icu' > /dev/null 2>&1
  # reltuples on a table nothing has counted, and after each writer.
  q suite 'DROP TABLE IF EXISTS zz_rt' > /dev/null 2>&1
  q suite 'CREATE TABLE zz_rt(k int)' > /dev/null
  fact reltuples_after_create "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'INSERT INTO zz_rt SELECT i FROM generate_series(1,1000) i' > /dev/null
  fact reltuples_after_insert "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'CREATE INDEX zz_rt_k ON zz_rt (k)' > /dev/null
  fact reltuples_after_create_index "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'ANALYZE zz_rt' > /dev/null
  fact reltuples_after_analyze "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'TRUNCATE zz_rt' > /dev/null
  fact reltuples_after_truncate "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  q suite 'INSERT INTO zz_rt SELECT i FROM generate_series(1,2000) i' > /dev/null
  q suite 'REINDEX INDEX zz_rt_k' > /dev/null
  fact reltuples_after_reindex "$(s suite "SELECT reltuples FROM pg_class WHERE relname='zz_rt'")"
  # The index's own reltuples, which is the third gate input: who writes it,
  # what each writer writes, and what a partial index gets.  A plain index and
  # a partial index on the same 10,000-row table, where the predicate selects
  # exactly one row in five.
  fact autovacuum_analyze_threshold "$(s suite 'SHOW autovacuum_analyze_threshold')"
  fact autovacuum_analyze_scale_factor "$(s suite 'SHOW autovacuum_analyze_scale_factor')"
  q suite 'DROP TABLE IF EXISTS zz_it' > /dev/null 2>&1
  q suite 'CREATE TABLE zz_it AS SELECT i::int AS k, (i % 5 = 0) AS hot
             FROM generate_series(1,10000) i' > /dev/null
  q suite 'CREATE INDEX zz_it_all ON zz_it (k)' > /dev/null
  q suite 'CREATE INDEX zz_it_part ON zz_it (k) WHERE hot' > /dev/null
  local ir
  ir() { s suite "SELECT reltuples FROM pg_class WHERE relname='$1'"; }
  fact idx_reltuples_after_build "plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'ANALYZE zz_it' > /dev/null
  fact idx_reltuples_after_analyze \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'DELETE FROM zz_it WHERE hot AND k % 50 <> 0' > /dev/null
  q suite 'VACUUM zz_it' > /dev/null
  fact idx_reltuples_after_vacuum \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'ANALYZE zz_it' > /dev/null
  fact idx_reltuples_after_analyze2 \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  # A VACUUM with nothing to delete is cleanup-only, and an index AM that
  # reports an estimated count leaves pg_class alone: the count can therefore
  # be older than the last VACUUM.
  q suite 'UPDATE zz_it SET k = k + 100000 WHERE k % 1000 = 0' > /dev/null
  q suite 'VACUUM zz_it' > /dev/null
  fact idx_reltuples_after_small_vacuum \
    "table=$(ir zz_it) plain=$(ir zz_it_all) partial=$(ir zz_it_part)"
  q suite 'REINDEX INDEX zz_it_part' > /dev/null
  fact idx_reltuples_after_reindex "partial=$(ir zz_it_part)"
  q suite 'DROP TABLE IF EXISTS zz_it' > /dev/null 2>&1
  # Transaction control inside DO, which the apply block needs for its COMMIT,
  # and the two commands that cannot be reached from inside one.
  fact commit_inside_do \
    "$(errf suite 'DO $x$ BEGIN PERFORM 1; COMMIT; END $x$;')"
  fact commit_inside_do_in_xact \
    "$(errf suite 'BEGIN; DO $x$ BEGIN PERFORM 1; COMMIT; END $x$; COMMIT;')"
  fact reindex_plain_inside_do \
    "$(errf suite 'DO $x$ BEGIN EXECUTE $q$REINDEX INDEX zz_rt_k$q$; END $x$;')"
  fact reindex_conc_inside_do \
    "$(errf suite 'DO $x$ BEGIN EXECUTE $q$REINDEX INDEX CONCURRENTLY zz_rt_k$q$; END $x$;')"
  fact reindex_conc_top_level "$(errf suite 'REINDEX INDEX CONCURRENTLY zz_rt_k;')"
  fact vacuum_inside_do \
    "$(errf suite 'DO $x$ BEGIN EXECUTE $q$VACUUM zz_rt$q$; END $x$;')"
  fact comment_expression \
    "$(errf suite "COMMENT ON INDEX zz_rt_k IS 'a' || 'b';")"
  fact comment_literal "$(errf suite "COMMENT ON INDEX zz_rt_k IS 'ab';")"
  q suite 'DROP TABLE IF EXISTS zz_rt' > /dev/null 2>&1
  cat "$OUT/facts.txt" >&2
}

# ---------------------------------------------------------------- suite ------
# The ported numbered suite, in eight steps: build, baseline, churn,
# auto-analyze, forge, decide, act, oracle.  Steps 2 and 7 run the page's apply
# block exactly as filed.
stage_suite() {
  say "the ported numbered suite: tests 1-17, 18-91 and controls 92-121"
  q suite 'DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public' > /dev/null
  q suite 'CREATE EXTENSION IF NOT EXISTS pgstattuple' > /dev/null
  f suite "$SQLD/harness.sql"        || die "harness failed"
  f suite "$SQLD/fixtures_build.sql" > "$OUT/suite_build.log" 2>&1 \
    || { tail -5 "$OUT/suite_build.log" >&2; die "build fixtures failed"; }
  local have13=no haveicu=no
  if [ "$(s suite 'SHOW server_version_num')" -ge 130000 ]; then
    f suite "$SQLD/fixtures_v13.sql" >> "$OUT/suite_build.log" 2>&1 \
      && have13=yes || note "v13 fixtures failed, skipped"
  else
    note "server is older than 13: the four deduplicate_items fixtures are skipped"
  fi
  if f suite "$SQLD/fixtures_icu.sql" >> "$OUT/suite_build.log" 2>&1; then
    haveicu=yes
  else
    note "no ICU in this build: the five ICU fixtures are skipped"
  fi
  printf 'v13_fixtures=%s icu_fixtures=%s\n' "$have13" "$haveicu" > "$OUT/suite_groups.txt"
  note "$(s suite 'SELECT count(*) || $$ planned fixtures$$ FROM plan')"

  q suite 'CALL take_snap($$built$$)' || die "snapshot built failed"
  say "baseline: the filed apply block, first run"
  f suite "$SQLD/apply.sql" > "$OUT/apply_init.log" 2>&1 \
    || { tail -5 "$OUT/apply_init.log" >&2; die "apply (init) failed"; }
  grep -c 'initialize=' "$OUT/apply_init.log" > /dev/null
  tail -1 "$OUT/apply_init.log" >&2
  q suite 'CALL take_snap($$init$$)' || die "snapshot init failed"

  say "churn"
  f suite "$SQLD/fixtures_churn.sql" > "$OUT/suite_churn.log" 2>&1 \
    || { tail -5 "$OUT/suite_churn.log" >&2; die "churn failed"; }
  [ "$have13" = yes ] && f suite "$SQLD/churn_v13.sql" >> "$OUT/suite_churn.log" 2>&1
  [ "$haveicu" = yes ] && f suite "$SQLD/churn_icu.sql" >> "$OUT/suite_churn.log" 2>&1

  # The mandatory-test rule: a fixture that moved more than 10 % of a table's
  # heap tuples gets an ANALYZE, because autovacuum would have run one.  The
  # churn ran in psql processes that have now exited, so their statistics are
  # flushed; one second of grace covers the collector interval of the older
  # server, whose reader also waits for a fresh file.
  say "simulated auto-analyze: the engine's own threshold on its own counters"
  sleep 1
  f suite "$SQLD/autoanalyze.sql" > "$OUT/autoanalyze.log" 2>&1 \
    || { tail -5 "$OUT/autoanalyze.log" >&2; die "simulated auto-analyze failed"; }
  t suite "SELECT /* wiki_btmaint_autoanalyze_report */ tbl, reltuples, mods,
                  threshold, mod_pct, would_autoanalyze
             FROM autoanl ORDER BY would_autoanalyze DESC, mods DESC, tbl" \
    > "$OUT/autoanalyze.txt" 2>&1
  { printf 'tables considered  %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl')"
    printf 'analyzed           %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl WHERE would_autoanalyze')"
    printf 'left alone         %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl WHERE NOT would_autoanalyze')"
    printf 'counters reset     %s of %s\n' \
      "$(s suite 'SELECT count(*) FROM autoanl_after WHERE mods = 0')" \
      "$(s suite 'SELECT count(*) FROM autoanl_after')"
    printf 'mod_pct of those analyzed, min/median/max %s\n' \
      "$(s suite "SELECT min(mod_pct) || ' / ' ||
                         percentile_disc(0.5) WITHIN GROUP (ORDER BY mod_pct) || ' / ' ||
                         max(mod_pct) FROM autoanl WHERE would_autoanalyze")"
    printf 'largest mod_pct left alone %s\n' \
      "$(s suite "SELECT COALESCE(max(mod_pct)::text, 'none')
                    FROM autoanl WHERE NOT would_autoanalyze")"; } \
    >> "$OUT/autoanalyze.txt"
  [ "$(s suite 'SELECT count(*) FROM autoanl WHERE would_autoanalyze')" -gt 0 ] \
    || die "no table crossed the auto-analyze threshold: the churn counters are not visible"
  tail -6 "$OUT/autoanalyze.txt" >&2

  say "forge: the catalog forgeries an ANALYZE would have overwritten"
  f suite "$SQLD/forge.sql" > "$OUT/forge.log" 2>&1 || die "forge failed"
  cat "$OUT/forge.log" >&2
  q suite 'CALL take_snap($$churned$$)' || die "snapshot churned failed"

  say "decide: the filed report statement, as filed and through the one-edit view"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d suite -f "$SQLD/report.sql" \
    > "$OUT/report_churned.txt" 2>&1 || die "filed report failed"
  f suite "$SQLD/plan_view.sql" || die "plan_v failed"
  f suite /dev/stdin <<'SQL' || die "recording the plan failed"
DELETE FROM truth;
INSERT INTO truth(idx, action, baseline, wasted_pct, notes, bytes_churned, cmd_report)
SELECT v.index_name, v.action, v.baseline, v.wasted_pct, v.notes, s.bytes,
       v.comment_command
  FROM plan_v v
  JOIN snap s ON s.phase = 'churned' AND s.idx = v.index_name;
SQL
  note "$(s suite 'SELECT count(*) || $$ decisions recorded$$ FROM truth')"

  say "act: the filed apply block, second run"
  f suite "$SQLD/apply.sql" > "$OUT/apply_act.log" 2>&1 \
    || { tail -20 "$OUT/apply_act.log" >&2; die "apply (act) failed"; }
  tail -1 "$OUT/apply_act.log" >&2
  q suite 'CALL take_snap($$applied$$)' || die "snapshot applied failed"
  f suite /dev/stdin <<'SQL' || die "recording the applied state failed"
UPDATE truth t
   SET bytes_applied = a.bytes,
       cmd_written = a.cmt,
       reindexed_by_heuristic = (a.bytes < t.bytes_churned)
  FROM snap a
 WHERE a.phase = 'applied' AND a.idx = t.idx;
SQL

  say "oracle: REINDEX INDEX on every fixture"
  f suite /dev/stdin <<'SQL' || die "ground truth failed"
SET /* wiki_btmaint_oracle_statement_timeout */ statement_timeout = '900s';
CALL ground_truth();
SQL
  note "$(s suite 'SELECT count(*) || $$ fixtures with an oracle$$ FROM truth WHERE bytes_fresh IS NOT NULL')"
}

# ---------------------------------------------------------------- edge -------
stage_edge() {
  say "the heuristic's own acceptance fixtures"
  q edge 'DROP SCHEMA IF EXISTS public CASCADE; CREATE SCHEMA public' > /dev/null
  q edge 'CREATE EXTENSION IF NOT EXISTS pgstattuple' > /dev/null
  f edge "$SQLD/edge_build.sql" > "$OUT/edge_build.log" 2>&1 \
    || { tail -5 "$OUT/edge_build.log" >&2; die "edge fixtures failed"; }
  f edge "$SQLD/edge_churn.sql" > "$OUT/edge_churn.log" 2>&1 \
    || { tail -5 "$OUT/edge_churn.log" >&2; die "edge churn failed"; }
  # The same >10 % rule, and then the forgeries an ANALYZE would have repaired.
  sleep 1
  f edge "$SQLD/edge_autoanalyze.sql" > "$OUT/edge_autoanalyze.log" 2>&1 \
    || { tail -5 "$OUT/edge_autoanalyze.log" >&2; die "edge auto-analyze failed"; }
  f edge "$SQLD/edge_forge.sql" > "$OUT/edge_forge.log" 2>&1 \
    || die "edge forge failed"
  t edge "SELECT /* wiki_btmaint_eautoanalyze_report */ tbl, reltuples, mods,
                 threshold, mod_pct, would_autoanalyze
            FROM eautoanl ORDER BY would_autoanalyze DESC, mods DESC, tbl" \
    > "$OUT/edge_autoanalyze.txt" 2>&1
  cat "$OUT/edge_forge.log" >> "$OUT/edge_autoanalyze.txt"
  f edge "$SQLD/plan_view.sql" || die "plan_v in edge failed"

  # What the filed statement decides, and what the raw pgstatindex call does to
  # the shapes the candidate filter is there to keep out.
  q edge 'DROP TABLE IF EXISTS eplan' > /dev/null 2>&1
  f edge /dev/stdin <<'SQL' || die "edge plan failed"
CREATE TABLE eplan AS
SELECT e.name, e.idx, e.kind, e.note, v.action, v.baseline, v.wasted_pct,
       v.size_ratio, v.tuple_ratio_now, v.idx_tuple_ratio_now, v.notes,
       v.comment_command, (v.index_name IS NOT NULL) AS is_candidate
  FROM ecase e LEFT JOIN plan_v v ON v.index_name = e.idx;
SQL
  t edge "SELECT /* wiki_btmaint_edge_parse */ name, idx, action, baseline, notes
            FROM eplan WHERE kind = 'parse' ORDER BY idx" > "$OUT/edge_parse.txt" 2>&1
  t edge "SELECT /* wiki_btmaint_edge_filter */ name, idx, is_candidate, action, notes
            FROM eplan WHERE kind = 'filter' ORDER BY idx" > "$OUT/edge_filter.txt" 2>&1
  t edge "SELECT /* wiki_btmaint_edge_gate */ e.name, e.idx, p.action,
                 round(pg_relation_size(e.idx::regclass)
                       / substring(obj_description(e.idx::regclass, 'pg_class')
                                   from '\"sz\":([0-9]+)')::numeric, 6) AS size_ratio,
                 round((SELECT t.reltuples::numeric FROM pg_class c
                          JOIN pg_index x ON x.indexrelid = c.oid
                          JOIN pg_class t ON t.oid = x.indrelid
                         WHERE c.oid = e.idx::regclass)
                       / substring(obj_description(e.idx::regclass, 'pg_class')
                                   from '\"tup\":([0-9]+)')::numeric, 6) AS tuple_ratio,
                 round((SELECT c.reltuples::numeric FROM pg_class c
                         WHERE c.oid = e.idx::regclass)
                       / substring(obj_description(e.idx::regclass, 'pg_class')
                                   from '\"itup\":([0-9]+)')::numeric, 6)
                     AS idx_tuple_ratio,
                 p.notes
            FROM ecase e JOIN eplan p ON p.idx = e.idx
           WHERE e.kind = 'gate' ORDER BY e.idx" > "$OUT/edge_gate.txt" 2>&1
  t edge "SELECT /* wiki_btmaint_edge_curve */ name, idx, action, wasted_pct
            FROM eplan WHERE kind = 'curve'
           ORDER BY length(name), name" > "$OUT/edge_curve.txt" 2>&1

  say "what pgstatindex does to the shapes the filter excludes"
  : > "$OUT/edge_refusals.txt"
  local ix
  for ix in e_am_hash e_am_gist e_am_spgist e_am_gin e_am_brin e_part_k e_inv_k; do
    printf '%-12s %s\n' "$ix" \
      "$(err edge "SELECT * FROM pgstatindex('$ix')" | head -1)" >> "$OUT/edge_refusals.txt"
  done
  printf '%-12s %s\n' e_am_btree \
    "$(s edge "SELECT 'row returned, leaf_pages=' || leaf_pages FROM pgstatindex('e_am_btree')")" \
    >> "$OUT/edge_refusals.txt"
  # A temporary index belonging to another session: the filter must exclude it,
  # and a direct call must refuse it.  The owning session is this psql process,
  # which stays alive while the second one asks by OID, because the asking
  # session cannot resolve another session's pg_temp schema by name.
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=0 -d edge > "$OUT/edge_temp.txt" 2>&1 <<SQL
CREATE TEMP TABLE e_tmp AS SELECT i::int AS k FROM generate_series(1, 50000) i;
CREATE INDEX e_tmp_k ON e_tmp (k);
ANALYZE e_tmp;
SELECT 'own session candidate rows: ' || count(*) FROM plan_v WHERE index_name = 'e_tmp_k';
SELECT 'own session pgstatindex leaf_pages: ' || leaf_pages FROM pgstatindex('e_tmp_k');
\o | tr -d '\n' > $OUT/edge_temp_oid.txt
SELECT 'e_tmp_k'::regclass::oid;
\o
\! $BIN/psql -X -At -q -d edge -c "SELECT 'other session candidate rows: ' || count(*) FROM plan_v WHERE index_name = 'e_tmp_k'"
\! $BIN/psql -X -At -q -d edge -c "SELECT * FROM pgstatindex(\$(cat $OUT/edge_temp_oid.txt)::oid::regclass)" 2>&1 | tail -1
SQL

  say "comment survival across both REINDEX forms"
  : > "$OUT/edge_survival.txt"
  f edge /dev/stdin <<'SQL'
DROP TABLE IF EXISTS e_surv CASCADE;
CREATE TABLE e_surv AS SELECT i::int AS k FROM generate_series(1, 200000) i;
ANALYZE e_surv;
CREATE INDEX e_surv_a ON e_surv (k);
CREATE INDEX e_surv_b ON e_surv ((k + 1));
COMMENT ON INDEX e_surv_a IS E'human note\n@btmaint:{"v":2,"sz":1,"tup":2,"itup":3}';
COMMENT ON INDEX e_surv_b IS E'human note\n@btmaint:{"v":2,"sz":1,"tup":2,"itup":3}';
SQL
  printf 'before      a=%s b=%s\n' \
    "$(s edge "SELECT md5(obj_description('e_surv_a'::regclass,'pg_class'))")" \
    "$(s edge "SELECT md5(obj_description('e_surv_b'::regclass,'pg_class'))")" \
    >> "$OUT/edge_survival.txt"
  printf 'oid before  a=%s b=%s\n' \
    "$(s edge "SELECT 'e_surv_a'::regclass::oid")" \
    "$(s edge "SELECT 'e_surv_b'::regclass::oid")" >> "$OUT/edge_survival.txt"
  q edge 'REINDEX INDEX e_surv_a' > /dev/null || die "plain reindex failed"
  q edge 'REINDEX INDEX CONCURRENTLY e_surv_b' > /dev/null \
    || printf 'concurrent reindex failed\n' >> "$OUT/edge_survival.txt"
  printf 'after       a=%s b=%s\n' \
    "$(s edge "SELECT md5(obj_description('e_surv_a'::regclass,'pg_class'))")" \
    "$(s edge "SELECT md5(obj_description('e_surv_b'::regclass,'pg_class'))")" \
    >> "$OUT/edge_survival.txt"
  printf 'oid after   a=%s b=%s\n' \
    "$(s edge "SELECT 'e_surv_a'::regclass::oid")" \
    "$(s edge "SELECT 'e_surv_b'::regclass::oid")" >> "$OUT/edge_survival.txt"
  printf 'text        %s\n' \
    "$(s edge "SELECT replace(obj_description('e_surv_b'::regclass,'pg_class'), chr(10), ' | ')")" \
    >> "$OUT/edge_survival.txt"

  say "privileges: a reader, and a MAINTAIN grantee where the privilege exists"
  : > "$OUT/edge_priv.txt"
  q edge 'DROP ROLE IF EXISTS btm_reader' > /dev/null 2>&1
  q edge 'CREATE ROLE btm_reader LOGIN' > /dev/null
  # stage_edge recreated the public schema, so its owner-only default ACL has
  # to be opened before the reader can even see a relation in it.
  q edge 'GRANT USAGE ON SCHEMA public TO btm_reader' > /dev/null
  q edge 'GRANT SELECT ON e_t TO btm_reader' > /dev/null
  q edge 'GRANT EXECUTE ON FUNCTION pgstatindex(regclass) TO btm_reader' > /dev/null
  q edge 'GRANT SELECT ON plan_v TO btm_reader' > /dev/null
  printf 'owner candidate rows      %s\n' \
    "$(s edge "SELECT count(*) FROM plan_v WHERE index_name = 'e_none'")" >> "$OUT/edge_priv.txt"
  printf 'reader rows for e_none    %s\n' \
    "$("$BIN/psql" -X -At -q -U btm_reader -d edge \
        -c "SELECT count(*) FROM plan_v WHERE index_name = 'e_none'" 2>&1 | tail -1)" \
    >> "$OUT/edge_priv.txt"
  printf 'reader action on e_none   %s\n' \
    "$("$BIN/psql" -X -At -q -F ' / ' -U btm_reader -d edge \
        -c "SELECT action, notes FROM plan_v WHERE index_name = 'e_none'" 2>&1 | tail -1)" \
    >> "$OUT/edge_priv.txt"
  printf 'reader COMMENT refusal    %s\n' \
    "$("$BIN/psql" -X -q -U btm_reader -d edge \
        -c "COMMENT ON INDEX e_none IS 'x'" 2>&1 | grep -E 'ERROR' | head -1)" \
    >> "$OUT/edge_priv.txt"
  printf 'reader REINDEX refusal    %s\n' \
    "$("$BIN/psql" -X -q -U btm_reader -d edge -c 'REINDEX INDEX e_none' 2>&1 \
        | grep -E 'ERROR' | head -1)" >> "$OUT/edge_priv.txt"
  if [ "$(s edge 'SHOW server_version_num')" -ge 160000 ]; then
    q edge 'GRANT MAINTAIN ON e_t TO btm_reader' > /dev/null \
      && printf 'MAINTAIN granted          yes\n' >> "$OUT/edge_priv.txt"
    printf 'MAINTAIN REINDEX          %s\n' \
      "$("$BIN/psql" -X -q -U btm_reader -d edge -c 'REINDEX INDEX e_none' 2>&1 \
          | grep -E 'ERROR' | head -1 || echo 'accepted')" >> "$OUT/edge_priv.txt"
    printf 'MAINTAIN COMMENT          %s\n' \
      "$("$BIN/psql" -X -q -U btm_reader -d edge -c "COMMENT ON INDEX e_none IS 'x'" 2>&1 \
          | grep -E 'ERROR' | head -1)" >> "$OUT/edge_priv.txt"
    printf 'MAINTAIN action on e_none %s\n' \
      "$("$BIN/psql" -X -At -q -U btm_reader -d edge \
          -c "SELECT action FROM plan_v WHERE index_name = 'e_none'" 2>&1 | tail -1)" \
      >> "$OUT/edge_priv.txt"
  else
    printf 'MAINTAIN privilege        %s\n' \
      "$(err edge "SELECT has_table_privilege('e_t','MAINTAIN')" | head -1)" >> "$OUT/edge_priv.txt"
  fi
  printf 'index owner = table owner %s\n' \
    "$(s edge "SELECT count(*) FROM pg_class c JOIN pg_index x ON x.indexrelid = c.oid
                JOIN pg_class t ON t.oid = x.indrelid
               WHERE c.relkind = 'i' AND c.relowner <> t.relowner")" >> "$OUT/edge_priv.txt"
  # ALTER INDEX ... OWNER TO does not raise: it warns and does nothing, so the
  # probe has to read WARNING as well as ERROR, and then re-read the owners.
  printf 'ALTER INDEX OWNER answer   %s\n' \
    "$("$BIN/psql" -X -q -d edge -c 'ALTER INDEX e_none OWNER TO btm_reader' 2>&1 \
        | grep -E 'WARNING|ERROR|HINT' | tr '\n' ' ')" >> "$OUT/edge_priv.txt"
  printf 'owners after that attempt  %s\n' \
    "$(s edge "SELECT c.relowner::regrole::text || ' / ' || t.relowner::regrole::text
                 FROM pg_class c JOIN pg_index x ON x.indexrelid = c.oid
                 JOIN pg_class t ON t.oid = x.indrelid WHERE c.relname = 'e_none'")" \
    >> "$OUT/edge_priv.txt"

  say "the lock COMMENT ON INDEX takes, and what dry_run writes"
  s edge "BEGIN; COMMENT /* wiki_btmaint_lockprobe */ ON INDEX e_none IS 'lock probe';
          SELECT l.mode || ' on ' || l.relation::regclass
            FROM pg_locks l WHERE l.relation = 'e_none'::regclass; ROLLBACK" \
    > "$OUT/edge_lock.txt" 2>&1
  local before after
  before=$(s edge "SELECT md5(string_agg(objoid || ':' || description, ',' ORDER BY objoid))
                     FROM pg_description WHERE classoid = 'pg_class'::regclass")
  sed 's/^    dry_run     boolean := false;/    dry_run     boolean := true; --EDIT/' \
    "$SQLD/apply.sql" > "$SQLD/apply_dry.sql"
  cmp -s "$SQLD/apply.sql" "$SQLD/apply_dry.sql" && die "the dry_run edit matched nothing"
  f edge "$SQLD/apply_dry.sql" > "$OUT/edge_dryrun.log" 2>&1 || die "dry run failed"
  after=$(s edge "SELECT md5(string_agg(objoid || ':' || description, ',' ORDER BY objoid))
                    FROM pg_description WHERE classoid = 'pg_class'::regclass")
  printf 'pg_description md5 before %s\npg_description md5 after  %s\nunchanged %s\n' \
    "$before" "$after" "$([ "$before" = "$after" ] && echo yes || echo NO)" \
    > "$OUT/edge_dryrun.txt"
  tail -1 "$OUT/edge_dryrun.log" >> "$OUT/edge_dryrun.txt"

  say "apply for real, then twice more, to see what settles"
  f edge "$SQLD/apply.sql" > "$OUT/edge_apply1.log" 2>&1 || die "edge apply 1 failed"
  f edge "$SQLD/apply.sql" > "$OUT/edge_apply2.log" 2>&1 || die "edge apply 2 failed"
  f edge "$SQLD/apply.sql" > "$OUT/edge_apply3.log" 2>&1 || die "edge apply 3 failed"
  { printf 'run 1 %s\n' "$(tail -1 "$OUT/edge_apply1.log")"
    printf 'run 2 %s\n' "$(tail -1 "$OUT/edge_apply2.log")"
    printf 'run 3 %s\n' "$(tail -1 "$OUT/edge_apply3.log")"; } > "$OUT/edge_idempotence.txt"

  # The comment text that was actually written, against the command the report
  # printed, and the human text that had to survive both.
  t edge "SELECT /* wiki_btmaint_edge_written */ e.idx,
                 length(obj_description(e.idx::regclass, 'pg_class')) AS len,
                 replace(left(obj_description(e.idx::regclass, 'pg_class'), 120),
                         chr(10), ' | ') AS comment_now
            FROM ecase e WHERE e.kind = 'parse' ORDER BY e.idx" \
    > "$OUT/edge_written.txt" 2>&1
  # No LIKE here: the human text contains a per cent sign, which LIKE would
  # read as a wildcard.  left() on the exact original is an equality test.
  s edge "WITH orig(txt) AS (VALUES ('quote '' backslash \ percent %s newline'
                                     || chr(10) || 'second line of the human note'))
          SELECT 'quote case survived: ' ||
                 (left(obj_description('e_quote'::regclass, 'pg_class'),
                       length(o.txt)) = o.txt)
            FROM orig o" >> "$OUT/edge_written.txt" 2>&1
  s edge "SELECT 'long note kept: ' ||
                 (left(obj_description('e_long'::regclass,'pg_class'), 8000)
                  = repeat('L', 8000))" >> "$OUT/edge_written.txt" 2>&1
  s edge "SELECT 'payload bytes: ' ||
                 length(substring(obj_description('e_none'::regclass,'pg_class')
                                  from '@btmaint:\{[^}]*\}'))" >> "$OUT/edge_written.txt" 2>&1

  say "does a dump carry the baseline"
  "$BIN/pg_dump" -d edge -t e_t --schema-only > "$OUT/edge_dump.sql" 2>&1
  printf 'COMMENT ON INDEX lines in the dump: %s\nwith a payload: %s\n' \
    "$(grep -c '^COMMENT ON INDEX' "$OUT/edge_dump.sql")" \
    "$(grep -c '@btmaint:' "$OUT/edge_dump.sql")" > "$OUT/edge_dump.txt"
  cat "$OUT/edge_dump.txt" >&2
}

# ---------------------------------------------------------------- cost -------
# What a run costs, in the two states an operator actually meets: a settled
# database where every index is skipped, and one where the size gate fires
# everywhere, which is the worst case because pgstatindex reads every page of
# every gated index.  The gated state is forged by halving each stored sz; the
# suite has already been scored by the time this stage runs, and the forgery is
# recorded so no later stage reads those comments as real baselines.
stage_cost() {
  say "cost: the filed report in a settled and in a fully gated database"
  : > "$OUT/cost.txt"
  local i
  # Settle first: the oracle's rebuilds left many indexes smaller than their
  # stored baseline, which is a refresh, not a skip.  One apply run takes those
  # back to a state where the report has nothing to do, which is the state an
  # operator's scheduled run meets almost every time.
  f suite "$SQLD/apply.sql" > "$OUT/cost_settle.log" 2>&1
  printf 'settled state\n' >> "$OUT/cost.txt"
  s suite "SELECT '  gated indexes: ' || count(*) FILTER (WHERE action <> 'skip') ||
                  ' of ' || count(*) FROM plan_v" >> "$OUT/cost.txt"
  for i in 1 2 3 4 5 6; do
    printf '  run %s %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite -c '\timing on' \
           -f "$SQLD/report.sql" 2>&1 | grep -E '^Time:' | tail -1)" >> "$OUT/cost.txt"
  done
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
    -c 'EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM plan_v' 2>&1 \
    | grep -m 1 -E 'Buffers: shared' | sed 's/^ */  total /' >> "$OUT/cost.txt"
  printf 'fully gated state (every stored sz halved)\n' >> "$OUT/cost.txt"
  # Only the size input is forged: both tuple counts are written at their
  # current values, so the fully gated reading isolates one gate rather than
  # three, and the worst case it measures is still every gated index read end
  # to end.
  f suite /dev/stdin > /dev/null 2>&1 <<'SQL'
DO $fg$
DECLARE r record;
BEGIN
  FOR r IN SELECT p.idx, pg_relation_size(p.idx::regclass) AS b,
                  c.reltuples::numeric AS itup, t.reltuples::numeric AS tup
             FROM plan p
             JOIN pg_class c ON c.relname = p.idx AND c.relkind = 'i'
             JOIN pg_index x ON x.indexrelid = c.oid
             JOIN pg_class t ON t.oid = x.indrelid LOOP
    EXECUTE format('COMMENT /* wiki_btmaint_cost_forgery */ ON INDEX %I IS %L',
                   r.idx, '@btmaint:{"v":2,"sz":' || (r.b / 2)::bigint
                          || ',"tup":' || round(GREATEST(r.tup, -1))
                          || ',"itup":' || round(GREATEST(r.itup, -1))
                          || ',"at":"2026-01-01T00:00:00+00"}');
  END LOOP;
END $fg$;
SQL
  s suite "SELECT '  gated indexes: ' || count(*) FILTER (WHERE action <> 'skip') ||
                  ' of ' || count(*) FROM plan_v" >> "$OUT/cost.txt"
  for i in 1 2 3 4 5 6; do
    printf '  run %s %s\n' "$i" \
      "$("$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite -c '\timing on' \
           -f "$SQLD/report.sql" 2>&1 | grep -E '^Time:' | tail -1)" >> "$OUT/cost.txt"
  done
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d suite \
    -c 'EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM plan_v' 2>&1 \
    | grep -m 1 -E 'Buffers: shared' | sed 's/^ */  total /' >> "$OUT/cost.txt"
  s suite "SELECT '  total index bytes: ' ||
                  pg_size_pretty(sum(pg_relation_size(idx::regclass))) FROM plan" \
    >> "$OUT/cost.txt"
  cat "$OUT/cost.txt" >&2
}

# ---------------------------------------------------------------- score ------
stage_score() {
  say "score"
  t suite "SELECT /* wiki_btmaint_verdict_rows */ num, leg, grp, idx, action,
                  wasted_pct, actual_pct, applied_pct, verdict, lost_by,
                  expected_stage, taken_stage, two_gate_stage, want_stage,
                  size_ratio, tuple_ratio, idx_tuple_ratio
             FROM verdicts ORDER BY num, leg" > "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_verdict_counts */ verdict, count(*)
             FROM verdicts GROUP BY 1 ORDER BY 2 DESC" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_group_counts */ grp,
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE action = 'reindex') AS reindexed,
                  count(*) FILTER (WHERE action = 'update')  AS updated,
                  count(*) FILTER (WHERE action = 'skip')    AS skipped,
                  count(*) FILTER (WHERE action = 'refresh') AS refreshed,
                  count(*) FILTER (WHERE verdict = 'FALSE NEGATIVE') AS false_neg,
                  round(avg(wasted_pct), 1) AS avg_wasted,
                  round(avg(actual_pct), 1) AS avg_actual
             FROM verdicts GROUP BY grp ORDER BY grp" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_accuracy */
                  count(*) AS measured,
                  round(avg(actual_pct - wasted_pct), 1)  AS mean_error,
                  round(min(actual_pct - wasted_pct), 1)  AS min_error,
                  round(max(actual_pct - wasted_pct), 1)  AS max_error,
                  count(*) FILTER (WHERE abs(actual_pct - wasted_pct) <= 15) AS within_15,
                  count(*) FILTER (WHERE wasted_pct > actual_pct) AS over_estimates
             FROM verdicts WHERE wasted_pct IS NOT NULL" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_action_counts */ action, count(*),
                  round(avg(actual_pct), 1) AS avg_actual
             FROM verdicts GROUP BY 1 ORDER BY 2 DESC" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_gate_agreement */
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE expected_stage = taken_stage) AS gate_agrees,
                  count(*) FILTER (WHERE want_stage IS NOT NULL
                                     AND want_stage = taken_stage) AS want_hit,
                  count(*) FILTER (WHERE want_stage IS NOT NULL
                                     AND want_stage <> taken_stage) AS want_miss
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_lost */ num, leg, idx, action, wasted_pct,
                  actual_pct, lost_by, size_ratio, tuple_ratio
             FROM verdicts WHERE lost_by IS NOT NULL ORDER BY actual_pct DESC" \
    > "$OUT/lost.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_disagree */ num, leg, idx, expected_stage,
                  taken_stage, baseline, size_ratio, tuple_ratio, notes
             FROM verdicts WHERE expected_stage <> taken_stage ORDER BY num, leg" \
    > "$OUT/disagree.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_want_miss */ num, leg, idx, want_stage,
                  taken_stage, action, wasted_pct, actual_pct, req
             FROM verdicts WHERE want_stage IS NOT NULL AND want_stage <> taken_stage
            ORDER BY num, leg" > "$OUT/want_miss.txt" 2>&1
  # Did step 2 store the baseline step 1 said it would?  For every action but
  # reindex the two sz values are the same state and must agree; for reindex
  # step 1 prints no command and the stored size must be the post-rebuild one.
  t suite "SELECT /* wiki_btmaint_cmd_match */ action,
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE sz_reported IS NOT NULL) AS reported,
                  count(*) FILTER (WHERE sz_reported = sz_written) AS sz_agrees,
                  count(*) FILTER (WHERE sz_written = bytes_applied) AS sz_is_current
             FROM verdicts GROUP BY action ORDER BY action" >> "$OUT/verdicts.txt" 2>&1
  # Which gate opened each measured fixture, and what the third one changed.
  t suite "SELECT /* wiki_btmaint_gate_attribution */
                  count(*) FILTER (WHERE taken_stage = 'measure') AS measured,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND size_gate_fires) AS by_size,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND tbl_gate_fires) AS by_table_tuples,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND idx_gate_fires) AS by_index_tuples,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND idx_gate_fires AND NOT size_gate_fires
                                     AND NOT tbl_gate_fires) AS index_gate_only,
                  count(*) FILTER (WHERE taken_stage = 'measure'
                                     AND NOT size_gate_fires AND NOT tbl_gate_fires
                                     AND NOT idx_gate_fires) AS no_gate
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_two_gate_delta */
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE two_gate_stage = taken_stage) AS same_as_two_gate,
                  count(*) FILTER (WHERE two_gate_stage = 'skip'
                                     AND taken_stage = 'measure') AS newly_measured,
                  count(*) FILTER (WHERE two_gate_stage = 'skip'
                                     AND taken_stage = 'measure'
                                     AND actual_pct >= 50) AS newly_measured_bloated,
                  count(*) FILTER (WHERE two_gate_stage = 'skip'
                                     AND taken_stage = 'measure'
                                     AND actual_pct < 50) AS newly_measured_healthy,
                  count(*) FILTER (WHERE two_gate_stage = 'measure'
                                     AND taken_stage <> 'measure') AS lost_measurements
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_newly_measured */ num, leg, idx, action,
                  wasted_pct, actual_pct, verdict, idx_tuple_ratio, req
             FROM verdicts WHERE two_gate_stage = 'skip' AND taken_stage = 'measure'
            ORDER BY actual_pct DESC" > "$OUT/newly_measured.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_itup_stored */
                  count(*) AS fixtures,
                  count(*) FILTER (WHERE itup_stored_matches) AS itup_matches_catalog,
                  count(*) FILTER (WHERE base_idx_tuples IS NULL) AS itup_missing,
                  count(*) FILTER (WHERE itup_written IS NOT NULL) AS itup_written_back
             FROM verdicts" >> "$OUT/verdicts.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_lost_detail */ num, leg, idx, actual_pct,
                  size_ratio, tuple_ratio, idx_tuple_ratio,
                  size_gate_fires, tbl_gate_fires, idx_gate_fires, req
             FROM verdicts WHERE lost_by IS NOT NULL
            ORDER BY actual_pct DESC" >> "$OUT/lost.txt" 2>&1
  t suite "SELECT /* wiki_btmaint_payload_health */
                  count(*) AS indexes,
                  count(*) FILTER (WHERE payload IS NOT NULL) AS with_payload,
                  count(*) FILTER (WHERE base_bytes IS NULL) AS unparseable,
                  count(*) FILTER (WHERE cmt !~ '@btmaint:') AS no_marker
             FROM snap WHERE phase = 'applied'" >> "$OUT/verdicts.txt" 2>&1
  tail -40 "$OUT/verdicts.txt" >&2
}

# ------------------------------------------------- expected server errors ---
# Every server-side error this suite provokes is deliberate: the refusals the
# candidate filter exists to prevent, the privilege refusals, and the ALTER
# INDEX OWNER refusal.  An allowed error is a pair, the message and a fragment
# of the statement that raised it, so the same message from another statement is
# not allowed.  Only the lines after the mark this run wrote are read.
UNEXPECTED_ERRORS=0
check_server_errors() {
  local log=$1
  [ -f "$log" ] || { note "no server log"; return 0; }
  grep -E '^[0-9-]+ [0-9:.]+ [A-Z]+ (ERROR|FATAL|PANIC)' "$log" \
    | grep -Ev 'is not a btree index|cannot access temporary tables of other sessions|index "[^"]+" is not valid|must be owner of index|permission denied for (table|index)|must be owner of table|cannot change owner of index|relation "[^"]+" does not exist|REINDEX CONCURRENTLY cannot run inside a transaction block|cannot be executed from a function|deduplicate_items|ICU is not supported|unrecognized privilege type|column "inherited" does not exist' \
    > "$OUT/server_errors.txt"
  UNEXPECTED_ERRORS=$(grep -c '' "$OUT/server_errors.txt")
  printf 'unexpected server errors: %s\n' "$UNEXPECTED_ERRORS"
  [ "$UNEXPECTED_ERRORS" -gt 0 ] && head -5 "$OUT/server_errors.txt"
  return 0
}

# ---------------------------------------------------------------- criteria ---
stage_criteria() {
  say "pass criteria"
  # Every counter comes through a file, not through -c: a dollar-quoted string
  # inside a double-quoted shell argument would have $$ replaced by the shell's
  # own process id before psql ever saw it.
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d suite -f - > "$OUT/counters.txt" 2>&1 <<'SQL'
SELECT 'planned=' || count(*) FROM plan;
SELECT '  ' || verdict || '=' || count(*) FROM verdicts GROUP BY verdict ORDER BY verdict;
SELECT '  gate agree=' || count(*) FILTER (WHERE expected_stage = taken_stage) ||
       ' disagree='    || count(*) FILTER (WHERE expected_stage <> taken_stage) ||
       ' want_hit='    || count(*) FILTER (WHERE want_stage = taken_stage) ||
       ' want_miss='   || count(*) FILTER (WHERE want_stage <> taken_stage)
  FROM verdicts;
SELECT '  action ' || action || '=' || count(*) FROM verdicts GROUP BY action ORDER BY action;
SELECT '  payload with=' || count(*) FILTER (WHERE payload IS NOT NULL) ||
       ' unparseable='   || count(*) FILTER (WHERE base_bytes IS NULL) ||
       ' no_marker='     || count(*) FILTER (WHERE cmt !~ '@btmaint:')
  FROM snap WHERE phase = 'applied';
SELECT '  false negatives=' || count(*) FILTER (WHERE verdict = 'FALSE NEGATIVE') ||
       ' of which the index tuple gate opened=' ||
       count(*) FILTER (WHERE verdict = 'FALSE NEGATIVE' AND idx_gate_fires)
  FROM verdicts;
SELECT '  measured=' || count(*) FILTER (WHERE taken_stage = 'measure') ||
       ' by size='   || count(*) FILTER (WHERE taken_stage = 'measure' AND size_gate_fires) ||
       ' by table='  || count(*) FILTER (WHERE taken_stage = 'measure' AND tbl_gate_fires) ||
       ' by index='  || count(*) FILTER (WHERE taken_stage = 'measure' AND idx_gate_fires) ||
       ' index only='|| count(*) FILTER (WHERE taken_stage = 'measure' AND idx_gate_fires
                                          AND NOT size_gate_fires AND NOT tbl_gate_fires)
  FROM verdicts;
SELECT '  versus the two-gate form: same=' ||
       count(*) FILTER (WHERE two_gate_stage = taken_stage) ||
       ' newly measured=' || count(*) FILTER (WHERE two_gate_stage = 'skip'
                                                AND taken_stage = 'measure') ||
       ' of those bloated=' || count(*) FILTER (WHERE two_gate_stage = 'skip'
                                                 AND taken_stage = 'measure'
                                                 AND actual_pct >= 50) ||
       ' healthy=' || count(*) FILTER (WHERE two_gate_stage = 'skip'
                                        AND taken_stage = 'measure'
                                        AND actual_pct < 50)
  FROM verdicts;
SELECT '  itup stored = catalog count: ' ||
       count(*) FILTER (WHERE itup_stored_matches) || ' of ' || count(*)
  FROM verdicts;
SELECT '  simulated auto-analyze: ' ||
       count(*) FILTER (WHERE would_autoanalyze) || ' of ' || count(*) ||
       ' tables analyzed, largest left alone ' ||
       COALESCE(max(mod_pct) FILTER (WHERE NOT would_autoanalyze)::text, 'none') || ' %'
  FROM autoanl;
SQL
  { printf '1. texts\n'; cat "$OUT/hashes.txt" 2>/dev/null
    printf '2. engine checks\n'; cat "$OUT/checks.txt" 2>/dev/null
    printf '3. fixture groups\n'; cat "$OUT/suite_groups.txt" 2>/dev/null
    printf '4. counters\n'; sed 's/^/   /' "$OUT/counters.txt" 2>/dev/null
    printf '5. cost\n'; sed 's/^/   /' "$OUT/cost.txt" 2>/dev/null
    printf '5a. simulated auto-analyze\n'
    sed 's/^/   /' "$OUT/autoanalyze.txt" 2>/dev/null
    printf '6. edge groups\n'
    for fl in edge_parse edge_filter edge_gate edge_curve edge_refusals \
              edge_survival edge_priv edge_lock edge_dryrun edge_idempotence \
              edge_written edge_dump edge_autoanalyze; do
      printf '   --- %s\n' "$fl"; sed 's/^/   /' "$OUT/$fl.txt" 2>/dev/null
    done
    printf '7. facts\n'; sed 's/^/   /' "$OUT/facts.txt" 2>/dev/null
  } > "$OUT/criteria.txt" 2>&1
  { printf '8. server errors\n'; check_server_errors "$OUT/server.log"; } >> "$OUT/criteria.txt" 2>&1
  tail -60 "$OUT/criteria.txt" >&2
  note "full criteria in $OUT/criteria.txt"
}

# ---------------------------------------------------------------- report -----
stage_report() {
  say "report written to $OUT"
  ls -1 "$OUT" >&2
}

# ---------------------------------------------------------------- stop -------
# -m fast disconnects clients and lets the checkpointer write a shutdown
# checkpoint, so the next start needs no recovery.  The stop is then confirmed
# the way the teardown rule asks, and the stage dies rather than report a stop
# that did not happen, so clean never deletes a live cluster.
stage_stop() {
  say "stop the server cleanly"
  [ -x "$BIN/pg_ctl" ] || { note "no server binary under $BIN"; return 0; }
  if [ -s "$DATA/postmaster.pid" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null 2>&1 \
      || die "pg_ctl -m fast stop failed"
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
  note "confirmed: no postmaster.pid, no postgres process on $DATA, socket empty"
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
  inside_tmp "$SANDBOX" || die "refusing to delete $SANDBOX outside .wiki-runtime/tmp/"
  rm -rf "$SANDBOX"; say "sandbox deleted"
}

main() {
  local stages=("$@")
  [ ${#stages[@]} -eq 0 ] && stages=(build check cluster sql texts exact facts suite \
                                     edge cost score criteria report)
  local st
  for st in "${stages[@]}"; do
    case $st in
      build|check|cluster|sql|texts|exact|facts|suite|edge|cost|score|criteria|report|stop|clean)
        "stage_$st" ;;
      *) die "unknown stage: $st" ;;
    esac
  done
}

main "$@"
```

## Context Reviewed

- `contrib/pgstattuple/pgstatindex.c`: the `IS_INDEX`/`IS_BTREE` macros, all four
  refusals, the `BAS_BULKREAD` strategy, the block-by-block classification of
  leaf, internal, empty and deleted pages, `max_avail` and `free_space`, the
  `index_size` arithmetic including the metapage, and the two `NaN` branches.
- `contrib/pgstattuple/pgstattuple--1.4--1.5.sql` and `pgstattuple.control`: the
  `pgstatindex(regclass)` and `pgstatindex(text)` declarations, the
  `REVOKE ... FROM PUBLIC` and the `GRANT ... TO pg_stat_scan_tables`.
- `doc/src/sgml/pgstattuple.sgml`: the output-column table and the metapage note.
- `src/backend/commands/comment.c`: `CommentObject`'s
  `get_object_address(..., ShareUpdateExclusiveLock)` and
  `check_object_ownership`, and `CreateComments`'s replace-or-delete behaviour
  including the empty-string reduction to NULL.
- `src/backend/catalog/objectaddress.c`: `check_object_ownership` for
  `OBJECT_INDEX`, and the relkind check that rejects a non-index.
- `src/include/catalog/pg_description.h`: the catalog's three-column key and the
  `objsubid = 0` rule for a whole-object comment.
- `src/backend/catalog/index.c`: `index_concurrently_swap`'s comment move,
  `index_update_stats` and its empty-table `reltuples` hack, both
  `index_update_stats` calls in `index_build`, and `reindex_index`'s `ShareLock`
  on the heap with `AccessExclusiveLock` on the index.
- `src/backend/commands/indexcmds.c`: `ExecReindex`'s
  `PreventInTransactionBlock("REINDEX CONCURRENTLY")`, `ReindexIndex`'s lock
  modes, and `RangeVarCallbackForReindexIndex`'s `ACL_MAINTAIN` check.
- `src/include/catalog/pg_class.h`: the `reltuples` definition and its `-1`
  sentinel.
- `src/include/access/nbtree.h` and `src/backend/access/nbtree/nbtsort.c`:
  `BTGetFillFactor`, `BTGetTargetPageFreeSpace` and `_bt_pagestate`'s leaf-page
  fill target.
- `src/backend/catalog/system_functions.sql`: `pg_relation_size(regclass)`
  defaulting to the main fork, and `obj_description(oid, name)`.
- `src/backend/utils/adt/dbsize.c`: `pg_relation_size`'s `try_relation_open` and
  its NULL return for an already-dropped relation.
- `src/backend/utils/misc/guc_tables.c`: `statement_timeout` and `lock_timeout`
  as `PGC_USERSET`.
- `src/backend/storage/lmgr/lock.c`: the `LockConflicts` row for
  `ShareUpdateExclusiveLock`.
- `src/backend/parser/gram.y`: `comment_text` as `Sconst | NULL_P`.
- `src/backend/executor/nodeFunctionscan.c` and `src/backend/executor/execScan.c`:
  a set-returning function materialized into a tuplestore, and the qualification
  applied to the rows it produced.
- `src/backend/commands/vacuum.c` and `src/backend/commands/analyze.c`:
  `vac_update_relstats` as the writer VACUUM and ANALYZE share, its `relpages`
  and `reltuples` assignments, `do_analyze_rel`'s per-index `vac_update_relstats`
  loop and its `ceil(tupleFract * totalrows)`, the `tupleFract = 1.0`
  initialization, `compute_index_stats`'s "no columns to analyze and not partial"
  skip, and the `numindexrows / numrows` fraction it computes for a partial index.
- `src/backend/access/heap/vacuumlazy.c`: `update_relstats_all_indexes` and its
  `estimated_count` skip, `lazy_cleanup_all_indexes`'s
  `scanned_pages < rel_pages` test, `BYPASS_THRESHOLD_PAGES` and the
  `lazy_vacuum` bypass decision.
- `src/backend/access/nbtree/nbtree.c`: `btvacuumcleanup`'s cleanup-only branch
  and its `estimated_count = true`, the heap-count clamp, and `btvacuumpage`'s two
  ways of counting (`nhtidslive` with a callback, items per page without).
- `src/backend/postmaster/autovacuum.c`: `relation_needs_vacanalyze`'s
  `anl_base_thresh`/`anl_scale_factor` resolution and the
  `doanalyze = (anltuples > anlthresh)` test the mandatory tests now simulate.
- `src/backend/utils/activity/pgstat_relation.c` and
  `src/backend/utils/activity/pgstat.c`: `pgstat_report_analyze`'s
  `mod_since_analyze = 0` reset, the pending-stats flush that adds
  `changed_tuples` to the same counter, and `PGSTAT_MIN_INTERVAL`.
- `src/backend/catalog/system_views.sql`: `pg_stat_all_tables`'s
  `n_mod_since_analyze` column.
- `doc/src/sgml/config.sgml`: the two autovacuum analyze parameters, their
  defaults and their per-table overrides.
- `doc/src/sgml/ref/comment.sgml` and `doc/src/sgml/ref/reindex.sgml`: the
  one-comment-per-object rule, the `SHARE UPDATE EXCLUSIVE` lock, the
  owner-only rule, and `REINDEX`'s `MAINTAIN` requirement.
- The sibling pages this one reuses or contrasts with:
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](btree-index-bloat-core-sql-only.md) for the numbered suite,
  [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17
  (unverified)](btree-bloat-with-pgstatindex.md) for the wasted-space definition
  and the candidate filters, and
  [Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored
  Baseline in PostgreSQL 17
  (unverified)](non-btree-index-inflation-comment-baseline.md) for the
  comment-stored baseline pattern on the other access methods.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstatindex` refuses a non-B-tree relation, another session's temp index and an invalid index, and one raised error aborts the statement | [pgstatindex.c#refusals](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L250); measured refusals for six shapes on both servers |
| `avg_leaf_density` is live-leaf-page density only, and `NaN` with no leaf pages | [pgstatindex.c#avg_leaf_density](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372) |
| `index_size` counts every page including the metapage | [pgstatindex.c#index_size](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L350-L357), [pgstattuple.sgml#metapage](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L273) |
| The rebuild target is `BLCKSZ * (100 - fillfactor) / 100` free per leaf page | [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145), [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671) |
| A comment is one string per object, keyed by OID, replaced whole | [pg_description.h#FormData](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L44-L57), [comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L133-L171), [comment.sgml#replaces](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L89-L93) |
| `COMMENT ON INDEX` takes `ShareUpdateExclusiveLock` and requires ownership | [comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L78), [objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2387-L2400), [comment.sgml#lock](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L98); measured `ShareUpdateExclusiveLock on e_none` and `must be owner of index e_none` |
| The comment text is a literal, never an expression | [gram.y#comment_text](../../../../raw/postgres-17/src/backend/parser/gram.y#L7219-L7222); measured `syntax error at or near "||"` |
| A concurrent rebuild moves the comment to the new index | [index.c#index_concurrently_swap-comment](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784); measured OID 18147 -> 18149 with the comment md5 unchanged |
| A rebuild refreshes both `reltuples` values from its own scans | [index.c#index_build-update-stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L3129-L3134), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2788-L2812); measured 1000 after `CREATE INDEX` and 2000 after `REINDEX` on the table, 2000 then 200 on a partial index |
| `reltuples = -1` means uncounted from v14 | [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66); measured `-1` on 17.11 and `0` on 12.2 for the same sequence |
| ANALYZE writes an index's `reltuples` as `ceil(tupleFract * totalrows)`, with the fraction refined only for an index with statistics columns or a predicate | [analyze.c#same-for-indexes](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), [analyze.c#tupleFract-init](../../../../raw/postgres-17/src/backend/commands/analyze.c#L443-L449), [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L827-L863), [analyze.c#tupleFract-from-sample](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953); measured 2000 for a partial index on a 10,000-row table, and 10000 for the plain index beside it, on both servers |
| VACUUM writes an index's `reltuples` only when the AM's count is not estimated, and `btvacuumcleanup` marks a cleanup-only count estimated | [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3069-L3099), [vacuumlazy.c#lazy_cleanup_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2349-L2360), [nbtree.c#btvacuumcleanup-estimated](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L874-L892), [nbtree.c#btvacuumpage-counting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1345-L1362); measured 200 after a drain plus `VACUUM`, and unchanged after a 10-row `UPDATE` plus `VACUUM` |
| A VACUUM skips index vacuuming when under 2 % of heap pages hold dead items | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1940) |
| Autovacuum analyzes when `n_mod_since_analyze` exceeds `50 + 0.1 * reltuples` at the defaults, which is the rule the mandatory tests simulate | [autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3060-L3096), [autovacuum.c#anl-thresholds](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3005-L3018), [guc_tables.c#autovacuum_analyze_threshold](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375), [guc_tables.c#autovacuum_analyze_scale_factor](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914), [config.sgml#autovacuum_analyze_scale_factor](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L8832-L8850); both defaults read back as 50 and 0.1 on each server, and the census analyzed 77 of 99 tables on 17.11 with the boundary at 9.8 % and 11.1 % |
| `mod_since_analyze` is reset by ANALYZE and added to by a later flush, so it can outlive the ANALYZE that reset it | [pgstat_relation.c#report_analyze-reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L338), [pgstat_relation.c#flush-mod_since_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L845-L860), [pgstat.c#PGSTAT_MIN_INTERVAL](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L110-L122), [pgstat.c#pgstat_report_stat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L655); measured as `e_del80` reading 400,000 mods after its own `ANALYZE` while `e_del90` read 0 |
| The counter the census reads is `pg_stat_all_tables.n_mod_since_analyze` | [system_views.sql#n_mod_since_analyze](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L689) |
| `REINDEX INDEX` needs `MAINTAIN` on the table in v17, and takes `AccessExclusiveLock` on the index with `ShareLock` on the table | [indexcmds.c#RangeVarCallbackForReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2905-L2912), [indexcmds.c#ReindexIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2822-L2829), [index.c#reindex_index-locks](../../../../raw/postgres-17/src/backend/catalog/index.c#L3600-L3612), [reindex.sgml#MAINTAIN](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L300-L316); measured `permission denied for index` then accepted after `GRANT MAINTAIN`, while `COMMENT` stayed refused |
| `REINDEX CONCURRENTLY` cannot run from a `DO` block | [indexcmds.c#ExecReindex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2736-L2738); measured `REINDEX CONCURRENTLY cannot be executed from a function` on both servers |
| A set-returning function in `FROM` runs before its qualification | [nodeFunctionscan.c#FunctionNext](../../../../raw/postgres-17/src/backend/executor/nodeFunctionscan.c#L59-L112), [execScan.c#ExecScan](../../../../raw/postgres-17/src/backend/executor/execScan.c#L164-L205); measured 57,964 buffer reads in a settled database before the repair, 0 after |
| `pg_relation_size` returns NULL for a dropped relation instead of raising | [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L368) |
| `ShareUpdateExclusiveLock` conflicts with itself | [lock.c#LockConflicts](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L76-L80) |
| Both timeouts are session-scoped | [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2612-L2620), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2623-L2631) |
| VACUUM and ANALYZE share one `reltuples` writer | [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1404-L1417), [analyze.c#vac_update_relstats-call](../../../../raw/postgres-17/src/backend/commands/analyze.c#L637-L645) |

## Open Questions

1. **Four false negatives remain, and the measurement is what loses them.**
   `p113a`, `p113c`, `p65` and `p67` are opened by the index-count gate on both
   servers and then read 0.0 % to 0.1 % wasted, because their dead index entries
   were never vacuumed away and `pgstatindex` reports a dense file. A fourth
   input would be needed - the stored entry count against the leaf-page capacity
   the file implies, or `n_dead_tup` on the table - and neither is in the brief or
   measured here.
2. **The third gate can fire on sampling noise.** ANALYZE estimates a partial
   index's entry count from the same sample it uses for column statistics, so a
   small or skewed subset can move the stored count without the index moving:
   `p120` was measured at `idx_tuple_ratio` 0.1941 on 17.11 while the same fixture
   read 1.0000 on 12.2. One extra `pgstatindex()` call is the cost each time, and
   how often it happens as selectivity and `default_statistics_target` vary is
   unmeasured.
3. **The v1-to-v2 migration is measured only as a single comment write.** Every
   index carrying an old payload re-initializes on the first run, which is one
   write and no measurement per index; a database where that first run overlaps a
   real maintenance window was not constructed, and neither was a mixed database
   holding both payload versions.
4. **The baseline ratchets upward on every `update`.** An index measured at 35 %
   wasted is not rebuilt and its baseline is reset to the larger size, so the next
   20 % is counted from there. The page measures the single-step behaviour; the
   multi-run drift over many cycles is not measured.
5. **A stale-high baseline from a restored dump.** `pg_dump` carries the payload,
   so restoring an old dump restores an old baseline. Rule 3 handles the shrink
   direction; the case where a restored baseline is *lower* than reality was not
   constructed.
6. **The 13, 14, 15 and 16 majors were not run.** The compatibility claim rests on
   two measured legs plus the feature facts those legs discovered. Nothing here
   proves the texts parse on 13 through 16, although no construct they use was
   introduced or removed between 12 and 17.
7. **One platform, one block size.** Both legs ran on Linux x86_64 at
   `database_block_size` 8192 and `max_data_alignment` 8. The two page-layout
   constants in the text (24 and 16) are derived from that layout.
8. **The uniform 90 % drain is this page's invention.** 74 of the ported fixtures
   had no churn of their own, so they were drained by heap block number to give
   the heuristic an "after" state. That is a faithful stress for a density-based
   decision but it is not the fixture the sibling page filed, and a differently
   shaped drain could move the `skip`/`measure` boundary for those rows.
9. **`max_reindex` defaults to 1000 in the filed text.** The published run let it
   rebuild 103 indexes in one invocation, one transaction per index. A production
   setting of 1 is recommended in the text's own comment but was not the setting
   measured.
10. **No concurrency test.** Two overlapping runs, and a run racing a
    `DROP INDEX`, are unmeasured; the lock modes are known but the outcomes are
    not.
11. **The simulated auto-analyze over-triggers, by construction.** Because a
    backend's pending statistics can be flushed after the `ANALYZE` that reset
    the counter, the census analyzes some tables that were just analyzed. That is
    the safe direction for making counts fresh, but it means the suite's
    "77 of 99 analyzed" is an upper bound on what an autovacuum launcher would
    have done, not an equality.
12. **The census is a single pass, not a loop.** A real launcher re-checks every
    `autovacuum_naptime`, so a fixture could be analyzed, churned again and
    analyzed again before a scheduled heuristic run. The suite analyzes once,
    between the churn and the decision.

## Source References

- [pgstatindex.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c)
- [pgstattuple--1.4--1.5.sql](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql)
- [pgstattuple.control](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control)
- [pgstattuple.sgml](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml)
- [comment.c](../../../../raw/postgres-17/src/backend/commands/comment.c)
- [objectaddress.c](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c)
- [pg_description.h](../../../../raw/postgres-17/src/include/catalog/pg_description.h)
- [index.c](../../../../raw/postgres-17/src/backend/catalog/index.c)
- [indexcmds.c](../../../../raw/postgres-17/src/backend/commands/indexcmds.c)
- [pg_class.h](../../../../raw/postgres-17/src/include/catalog/pg_class.h)
- [nbtree.h](../../../../raw/postgres-17/src/include/access/nbtree.h)
- [nbtsort.c](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c)
- [system_functions.sql](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql)
- [dbsize.c](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c)
- [guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c)
- [lock.c](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c)
- [gram.y](../../../../raw/postgres-17/src/backend/parser/gram.y)
- [nodeFunctionscan.c](../../../../raw/postgres-17/src/backend/executor/nodeFunctionscan.c)
- [execScan.c](../../../../raw/postgres-17/src/backend/executor/execScan.c)
- [vacuum.c](../../../../raw/postgres-17/src/backend/commands/vacuum.c)
- [analyze.c](../../../../raw/postgres-17/src/backend/commands/analyze.c)
- [vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c)
- [nbtree.c](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c)
- [autovacuum.c](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c)
- [pgstat_relation.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c)
- [pgstat.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c)
- [system_views.sql](../../../../raw/postgres-17/src/backend/catalog/system_views.sql)
- [config.sgml](../../../../raw/postgres-17/doc/src/sgml/config.sgml)
- [comment.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml)
- [reindex.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml)

## Navigation

- [v17/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)](btree-index-bloat-core-sql-only.md)
- [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)](btree-bloat-with-pgstatindex.md)
- [Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17 (unverified)](non-btree-index-inflation-comment-baseline.md)
- [A COMMENT-Stored Baseline and Normalized Index Growth for Finding GIN Indexes That Need REINDEX CONCURRENTLY in PostgreSQL 17 (unverified)](gin-reindex-normalized-growth-comment-baseline.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
