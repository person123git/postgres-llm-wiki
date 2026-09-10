---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The statement](#the-statement)
  - [How to read the output](#how-to-read-the-output)
  - [Follow-up: no threshold, no verdict column](#follow-up-no-threshold-no-verdict-column)
  - [Follow-up: wasted space measured against the fillfactor](#follow-up-wasted-space-measured-against-the-fillfactor)
  - [What one pgstatindex call actually measures](#what-one-pgstatindex-call-actually-measures)
  - [Why every candidate filter is there](#why-every-candidate-filter-is-there)
  - [The one behavioural difference between 12 and 17](#the-one-behavioural-difference-between-12-and-17)
  - [The model, from avg_leaf_density to a rebuilt size](#the-model-from-avg_leaf_density-to-a-rebuilt-size)
  - [Why the two page-layout constants are safe](#why-the-two-page-layout-constants-are-safe)
  - [wasted_vs_fillfactor is not est_reclaimable](#wasted_vs_fillfactor-is-not-est_reclaimable)
  - [NaN is the trap](#nan-is-the-trap)
  - [Accuracy against REINDEX INDEX](#accuracy-against-reindex-index)
  - [The three shapes it gets wrong](#the-three-shapes-it-gets-wrong)
  - [Deduplication changes the input, not the arithmetic](#deduplication-changes-the-input-not-the-arithmetic)
  - [What it costs to run](#what-it-costs-to-run)
  - [Privileges](#privileges)
  - [Locking, timeouts, and the concurrent-drop race](#locking-timeouts-and-the-concurrent-drop-race)
  - [Everything the two servers agreed on](#everything-the-two-servers-agreed-on)
  - [Re-measured from a published script](#re-measured-from-a-published-script)
  - [How this was measured](#how-this-was-measured)
- [Measurement Script](#measurement-script)
  - [How to use the two leg scripts](#how-to-use-the-two-leg-scripts)
  - [The PostgreSQL 17 leg script](#the-postgresql-17-leg-script)
  - [The PostgreSQL 12 leg script](#the-postgresql-12-leg-script)
  - [The last run](#the-last-run)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)
## Question

In PostgreSQL 17: provide SQL that uses `pgstatindex` only, for B-tree indexes,
to report bloat and wasted space. Make sure that it works on v12 and v17.

Prompt note: the request was filed as `in postgresql 17 , question:  provide sql
that using pgstatindex only for btree indexes to report bloat, wasted space,
make sure that it works on v12 and v17`. It had lowercase `postgresql` and `sql`,
a space before a comma, a double space, `that using` for `that uses`, `btree` for
`B-tree`, and two comma splices; the asker approved the corrected restatement
above. Four scoping answers are recorded with it: `pgstatindex` is the only
measurement function, while `pg_class`, `pg_index`, `pg_am`, `pg_namespace` and
`pg_relation_size` may be read to choose, size and label indexes; the deliverable
is one statement that runs unchanged on both majors; the statement was executed on
isolated 12.2 and 17.11 servers and scored against `REINDEX INDEX`; and it is filed
as a new page rather than a follow-up on
[Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17](btree-index-bloat-core-sql-only.md),
which is deliberately the no-contrib method.

Follow-up: remove `alert_pct`. Report only the index statistics and the estimate
of wasted space.

> Prompt note: filed as an approved corrected restatement of `in postgresql 17 ,
> for question: B-Tree Bloat and Wasted Space From pgstatindex Alone, on
> PostgreSQL 12 and 17 (unverified) , remove the alert_pct , just report on the
> index stats and estimation of wasted space`, per the repository's
> prompt-hygiene rule; the original had `agents.md` for AGENTS.md, lowercase
> `postgresql`, three spaces before commas, and a comma splice. Three scoping
> answers are recorded with it: the `status` column goes with `alert_pct`,
> because the parameter existed only to drive it; the `notes` column stays
> exactly as it was; and both retained servers were restarted, their fixtures
> rebuilt, and the amended text run on each.

Second follow-up: adjust every wasted-space-related calculation to the index
fillfactor.

> Prompt note: filed as an approved corrected restatement of `follow agents.md,
> in postgresql 17 , for question: B-Tree Bloat and Wasted Space From
> pgstatindex Alone, on PostgreSQL 12 and 17 (unverified) , adjust any wasted
> space related calculation to the index fillfactor`, per the repository's
> prompt-hygiene rule; the original had `agents.md` for AGENTS.md, lowercase
> `postgresql`, two spaces before commas, and unhyphenated `wasted space
> related`. Four scoping answers are recorded with it: the baseline is the
> build-code target density the estimate already uses,
> `(leaf_capacity - BLCKSZ * (100 - fillfactor) / 100) / leaf_capacity`, not the
> literal `fillfactor / 100`, so both columns rest on one target; an index denser
> than that target reports zero rather than a negative number; the two output
> columns are renamed `wasted_vs_fillfactor` and `wasted_ff_pct` so the baseline
> is in the name and no old output is silently reinterpreted; and both retained
> servers were restarted and both texts run on each.

Review: following AGENTS.md, review this question page for PostgreSQL 17.

> Prompt note: filed as an approved corrected restatement of `follow agents.md,
> in postgresql 17,  review question: B-Tree Bloat and Wasted Space From
> pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)`, per the
> repository's prompt-hygiene rule; the original had `agents.md` for AGENTS.md,
> lowercase `postgresql`, a double space after the comma, and no finite verb.
> Four scoping answers are recorded with it: re-read every citation against the
> pinned checkout **and** rebuild both servers and re-measure; repair whatever
> the review finds, in place; and add the mandatory `## Measurement Script`
> section with a runnable script, then run it. What the review found is
> [Re-measured from a published script](#re-measured-from-a-published-script),
> the corrections carried in
> [Locking, timeouts, and the concurrent-drop race](#locking-timeouts-and-the-concurrent-drop-race),
> [The one behavioural difference between 12 and 17](#the-one-behavioural-difference-between-12-and-17)
> and [Why the two page-layout constants are safe](#why-the-two-page-layout-constants-are-safe),
> and the new [Measurement Script](#measurement-script) section.

## Answer

### The statement

One statement, eight stages, no measurement function other than `pgstatindex`. It
returned 27 rows on the 12.2 server and 28 on the 17.11 server, from the same text.

```sql
-- B-tree bloat and wasted space from pgstatindex alone.
-- Runs unchanged on PostgreSQL 12 and 17.
--
--   params    the size prefilter and the two page-layout constants
--   cand      every index pgstatindex can be called on without raising
--   measured  one pgstatindex() call per candidate
--   modelled  per-index constants: leaf capacity, fillfactor, target free space
--   sized     bytes holding entries, dead-page bytes, the fillfactor target
--   est       leaf pages a rebuild at this index's fillfactor would need
--   final     the modelled rebuilt size
--
-- wasted_vs_fillfactor measures the file against a rebuild at this index's own
-- fillfactor: free bytes in live leaf pages beyond what such a build leaves,
-- never below zero, plus every empty and deleted page.  est_reclaimable is the
-- same target read as a file size, which is what REINDEX gives back.
-- The statement sets no threshold and reaches no verdict.  It reports the
-- measurements and the estimate, ordered by est_reclaimable, largest first.

SET statement_timeout = '15min';
SET lock_timeout = '5s';

WITH params AS (
    SELECT current_setting('block_size')::bigint AS bs,
           24::bigint      AS page_header,     -- SizeOfPageHeaderData
           16::bigint      AS btree_special,   -- MAXALIGN(sizeof(BTPageOpaqueData))
           1048576::bigint AS min_index_bytes  -- skip anything smaller
),
cand AS MATERIALIZED (
    SELECT c.oid       AS idx_oid,
           n.nspname   AS schema_name,
           c.relname   AS index_name,
           t.relname   AS table_name,
           (SELECT o.option_value::int
              FROM pg_options_to_table(c.reloptions) o
             WHERE o.option_name = 'fillfactor') AS fillfactor_opt
      FROM pg_class c
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_am a        ON a.oid = c.relam
     CROSS JOIN params p
     WHERE a.amname = 'btree'                        -- pgstatindex takes no other AM
       AND c.relkind = 'i'                           -- not 'I', a partitioned index has no storage
       AND x.indisvalid AND x.indisready AND x.indislive
       AND NOT pg_is_other_temp_schema(c.relnamespace)
       AND (c.relpersistence <> 'u' OR NOT pg_is_in_recovery())
       AND pg_relation_size(c.oid) >= p.min_index_bytes
),
measured AS (
    SELECT c.*, s.*
      FROM cand c, LATERAL pgstatindex(c.idx_oid::regclass) s
),
modelled AS (
    SELECT m.*,
           p.bs,
           p.bs - p.page_header - p.btree_special AS leaf_capacity,
           COALESCE(m.fillfactor_opt, 90)         AS fillfactor,
           -- what a build leaves free on a leaf page: BLCKSZ * (100 - fillfactor) / 100
           (p.bs * (100 - COALESCE(m.fillfactor_opt, 90))) / 100 AS target_free,
           m.empty_pages + m.deleted_pages        AS dead_pages,
           -- an index with no leaf pages reports NaN; NaN sorts above every
           -- number, so it must never reach a comparison
           CASE WHEN m.leaf_pages > 0 AND m.avg_leaf_density <> 'NaN'::float8
                THEN (m.avg_leaf_density / 100)::numeric
                ELSE 0::numeric END               AS density
      FROM measured m CROSS JOIN params p
),
sized AS (
    SELECT d.*,
           d.leaf_pages * d.leaf_capacity                       AS leaf_bytes,
           round(d.leaf_pages * d.leaf_capacity * d.density)    AS live_leaf_bytes,
           d.dead_pages * d.bs                                  AS dead_bytes,
           (d.leaf_capacity - d.target_free)::numeric / d.leaf_capacity AS target_density
      FROM modelled d
),
est AS (
    SELECT s.*,
           -- free leaf bytes a rebuild at this fillfactor would not leave,
           -- never negative, plus pages that hold nothing at any fillfactor
           GREATEST(round(s.leaf_bytes * s.target_density) - s.live_leaf_bytes, 0)
               + s.dead_bytes AS wasted_vs_fillfactor,
           CASE WHEN s.leaf_pages = 0 THEN 0
                ELSE ceil(s.leaf_pages * s.density / s.target_density) END
               AS est_leaf_pages
      FROM sized s
),
final AS (
    SELECT e.*,
           (1 + e.est_leaf_pages
              + CASE WHEN e.leaf_pages = 0 THEN 0
                     ELSE round(e.internal_pages * e.est_leaf_pages / e.leaf_pages) END
           ) * e.bs AS est_rebuilt_bytes
      FROM est e
)
SELECT /* wiki_btree_bloat_pgstatindex_12_17 */
       f.schema_name,
       f.index_name,
       f.table_name,
       pg_size_pretty(f.index_size) AS index_size,
       f.leaf_pages,
       f.dead_pages,
       CASE WHEN f.leaf_pages > 0 THEN round(f.avg_leaf_density::numeric, 2) END
           AS avg_leaf_density,
       CASE WHEN f.leaf_pages > 0 THEN round(f.leaf_fragmentation::numeric, 2) END
           AS leaf_fragmentation,
       pg_size_pretty(f.wasted_vs_fillfactor::bigint) AS wasted_vs_fillfactor,
       round(100 * f.wasted_vs_fillfactor / f.index_size, 1) AS wasted_ff_pct,
       pg_size_pretty(f.est_rebuilt_bytes::bigint) AS est_rebuilt_size,
       pg_size_pretty((f.index_size - f.est_rebuilt_bytes)::bigint) AS est_reclaimable,
       round(100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size, 1)
           AS est_reclaimable_pct,
       array_to_string(array_remove(ARRAY[
           CASE WHEN f.leaf_pages = 0 THEN 'no leaf pages' END,
           CASE WHEN f.version < 4 THEN 'metapage version ' || f.version END,
           CASE WHEN f.fillfactor <> 90 THEN 'fillfactor ' || f.fillfactor END,
           CASE WHEN f.leaf_pages > 0 AND f.leaf_fragmentation >= 30
                THEN 'fragmented, not wasted space' END,
           CASE WHEN 100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size <= -1
                THEN 'denser than a rebuild would leave it' END,
           CASE WHEN f.dead_pages > 0
                 AND f.dead_bytes >= (f.index_size - f.est_rebuilt_bytes) / 2
                 AND f.index_size > f.est_rebuilt_bytes
                THEN 'reclaim is mostly empty/deleted pages' END
       ], NULL), '; ') AS notes
  FROM final f
 ORDER BY f.index_size - f.est_rebuilt_bytes DESC;
```

It needs `CREATE EXTENSION pgstattuple` in the database being examined. Both
checkouts ship the same `pgstattuple` 1.5 control file and the same install
script, so the function signature is the same on both
([pgstattuple.control:1-5](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).

`statement_timeout` and `lock_timeout` are both `PGC_USERSET`
([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620),
[guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)),
so they apply at session/transaction scope and need no reload or restart. The
statement timeout is minutes, not seconds, because this report reads every page
of every index it reports on; see [What it costs to run](#what-it-costs-to-run).

### How to read the output

Read `est_reclaimable_pct` first. It is the modelled answer to "how much smaller
would `REINDEX INDEX` make this file", and on the fixture suite it landed within
one point of the truth for 91 of 94 indexes on 12.2 and 89 of 93 on 17.11. The
statement itself reaches no verdict. It carries no threshold, labels no row, and
returns the measurements and the estimate ordered by estimated reclaim, largest
first. Why the column that used to carry a verdict is gone, and where the
threshold belongs instead, is
[Follow-up: no threshold, no verdict column](#follow-up-no-threshold-no-verdict-column).

Read the other columns as supporting detail:

| Column | Means | Watch for |
|---|---|---|
| `wasted_vs_fillfactor`, `wasted_ff_pct` | Free bytes inside live leaf pages beyond what a build at this index's fillfactor leaves, never below zero, plus every empty and deleted page | A fresh index reports 0.0 at any fillfactor. It counts payload bytes, so it runs about `(leaf_capacity - target_free) / block_size` of `est_reclaimable_pct` — 0.895 of it at 8192/90 |
| `avg_leaf_density` | Share of leaf-page space holding entries | Low density is the usual bloat signal, but it is blind to whole pages that hold nothing, and a low number is normal at a low fillfactor |
| `dead_pages` | `empty_pages + deleted_pages` | These are 100% waste at any fillfactor and invisible to `avg_leaf_density`. A measured index read 89.94% density and was still 69.9% reclaimable |
| `leaf_fragmentation` | Share of leaves whose right sibling sits at a lower block number | Not wasted space at all. Physical disorder that costs sequential-scan I/O; the note says so |
| `notes` | Why a row looks odd | `no leaf pages`, `fillfactor N`, `fragmented, not wasted space`, `denser than a rebuild would leave it`, `reclaim is mostly empty/deleted pages` |

Two rows from the 17.11 run show why both percentages exist:

```text
 index_name  | index_size | leaf_pages | dead_pages | avg_leaf_density | wasted_ff_pct | est_reclaimable_pct | notes
 i_delhead   | 21 MB      |        821 |       1918 |            89.94 |          69.9 |                69.9 | reclaim is mostly empty/deleted pages
 i_ff10      | 206 MB     |      26316 |          0 |             9.62 |           0.0 |                -0.5 | fillfactor 10
```

`i_delhead` has textbook-perfect leaves and is two thirds reclaimable, and here
the two columns agree to the tenth because every wasted byte is in a whole page
that holds nothing. `i_ff10` looks catastrophic by density and is exactly the
size its owner asked for, so both columns say there is nothing to take back.

### Follow-up: no threshold, no verdict column

The statement measures and estimates; it no longer judges. Five edits took out
the threshold and the column it drove, and nothing else in the text moved:

| Where | Was | Is |
|---|---|---|
| `params` | `20::numeric AS alert_pct` | gone, so `min_index_bytes` is the last entry |
| `modelled` | `p.alert_pct,` carried it down the pipeline | gone |
| presentation `SELECT` | `CASE WHEN 100 * (index_size - est_rebuilt_bytes) / index_size >= f.alert_pct THEN 'rebuild candidate' ELSE 'ok' END AS status` | gone |
| header comment | `Alert on est_reclaimable_pct; read wasted_pct for composition only.` | the statement sets no threshold and reaches no verdict, and returns rows ordered by `est_reclaimable`, largest first |
| stage list | `params  tunables and the two page-layout constants` | `params  the size prefilter and the two page-layout constants` |

The text drops from 125 lines and 6,002 bytes to 122 lines and 5,839 bytes, and
the output from 15 columns to 14. `notes` is untouched.

**Nothing else the statement returns moved.** Both servers were restarted from
the retained sandbox, both fixture scripts were re-run, and both texts were then
executed on each server:

| Check | 12.2 | 17.11 |
|---|---|---|
| Rows returned, either text | 27 | 28 |
| Filed text, `rebuild candidate` / `ok` | 15 / 12 | 15 / 13 |
| Amended output against the filed output with column 14 cut | identical, 2,448 bytes | identical, 2,531 bytes |
| Columns exposed by the internal `final` stage, filed against amended | 29 against 28, and `alert_pct` is the only one missing | 29 against 28, and `alert_pct` is the only one missing |
| `EXCEPT` in both directions over the 28 shared columns | 0 rows, 214 indexes | 0 rows, 220 indexes |

The row check compares `psql -A -F '|'` output with field 14 removed from the
filed run. The column check builds one view per text over the internal `final`
stage, generated mechanically from each text with the two `SET` lines dropped
and `min_index_bytes` set to 0, so every index in the database is compared and
not only those over a megabyte.

**The same run reproduces the report this page already filed.** On 17.11 the
filed text returned the archived table character for character. On 12.2 it
returned the same 27 rows with two of them swapped: `i_expr` and `i_text_del`
are both 27 MB reclaimable at 79.4%, and `ORDER BY index_size -
est_rebuilt_bytes DESC` has no tie-break, so tied rows may arrive in either
order. That belongs to the filed statement, not to this edit.

**Cost is unchanged**, which is what a removed `CASE` over an already-computed
expression should cost: the same plan shape on both servers (4 `CTE Scan` nodes,
68 plan lines on 17.11 and 60 on 12.2), `EXPLAIN (ANALYZE, BUFFERS)` execution
of 136.3 ms against 135.7 ms on 17.11 and 134.7 ms against 127.9 ms on 12.2, and
six interleaved end-to-end runs of each text spanning 131.5-143.8 ms against
132.2-137.6 ms on 17.11 and 132.6-140.2 ms against 128.7-139.4 ms on 12.2.

What a reader loses is the label, not the ranking. On this fixture suite the 15
rows the filed text called `rebuild candidate` are exactly the first 15 rows of
the amended output, in the same order, on both servers. That coincidence is not
a rule: the sort is on reclaimable **bytes** and the old label was on reclaimable
**percent**, so a small, badly bloated index can sort below a large, healthy one.
A caller that wants a threshold applies it to `est_reclaimable_pct` at the call
site, where it can differ per environment and per index size, instead of being
frozen at 20 inside a report whose job is to measure.

### Follow-up: wasted space measured against the fillfactor

Every wasted-space calculation is now rebased on the index's own fillfactor, so
a correctly built index reports no waste whatever its fillfactor is. The filed
column measured the file against perfect packing, which meant a healthy default
index always reported about 10% wasted and a deliberate `fillfactor = 10` index
reported 89.6% — a number that described the DBA's own instruction, not a
problem. Four edits, in one CTE and one `SELECT` list:

| Where | Was | Is |
|---|---|---|
| `est` | `s.leaf_bytes - s.live_leaf_bytes + s.dead_bytes AS wasted_space` | `GREATEST(round(s.leaf_bytes * s.target_density) - s.live_leaf_bytes, 0) + s.dead_bytes AS wasted_vs_fillfactor` |
| presentation `SELECT` | `pg_size_pretty(f.wasted_space::bigint) AS wasted_space` | `pg_size_pretty(f.wasted_vs_fillfactor::bigint) AS wasted_vs_fillfactor` |
| presentation `SELECT` | `round(100 * f.wasted_space / f.index_size, 1) AS wasted_pct` | `round(100 * f.wasted_vs_fillfactor / f.index_size, 1) AS wasted_ff_pct` |
| header comment, stage list | `wasted_space measures the file against perfect packing` | the fillfactor-relative definition, and `sized` now names the target |

The text grows from 122 lines and 5,839 bytes to 126 lines and 6,154 bytes, and
the output keeps its 14 columns. `notes` is untouched, and so is every other
expression in the statement.

Three decisions are worth stating, because each could have gone the other way:

- **The baseline is `target_density`, not `fillfactor / 100`.** It is the same
  `(leaf_capacity - BLCKSZ * (100 - fillfactor) / 100) / leaf_capacity` the
  rebuild estimate already uses — 89.95% at 8192 and fillfactor 90, not 90.00% —
  so the two columns now measure against one target instead of two that differ
  by 0.05 points ([nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671),
  [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)).
- **The leaf term is clamped at zero.** An index denser than its target is not
  holding negative waste; it is denser than a rebuild would leave it, which the
  `notes` column already says. `GREATEST(..., 0)` applies to the leaf term only,
  so empty and deleted pages still count in full — they hold nothing at any
  fillfactor.
- **The columns are renamed.** `wasted_space` and `wasted_pct` meant something
  else in the two earlier versions of this report, and an archived output should
  not be silently reinterpreted. The baseline is now in the name.

**Measured on both restarted servers, and nothing outside those two fields
moved.** Both clusters reproduced their filed output byte for byte before the
comparison started, so this is a comparison of two texts and not of two database
states:

| Check | 12.2 | 17.11 |
|---|---|---|
| Rows returned, either text | 27 | 28 |
| Output columns, either text | 14 | 14 |
| The 12 untouched fields, filed against fillfactor-relative | identical, 2,109 bytes | identical, 2,180 bytes |
| Columns exposed by the internal `final` stage | 28 against 28; `wasted_space` out, `wasted_vs_fillfactor` in | 28 against 28; same one-for-one swap |
| `EXCEPT` in both directions over the 27 shared columns | 0 rows, 214 indexes | 0 rows, 220 indexes |
| Rows byte-identical across the two servers, new text | 24 of 27 shared | 24 of 27 shared |
| Rows where the new column exceeds the old one | 0 of 214 | 0 of 220 |
| Rows where the new column is negative | 0 of 214 | 0 of 220 |

The same 24-of-27 cross-server agreement holds under the new text as under the
filed one, with the same three exceptions — the two duplicate-key indexes and
the non-deterministic churn fixture.

**What actually changed, on the 17.11 report.** No row read `0.0` under the
filed text; eleven of 28 do now, and every remaining row fell by roughly a tenth
of the file:

```text
 index_name  | index_size | avg_leaf_density | old wasted_pct | wasted_ff_pct | est_reclaimable_pct | notes
 i_del90     | 21 MB      |             9.27 |           89.9 |          79.9 |                89.7 |
 i_delhead   | 21 MB      |            89.94 |           72.9 |          69.9 |                69.9 | reclaim is mostly empty/deleted pages
 i_del50     | 21 MB      |            45.18 |           54.3 |          44.4 |                49.7 |
 i_ff100     | 19 MB      |            99.86 |            0.1 |           0.1 |                 0.1 | fillfactor 100
 i_fresh     | 21 MB      |            90.06 |            9.8 |           0.0 |                -0.1 |
 i_ff50      | 39 MB      |            49.85 |           49.7 |           0.0 |                -0.2 | fillfactor 50
 i_ff10      | 206 MB     |             9.62 |           89.6 |           0.0 |                -0.5 | fillfactor 10
 i_dup_ins   | 6368 kB    |            95.94 |            4.0 |           0.0 |                -6.7 | denser than a rebuild would leave it
```

Both servers produced this pattern; 12.2 has ten such rows out of 27, the one
difference being the 17-only `deduplicate_items = off` fixture.

The `fillfactor = 100` rows are the exception that proves the arithmetic: at
fillfactor 100 the target free space is zero, so
`target_density` is exactly 1 and the new column equals the old one to the byte.
That happened for 110 of 214 indexes on 12.2 and 110 of 220 on 17.11 — two
`fillfactor = 100` fixtures plus the 104 and 108 indexes with no leaf pages,
where both definitions reduce to the dead-page term.

**A rebuilt index must report zero, and it does.** After `REINDEX INDEX` over
every scored index, the new column was measured again on both servers:

| Post-`REINDEX` residual | 12.2 | 17.11 |
|---|---|---|
| Indexes scored | 97 | 96 |
| Exactly `0` bytes | 81 | 76 |
| At or below 0.1% | 87 | 85 |
| Worst residual, all indexes | 44.6%, 7,309 bytes | 44.6%, 7,309 bytes |
| Worst residual, indexes the report shows (≥ 1 MB) | 0.4%, 35,252 bytes | 0.4%, 35,252 bytes |

The 44.6% is the honest limit of the definition and it is the same fixture on
both servers: `c_one_idx`, a one-row index whose single leaf page is 0.29% dense.
A rebuild cannot make one tuple fill 89.95% of a page, so the column claims
7,309 wasted bytes that no operation will ever return. Every index in that state
has three leaf pages or fewer and sits far below the statement's 1 MB
`min_index_bytes` prefilter, which is why the report itself never shows one; the
worst residual among the rows it does print is 0.4%. The old baseline had the
same blind spot and read worse on the same index: 8,128 bytes and 49.6%.

**The two columns are related, not redundant.** For waste that sits inside live
leaf pages, the new column is a fixed fraction of the reclaim estimate, because
it counts payload bytes while `est_reclaimable` counts whole file pages including
each one's 24-byte header and 16-byte special area:

```text
wasted_vs_fillfactor / est_reclaimable  ->  (leaf_capacity - target_free) / block_size
                                        =   (8152 - 819) / 8192  =  0.8951   at 8192/90
```

Measured over the 15 indexes with more than a megabyte of estimated reclaim,
identically on both servers: `0.8868` to `0.8921` for the thirteen with no dead
pages, `1.0001` for `i_delhead`, whose waste is 1,918 whole dead pages, and
`0.8321` for `i_wide`, which mixes 44 dead pages with wide-tuple leaves. So the
ratio reads as a composition signal: near 0.89 the waste is inside pages, at 1.0
it is whole pages, and a rebuild is the only way to return either.

**At a low fillfactor the two columns diverge, and the new one is the smaller.**
Every fixture in the original suite that carries real waste has the default
fillfactor 90, so four more were built for this change — a table filled, indexed
at a stated fillfactor, then nine tenths of the rows deleted and the table
vacuumed. Both servers produced these four rows identically:

```text
 index_name     | ff  | leaf | dead | density | target | old wasted_pct | wasted_ff_pct | est_pct | actual_pct | ratio pred/meas | after
 i_ff100_del90  | 100 |  493 |    0 |   10.25 | 100.00 |           88.6 |          88.6 |    89.5 |       89.5 | 0.9951 / 0.9895 |   1.5
 i_ff50_del90   |  50 |  991 |    0 |    5.25 |  49.75 |           93.7 |          44.0 |    89.3 |       89.8 | 0.4951 / 0.4931 |   0.4
 i_ff10_del90   |  10 | 1579 |    0 |    1.23 |   9.57 |           97.8 |           8.3 |    87.1 |       89.9 | 0.0952 / 0.0948 |   0.0
 i_ff50_delhead |  50 |  100 |  894 |   49.36 |  49.75 |           94.7 |          89.7 |    89.7 |       89.8 | 0.4951 / 1.0004 |   0.4
```

The predicted ratio holds across the whole fillfactor range, to four decimal
places on both servers, and the consequence is blunt: **`i_ff10_del90` is 89.9%
reclaimable and reports 8.3% wasted.** That is the definition working, not
failing. At fillfactor 10 the index is *supposed* to be nine tenths free space,
so the bytes that a rebuild would not leave free are a small share of the file
even though the rebuild takes it from 1,579 leaf pages to 158. A reader who wants
"how much disk will `REINDEX` give back" must read `est_reclaimable_pct`, at any
fillfactor; `wasted_ff_pct` answers "how much of this file is space its own
fillfactor does not justify", and the lower the fillfactor the further apart
those two questions are. The old baseline hid the difference by reporting 97.8%
for the same index, which was neither answer. `i_ff50_delhead` shows the other
end: its waste is 894 whole dead pages, the ratio goes to `1.0004`, and the two
columns agree at 89.7%.

The last column is the post-`REINDEX` residual, and `i_ff100_del90`'s 1.5% is
worth naming: rebuilt, it holds 20,000 rows in 50 leaf pages at 98.42% density,
because the rightmost page of any build takes whatever is left over. Against a
100% target that partial page is 6,440 bytes, and on a 416 kB index that is 1.5%.
The effect is per-index, not per-byte, so it shrinks as the index grows.

These four fixtures also found the reclaim estimate's own worst under-estimate
for a vacuumed index anywhere on this page: `i_ff10_del90` at `87.1` against an
actual `89.9`, `−2.8` points, where every fixture in the original suite came
within `1.0`. The likely cause is in the same numbers. Every non-rightmost page
holds a high key in item 1
([nbtree.h#P_HIKEY](../../../../raw/postgres-17/src/include/access/nbtree.h#L348-L369)),
which a rebuild into 158 pages writes 158 times and the 1,579-page original
carries 1,579 times, and `avg_leaf_density` counts that per-page overhead as
occupied space because `PageGetFreeSpace` reports only what is unallocated
([pgstatindex.c#leaf-accounting](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L324),
[bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)).
At a density of 1.23% that overhead is most of the measured payload, so the model
believes there is more to re-pack than there is. The error is in the safe
direction; the per-page accounting that would confirm the size of each term was
not done, and it is recorded under [Open Questions](#open-questions).

**Cost is unchanged.** The plan shape is identical on both servers (4 `CTE Scan`
nodes; 68 plan lines on 17.11, 60 on 12.2) and so is the page count: the two
texts read exactly 108,021 buffers on 17.11 and 108,327 on 12.2, differing only
in the hit/read split. `EXPLAIN (ANALYZE, BUFFERS)` execution was 131.1 ms
against 120.9 ms on 17.11 and 123.6 ms against 117.5 ms on 12.2, and six
interleaved end-to-end runs of each text spanned 124.0-127.7 ms against
122.1-125.6 ms on 17.11 and 120.6-129.2 ms against 121.2-123.8 ms on 12.2 — the
fillfactor-relative text is at or just inside the filed text's range, which for
one multiplication over an already-materialized CTE is noise.

**What this column does not become.** It is still a description of the file, not
a prescription for it. `pgstatindex` cannot see entries that are deleted but not
yet vacuumed, so `i_novac` now reads `0.0` where it used to read 9.8, on an index
a rebuild would shrink by 90%; the 9.8 was never a signal, but zero is a flatter
way to be wrong. And fillfactor is a build-time and rightmost-split target, not
a property a growing index holds: an ordinary leaf split divides 50:50, and a
page full of one value splits at `BTREE_SINGLEVAL_FILLFACTOR`, 96%
([nbtsplitloc.c#fillfactormult](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L279-L335),
[nbtsplitloc.c#SPLIT_SINGLE_VALUE](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416),
[nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)).
The column measures the file against what a rebuild would produce, which is the
only fillfactor-relative question `REINDEX` can answer.

### What one pgstatindex call actually measures

`pgstatindex` opens the index with `AccessShareLock`, reads the metapage, then
walks every remaining block under a shared buffer lock with a `BAS_BULKREAD`
strategy, and buckets each page into deleted, half-dead (`empty_pages`), leaf, or
internal
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L331),
[nbtree.h#P_ISLEAF](../../../../raw/postgres-17/src/include/access/nbtree.h#L212-L227)).
Three details drive the model:

- **`index_size` is the whole file**, computed as `(1 + leaf + internal + deleted
  + empty) * BLCKSZ`
  ([pgstatindex.c#index_size](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L357)).
  Measured: it equalled `pg_relation_size()` for 218 of 218 candidate indexes on
  17.11 and 212 of 212 on 12.2.
- **`avg_leaf_density` covers live leaves only.** It is
  `100 - free_space / max_avail * 100`, where both sums are accumulated only in
  the leaf branch of the loop; deleted and half-dead pages contribute to neither
  ([pgstatindex.c#leaf-accounting](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L324),
  [pgstatindex.c#avg_leaf_density](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367)).
  That is why the statement adds `dead_pages * bs` separately.
- **`leaf_fragmentation` counts pages, not bytes**: leaves whose `btpo_next` is a
  lower block number, over `leaf_pages`
  ([pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323),
  [pgstatindex.c#leaf_fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L368-L372)).

Nothing here reads `pg_class.reltuples`, `pg_statistic` or the cumulative
statistics views, so the report does not care whether the table was ever analyzed.
That is the main thing it buys over a catalog-only estimator, whose accuracy rests
on statistics freshness and on the `reltuples = -1` sentinel; both hazards are
measured in
[Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17](btree-index-bloat-core-sql-only.md).

The reason `REINDEX` is the remedy and `VACUUM` is not: the nbtree code contains
no call to `RelationTruncate` or `smgrtruncate` (0 matches under
`src/backend/access/nbtree/`). A vacuum puts deleted pages into the free space map
for reuse instead
([nbtree.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1170),
[nbtree.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059)).
Measured on both servers: after deleting 70% of a table and vacuuming, its index
still occupied 22,487,040 bytes with 1,918 dead pages, and `REINDEX` took it to
6,758,400.

### Why every candidate filter is there

Every filter in `cand` exists because `pgstatindex` raises on that shape, and one
raised call aborts the whole statement. Each was reproduced on both servers:

| Filter | What happens without it |
|---|---|
| `a.amname = 'btree'` | `ERROR: relation "s_hash" is not a btree index` — reproduced for hash, GIN, GiST, SP-GiST and BRIN ([pgstatindex.c#IS_BTREE](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228)) |
| `c.relkind = 'i'` | A partitioned index is `'I'`, so `IS_INDEX` is false and the call fails the same test a non-B-tree index fails: `ERROR: relation "i_part" is not a btree index`, reproduced on both servers. Upstream's expected output covers the neighbours rather than this exact call — `pgstattuple` on a partitioned index, `pgstatindex` on a partitioned table ([pg_class.h#RELKIND_PARTITIONED_INDEX](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L165-L173), [pgstatindex.c:70](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70), [pgstattuple.out#partitioned](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L155-L171)) |
| `indisvalid AND indisready AND indislive` | On 17.11, `ERROR: index "i_invalid" is not valid`. On 12.2 the same index returns a row. See the next section ([pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250), [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)) |
| `NOT pg_is_other_temp_schema(...)` | `ERROR: cannot access temporary tables of other sessions`. Measured: one other session holding a 4.4 MB temp index moved the candidate count from 27 to 28 on 17.11 and 26 to 27 on 12.2, and the statement still returned every row with that session open ([pgstatindex.c#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L230-L238), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669)) |
| `relpersistence <> 'u' OR NOT pg_is_in_recovery()` | Untested belt and braces. The planner refuses unlogged relations during recovery ([plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)) but a function call never goes through that path, and `pgstatindex_impl` has no such guard, so on a standby it would read whatever is in the file. On a primary, unlogged indexes are read normally and one is in the fixture output |
| `pg_relation_size(c.oid) >= min_index_bytes` | Not a correctness filter, a cost filter. It is the only place a non-`pgstatindex` measurement appears, and it is a `stat()`-level answer, not a page read. It does open the relation, with `try_relation_open`, which returns NULL rather than raising when the index has gone — so it doubles as a shield against a concurrent drop, measured in [Locking, timeouts, and the concurrent-drop race](#locking-timeouts-and-the-concurrent-drop-race) ([dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371)). Replace it with `c.relpages` for a strictly-catalog prefilter, at the price of trusting a stale number and losing that shield |

The index is passed by OID, not by name (`pgstatindex(c.idx_oid::regclass)`). That
is deliberate: see [Privileges](#privileges).

`cand` is `AS MATERIALIZED` — v12 syntax, and the earliest major this statement
claims — so the candidate list is fixed before the first page is read.

### The one behavioural difference between 12 and 17

**An invalid index is the only shape where the two pinned servers disagree**,
and it is the reason the `indisvalid` filter is not optional. Read it as a
difference between 12.2 and 17.11, not between the majors: the commit that
added the check says it was back-patched, so a late-enough 12 minor refuses an
invalid index too. See below.

On the 17.11 server, `pgstatindex` on an index with `indisvalid = false` fails:

```text
ERROR:  index "i_invalid" is not valid
```

On the 12.2 server, the identical call on the identical fixture returns a row:

```text
 version | tree_level | index_size | root_block_no | internal_pages | leaf_pages | empty_pages | deleted_pages | avg_leaf_density | leaf_fragmentation
       4 |          2 |    6758400 |           290 |              4 |        820 |           0 |             0 |            90.05 |                  0
```

The check is `13503eb5905`, "Diagnose !indisvalid in more SQL functions"
(2023-10-30), whose earliest containing release tag in this checkout is
`REL_17_0`; it also states its own reasoning, that a `!indisready` index could
lead to `ERRCODE_DATA_CORRUPTED` and that an `indisready && !indisvalid` index
gives confusing results because its size can be too low for a valid index of the
table
([pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250)).

**The measured difference is between these two minors, not between the two
majors.** The same commit message, in this checkout's own history, ends
"Back-patch to v11 (all supported versions)", and 12 was supported in October
2023. So the pinned 12.2 of February 2020 predates the check and returns a row,
while a 12 minor from the back-patch onwards raises the same error 17.11 does.
Which minor that is cannot be read from this page's evidence base; see
[Open Questions](#open-questions).

Filtering the index out is right on both majors: on 17 it prevents an abort, and
on 12 it prevents a half-built `CREATE INDEX CONCURRENTLY` leftover from being
reported as a healthy index. Failed concurrent builds are exactly how such indexes
appear; see
[How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md).

Everything else matched. Both servers produced the same message text for
non-B-tree access methods, partitioned indexes, tables, views, sequences,
another session's temp index, and a stale OID
(`ERROR: could not open relation with OID 2147483647`).

### The model, from avg_leaf_density to a rebuilt size

The estimate is a ratio, so most constants cancel:

```text
payload_leaf_pages  = leaf_pages * avg_leaf_density / 100
est_leaf_pages      = ceil(payload_leaf_pages / target_density)
est_internal_pages  = round(internal_pages * est_leaf_pages / leaf_pages)
est_rebuilt_bytes   = (1 + est_leaf_pages + est_internal_pages) * block_size
est_reclaimable     = index_size - est_rebuilt_bytes
```

`target_density` is not the fillfactor. It is what the build code actually leaves
behind. A sorted build closes a leaf page when the remaining free space drops
below `BLCKSZ * (100 - fillfactor) / 100`
([nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671),
[nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145),
[nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L860)),
so the density a rebuild reaches is `(leaf_capacity - target_free) / leaf_capacity`
and not `fillfactor / 100`. At `block_size` 8192 and the default fillfactor of 90
([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202),
[reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194))
that is `(8152 - 819) / 8152 = 89.95%`, marginally below the fillfactor, which
makes the estimate conservative by construction.

Measured fresh-build densities, byte-identical on both servers over a 200,000-row
`int` index, against what the formula predicts:

| fillfactor | modelled target density | measured `avg_leaf_density` | reported `est_reclaimable_pct` |
|---|---|---|---|
| 100 | 100.00 | 99.82 | 0.0 |
| 90 | 89.95 | 90.00 | −0.2 |
| 50 | 49.75 | 49.81 | −0.2 |
| 10 | 9.57 | 9.62 | −0.5 |

`est_internal_pages` uses `round`, not `ceil`, because it is a proportional
estimate rather than a capacity bound; internal pages were under 0.5% of every
fixture. Non-leaf pages are built to a fixed 70%
([nbtree.h#BTREE_NONLEAF_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)),
which the model does not use.

### Why the two page-layout constants are safe

`leaf_capacity` is `block_size - 24 - 16`, which reproduces `pgstatindex`'s own
`max_avail`, computed as `pd_special - SizeOfPageHeaderData`
([pgstatindex.c#max_avail](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L314)).
The 24 is `SizeOfPageHeaderData`
([bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214));
the 16 is `MAXALIGN(sizeof(BTPageOpaqueData))`, whose five fields are two
`BlockNumber`, one `uint32` and two `uint16` — `btpo_flags`, and
`btpo_cycleid`, which is declared as a `BTCycleId`
([nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71),
[nbtree.h:29](../../../../raw/postgres-17/src/include/access/nbtree.h#L29)).
The same subtraction appears in core as
`BLCKSZ - SizeOfPageHeaderData - sizeof(BTPageOpaqueData)`
([nbtree.h#MaxTIDsPerBTreePage](../../../../raw/postgres-17/src/include/access/nbtree.h#L185-L187)).

Rather than trust the arithmetic, both servers were asked to imply the constant
from pages with known contents. One `int4` key on a root leaf page leaves
`8160 - 28 - 4 = 8128` free, and an empty leaf page leaves `8176 - 24 - 4 = 8148`;
the 4 is the line pointer that `PageGetFreeSpace` deducts
([bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)).
Both servers reported `0.29` and `0.05`, implying `max_avail` of 8151.6 and
8152.1 against the statement's 8152.

The statement reads `block_size` from the server, so a cluster built at another
`BLCKSZ` scales; the 24 and the 16 do not depend on `BLCKSZ`. Nothing was run at a
block size other than 8192.

### wasted_vs_fillfactor is not est_reclaimable

```text
target_density       = (leaf_capacity - block_size * (100 - fillfactor) / 100)
                       / leaf_capacity
wasted_vs_fillfactor = max(leaf_pages * leaf_capacity
                             * (target_density - avg_leaf_density/100), 0)
                     + (empty_pages + deleted_pages) * block_size
```

That is the file measured against the density a rebuild at this index's own
fillfactor reaches, which is the only fillfactor-relative question `REINDEX` can
answer. It counts bytes that hold nothing **and that a rebuild would not leave
empty**, so a correctly packed index reports zero at any fillfactor, and the
`fillfactor = 10` fixture that used to report 89.6% now reports 0.0.

`est_reclaimable` answers a different question with the same target: not "how
many bytes in this file are surplus" but "how large would the file be after a
rebuild". The gap between them is per-page overhead. `wasted_vs_fillfactor`
counts payload bytes inside leaf pages; `est_reclaimable` counts whole 8 kB file
pages, each carrying a 24-byte header and a 16-byte special area that the payload
figure excludes. For in-page waste that makes the first a fixed fraction
`(leaf_capacity - target_free) / block_size` of the second — 0.8951 at 8192 and
fillfactor 90, measured at 0.8868 to 0.8921 — and for whole dead pages the two
coincide, because a dead page wastes all 8,192 of its bytes and gives all 8,192
back. Both are in the output because a reader who wants to know how much of the
file is surplus and a reader who wants to know how much disk a rebuild returns
are asking different things.

An earlier version of this report measured the same free bytes against 100%
packing under the names `wasted_space` and `wasted_pct`. That number was the
literal reading of "wasted space" but it was mostly a restatement of the
fillfactor: a healthy default index reported about 10%. See
[Follow-up: wasted space measured against the fillfactor](#follow-up-wasted-space-measured-against-the-fillfactor).

### NaN is the trap

An index with no leaf pages returns `NaN` for both `avg_leaf_density` and
`leaf_fragmentation`, because the C code guards on `max_avail > 0` and
`leaf_pages > 0`
([pgstatindex.c#NaN](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
Upstream's own expected output records this for an empty index
([pgstattuple.out#empty-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)).

In PostgreSQL, `NaN` compares greater than every number. Measured on both servers:

```text
 is_nan | nan_over_threshold | numeric_nan_over_threshold
 t      | t                  | t
```

So a naive `WHERE bloat_pct > 20` reports every empty index as maximally bloated.
The statement converts density to `0` behind a `leaf_pages > 0` test before any
arithmetic, prints `NULL` instead of `NaN`, and tags the row `no leaf pages`. In
the scoring run 62 of 94 indexes on 12.2 and 59 of 93 on 17.11 had `NaN` density —
mostly empty catalog TOAST indexes — and every one of them reported exactly `0.0`
estimated reclaim against `0.0` actual.

### Accuracy against REINDEX INDEX

Ground truth is `pg_relation_size` before and after `REINDEX INDEX`, run over
every scored index on each server. The estimator was scored from a view generated
mechanically from this page's statement text, with only two edits, both printed by
the generator: the two `SET` lines dropped, and `min_index_bytes` set to 0 so
sub-megabyte fixtures are scored too.

| Index | 12.2 est / actual | 17.11 est / actual | density | dead pages |
|---|---|---|---|---|
| `i_del90` 90% deleted, vacuumed | 89.7 / 89.9 | 89.7 / 89.9 | 9.27 | 0 |
| `i_partial` partial, 90% of matches deleted | 89.6 / 89.8 | 89.6 / 89.8 | 9.27 | 0 |
| `i_uniq` unique, 7 of 8 deleted | 87.2 / 87.4 | 87.2 / 87.4 | 11.52 | 0 |
| `t_part_1_id_idx` partition leaf index | 85.3 / 85.6 | 85.3 / 85.6 | 13.11 | 0 |
| `t_part_2_id_idx` partition leaf index | 85.3 / 85.6 | 85.3 / 85.6 | 13.11 | 0 |
| `i_unlogged` unlogged table | 83.0 / 83.3 | 83.0 / 83.3 | 15.25 | 0 |
| `i_expr` expression key | 79.4 / 79.9 | 79.4 / 79.9 | 18.55 | 0 |
| `i_text_del` `text` key | 79.4 / 79.9 | 79.4 / 79.9 | 18.55 | 0 |
| `i_incl` `INCLUDE` column | 74.8 / 75.0 | 74.8 / 75.0 | 22.62 | 0 |
| `i_wide` 400-byte keys | 69.9 / 75.0 | 69.9 / 75.0 | 27.23 | 44 |
| `i_delhead` contiguous head deleted | 69.9 / 69.9 | 69.9 / 69.9 | 89.94 | 1918 |
| `i_multi` three-column key | 66.4 / 66.6 | 66.4 / 66.6 | 30.22 | 0 |
| `t_toast_pkey` TOAST table primary key | 64.9 / 63.2 | 64.9 / 63.2 | 30.02 | 0 |
| `i_del50` half deleted | 49.7 / 49.9 | 49.7 / 49.9 | 45.18 | 0 |
| `i_churn` update churn on the key | 46.2 / 46.4 | 46.2 / 46.9 | 48.36 | 0 |
| `i_frag` reverse-order inserts | 44.0 / 44.3 | 44.0 / 44.3 | 50.34 | 0 |
| `i_ff100` fresh, fillfactor 100 | 0.1 / 0.0 | 0.1 / 0.0 | 99.86 | 0 |
| `i_text` fresh `text` | 0.0 / 0.0 | 0.0 / 0.0 | 89.98 | 0 |
| `i_fresh` fresh, default fillfactor | −0.1 / 0.0 | −0.1 / 0.0 | 90.06 | 0 |
| `i_ff50` fresh, fillfactor 50 | −0.2 / 0.0 | −0.2 / 0.0 | 49.85 | 0 |
| `i_ff10` fresh, fillfactor 10 | −0.5 / 0.0 | −0.5 / 0.0 | 9.62 | 0 |
| `t_empty_pkey` empty table | 0.0 / 0.0 | 0.0 / 0.0 | NaN | 0 |
| `i_dup` 10 distinct values, built by `CREATE INDEX` | −0.3 / 0.0 | 0.0 / 0.0 | 89.91 | 0 |
| `i_dup_ins` same data, built by inserts | −6.6 / −6.4 | −6.7 / −6.8 | 95.94 | 0 |
| `i_novac` 90% deleted, **not** vacuumed | −0.1 / 89.9 | −0.1 / 89.9 | 90.06 | 0 |
| `i_dedup_off` duplicates, `deduplicate_items = off` | not constructible | −0.3 / 69.1 | 90.16 | 0 |

Totals over every scored index, including the 60-odd empty catalog TOAST indexes
not listed above: **94 indexes on 12.2, 91 within 1.0 point and 92 within 2.0**;
**93 indexes on 17.11, 89 within 1.0 point and 90 within 2.0**. The largest
over-estimate on both servers is the same `+1.7` points, on a 456 kB TOAST
primary key; every other row over-estimates by at most `+0.1`. Over-estimates are
the dangerous direction, because they promise space a rebuild will not return,
and on this suite they are bounded by two points on indexes small enough that the
per-page rounding dominates.

### The three shapes it gets wrong

All three are under-estimates: the index is more reclaimable than the report says.

1. **Entries deleted but not yet vacuumed** (`i_novac`, −90.0 points on both
   servers). Deleting 90% of the rows changes nothing on the index pages until a
   vacuum removes the entries, so the density is a healthy 90.06 while a rebuild
   takes the file from 22,487,040 to 2,260,992 bytes. `pgstatindex` counts bytes,
   not liveness, and cannot see this. A vacuum first, then this report, is the
   only fix — which is also the right operational order.
2. **Deduplication that the current index is not using** (`i_dedup_off`, −69.4
   points, 17.11 only). See the next section.
3. **Wide keys** (`i_wide`, −5.1 points on both). With 400-byte tuples a page is
   closed when the free space falls below 819 bytes, so on average it ends up
   several hundred bytes fuller than the model's bound; the rebuilt index measured
   92.77% density against the modelled 89.95%. The error is bounded by the tuple
   size over the leaf capacity, and it is always in the safe direction.

### Deduplication changes the input, not the arithmetic

B-tree deduplication arrived after 12, and it is the largest source of
same-statement, different-numbers between the two servers. Two fixtures with a
key of ten distinct values over a million rows:

| Fixture | 12.2 size | 17.11 size | 12.2 est / actual | 17.11 est / actual |
|---|---|---|---|---|
| `i_dup`, built by `CREATE INDEX` | 21 MB | 6800 kB | −0.3 / 0.0 | 0.0 / 0.0 |
| `i_dup_ins`, built empty then filled | 20 MB | 6368 kB | −6.6 / −6.4 | −6.7 / −6.8 |

The estimator is right on both servers. What changed is the index: on 17.11 the
duplicates are already posting lists, so the payload `pgstatindex` measures is
already the compressed payload, and a rebuild reproduces it.

The failure case is an index whose pages are **not** deduplicated but whose
rebuild would be. `i_dedup_off` builds one deliberately with
`deduplicate_items = off`
([nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1151)),
then turns the option back on: the report reads 90.16% density and `−0.3%`, and
`REINDEX` takes the file from 22,519,808 to 6,963,200 bytes, 69.1% reclaimed. The
reloption does not exist on 12.2, so the fixture is unconstructible there.

The realistic version of this is a cluster upgraded from 12: those indexes cannot
deduplicate until they are rebuilt, and no `pgstatindex` column distinguishes them
(they report `version` 4 like everything else). That case, and the core-SQL test
for it, is
[Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade From PostgreSQL 12 to 17](btree-deduplication-after-pg-upgrade.md).

### What it costs to run

This report reads every page of every index it reports on. That is the price of
not depending on statistics. `EXPLAIN (ANALYZE, BUFFERS)` attributes it precisely,
because the reads go through the buffer manager:

| Server | Indexes | Buffers at the `pgstatindex` function scan | Execution |
|---|---|---|---|
| 12.2 | 27 | 6,693 hit + 99,658 read (~831 MB) | 124.2 ms |
| 17.11 | 28 | 5,616 hit + 99,797 read (~824 MB) | 132.6 ms |

End to end through psql, warm: 153.1 ms then 135.2 ms on 12.2, 147.5 ms then
142.2 ms on 17.11. These are millisecond numbers only because the fixture database
is under a gigabyte and the pages were in the OS cache; the cost scales with the
bytes of index, not the number of indexes.

Two mitigations are built in. `min_index_bytes` skips small indexes using a
`stat()`-level size, before any page is read. And the scan uses a `BAS_BULKREAD`
strategy, a 256 kB ring
([freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L540-L574)),
so a large report does not evict the buffer pool; the `written=832`/`written=874`
counters in the same plans are the ring writing back its own dirty buffers.

### Privileges

`pgstattuple` 1.5 revokes `EXECUTE` from `PUBLIC` and grants it to
`pg_stat_scan_tables`
([pgstattuple--1.4--1.5.sql#grants](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstattuple.sgml#access](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L15-L24)).
The `_v1_5` entry points carry no `superuser()` check; only the pre-1.5 symbols do
([pgstatindex.c#pgstatindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L162-L180),
[pgstatindex.c#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L144-L160)).

Measured identically on both servers with a login role `mon` whose only privilege
is membership in `pg_stat_scan_tables`:

- `mon` runs the whole statement and gets every row, including indexes on tables
  in a schema it has no `USAGE` on.
- `pgstatindex('bl.i_del90')` by **name** fails for `mon` with
  `ERROR: permission denied for schema bl`, because resolving the name needs
  schema `USAGE`. This is why the statement passes `c.idx_oid::regclass`: the OID
  overload never resolves a name.
- A role without the membership gets
  `ERROR: permission denied for function pgstatindex`.

So the grant needed is exactly `GRANT pg_stat_scan_tables TO <role>`, and it
carries page-level visibility into every table in the database. No `SELECT`
privilege on the indexed tables is involved at any point.

### Locking, timeouts, and the concurrent-drop race

`pgstatindex` takes `AccessShareLock` on the index
([pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213)),
so it waits behind anything holding `AccessExclusiveLock`. With another session
sitting on an uncommitted `DROP INDEX`, `lock_timeout = '2s'` cancelled the call
at 2000.9 ms on 17.11 and 2001.0 ms on 12.2:

```text
ERROR:  canceling statement due to lock timeout
```

A drop that commits while the statement is running has two outcomes, and which
one you get depends on where the statement is when the drop commits.

**Where `cand` is still running, the row is dropped and the report survives.**
`cand` sizes each candidate with `pg_relation_size`, which opens the relation
with `try_relation_open` and returns NULL when it has gone, on purpose: the
comment above it says returning NULL "for already-dropped tables" beats
throwing "and abort the whole query"
([dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371)).
`NULL >= min_index_bytes` is NULL, not true, so the candidate silently
disappears. Measured on both servers, with a second session holding
`BEGIN; DROP INDEX bl.i_uniq;` for three seconds: the report waited for the
lock, then returned normally, one row shorter — 24 rows against 25 on 17.11 and
23 against 24 on 12.2, with no error.

**Where the drop commits after `cand` has sized that index and before
`pgstatindex` opens it, the whole statement aborts.** `pgstatindexbyid_v1_5`
uses `relation_open`, which raises rather than returning NULL
([pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213)):

```text
ERROR:  could not open relation with OID 16897
```

That window is the gap between the size check and the per-row call, and on this
fixture suite it is small: the whole report runs in about 80 ms warm, `cand`
finishes inside the first 5 ms of it, and the target index is read last. The
2026-09-10 re-run swept the drop across ten delays from 5 ms to 120 ms on each
server, always against the index `cand` materializes last, and **never landed in
it**: below about 70 ms the drop was taken before `cand` finished and the row
vanished silently, and above it the report had already finished. The error above
is the one the original run recorded, and the source says it is reachable; this
page no longer claims a reproduction of it. See
[Open Questions](#open-questions).

So the operational advice is weaker than "unfixable", but it is the same advice:
one failed index still loses the whole report when the timing is unlucky. On a
database with heavy DDL, run the report against a candidate list that excludes
tables under migration, or drive `pgstatindex` from a loop that catches the
error per index and keeps going.

### Everything the two servers agreed on

Of the 27 rows the 12.2 report produced and the 28 from 17.11, **24 are identical
character for character across the two servers**, including `leaf_pages`,
`avg_leaf_density`, `wasted_vs_fillfactor`, `est_reclaimable` and `notes`. The
exceptions are the two duplicate-key indexes (deduplication) and `i_churn`, where
the churn fixture is not deterministic (2,551 leaves against 2,550, and 48.38%
against 48.36%). Both servers also produced the same error text for every rejected
shape, the same `+1.7` worst over-estimate, the same `−90.0` and `−5.1`
under-estimates, the same implied leaf capacity, and the same fresh-build
densities at four fillfactors. The count is 24 under the fillfactor-relative text
and was 24 under both earlier texts, with the same three exceptions each time.

### Re-measured from a published script

On 2026-09-10 this page was reviewed and the whole measurement was run again
from scratch, from the two scripts now filed under
[Measurement Script](#measurement-script). Both servers were rebuilt from the
two pins in an empty sandbox, the fixture suite was rebuilt from published DDL,
and the filed statement was extracted from this page and hash-checked before it
ran. The earlier runs used a fixture suite that was never published and whose
sandbox no longer exists, so this is a reconstruction, not a replay; see
[Open Questions](#open-questions).

**Environment.** Linux x86_64, `block_size` 8192, `max_data_alignment` 8,
`initdb --locale=C --encoding=UTF8`, isolated clusters on ports 55417 and
55412, `autovacuum = off`, `fsync = off`, `shared_buffers = 512MB`. 17.11 was
configured `--enable-debug --with-icu --with-readline --with-zlib` and passed
**All 225 core tests** plus `contrib/pgstattuple`; 12.2 the same without
`--with-icu`, **All 192** plus `contrib/pgstattuple`. The fixture database
came to 1828 MB with 39 indexes in schema `bl` on 17.11, and 1801 MB with 38 on
12.2 — the difference is the 17-only `deduplicate_items` fixture. A full leg
takes about 86 seconds from a built tree.

**The statement itself is unchanged and still runs on both.** The `sql` block
hashes to `9d2e3a2c73c8…`, 126 lines and 6,154 bytes, and executed **unmodified**
on 12.2, returning 28 rows against 29 on 17.11, 14 columns on each. The two
superseded texts recover from history at `f5b995d3c5d5…` (122 lines, 5,839
bytes) and `da4f4277b24e…` (125 lines, 6,002 bytes), which confirms all three
text sizes the follow-up sections quote, and a diff of them confirms both edit
tables: five edits for `alert_pct`, four for the fillfactor rebase, and nothing
else moved either time.

**What reproduced exactly, on both servers.**

| Claim | Filed | 2026-09-10 |
|---|---|---|
| `i_delhead` after VACUUM | 22,487,040 bytes, 1,918 dead pages | identical on both servers |
| `i_delhead` reading | 821 leaf, 89.94 density, 69.9 / 69.9 | identical |
| Implied `max_avail` from two known pages | 8151.6 and 8152.1, from 0.29 % and 0.05 % | identical on both servers |
| `index_size` equals `pg_relation_size` | 218 of 218, 212 of 212 | 201 of 201 and 195 of 195 |
| `NaN > 20` for `float8` and `numeric` | true, true | true, true |
| The invalid index on 12.2 | `4 \| 2 \| 6758400 \| 290 \| 4 \| 820 \| 0 \| 0 \| 90.05 \| 0` | the same row, digit for digit |
| The invalid index on 17.11 | `ERROR: index "i_invalid" is not valid` | same |
| Refusals: hash, GIN, GiST, SP-GiST, BRIN, partitioned index, table, view, sequence, stale OID | ten messages | ten identical messages, both servers |
| Another session's temp index | refused, report unaffected | `ERROR: cannot access temporary tables of other sessions`, candidates 201 against 202 unfiltered, report unaffected |
| `lock_timeout` cancels the wait | 2000.9 / 2001.0 ms | 2006.1 / 2005.5 ms, same message |
| Privileges: member by name, member by OID, non-member | three outcomes | three identical outcomes, both servers |
| Plan shape | 4 `CTE Scan` nodes | 4 on both |
| Worst post-`REINDEX` residual | 44.6 %, 7,309 bytes | 44.6 %, 7,309 bytes, on six one-leaf-page indexes including `c_one_idx` |
| Deduplication sizes, `i_dup` / `i_dup_ins` | 21 MB / 20 MB on 12.2, 6800 kB / 6368 kB on 17.11 | identical |

**The accuracy table, re-scored.** Every fixture, `est_reclaimable_pct` against
a measured `REINDEX INDEX`, both servers:

| Index | 12.2 est / actual | 17.11 est / actual | density | dead pages |
|---|---|---|---|---|
| `i_ff10_del90` | 87.1 / 90.0 | 87.1 / 90.0 | 1.23 | 0 |
| `i_del90` | 89.7 / 89.9 | 89.7 / 89.9 | 9.27 | 0 |
| `i_partial` | 89.7 / 89.9 | 89.7 / 89.9 | 9.27 | 0 |
| `i_novac` | −0.1 / 89.9 | −0.1 / 89.9 | 90.06 | 0 |
| `i_ff50_del90` | 89.3 / 89.8 | 89.3 / 89.8 | 5.25 | 0 |
| `i_ff50_delhead` | 89.7 / 89.8 | 89.7 / 89.8 | 49.36 | 894 |
| `i_ff100_del90` | 89.5 / 89.5 | 89.5 / 89.5 | 10.25 | 0 |
| `i_uniq` | 87.1 / 87.4 | 87.1 / 87.4 | 11.52 | 0 |
| `t_part_1_id_idx`, `t_part_2_id_idx` | 85.3 / 85.6 | 85.3 / 85.6 | 13.11 | 0 |
| `i_unlogged` | 83.0 / 83.2 | 83.0 / 83.2 | 15.26 | 0 |
| `i_text_del` | 79.3 / 80.0 | 79.3 / 80.0 | 18.55 | 0 |
| `i_expr` | 79.7 / 79.9 | 79.7 / 79.9 | 18.25 | 0 |
| `i_incl` | 74.7 / 74.9 | 74.7 / 74.9 | 22.74 | 0 |
| `i_wide` | 65.0 / 70.0 | 65.0 / 70.0 | 31.53 | 12 |
| `i_delhead` | 69.9 / 69.9 | 69.9 / 69.9 | 89.94 | 1918 |
| `i_dedup_off` | not constructible | −0.3 / 69.1 | 90.16 | 0 |
| `i_multi` | 66.4 / 66.6 | 66.4 / 66.6 | 30.17 | 0 |
| `t_toast_pkey` | 64.9 / 63.2 | 64.9 / 63.2 | 30.02 | 0 |
| `c_zero_idx` | 0.0 / 50.0 | 0.0 / 50.0 | 0.05 | 0 |
| `i_churn` | 49.9 / 50.0 | 49.9 / 50.0 | 90.06 | 2741 |
| `i_del50` | 49.7 / 49.9 | 49.7 / 49.9 | 45.18 | 0 |
| `i_frag` | 44.0 / 44.3 | 44.0 / 44.3 | 50.34 | 0 |
| `i_ff100` | 0.1 / 0.0 | 0.1 / 0.0 | 99.86 | 0 |
| `i_text`, `c_one_idx`, `t_empty_pkey` | 0.0 / 0.0 | 0.0 / 0.0 | 89.99, 0.29, NaN | 0 |
| `i_dup` | −0.3 / 0.0 | 0.0 / 0.0 | 89.91 | 0 |
| `i_fresh` | −0.1 / 0.0 | −0.1 / 0.0 | 90.06 | 0 |
| `i_ff50` | −0.2 / 0.0 | −0.2 / 0.0 | 49.85 | 0 |
| `i_ff10` | −0.5 / 0.0 | −0.5 / 0.0 | 9.62 | 0 |
| `i_dup_ins` | −6.6 / −6.4 | −6.7 / −6.8 | 95.94 | 0 |

Eighteen of these rows match the filed table to the tenth of a point, including
every one the filed page called out: the `+1.7` over-estimate lands on the same
456 kB `t_toast_pkey` at `+1.8`, `i_novac` misses by `−90.1` against a filed
`−90.0`, `i_dedup_off` by `−69.3` against `−69.4`, `i_wide` by `−5.0` against
`−5.1`, and `i_ff10_del90` reads `87.1` against an actual `90.0`, the `−2.8`
the fillfactor section reports.

**Four things the re-run changes.**

1. **The concurrent-drop abort did not reproduce**, and the reason is in the
   source. See
   [Locking, timeouts, and the concurrent-drop race](#locking-timeouts-and-the-concurrent-drop-race).
2. **A fourth shape the estimate gets wrong.** `c_zero_idx`, an index whose
   single leaf page holds nothing, reports `0.0` against an actual `50.0`: the
   rebuild takes it from two pages to one, and a model that rounds a nearly
   empty leaf up to one page plus a metapage cannot see that. It is the
   small-index rounding limit of
   [The three shapes it gets wrong](#the-three-shapes-it-gets-wrong), on the
   reclaim column rather than the waste column.
3. **The scored population has to be stated, because three different ones
   appear on this page.** This run pins it: 201 indexes on 17.11 and 195 on
   12.2, being every candidate the harness view returns with
   `min_index_bytes` at 0. Of those, 32 and 31 are fixtures in schema `bl`,
   and the rest are catalog and TOAST indexes; 89 and 85 have no leaf pages
   and score `0.0` against an actual `0.0`. Split that way, the fixtures are
   26 of 32 and 26 of 31 within one point, 27 of each within two, worst
   over-estimate `+1.8` and worst under-estimate `−90.1` on both.
4. **Scoring catalog indexes measures the harness, not the estimator.** The
   scoring pass issues one `REINDEX INDEX` per index, and every one of those
   writes to `pg_class`, so a catalog index can grow between the snapshot and
   its own rebuild. On 12.2 that produced the run's two largest over-estimates,
   `+20.0` on a 40 kB `pg_class_oid_index` and `+14.3` on
   `pg_class_relname_nsp_index`; on 17.11 no catalog index over-estimated at
   all. Any "largest over-estimate" figure that includes catalogs is partly a
   statement about the measurement order.

**Cross-server agreement.** Of the 28 rows both reports produced, **25 are
identical character for character**. The three that differ are the two
duplicate-key indexes, which is deduplication, and one TOAST index whose name
embeds an OID. `i_churn` now agrees too, because the published churn fixture
updates every row rather than half of them and is deterministic.

**Cost.** 68 plan lines on 17.11 and 65 on 12.2, 4 `CTE Scan` nodes on each,
`EXPLAIN (ANALYZE, BUFFERS)` execution of 140.5 ms and 135.6 ms, the
`pgstatindex` function scan reading 13,216 hit + 98,197 read on 17.11 and
comparable on 12.2. Six interleaved end-to-end runs of the filed text and the
superseded text spanned 140.9–153.8 ms against 143.9–151.7 ms on 17.11 and
135.0–150.5 ms against 133.5–142.1 ms on 12.2 — the same overlapping ranges the
follow-up sections report.

### How this was measured

The original run, on 2026-09-08 and 2026-09-09, used two isolated servers on
one host, each initialised for it:

- **12.2**, `server_version_num` 120002, built from this repo's v12 pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`, with `contrib/pgstattuple` compiled
  from the same checkout through PGXS into a copy of the source tree, never into
  `raw/`.
- **17.11**, `server_version_num` 170011, from the `--with-icu --enable-debug`
  install of this page's pin `786db8dcf168bd9df8f55047337525ac19118b1c`.

Both at `block_size` 8192, `autovacuum = off`, `fsync = off`, one shared fixture
script of 24 index shapes plus one 17-only deduplication fixture, `REINDEX INDEX`
as ground truth, and `pgstatindex` as the only measurement tool — no
`pageinspect`, no `pgstattuple()`, no `amcheck`. That fixture script was never
published and its sandbox no longer exists; the reconstruction that replaces it
is in [Measurement Script](#measurement-script), and the difference between the
two is under [Open Questions](#open-questions). Nothing is retained: the
scripts create `.wiki-runtime/tmp/pgsi/` and their `clean` stages delete it.

The follow-up that removed `alert_pct` restarted those same two clusters and
re-ran both fixture scripts first, because the scoring pass that produced the
accuracy tables ends by rebuilding every index it scores, which leaves the
database in a rebuilt state that no longer reproduces the report. After the
rebuild the filed text returned its filed rows again on both servers, which is
what makes the before-and-after comparison a comparison of the two texts rather
than of two database states. The amended text was derived from the filed one by
a script that asserts each of the five edits appears exactly once and prints it,
and the page's SQL block was then verified byte-identical to the executed file
(5,839 bytes, 122 lines).

The follow-up that rebased wasted space on the fillfactor restarted the same two
clusters again and needed no fixture rebuild: both servers returned their filed
output byte for byte on the first run (2,448 bytes over 27 rows on 12.2, 2,531
over 28 on 17.11), because the previous follow-up ended with the fixtures rebuilt
rather than reindexed. The fillfactor-relative text was derived from the filed
one by the same kind of script — four edits, each asserted to appear exactly once
and printed — and the page's SQL block was verified byte-identical to the
executed file (6,154 bytes, 126 lines). Presentation identity is measured by
cutting the two changed fields out of both outputs; internal identity by
generating one view per text over the `final` stage with `min_index_bytes` set to
0, so all 214 and 220 indexes in each database are compared and not only those
over a megabyte. The post-`REINDEX` residual pass is destructive and ran last, in
the order the earlier passes need: report, comparison, timing, then rebuild.

## Measurement Script

Two scripts produce every number this page takes from a running server, one per
version leg: `bloat_pgstatindex_v17.sh` for 17.11 and `bloat_pgstatindex_v12.sh`
for 12.2. Both are filed in full below, in Bash and SQL only, and both ran end
to end on 2026-09-10 on Linux x86_64 from an empty sandbox. What that run
measured is [Re-measured from a published script](#re-measured-from-a-published-script).

The sections written before 2026-09-10 report numbers from a harness that was
never published and whose sandbox has since been deleted. These scripts replace
it and cover the same families; where a figure moved, the re-run section says
so, and the unpublished fixture recipe is [an open question](#open-questions).

### How to use the two leg scripts

| Item | The 17 leg, `bloat_pgstatindex_v17.sh` | The 12 leg, `bloat_pgstatindex_v12.sh` |
|---|---|---|
| Purpose | builds 17.11 from this page's pin, runs its regression suites, builds the fixture suite, executes this page's `sql` block exactly as filed, and measures every family the page reports: refusals, `NaN`, the implied leaf capacity, fresh-build densities, privileges, the lock and drop races, statement cost, and the scoring pass against a measured `REINDEX INDEX` | the same on 12.2 from the v12 pin, and it answers the one question the 17 leg cannot: whether the exact filed text executes unmodified on the oldest major this page claims. The one fixture it cannot build is the `deduplicate_items` index, and it records that |
| Invocation | `bash bloat_pgstatindex_v17.sh [stage ...]`, run from the repository root | `bash bloat_pgstatindex_v12.sh [stage ...]`, run from the repository root |
| Stages | 13 stages plus `stop` and `clean`; see [the stages](#the-stages-both-legs). With no argument every stage runs, in the table's order | the same 13, with the same meanings |
| Environment | 8 variables, all with defaults; see [what the scripts read](#what-the-scripts-read-from-the-environment) | the same 8, two of them named for this leg |
| Prerequisites | a C toolchain, ICU, readline and zlib headers, `git`, `sha256sum`, and the pinned checkout at `raw/postgres-17`; see [Prerequisites](#prerequisites) | the same without ICU: this leg does not pass `--with-icu`, because no fixture here needs a collation provider and ICU is opt-in on that major |
| Output | under `$SANDBOX/out`; **read `summary`'s output or `exact17.txt` first**, then `scores17.txt` for the accuracy table and `residual17.txt` for the post-rebuild residual. Build and check logs are copied there so they survive `clean` | the same directory, with `12` in every name: `exact12.txt`, `scores12.txt`, `residual12.txt` |
| Runtime | **86 seconds** for `fixtures report facts cost priv score residual race summary` from a built tree, measured. A full run adds the build and `make check`, which dominate it and were not separately timed on the recorded host | **86 seconds** for the same nine stages from a built tree, measured; the same caveat on a full run |
| Cleanup | `bash bloat_pgstatindex_v17.sh clean` stops the server with `pg_ctl -m fast -w stop`, confirms the teardown — no `postmaster.pid`, no postgres process on the data directory, an empty socket directory — and only then deletes `$SANDBOX`, after checking it is inside `$WIKI_ROOT/.wiki-runtime/tmp/`. `stop` does the first two and keeps everything. **`out/` is inside `$SANDBOX`, so copy it out first** | `clean` stops the 12 server the same way and deletes only this leg's `build12`, `install12`, `data12`, `sock12` and `sql12`, because the 17 leg owns the shared `out/`. Run the 17 leg's `clean` last to remove the sandbox |

Save the two fenced blocks below under those names and run them from the
repository root: `WIKI_ROOT` defaults to `$PWD`, and everything else — this
page, both pinned checkouts and the sandbox — is resolved beneath it.

```sh
bash bloat_pgstatindex_v17.sh                     # every stage, in order
bash bloat_pgstatindex_v17.sh score residual      # selected stages
bash bloat_pgstatindex_v12.sh report              # just the 12.2 execution result
bash bloat_pgstatindex_v17.sh clean               # stop and delete the sandbox
```

#### The stages, both legs

Every stage is idempotent and re-runnable on its own once the stages it needs
have run. The default order is the order of this table, and that order matters
in one place: `score` rebuilds every index in the database, so `report`,
`facts`, `cost` and `priv` have to precede it, and `residual` and `race` follow
it.

| Stage | What it does | Needs first |
|---|---|---|
| `build` | configures the pinned checkout out of tree under `$SANDBOX/build17`, installs into `$SANDBOX/install17`, then builds and installs `contrib/pgstattuple`; skips everything when the binary already exists. Copies `configure.log`, `make.log` and `install.log` into `out/` on the failure path too, because `clean` deletes the build tree | nothing |
| `check` | `make check` plus the `pgstattuple` check, one result line each into `out/checks17.txt`, then copies every `check_*.log` and any `regression.diffs` into `out/` | `build` |
| `cluster` | `initdb --locale=C --encoding=UTF8`, writes the settings below into `postgresql.conf`, starts on `PORT`, creates the database and `CREATE EXTENSION pgstattuple`, and records `uname -sm`, the version, `block_size` and `max_data_alignment` into `out/platform17.txt` | `build` |
| `texts` | extracts `sql` block 1 from this page, checks its SHA-256 against `BASE_SQL`, generates the harness view from it with exactly two edits, both printed, and recovers the superseded text from `OLD_REV` and checks it against `BASE_PREV` | `cluster` |
| `fixtures` | drops and rebuilds schema `bl`: 24 index shapes, the four fillfactor fixtures, two known-content pages, the shapes `pgstatindex` refuses, an index the catalog says is not valid, and — only where the server accepts the reloption — the `deduplicate_items = off` fixture | `cluster` |
| `report` | runs the filed text **as filed**, both `SET` lines included, and records whether it executed, how many rows and columns it returned and how many bytes | `texts`, `fixtures` |
| `facts` | the version-local facts: candidate count, `index_size` against `pg_relation_size`, the two `NaN` comparisons, ten refusals, the invalid index, fresh-build density at four fillfactors, the implied `max_avail` from two known-content pages, the post-VACUUM size of the head-deleted index, and another session's temp index | `texts`, `fixtures` |
| `cost` | `EXPLAIN (ANALYZE, BUFFERS)` of the filed text, then six interleaved end-to-end runs of the filed and superseded texts | `texts` |
| `priv` | creates two login roles, grants `pg_stat_scan_tables` to one, and reads the index three ways: the whole statement, by name, and by OID | `texts`, `fixtures` |
| `score` | snapshots the harness view into `bl_before`, generates and runs one `REINDEX INDEX` per index, snapshots the sizes into `bl_after`, and writes `out/scores17.txt`: one row per fixture plus totals for three populations — all, schema `bl`, and catalog/TOAST. **Destructive**: it rebuilds every index in the database | `texts`, `fixtures` |
| `residual` | re-reads the harness view over exactly the population `score` scored and writes the post-`REINDEX` residual of `wasted_vs_fillfactor`, the ratio between the two percentage columns, and the coincide and clamp counts | `score` |
| `race` | the `lock_timeout` cancellation, then the two concurrent-drop timings: a drop that commits while `cand` runs, and a sweep of ten delays aimed at the window between `cand` and the per-index call | `texts`, `fixtures` |
| `summary` | prints what landed in `out/` and the small result files | nothing |
| `stop` | stops the server with `pg_ctl -m fast -w stop`, so the checkpointer writes a shutdown checkpoint and the next start needs no recovery, then confirms no `postmaster.pid`, no postgres process on the data directory and an empty socket directory. It dies rather than report a stop that did not happen | `cluster` |
| `clean` | `stop`, then deletes the sandbox after checking it is inside `$WIKI_ROOT/.wiki-runtime/tmp/`; because `stop` dies on a failed teardown, `clean` never deletes a live cluster | nothing |

#### What the scripts read from the environment

| Variable | Default | Read by | Meaning |
|---|---|---|---|
| `WIKI_ROOT` | `$PWD` | both | the repository root; everything else is resolved beneath it |
| `PAGE` | `$WIKI_ROOT/wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md` | both | the page the `sql` block is extracted from |
| `SANDBOX` | `$WIKI_ROOT/.wiki-runtime/tmp/pgsi` | both | build, install, data, socket, SQL and output directories; the only tree either script writes |
| `JOBS` | `4` | both | `make -j` parallelism |
| `ROWS` | `1000000` | both | the base fixture size. The fillfactor fixtures use `ROWS / 5` and the wide-key fixture `ROWS / 10` |
| `OLD_REV` | `cbbbd16` | both | the revision of this page holding the superseded statement text |
| `SRC` / `SRC12` | `$WIKI_ROOT/raw/postgres-17` / `-12` | 17 leg / 12 leg | the pinned checkout, read only |
| `PORT` / `PORT12` | `55417` / `55412` | 17 leg / 12 leg | the cluster's port |

Both scripts export `PGPORT`, `PGHOST` and `PGDATABASE` for their own `psql`
calls, so a value in the caller's environment is overridden rather than
honoured.

The cluster settings each `cluster` stage writes, with the apply scope each one
needs. All of them are written to `postgresql.conf` before the first start, so
every one is in force from the first connection, and none of them is changed
again while the cluster is up:

| Setting | Value | Context | Apply scope |
|---|---|---|---|
| `listen_addresses` | `''` | `PGC_POSTMASTER` | restart ([guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4436-L4445)) |
| `port` | `55417` / `55412` | `PGC_POSTMASTER` | restart ([guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2393-L2401)) |
| `unix_socket_directories` | inside the sandbox | `PGC_POSTMASTER` | restart ([guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4425-L4434)) |
| `shared_buffers` | `512MB` | `PGC_POSTMASTER` | restart ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270)) |
| `logging_collector` | `off` | `PGC_POSTMASTER` | restart ([guc_tables.c#logging_collector](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1640-L1648)) |
| `fsync` | `off` | `PGC_SIGHUP` | reload ([guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1096-L1107)) |
| `autovacuum` | `off` | `PGC_SIGHUP` | reload ([guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)) |
| `maintenance_work_mem` | `256MB` | `PGC_USERSET` | session or transaction ([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)) |
| `max_parallel_maintenance_workers` | `0` | `PGC_USERSET` | session or transaction ([guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)) |

`autovacuum` is off so that no background vacuum changes a fixture between the
report and the scoring pass; `fsync` is off because the cluster is disposable.
Both scripts also set `statement_timeout` and `lock_timeout` per session, both
`PGC_USERSET`
([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620),
[guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)).
These contexts are read from the v17 GUC table, which is this page's version;
the 12.2 contexts are not citable here, and that is
[an open question](#open-questions).

#### Prerequisites

- A C toolchain and `make`. Both legs build their own server; no installed
  PostgreSQL is used or needed.
- Development headers for ICU, readline and zlib for the 17 leg; readline and
  zlib alone for the 12 leg.
- The two pinned checkouts, at `raw/postgres-17` and `raw/postgres-12`. Both
  stay read-only: each build is a VPATH build in a directory under
  `.wiki-runtime/tmp/`, which is the form the documentation describes
  ([installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L425-L436)).
- `git`, with `OLD_REV` reachable, since `texts` recovers the superseded
  statement with `git show`.
- `sha256sum`, and a Bash new enough for arrays and `${var:-default}`. Nothing
  else: no Python, no `awk`, no `perl`, no `jq`.
- Free TCP ports 55417 and 55412, or `PORT`/`PORT12` set to free ones. Both
  clusters set `listen_addresses = ''` and listen on a Unix socket inside the
  sandbox, so the port is reserved but never bound on TCP.
- `contrib/pgstattuple`, built and installed from the same tree as the server,
  into the disposable cluster only.
- Disk for two source builds, two clusters and the fixtures: the recorded run
  left a 1828 MB and a 1801 MB database, and `score` briefly doubles the
  largest index it rebuilds.

**Every statement either script sends is disposable.** The fixture stage drops
and rebuilds a whole schema, writes `indisvalid = false` into `pg_index` by
hand, creates and drops two login roles, drops and recreates indexes during the
race stage, and rebuilds every index in the database during the scoring pass.
Never point `SANDBOX`, `PORT`, `PORT12` or `PGHOST` at a cluster anyone cares
about.

### The PostgreSQL 17 leg script

```sh
#!/usr/bin/env bash
#
# bloat_pgstatindex_v17.sh - the PostgreSQL 17 leg of the measurement behind
# "B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and
# 17".  Bash and SQL only: build the pinned 17 checkout out of tree, run its
# regression suites, start an isolated cluster, build the published fixture
# suite, run this page's statement exactly as filed, and score every estimate
# against a measured REINDEX INDEX.
#
# The pinned checkout is read only.  Everything this script writes lives under
# $SANDBOX, and `clean` deletes it.
#
# Usage, from the repository root:
#   bash bloat_pgstatindex_v17.sh                 # every stage, in order
#   bash bloat_pgstatindex_v17.sh score residual  # selected stages
#   bash bloat_pgstatindex_v17.sh clean           # stop and delete the sandbox
#
# Stages: build check cluster texts fixtures report facts score residual cost
#         priv race summary stop clean
#
# Environment: WIKI_ROOT PAGE SRC SANDBOX PORT JOBS ROWS OLD_REV
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-17}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/pgsi}"
PORT="${PORT:-55417}"
JOBS="${JOBS:-4}"
ROWS="${ROWS:-1000000}"
OLD_REV="${OLD_REV:-cbbbd16}"

BUILD="$SANDBOX/build17"; INST="$SANDBOX/install17"; DATA="$SANDBOX/data17"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql"; SOCK="$SANDBOX/sock17"; BIN="$INST/bin"
DB=bloat17
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE=postgres

# SHA-256 of the sql blocks this page files.  BASE_SQL is the filed statement;
# BASE_PREV is the text filed before wasted space was rebased on the
# fillfactor, recovered from OLD_REV for the cost comparison.
BASE_SQL=9d2e3a2c73c81f3efb848b61f4bf307365ce0da56d780baee08fa9a9606f6c91
BASE_PREV=f5b995d3c5d51dddd1378e4e1ac31f9ad180cec0cc811a803186b86a1fb721e9

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# -X ignores ~/.psqlrc; ON_ERROR_STOP is on every helper, because without it a
# failed statement inside a -f script leaves the exit status 0.
q()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }
s()  { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }
t()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" -c "$1"; }
fl() { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -f "$1"; }
# err() runs a statement that is expected to fail and prints the message only.
err() { "$BIN/psql" -X -At -q -d "$DB" -c "$1" 2>&1 | grep -E '^(ERROR|FATAL)' | head -1; }

# The fence is assembled at run time so that this script contains no literal
# Markdown fence and can therefore live inside one.
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

# ---------------------------------------------------------------- build ------
stage_build() {
  say "build 17 out of tree from $SRC"
  [ -x "$BIN/postgres" ] && { note "already built, skipping"; return 0; }
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC; set SRC or run from the repository root"
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --enable-debug \
      --with-icu --with-readline --with-zlib > configure.log 2>&1 ) \
    || { cp "$BUILD/configure.log" "$OUT/configure17.log" 2>/dev/null; die "configure failed, see $OUT/configure17.log"; }
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { cp "$BUILD"/*.log "$OUT/" 2>/dev/null; grep -m3 'error:' "$BUILD/make.log" >&2; die "make failed"; }
  ( cd "$BUILD" && make -C contrib/pgstattuple -j"$JOBS" >> install.log 2>&1 \
      && make -C contrib/pgstattuple install >> install.log 2>&1 ) || die "contrib/pgstattuple failed"
  local l
  for l in configure make install; do cp "$BUILD/$l.log" "$OUT/${l}17.log" 2>/dev/null; done
  note "$("$BIN/postgres" --version)"
}

stage_check() {
  say "regression suites, 17"
  mkdir -p "$OUT"
  : > "$OUT/checks17.txt"
  ( cd "$BUILD" && make check > check_core.log 2>&1 )
  printf 'core=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_core.log" | tail -1)" \
    >> "$OUT/checks17.txt"
  ( cd "$BUILD" && make -C contrib/pgstattuple check > check_pgstattuple.log 2>&1 )
  printf 'pgstattuple=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_pgstattuple.log" | tail -1)" \
    >> "$OUT/checks17.txt"
  local l d
  for l in "$BUILD"/check_*.log; do [ -f "$l" ] && cp "$l" "$OUT/17_$(basename "$l")"; done
  for d in "$BUILD"/src/test/regress/regression.diffs "$BUILD"/contrib/pgstattuple/regression.diffs; do
    [ -f "$d" ] && cp "$d" "$OUT/diffs17_$(basename "$(dirname "$d")").txt"
  done
  cat "$OUT/checks17.txt" >&2
}

# -------------------------------------------------------------- cluster ------
# Cluster settings and their apply scope, all written to postgresql.conf before
# the first start, so every one of them is in force from the first connection:
#   listen_addresses, port, unix_socket_directories, shared_buffers,
#   logging_collector  -> PGC_POSTMASTER, restart
#   fsync, autovacuum                                   -> PGC_SIGHUP, reload
#   maintenance_work_mem, max_parallel_maintenance_workers -> PGC_USERSET,
#                                                          session scope
# autovacuum is off so that no background vacuum changes a fixture between the
# report and the scoring pass.
stage_cluster() {
  say "isolated 17 cluster on port $PORT"
  mkdir -p "$OUT" "$SQLD" "$SOCK"
  if [ ! -f "$DATA/PG_VERSION" ]; then
    "$BIN/initdb" -D "$DATA" --locale=C --encoding=UTF8 > "$OUT/initdb17.log" 2>&1 \
      || die "initdb failed, see $OUT/initdb17.log"
    cat >> "$DATA/postgresql.conf" <<CONF
listen_addresses = ''
port = $PORT
unix_socket_directories = '$SOCK'
shared_buffers = 512MB
maintenance_work_mem = 256MB
max_parallel_maintenance_workers = 0
autovacuum = off
fsync = off
logging_collector = off
CONF
  fi
  if ! "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server17.log" -w start > /dev/null 2>&1 \
      || die "server did not start, see $OUT/server17.log"
  fi
  "$BIN/psql" -X -At -q -d postgres -c "SELECT 1" > /dev/null 2>&1 || die "cannot connect"
  "$BIN/psql" -X -At -q -d postgres -c \
    "SELECT count(*) FROM pg_database WHERE datname = '$DB'" | grep -q '^1$' \
    || "$BIN/createdb" "$DB" || die "createdb failed"
  q "CREATE EXTENSION IF NOT EXISTS pgstattuple;" > /dev/null || die "pgstattuple not installed"
  {
    printf 'uname=%s\n' "$(uname -sm)"
    printf 'server_version_num=%s\n' "$(s 'SHOW server_version_num')"
    printf 'server_version=%s\n' "$(s 'SHOW server_version')"
    printf 'block_size=%s\n' "$(s 'SHOW block_size')"
    printf 'max_data_alignment=%s\n' "$("$BIN/pg_controldata" -D "$DATA" | grep -i 'maximum data alignment' | tr -s ' ' | cut -d' ' -f4)"
    printf 'pgstattuple=%s\n' "$(s "SELECT extversion FROM pg_extension WHERE extname = 'pgstattuple'")"
  } > "$OUT/platform17.txt"
  cat "$OUT/platform17.txt" >&2
}

# ---------------------------------------------------------------- texts -----
# Two texts come out of the page and one out of git history:
#   report.sql   the filed statement, byte for byte, hash-checked
#   view.sql     the same text as a view over the internal `final` stage, with
#                exactly two edits, both printed: the two SET lines dropped and
#                min_index_bytes set to 0 so sub-megabyte indexes are scored
#   prev.sql     the text filed before wasted space was rebased on the
#                fillfactor, recovered from OLD_REV for the cost comparison
gen_view() {                       # gen_view <view> < text
  local view=$1 line tail=0
  printf 'DROP VIEW IF EXISTS %s CASCADE;\nCREATE VIEW %s AS\n' "$view" "$view"
  while IFS= read -r line; do
    case $line in
      "SET statement_timeout"*|"SET lock_timeout"*)
        printf '   harness edit: dropped %s\n' "$line" >&2; continue ;;
      *"AS min_index_bytes"*)
        printf '   harness edit: %s -> 0\n' "$(printf '%s' "$line" | tr -s ' ')" >&2
        printf '           0::bigint AS min_index_bytes  -- harness: score every index\n'
        continue ;;
      "SELECT /* wiki_btree_bloat_pgstatindex_12_17 */") tail=1; continue ;;
    esac
    [ "$tail" = 1 ] && continue
    printf '%s\n' "$line"
  done
  printf 'SELECT f.* FROM final f;\n'
}

stage_texts() {
  say "extract the statement from $PAGE"
  mkdir -p "$SQLD" "$OUT"
  [ -f "$PAGE" ] || die "no page at $PAGE; set PAGE or run from the repository root"
  md_block sql 1 "$PAGE" > "$SQLD/report.sql"
  local got
  got=$(sha256sum < "$SQLD/report.sql" | cut -d' ' -f1)
  [ "$got" = "$BASE_SQL" ] || die "sql block 1 hashes $got, expected $BASE_SQL"
  note "filed text: $(wc -l < "$SQLD/report.sql") lines, $(wc -c < "$SQLD/report.sql") bytes, sha256 ${got:0:12}"
  gen_view bloat_final < "$SQLD/report.sql" > "$SQLD/view.sql"
  # The statement alone, without the two SET lines, for EXPLAIN.
  grep -v '^SET ' "$SQLD/report.sql" > "$SQLD/bare.sql"
  if git -C "$WIKI_ROOT" cat-file -e "$OLD_REV:wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md" 2>/dev/null; then
    git -C "$WIKI_ROOT" show "$OLD_REV:wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md" > "$SQLD/page_prev.md"
    md_block sql 1 "$SQLD/page_prev.md" > "$SQLD/prev.sql"
    got=$(sha256sum < "$SQLD/prev.sql" | cut -d' ' -f1)
    [ "$got" = "$BASE_PREV" ] || die "superseded text hashes $got, expected $BASE_PREV"
    grep -v '^SET ' "$SQLD/prev.sql" > "$SQLD/prev_bare.sql"
    note "superseded text: $(wc -l < "$SQLD/prev.sql") lines, $(wc -c < "$SQLD/prev.sql") bytes, sha256 ${got:0:12}"
  else
    note "no revision $OLD_REV in this repository; the cost comparison will be skipped"
  fi
}

# ------------------------------------------------------------- fixtures -----
# Every statement in this stage is DISPOSABLE.  It drops and rebuilds schema
# bl in a throwaway database and is not meant for a database anyone cares
# about.  The two session GUCs it sets are PGC_USERSET: session scope, no
# reload and no restart.
stage_fixtures() {
  say "build the fixture suite in $DB (rows=$ROWS)"
  cat > "$SQLD/fixtures.sql" <<'SQL'
-- DISPOSABLE fixture suite for the pgstatindex bloat report.
-- Every object lives in schema bl of a throwaway database.
SET statement_timeout = '30min';   -- PGC_USERSET, session scope
SET lock_timeout      = '30s';     -- PGC_USERSET, session scope
SET client_min_messages = warning;

DROP SCHEMA IF EXISTS bl CASCADE;
CREATE SCHEMA bl;

-- 1. scattered deletes at five fractions, all vacuumed afterwards ----------
CREATE TABLE bl.t_del90 (id int);
INSERT INTO bl.t_del90 SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_del90 ON bl.t_del90 (id);
DELETE FROM bl.t_del90 WHERE id % 10 <> 0;
VACUUM bl.t_del90;

CREATE TABLE bl.t_del50 (id int);
INSERT INTO bl.t_del50 SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_del50 ON bl.t_del50 (id);
DELETE FROM bl.t_del50 WHERE id % 2 <> 0;
VACUUM bl.t_del50;

-- 2. the same shape, deliberately not vacuumed -----------------------------
CREATE TABLE bl.t_novac (id int);
INSERT INTO bl.t_novac SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_novac ON bl.t_novac (id);
DELETE FROM bl.t_novac WHERE id % 10 <> 0;

-- 3. a contiguous head deleted: whole pages empty out ----------------------
CREATE TABLE bl.t_delhead (id int);
INSERT INTO bl.t_delhead SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_delhead ON bl.t_delhead (id);
DELETE FROM bl.t_delhead WHERE id <= (:rows * 7) / 10;
VACUUM bl.t_delhead;

-- 4. partial, unique, expression, INCLUDE, multicolumn, text, wide ---------
CREATE TABLE bl.t_partial (id int, flag boolean);
INSERT INTO bl.t_partial SELECT g, true FROM generate_series(1, :rows) g;
CREATE INDEX i_partial ON bl.t_partial (id) WHERE flag;
DELETE FROM bl.t_partial WHERE flag AND id % 10 <> 0;
VACUUM bl.t_partial;

CREATE TABLE bl.t_uniq (id int);
INSERT INTO bl.t_uniq SELECT g FROM generate_series(1, :rows) g;
CREATE UNIQUE INDEX i_uniq ON bl.t_uniq (id);
DELETE FROM bl.t_uniq WHERE id % 8 <> 0;
VACUUM bl.t_uniq;

CREATE TABLE bl.t_expr (id int);
INSERT INTO bl.t_expr SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_expr ON bl.t_expr ((id * 2));
DELETE FROM bl.t_expr WHERE id % 5 <> 0;
VACUUM bl.t_expr;

CREATE TABLE bl.t_incl (id int, v int);
INSERT INTO bl.t_incl SELECT g, g FROM generate_series(1, :rows) g;
CREATE INDEX i_incl ON bl.t_incl (id) INCLUDE (v);
DELETE FROM bl.t_incl WHERE id % 4 <> 0;
VACUUM bl.t_incl;

CREATE TABLE bl.t_multi (a int, b int, c int);
INSERT INTO bl.t_multi SELECT g, g, g FROM generate_series(1, :rows) g;
CREATE INDEX i_multi ON bl.t_multi (a, b, c);
DELETE FROM bl.t_multi WHERE a % 3 <> 0;
VACUUM bl.t_multi;

CREATE TABLE bl.t_text_del (id int, v text);
INSERT INTO bl.t_text_del SELECT g, md5(g::text) FROM generate_series(1, :rows) g;
CREATE INDEX i_text_del ON bl.t_text_del (v);
DELETE FROM bl.t_text_del WHERE id % 5 <> 0;
VACUUM bl.t_text_del;

CREATE TABLE bl.t_wide (id int, v text);
INSERT INTO bl.t_wide SELECT g, rpad(md5(g::text), 400, 'w') FROM generate_series(1, :wide_rows) g;
CREATE INDEX i_wide ON bl.t_wide (v);
DELETE FROM bl.t_wide WHERE id % 10 >= 3;
VACUUM bl.t_wide;

-- 5. unlogged, partitioned, TOAST-owning, empty ----------------------------
CREATE UNLOGGED TABLE bl.t_unlogged (id int);
INSERT INTO bl.t_unlogged SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_unlogged ON bl.t_unlogged (id);
DELETE FROM bl.t_unlogged WHERE id % 6 <> 0;
VACUUM bl.t_unlogged;

CREATE TABLE bl.t_part (id int) PARTITION BY RANGE (id);
CREATE TABLE bl.t_part_1 PARTITION OF bl.t_part FOR VALUES FROM (1) TO (500001);
CREATE TABLE bl.t_part_2 PARTITION OF bl.t_part FOR VALUES FROM (500001) TO (1000001);
INSERT INTO bl.t_part SELECT g FROM generate_series(1, 1000000) g;
CREATE INDEX i_part ON bl.t_part (id);
DELETE FROM bl.t_part WHERE id % 7 <> 0;
VACUUM bl.t_part_1;
VACUUM bl.t_part_2;

CREATE TABLE bl.t_toast (id int PRIMARY KEY, v text);
ALTER TABLE bl.t_toast ALTER COLUMN v SET STORAGE EXTERNAL;
INSERT INTO bl.t_toast SELECT g, rpad(md5(g::text), 4000, 'p') FROM generate_series(1, 20000) g;
DELETE FROM bl.t_toast WHERE id % 3 <> 0;
VACUUM bl.t_toast;

CREATE TABLE bl.t_empty (id int PRIMARY KEY);

-- 6. churn and reverse-order inserts ---------------------------------------
CREATE TABLE bl.t_churn (id int);
INSERT INTO bl.t_churn SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_churn ON bl.t_churn (id);
UPDATE bl.t_churn SET id = id + :rows;
VACUUM bl.t_churn;

CREATE TABLE bl.t_frag (id int);
CREATE INDEX i_frag ON bl.t_frag (id);
INSERT INTO bl.t_frag SELECT g FROM generate_series(:rows, 1, -1) g;

-- 7. fresh builds at four fillfactors, plus a fresh text index -------------
CREATE TABLE bl.t_fresh (id int);
INSERT INTO bl.t_fresh SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_fresh  ON bl.t_fresh (id);
CREATE INDEX i_ff100  ON bl.t_fresh (id) WITH (fillfactor = 100);
CREATE INDEX i_ff50   ON bl.t_fresh (id) WITH (fillfactor = 50);
CREATE INDEX i_ff10   ON bl.t_fresh (id) WITH (fillfactor = 10);

CREATE TABLE bl.t_text (v text);
INSERT INTO bl.t_text SELECT md5(g::text) FROM generate_series(1, :rows) g;
CREATE INDEX i_text ON bl.t_text (v);

-- 8. duplicates: built by CREATE INDEX, and built by inserts ---------------
CREATE TABLE bl.t_dup (v int);
INSERT INTO bl.t_dup SELECT g % 10 FROM generate_series(1, :rows) g;
CREATE INDEX i_dup ON bl.t_dup (v);

CREATE TABLE bl.t_dup_ins (v int);
CREATE INDEX i_dup_ins ON bl.t_dup_ins (v);
INSERT INTO bl.t_dup_ins SELECT g % 10 FROM generate_series(1, :rows) g;

-- 9. deduplication turned off at build time, then turned back on.
-- The reloption does not exist before PostgreSQL 13, so this fixture is
-- built only where the server accepts it; the 12 leg records its absence.
DO $dd$
BEGIN
  IF current_setting('server_version_num')::int >= 130000 THEN
    EXECUTE 'CREATE TABLE bl.t_dedup (v int)';
    EXECUTE 'INSERT INTO bl.t_dedup SELECT g % 10 FROM generate_series(1, ' ||
            current_setting('bl.rows') || ') g';
    EXECUTE 'CREATE INDEX i_dedup_off ON bl.t_dedup (v) WITH (deduplicate_items = off)';
    EXECUTE 'ALTER INDEX bl.i_dedup_off SET (deduplicate_items = on)';
  END IF;
END
$dd$;

-- 10. the four fillfactor fixtures: built at a stated fillfactor, then nine
-- tenths of the rows deleted and the table vacuumed ------------------------
CREATE TABLE bl.t_ffdel (id int);
INSERT INTO bl.t_ffdel SELECT g FROM generate_series(1, :ff_rows) g;
CREATE INDEX i_ff100_del90 ON bl.t_ffdel (id) WITH (fillfactor = 100);
CREATE INDEX i_ff50_del90  ON bl.t_ffdel (id) WITH (fillfactor = 50);
CREATE INDEX i_ff10_del90  ON bl.t_ffdel (id) WITH (fillfactor = 10);
DELETE FROM bl.t_ffdel WHERE id % 10 <> 0;
VACUUM bl.t_ffdel;

CREATE TABLE bl.t_ffhead (id int);
INSERT INTO bl.t_ffhead SELECT g FROM generate_series(1, :ff_rows) g;
CREATE INDEX i_ff50_delhead ON bl.t_ffhead (id) WITH (fillfactor = 50);
DELETE FROM bl.t_ffhead WHERE id <= (:ff_rows * 9) / 10;
VACUUM bl.t_ffhead;

-- 11. two known-content pages, for the implied leaf capacity ---------------
CREATE TABLE bl.c_one (id int);
INSERT INTO bl.c_one VALUES (1);
CREATE INDEX c_one_idx ON bl.c_one (id);

CREATE TABLE bl.c_zero (id int);
INSERT INTO bl.c_zero VALUES (1);
CREATE INDEX c_zero_idx ON bl.c_zero (id);
DELETE FROM bl.c_zero;
VACUUM bl.c_zero;

-- 12. shapes pgstatindex refuses -------------------------------------------
CREATE TABLE bl.s_other (id int, v text, g_point point, r int4range);
INSERT INTO bl.s_other SELECT g, md5(g::text), point(g, g), int4range(g, g + 10)
  FROM generate_series(1, 20000) g;
CREATE INDEX s_hash   ON bl.s_other USING hash   (id);
CREATE INDEX s_gin    ON bl.s_other USING gin    (to_tsvector('simple', v));
CREATE INDEX s_gist   ON bl.s_other USING gist   (g_point);
CREATE INDEX s_spgist ON bl.s_other USING spgist (g_point);
CREATE INDEX s_brin   ON bl.s_other USING brin   (id);
CREATE VIEW  bl.s_view AS SELECT 1 AS one;
CREATE SEQUENCE bl.s_seq;

-- 13. an index the catalog says is not valid.  Only a disposable cluster
-- may have its catalog written to by hand like this.
CREATE TABLE bl.t_invalid (id int);
INSERT INTO bl.t_invalid SELECT g FROM generate_series(1, 300000) g;
CREATE INDEX i_invalid ON bl.t_invalid (id);
UPDATE pg_index SET indisvalid = false WHERE indexrelid = 'bl.i_invalid'::regclass;
SQL
  local ffrows=$((ROWS / 5)) widerows=$((ROWS / 10))
  # psql does not substitute :variables inside a dollar-quoted body, so the
  # version-guarded fixture reads its row count from a database-level GUC.
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" \
    -c "ALTER DATABASE $DB SET bl.rows = '$ROWS'" > /dev/null || die "cannot set bl.rows"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" \
    -v rows="$ROWS" -v ff_rows="$ffrows" -v wide_rows="$widerows" \
    -f "$SQLD/fixtures.sql" || die "fixtures failed"
  note "indexes in bl: $(s "SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace WHERE n.nspname = 'bl' AND c.relkind IN ('i','I')")"
  note "database size: $(s "SELECT pg_size_pretty(pg_database_size(current_database()))")"
}

# --------------------------------------------------------------- report -----
# Runs the filed text exactly as filed, including its two SET lines.
stage_report() {
  say "run the filed statement"
  [ -f "$SQLD/report.sql" ] || die "run the texts stage first"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -P footer=off -A -F '|' -d "$DB" \
    -f "$SQLD/report.sql" > "$OUT/report17.txt" 2> "$OUT/report17.err"
  local rc=$?
  if [ "$rc" != 0 ]; then
    printf 'exact_text=refused\n' > "$OUT/exact17.txt"
    head -5 "$OUT/report17.err" >> "$OUT/exact17.txt"
    cat "$OUT/exact17.txt" >&2
    die "the filed text did not run; see $OUT/report17.err"
  fi
  printf 'exact_text=executes\n' > "$OUT/exact17.txt"
  # footer=off leaves one header line and one line per row.
  local rows cols
  rows=$(grep -c '^' "$OUT/report17.txt")
  rows=$((rows - 1))
  cols=$(head -1 "$OUT/report17.txt" | tr '|' '\n' | grep -c '^')
  {
    printf 'report_rows=%s\n' "$rows"
    printf 'report_columns=%s\n' "$cols"
    printf 'report_bytes=%s\n' "$(wc -c < "$OUT/report17.txt")"
  } >> "$OUT/exact17.txt"
  cat "$OUT/exact17.txt" >&2
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" \
    -f "$SQLD/report.sql" > "$OUT/report17_pretty.txt" 2>&1
}

# ---------------------------------------------------------------- facts -----
stage_facts() {
  say "version-local facts, refusals and page arithmetic"
  fl "$SQLD/view.sql" > /dev/null || die "harness view failed"
  : > "$OUT/facts17.txt"
  local f="$OUT/facts17.txt"
  {
    printf 'server_version_num=%s\n' "$(s 'SHOW server_version_num')"
    printf 'block_size=%s\n' "$(s 'SHOW block_size')"
    printf 'candidates=%s\n' "$(s 'SELECT count(*) FROM bloat_final')"
    printf 'index_size_equals_relation_size=%s of %s\n' \
      "$(s 'SELECT count(*) FROM bloat_final f WHERE f.index_size = pg_relation_size(f.idx_oid)')" \
      "$(s 'SELECT count(*) FROM bloat_final')"
    printf 'nan_density_indexes=%s\n' "$(s 'SELECT count(*) FROM bloat_final WHERE leaf_pages = 0')"
    printf 'nan_float8_gt_20=%s\n' "$(s "SELECT ('NaN'::float8 > 20)::text")"
    printf 'nan_numeric_gt_20=%s\n' "$(s "SELECT ('NaN'::numeric > 20)::text")"
    printf 'dedup_fixture_present=%s\n' "$(s "SELECT (to_regclass('bl.i_dedup_off') IS NOT NULL)::text")"
  } >> "$f"

  # Every refusal the cand filters exist for.
  {
    printf 'err_hash=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_hash'::regclass)")"
    printf 'err_gin=%s\n'       "$(err "SELECT * FROM pgstatindex('bl.s_gin'::regclass)")"
    printf 'err_gist=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_gist'::regclass)")"
    printf 'err_spgist=%s\n'    "$(err "SELECT * FROM pgstatindex('bl.s_spgist'::regclass)")"
    printf 'err_brin=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_brin'::regclass)")"
    printf 'err_partitioned=%s\n' "$(err "SELECT * FROM pgstatindex('bl.i_part'::regclass)")"
    printf 'err_table=%s\n'     "$(err "SELECT * FROM pgstatindex('bl.t_del90'::regclass)")"
    printf 'err_view=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_view'::regclass)")"
    printf 'err_sequence=%s\n'  "$(err "SELECT * FROM pgstatindex('bl.s_seq'::regclass)")"
    printf 'err_stale_oid=%s\n' "$(err "SELECT * FROM pgstatindex(2147483647::oid::regclass)")"
  } >> "$f"

  # The invalid index: an error on 17, a row on 12.
  local inv
  inv=$(err "SELECT * FROM pgstatindex('bl.i_invalid'::regclass)")
  if [ -n "$inv" ]; then
    printf 'invalid_index=refused %s\n' "$inv" >> "$f"
  else
    printf 'invalid_index=row %s\n' \
      "$(s "SELECT version || '|' || tree_level || '|' || index_size || '|' || root_block_no || '|' || internal_pages || '|' || leaf_pages || '|' || empty_pages || '|' || deleted_pages || '|' || avg_leaf_density || '|' || leaf_fragmentation FROM pgstatindex('bl.i_invalid'::regclass)")" >> "$f"
  fi

  # Fresh-build density at four fillfactors, and the reported estimate.
  printf 'fresh_builds fillfactor|target_density|avg_leaf_density|est_reclaimable_pct\n' >> "$f"
  s "SELECT f.fillfactor || '|' || round(100 * f.target_density, 2) || '|' ||
            round(f.avg_leaf_density::numeric, 2) || '|' ||
            round(100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size, 1)
       FROM bloat_final f
      WHERE f.index_name IN ('i_ff100', 'i_fresh', 'i_ff50', 'i_ff10')
      ORDER BY f.fillfactor DESC" >> "$f"

  # Leaf capacity implied by two pages of known contents.  A root leaf page
  # holding one int4 entry leaves 8192 - 16 special - 24 header - 16 tuple
  # - 4 line pointer = 8128 free; an empty leaf page leaves 8148.
  printf 'implied_max_avail one_tuple|empty_leaf\n' >> "$f"
  s "SELECT round(8128 / (1 - one.d / 100)::numeric, 1) || '|' ||
            round(8148 / (1 - zero.d / 100)::numeric, 1)
       FROM (SELECT avg_leaf_density::numeric AS d FROM pgstatindex('bl.c_one_idx'::regclass)) one,
            (SELECT avg_leaf_density::numeric AS d FROM pgstatindex('bl.c_zero_idx'::regclass)) zero" >> "$f"
  printf 'known_page_densities one|zero\n' >> "$f"
  s "SELECT (SELECT round(avg_leaf_density::numeric, 2) FROM pgstatindex('bl.c_one_idx'::regclass)) || '|' ||
            (SELECT round(avg_leaf_density::numeric, 2) FROM pgstatindex('bl.c_zero_idx'::regclass))" >> "$f"

  # VACUUM does not give index pages back; REINDEX does.
  printf 'delhead_after_vacuum=%s dead_pages=%s\n' \
    "$(s "SELECT index_size FROM pgstatindex('bl.i_delhead'::regclass)")" \
    "$(s "SELECT empty_pages + deleted_pages FROM pgstatindex('bl.i_delhead'::regclass)")" >> "$f"

  # Another session's temp index, with and without the filter.  Each -c is
  # its own transaction, so the temp relation is committed and visible to
  # this session while the sleeping one still owns it; a single multi-
  # statement -c would keep the catalog rows uncommitted and invisible.
  "$BIN/psql" -X -q -d "$DB" \
    -c "CREATE TEMP TABLE tmp_other(id int)" \
    -c "INSERT INTO tmp_other SELECT g FROM generate_series(1, 300000) g" \
    -c "CREATE INDEX tmp_other_idx ON tmp_other(id)" \
    -c "SELECT pg_sleep(25)" > /dev/null 2>&1 &
  local other=$!
  sleep 8
  {
    printf 'other_temp_candidates_filtered=%s\n' "$(s 'SELECT count(*) FROM bloat_final')"
    printf 'other_temp_candidates_unfiltered=%s\n' \
      "$(s "SELECT count(*) FROM pg_class c JOIN pg_index x ON x.indexrelid = c.oid
              JOIN pg_am a ON a.oid = c.relam
             WHERE a.amname = 'btree' AND c.relkind = 'i'
               AND x.indisvalid AND x.indisready AND x.indislive")"
    printf 'other_temp_index_size=%s\n' \
      "$(s "SELECT coalesce(pg_size_pretty(max(pg_relation_size(c.oid))), 'none')
              FROM pg_class c WHERE c.relname = 'tmp_other_idx' AND c.relkind = 'i'")"
    printf 'other_temp_error=%s\n' \
      "$(err "SELECT * FROM pgstatindex((SELECT c.oid FROM pg_class c WHERE c.relname = 'tmp_other_idx' AND c.relkind = 'i' LIMIT 1)::regclass)")"
    printf 'report_rows_with_other_session=%s\n' \
      "$(( $("$BIN/psql" -X -q -A -F '|' -P footer=off -v ON_ERROR_STOP=1 -d "$DB" -f "$SQLD/report.sql" 2>/dev/null | grep -c '^') - 1 ))"
  } >> "$f"
  wait "$other" 2>/dev/null
  cat "$f" >&2
}

# ---------------------------------------------------------------- score -----
# Ground truth is pg_relation_size before and after REINDEX INDEX, over every
# index the harness view can see.  This stage is destructive: it rebuilds
# every index in the database, so it must run after report, facts and cost.
stage_score() {
  say "score every estimate against a measured REINDEX INDEX"
  fl "$SQLD/view.sql" > /dev/null || die "harness view failed"
  q "DROP VIEW IF EXISTS bl_scored;
     DROP TABLE IF EXISTS bl_before, bl_after, bl_residual;
     CREATE TABLE bl_before AS
       SELECT /* wiki_pgsi_score_before */
              f.idx_oid, f.schema_name, f.index_name, f.index_size,
              f.est_rebuilt_bytes, f.wasted_vs_fillfactor, f.avg_leaf_density,
              f.leaf_pages, f.dead_pages, f.fillfactor, f.target_density,
              f.leaf_bytes, f.live_leaf_bytes, f.dead_bytes,
              pg_relation_size(f.idx_oid) AS before_bytes
         FROM bloat_final f;" > /dev/null || die "bl_before failed"
  note "indexes to score: $(s 'SELECT count(*) FROM bl_before')"
  s "SELECT 'REINDEX INDEX ' || idx_oid::regclass || ';' FROM bl_before ORDER BY idx_oid" \
    > "$SQLD/reindex.sql"
  fl "$SQLD/reindex.sql" > /dev/null || die "REINDEX pass failed"
  q "CREATE TABLE bl_after AS
       SELECT /* wiki_pgsi_score_after */
              b.idx_oid, pg_relation_size(b.idx_oid) AS after_bytes
         FROM bl_before b;
     CREATE VIEW bl_scored AS
       SELECT b.*, a.after_bytes,
              100 * (b.index_size - b.est_rebuilt_bytes) / b.index_size AS est_pct,
              100 * (b.before_bytes - a.after_bytes)::numeric / b.before_bytes AS actual_pct,
              100 * (b.index_size - b.est_rebuilt_bytes) / b.index_size
              - 100 * (b.before_bytes - a.after_bytes)::numeric / b.before_bytes AS delta
         FROM bl_before b JOIN bl_after a USING (idx_oid);" > /dev/null \
    || die "post-reindex pass failed"

  {
    printf 'index|size|est_pct|actual_pct|delta|density|dead\n'
    s "SELECT index_name || '|' || index_size || '|' || round(est_pct, 1) || '|' ||
              round(actual_pct, 1) || '|' || round(delta, 1) || '|' ||
              coalesce(round(avg_leaf_density::numeric, 2)::text, 'NaN') || '|' || dead_pages
         FROM bl_scored WHERE schema_name = 'bl' ORDER BY actual_pct DESC, index_name"
    # Three populations, because the scoring pass rewrites pg_class while it
    # runs: a catalog index can grow between the snapshot and its own
    # REINDEX, which is a property of the harness and not of the estimator.
    printf '\n-- totals, by population\n'
    s "SELECT 'all: scored=' || count(*) ||
              ' within_1.0=' || count(*) FILTER (WHERE abs(delta) <= 1.0) ||
              ' within_2.0=' || count(*) FILTER (WHERE abs(delta) <= 2.0) ||
              ' max_over=' || round(max(delta), 1) || ' max_under=' || round(min(delta), 1)
         FROM bl_scored
        UNION ALL
       SELECT 'fixtures (schema bl): scored=' || count(*) ||
              ' within_1.0=' || count(*) FILTER (WHERE abs(delta) <= 1.0) ||
              ' within_2.0=' || count(*) FILTER (WHERE abs(delta) <= 2.0) ||
              ' max_over=' || round(max(delta), 1) || ' max_under=' || round(min(delta), 1)
         FROM bl_scored WHERE schema_name = 'bl'
        UNION ALL
       SELECT 'catalog and TOAST: scored=' || count(*) ||
              ' within_1.0=' || count(*) FILTER (WHERE abs(delta) <= 1.0) ||
              ' max_over=' || round(max(delta), 1)
         FROM bl_scored WHERE schema_name <> 'bl'
        UNION ALL
       SELECT 'no leaf pages: ' || count(*) || ', all at est ' ||
              round(max(abs(est_pct)), 1) || ' and actual ' || round(max(abs(actual_pct)), 1)
         FROM bl_scored WHERE leaf_pages = 0"
    printf -- '-- the five largest over-estimates\n'
    s "SELECT schema_name || '.' || index_name || ' ' || pg_size_pretty(index_size) ||
              ' +' || round(delta, 1)
         FROM bl_scored ORDER BY delta DESC LIMIT 5"
    printf -- '-- the three largest under-estimates\n'
    s "SELECT schema_name || '.' || index_name || ' ' || pg_size_pretty(index_size) ||
              ' ' || round(delta, 1)
         FROM bl_scored ORDER BY delta LIMIT 3"
  } > "$OUT/scores17.txt"
  tail -16 "$OUT/scores17.txt" >&2
}

# ------------------------------------------------------------- residual -----
# A rebuilt index must report no waste at its own fillfactor.  Reads the
# tables the score stage left behind.
stage_residual() {
  say "post-REINDEX residual of wasted_vs_fillfactor"
  s "SELECT 1 FROM bl_before LIMIT 1" > /dev/null 2>&1 || die "run the score stage first"
  # Read the same population the score stage scored, not whatever the view
  # sees now: creating bl_before itself adds a TOAST index to the database.
  q "DROP TABLE IF EXISTS bl_residual;
     CREATE TABLE bl_residual AS
       SELECT /* wiki_pgsi_score_residual */
              f.idx_oid, f.wasted_vs_fillfactor AS residual_bytes, f.index_size,
              f.avg_leaf_density, f.leaf_pages
         FROM bloat_final f
        WHERE f.idx_oid IN (SELECT idx_oid FROM bl_before);" > /dev/null \
    || die "residual pass failed"
  {
    printf -- '-- post-REINDEX residual, over the population the score stage scored\n'
    s "SELECT 'scored=' || count(*) ||
              ' exactly_zero=' || count(*) FILTER (WHERE residual_bytes = 0) ||
              ' at_or_below_0.1pct=' || count(*) FILTER (WHERE 100 * residual_bytes / index_size <= 0.1) ||
              ' worst_pct=' || round(max(100 * residual_bytes / index_size), 1)
         FROM bl_residual"
    printf -- '-- worst residual, and the worst among indexes the report prints\n'
    s "SELECT r.idx_oid::regclass || ' ' || round(100 * r.residual_bytes / r.index_size, 1) ||
              '% ' || r.residual_bytes || ' bytes, leaf_pages ' || r.leaf_pages ||
              ', density ' || coalesce(round(r.avg_leaf_density::numeric, 2)::text, 'NaN')
         FROM bl_residual r ORDER BY 100 * r.residual_bytes / r.index_size DESC LIMIT 3"
    s "SELECT 'worst at or above 1 MB: ' || r.idx_oid::regclass || ' ' ||
              round(100 * r.residual_bytes / r.index_size, 1) || '% ' || r.residual_bytes || ' bytes'
         FROM bl_residual r WHERE r.index_size >= 1024 * 1024
        ORDER BY 100 * r.residual_bytes / r.index_size DESC LIMIT 1"
    printf -- '-- how many one-leaf-page indexes carry the worst residual\n'
    s "SELECT 'at_44.6pct=' || count(*) || ', all with leaf_pages ' || max(leaf_pages)
         FROM bl_residual WHERE round(100 * residual_bytes / index_size, 1) = 44.6"
    printf -- '-- the ratio between the two percentage columns, before the rebuild\n'
    s "SELECT b.index_name || ' ' ||
              round(b.wasted_vs_fillfactor / (b.index_size - b.est_rebuilt_bytes), 4) ||
              ' dead_pages ' || b.dead_pages
         FROM bl_before b
        WHERE b.index_size - b.est_rebuilt_bytes > 1024 * 1024
        ORDER BY 1"
    printf -- '-- indexes where the two definitions coincide: fillfactor 100, or no leaf pages\n'
    s "SELECT 'coincide=' || count(*) FILTER (WHERE target_density = 1 OR leaf_pages = 0) ||
              ' of ' || count(*) || ' (fillfactor 100: ' ||
              count(*) FILTER (WHERE target_density = 1) || ', no leaf pages: ' ||
              count(*) FILTER (WHERE leaf_pages = 0) || ')'
         FROM bl_before"
    printf -- '-- clamped rows: leaves denser than their own target, so the leaf term is 0\n'
    s "SELECT 'clamped=' || count(*) FILTER (WHERE leaf_pages > 0
                AND round(leaf_bytes * target_density) <= live_leaf_bytes) ||
              ' of ' || count(*) FILTER (WHERE leaf_pages > 0) || ' with leaf pages'
         FROM bl_before"
  } > "$OUT/residual17.txt"
  cat "$OUT/residual17.txt" >&2
}

# ----------------------------------------------------------------- cost -----
stage_cost() {
  say "what the statement costs to run"
  : > "$OUT/cost17.txt"
  local f="$OUT/cost17.txt"
  printf 'EXPLAIN (ANALYZE, BUFFERS) of the filed text\n' >> "$f"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" \
    -c "SET statement_timeout = '15min'; SET lock_timeout = '5s';" \
    -c "EXPLAIN (ANALYZE, BUFFERS) $(cat "$SQLD/bare.sql")" >> "$f" 2>&1
  printf 'plan_lines=%s\n' "$(grep -c '^' "$f")" >> "$f"
  printf 'cte_scans=%s\n' "$(grep -c 'CTE Scan' "$f")" >> "$f"
  local i a b
  printf 'six interleaved end-to-end runs, filed text then superseded text (ms)\n' >> "$f"
  for i in 1 2 3 4 5 6; do
    a=$(date +%s%N)
    "$BIN/psql" -X -q -o /dev/null -d "$DB" -f "$SQLD/report.sql" > /dev/null 2>&1
    b=$(date +%s%N)
    printf 'filed=%s.%s ' $(( (b - a) / 1000000 )) $(( ((b - a) / 100000) % 10 )) >> "$f"
    if [ -f "$SQLD/prev_bare.sql" ]; then
      a=$(date +%s%N)
      "$BIN/psql" -X -q -o /dev/null -d "$DB" -f "$SQLD/prev.sql" > /dev/null 2>&1
      b=$(date +%s%N)
      printf 'prev=%s.%s' $(( (b - a) / 1000000 )) $(( ((b - a) / 100000) % 10 )) >> "$f"
    fi
    printf '\n' >> "$f"
  done
  printf 'database_size=%s\n' "$(s 'SELECT pg_size_pretty(pg_database_size(current_database()))')" >> "$f"
  grep -E 'Execution Time|plan_lines|cte_scans|filed=|Buffers: shared' "$f" | head -20 >&2
}

# ----------------------------------------------------------------- priv -----
stage_priv() {
  say "who may run it"
  : > "$OUT/priv17.txt"
  local f="$OUT/priv17.txt"
  q "DROP ROLE IF EXISTS mon; DROP ROLE IF EXISTS nomon;
     CREATE ROLE mon LOGIN; CREATE ROLE nomon LOGIN;
     GRANT pg_stat_scan_tables TO mon;" > /dev/null 2>&1 || die "role setup failed"
  # The statement passes an OID column cast to regclass, which resolves no
  # name.  Writing 'bl.i_del90'::regclass in a test would resolve one, so the
  # OID is read here, as a number, and substituted.
  local oid
  oid=$(s "SELECT 'bl.i_del90'::regclass::oid")
  {
    printf 'index_oid=%s\n' "$oid"
    printf 'mon_schema_usage=%s\n' \
      "$(s "SELECT has_schema_privilege('mon', 'bl', 'USAGE')::text")"
    printf 'mon_rows_from_statement=%s\n' \
      "$("$BIN/psql" -X -At -q -U mon -d "$DB" -f "$SQLD/report.sql" 2>&1 | grep -c '|')"
    printf 'mon_by_name=%s\n' \
      "$("$BIN/psql" -X -At -q -U mon -d "$DB" -c "SELECT * FROM pgstatindex('bl.i_del90')" 2>&1 | grep -E '^(ERROR|FATAL)' | head -1)"
    printf 'mon_by_oid_leaf_pages=%s\n' \
      "$("$BIN/psql" -X -At -q -U mon -d "$DB" -c "SELECT leaf_pages FROM pgstatindex($oid::regclass)" 2>&1 | head -1)"
    printf 'nomon_by_oid=%s\n' \
      "$("$BIN/psql" -X -At -q -U nomon -d "$DB" -c "SELECT leaf_pages FROM pgstatindex($oid::regclass)" 2>&1 | head -1)"
  } >> "$f"
  q "DROP ROLE IF EXISTS mon; DROP ROLE IF EXISTS nomon;" > /dev/null
  cat "$f" >&2
}

# ----------------------------------------------------------------- race -----
# Two shapes of interference, both driven from a second session:
#   lock_timeout   an uncommitted DROP INDEX holds AccessExclusiveLock and the
#                  call is cancelled at the timeout
#   committed drop the same drop commits while the report waits, so
#                  relation_open finds nothing behind the lock
stage_race() {
  say "locking and the concurrent-drop race"
  : > "$OUT/race17.txt"
  local f="$OUT/race17.txt" a b msg
  "$BIN/psql" -X -q -d "$DB" -c \
    "BEGIN; DROP INDEX bl.i_del50; SELECT pg_sleep(8); ROLLBACK;" > /dev/null 2>&1 &
  local holder=$!
  sleep 2
  a=$(date +%s%N)
  msg=$(err "SET lock_timeout = '2s'; SELECT * FROM pgstatindex('bl.i_del50'::regclass)")
  b=$(date +%s%N)
  printf 'lock_timeout_error=%s\n' "$msg" >> "$f"
  printf 'lock_timeout_ms=%s.%s\n' $(( (b - a) / 1000000 )) $(( ((b - a) / 100000) % 10 )) >> "$f"
  wait "$holder" 2>/dev/null

  # Case 1: the drop commits while cand is still being materialized.  cand
  # sizes each candidate with pg_relation_size, which opens the relation with
  # try_relation_open and returns NULL when it has gone, so the row is
  # filtered out and the report survives, one row shorter.
  local before after
  before=$(( $("$BIN/psql" -X -q -A -F '|' -P footer=off -d "$DB" -f "$SQLD/report.sql" 2>/dev/null | grep -c '^') - 1 ))
  "$BIN/psql" -X -q -d "$DB" -c \
    "BEGIN; DROP INDEX bl.i_uniq; SELECT pg_sleep(3); COMMIT;" > /dev/null 2>&1 &
  local dropper=$!
  sleep 1
  a=$(date +%s%N)
  # psql prefixes an error from -f with "psql:<file>:<line>: ", so the match
  # cannot be anchored at the start of the line; the prefix is then cut.
  "$BIN/psql" -X -q -A -F '|' -P footer=off -d "$DB" -f "$SQLD/report.sql" \
    > "$OUT/race_during.txt" 2>&1
  b=$(date +%s%N)
  wait "$dropper" 2>/dev/null
  msg=$(grep -E '(ERROR|FATAL):' "$OUT/race_during.txt" | head -1 | sed 's/^psql:[^ ]* //')
  after=$(( $(grep -c '^' "$OUT/race_during.txt") - 1 ))
  printf 'drop_during_cand_error=%s\n' "${msg:-none}" >> "$f"
  printf 'drop_during_cand_rows=%s (was %s)\n' "$after" "$before" >> "$f"
  printf 'drop_during_cand_ms=%s\n' $(( (b - a) / 1000000 )) >> "$f"
  q "CREATE UNIQUE INDEX IF NOT EXISTS i_uniq ON bl.t_uniq (id);" > /dev/null 2>&1

  # Case 2: the drop lands after cand has sized that index and released its
  # lock, but before pgstatindex opens it.  That window is short, so the
  # delay is swept until the abort appears.
  # The target is the fixture index cand materializes last, so the window
  # between its size check and its pgstatindex call is the whole loop rather
  # than a few milliseconds.  The dropping session connects first and holds
  # the delay inside the server, as pg_sleep in the same command, so psql's
  # start-up cost is out of the critical path.
  local target defn d attempts=0 hit=
  target=$(s "SELECT index_name FROM (SELECT index_name, row_number() OVER () AS rn
                                        FROM bloat_final
                                       WHERE schema_name = 'bl'
                                         AND index_size >= 1024 * 1024) q
               ORDER BY rn DESC LIMIT 1")
  defn=$(s "SELECT pg_get_indexdef('bl.$target'::regclass)")
  # The reachable window is (end of cand, pgstatindex on the target).  On a
  # warm cache the whole report is under 100 ms, so that window sits inside
  # psql's own start-up skew; the sweep is recorded whether or not it lands.
  printf 'drop_after_cand_target=%s\n' "$target" >> "$f"
  for d in 0.005 0.02 0.04 0.06 0.07 0.075 0.08 0.09 0.10 0.12; do
    a=$(date +%s%N)
    "$BIN/psql" -X -q -d "$DB" \
      -c "SELECT pg_sleep($d)" \
      -c "BEGIN; DROP INDEX bl.$target; SELECT pg_sleep(2); COMMIT;" > /dev/null 2>&1 &
    local dp=$!
    "$BIN/psql" -X -q -d "$DB" -f "$SQLD/report.sql" > "$OUT/race_after.txt" 2>&1
    b=$(date +%s%N)
    wait "$dp" 2>/dev/null
    attempts=$((attempts + 1))
    msg=$(grep -E '(ERROR|FATAL):' "$OUT/race_after.txt" | head -1 | sed 's/^psql:[^ ]* //')
    q "$defn" > /dev/null 2>&1
    case $msg in
      *"could not open relation"*)
        hit="$msg after $(( (b - a) / 1000000 )) ms, drop taken ${d}s in"; break ;;
    esac
  done
  printf 'drop_after_cand_error=%s\n' "${hit:-not reproduced in $attempts attempts}" >> "$f"
  printf 'drop_after_cand_attempts=%s\n' "$attempts" >> "$f"
  q "CREATE UNIQUE INDEX IF NOT EXISTS i_uniq ON bl.t_uniq (id);" > /dev/null 2>&1
  cat "$f" >&2
}

# -------------------------------------------------------------- summary -----
stage_summary() {
  say "what landed in $OUT"
  ls -la "$OUT" >&2
  local x
  for x in platform17 exact17 facts17 cost17 priv17 race17; do
    [ -f "$OUT/$x.txt" ] && { printf '\n---- %s\n' "$x" >&2; cat "$OUT/$x.txt" >&2; }
  done
  [ -f "$OUT/scores17.txt" ] && { printf '\n---- scores17 (tail)\n' >&2; tail -14 "$OUT/scores17.txt" >&2; }
  [ -f "$OUT/residual17.txt" ] && { printf '\n---- residual17\n' >&2; cat "$OUT/residual17.txt" >&2; }
}

# ---------------------------------------------------------------- stop -------
stage_stop() {
  say "stop the 17 cluster"
  [ -d "$DATA" ] || { note "no data directory"; return 0; }
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  fi
  [ -f "$DATA/postmaster.pid" ] && die "postmaster.pid still present in $DATA"
  pgrep -f "postgres.*-D $DATA" > /dev/null 2>&1 && die "a postgres process still runs on $DATA"
  [ -n "$(ls -A "$SOCK" 2>/dev/null)" ] && die "socket directory $SOCK is not empty"
  note "stopped: no postmaster.pid, no process, empty socket directory"
}

stage_clean() {
  stage_stop || exit 1
  case "$SANDBOX" in
    "$WIKI_ROOT"/.wiki-runtime/tmp/?*) ;;
    *) die "refusing to delete $SANDBOX: not under $WIKI_ROOT/.wiki-runtime/tmp/" ;;
  esac
  say "delete $SANDBOX"
  rm -rf "$SANDBOX"
}

# ------------------------------------------------------------ dispatcher -----
STAGES_DEFAULT="build check cluster texts fixtures report facts cost priv score residual race summary"
run_stage() {
  case "$1" in
    build|check|cluster|texts|fixtures|report|facts|score|residual|cost|priv|race|summary|stop|clean)
      "stage_$1" ;;
    *) die "unknown stage: $1" ;;
  esac
}
main() {
  local st
  if [ "$#" -eq 0 ]; then set -- $STAGES_DEFAULT; fi
  for st in "$@"; do run_stage "$st" || die "stage $st failed"; done
  say "done: $*"
}
main "$@"
```

### The PostgreSQL 12 leg script

```sh
#!/usr/bin/env bash
#
# bloat_pgstatindex_v12.sh - the PostgreSQL 12 leg of the measurement behind
# "B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and
# 17".  It answers one question the 17 leg cannot: does the exact filed text
# run unmodified on the oldest major it claims, and does it report the same
# numbers there?
#
# Nothing here assumes what PostgreSQL 12 does.  The statement is executed as
# filed, byte for byte, and the outcome is recorded as a result in its own
# right.  The pinned checkout is read only; everything this script writes
# lives under $SANDBOX.
#
# Usage, from the repository root:
#   bash bloat_pgstatindex_v12.sh                 # every stage, in order
#   bash bloat_pgstatindex_v12.sh exact           # just the parse result
#   bash bloat_pgstatindex_v12.sh clean           # stop and delete this leg
#
# Stages: build check cluster texts fixtures report facts score residual cost
#         priv race summary stop clean
#
# Environment: WIKI_ROOT PAGE SRC12 SANDBOX PORT12 JOBS ROWS
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
PAGE="${PAGE:-$WIKI_ROOT/wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md}"
SRC12="${SRC12:-$WIKI_ROOT/raw/postgres-12}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/pgsi}"
PORT12="${PORT12:-55412}"
JOBS="${JOBS:-4}"
ROWS="${ROWS:-1000000}"
OLD_REV="${OLD_REV:-cbbbd16}"

BUILD="$SANDBOX/build12"; INST="$SANDBOX/install12"; DATA="$SANDBOX/data12"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql12"; SOCK="$SANDBOX/sock12"; BIN="$INST/bin"
DB=bloat12
export PGPORT="$PORT12" PGHOST="$SOCK" PGDATABASE=postgres

BASE_SQL=9d2e3a2c73c81f3efb848b61f4bf307365ce0da56d780baee08fa9a9606f6c91
BASE_PREV=f5b995d3c5d51dddd1378e4e1ac31f9ad180cec0cc811a803186b86a1fb721e9

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

q()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }
s()  { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }
t()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" -c "$1"; }
fl() { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -f "$1"; }
err() { "$BIN/psql" -X -At -q -d "$DB" -c "$1" 2>&1 | grep -E '^(ERROR|FATAL)' | head -1; }

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

# ---------------------------------------------------------------- build ------
# 12.2 is configured without ICU: this page's fixtures need no collation
# provider, and --with-icu is opt-in on this major, so leaving it off avoids
# the TRUE/FALSE macros ICU 68 removed.
stage_build() {
  say "build 12.2 out of tree from $SRC12"
  [ -x "$BIN/postgres" ] && { note "already built, skipping"; return 0; }
  [ -x "$SRC12/configure" ] || die "no pinned checkout at $SRC12; set SRC12 or run from the repository root"
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" --enable-debug \
      --with-readline --with-zlib > configure.log 2>&1 ) \
    || { cp "$BUILD/configure.log" "$OUT/configure12.log" 2>/dev/null; die "configure failed, see $OUT/configure12.log"; }
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { cp "$BUILD"/*.log "$OUT/" 2>/dev/null; grep -m3 'error:' "$BUILD/make.log" >&2; die "make failed"; }
  ( cd "$BUILD" && make -C contrib/pgstattuple -j"$JOBS" >> install.log 2>&1 \
      && make -C contrib/pgstattuple install >> install.log 2>&1 ) || die "contrib/pgstattuple failed"
  local l
  for l in configure make install; do cp "$BUILD/$l.log" "$OUT/${l}12.log" 2>/dev/null; done
  note "$("$BIN/postgres" --version)"
}

stage_check() {
  say "regression suites, 12.2"
  mkdir -p "$OUT"
  : > "$OUT/checks12.txt"
  ( cd "$BUILD" && make check > check_core.log 2>&1 )
  printf 'core=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_core.log" | tail -1)" \
    >> "$OUT/checks12.txt"
  ( cd "$BUILD" && make -C contrib/pgstattuple check > check_pgstattuple.log 2>&1 )
  printf 'pgstattuple=%s %s\n' "$?" \
    "$(grep -Eo 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests (passed|failed)' "$BUILD/check_pgstattuple.log" | tail -1)" \
    >> "$OUT/checks12.txt"
  local l d
  for l in "$BUILD"/check_*.log; do [ -f "$l" ] && cp "$l" "$OUT/12_$(basename "$l")"; done
  for d in "$BUILD"/src/test/regress/regression.diffs "$BUILD"/contrib/pgstattuple/regression.diffs; do
    [ -f "$d" ] && cp "$d" "$OUT/diffs12_$(basename "$(dirname "$d")").txt"
  done
  cat "$OUT/checks12.txt" >&2
}

# -------------------------------------------------------------- cluster ------
# Same settings as the 17 leg.  Their apply scopes are read from the v17 GUC
# table on the page, which is a v17 page; the 12 contexts are not cited there.
stage_cluster() {
  say "isolated 12.2 cluster on port $PORT12"
  mkdir -p "$OUT" "$SQLD" "$SOCK"
  if [ ! -f "$DATA/PG_VERSION" ]; then
    "$BIN/initdb" -D "$DATA" --locale=C --encoding=UTF8 > "$OUT/initdb12.log" 2>&1 \
      || die "initdb failed, see $OUT/initdb12.log"
    cat >> "$DATA/postgresql.conf" <<CONF
listen_addresses = ''
port = $PORT12
unix_socket_directories = '$SOCK'
shared_buffers = 512MB
maintenance_work_mem = 256MB
max_parallel_maintenance_workers = 0
autovacuum = off
fsync = off
logging_collector = off
CONF
  fi
  if ! "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server12.log" -w start > /dev/null 2>&1 \
      || die "server did not start, see $OUT/server12.log"
  fi
  "$BIN/psql" -X -At -q -d postgres -c "SELECT 1" > /dev/null 2>&1 || die "cannot connect"
  "$BIN/psql" -X -At -q -d postgres -c \
    "SELECT count(*) FROM pg_database WHERE datname = '$DB'" | grep -q '^1$' \
    || "$BIN/createdb" "$DB" || die "createdb failed"
  q "CREATE EXTENSION IF NOT EXISTS pgstattuple;" > /dev/null || die "pgstattuple not installed"
  {
    printf 'uname=%s\n' "$(uname -sm)"
    printf 'server_version_num=%s\n' "$(s 'SHOW server_version_num')"
    printf 'server_version=%s\n' "$(s 'SHOW server_version')"
    printf 'block_size=%s\n' "$(s 'SHOW block_size')"
    printf 'max_data_alignment=%s\n' "$("$BIN/pg_controldata" -D "$DATA" | grep -i 'maximum data alignment' | tr -s ' ' | cut -d' ' -f4)"
    printf 'pgstattuple=%s\n' "$(s "SELECT extversion FROM pg_extension WHERE extname = 'pgstattuple'")"
  } > "$OUT/platform12.txt"
  cat "$OUT/platform12.txt" >&2
}

# ---------------------------------------------------------------- texts -----
# Two texts come out of the page and one out of git history:
#   report.sql   the filed statement, byte for byte, hash-checked
#   view.sql     the same text as a view over the internal `final` stage, with
#                exactly two edits, both printed: the two SET lines dropped and
#                min_index_bytes set to 0 so sub-megabyte indexes are scored
#   prev.sql     the text filed before wasted space was rebased on the
#                fillfactor, recovered from OLD_REV for the cost comparison
gen_view() {                       # gen_view <view> < text
  local view=$1 line tail=0
  printf 'DROP VIEW IF EXISTS %s CASCADE;\nCREATE VIEW %s AS\n' "$view" "$view"
  while IFS= read -r line; do
    case $line in
      "SET statement_timeout"*|"SET lock_timeout"*)
        printf '   harness edit: dropped %s\n' "$line" >&2; continue ;;
      *"AS min_index_bytes"*)
        printf '   harness edit: %s -> 0\n' "$(printf '%s' "$line" | tr -s ' ')" >&2
        printf '           0::bigint AS min_index_bytes  -- harness: score every index\n'
        continue ;;
      "SELECT /* wiki_btree_bloat_pgstatindex_12_17 */") tail=1; continue ;;
    esac
    [ "$tail" = 1 ] && continue
    printf '%s\n' "$line"
  done
  printf 'SELECT f.* FROM final f;\n'
}

stage_texts() {
  say "extract the statement from $PAGE"
  mkdir -p "$SQLD" "$OUT"
  [ -f "$PAGE" ] || die "no page at $PAGE; set PAGE or run from the repository root"
  md_block sql 1 "$PAGE" > "$SQLD/report.sql"
  local got
  got=$(sha256sum < "$SQLD/report.sql" | cut -d' ' -f1)
  [ "$got" = "$BASE_SQL" ] || die "sql block 1 hashes $got, expected $BASE_SQL"
  note "filed text: $(wc -l < "$SQLD/report.sql") lines, $(wc -c < "$SQLD/report.sql") bytes, sha256 ${got:0:12}"
  gen_view bloat_final < "$SQLD/report.sql" > "$SQLD/view.sql"
  # The statement alone, without the two SET lines, for EXPLAIN.
  grep -v '^SET ' "$SQLD/report.sql" > "$SQLD/bare.sql"
  if git -C "$WIKI_ROOT" cat-file -e "$OLD_REV:wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md" 2>/dev/null; then
    git -C "$WIKI_ROOT" show "$OLD_REV:wiki/v17/questions/indexing/btree-bloat-with-pgstatindex.md" > "$SQLD/page_prev.md"
    md_block sql 1 "$SQLD/page_prev.md" > "$SQLD/prev.sql"
    got=$(sha256sum < "$SQLD/prev.sql" | cut -d' ' -f1)
    [ "$got" = "$BASE_PREV" ] || die "superseded text hashes $got, expected $BASE_PREV"
    grep -v '^SET ' "$SQLD/prev.sql" > "$SQLD/prev_bare.sql"
    note "superseded text: $(wc -l < "$SQLD/prev.sql") lines, $(wc -c < "$SQLD/prev.sql") bytes, sha256 ${got:0:12}"
  else
    note "no revision $OLD_REV in this repository; the cost comparison will be skipped"
  fi
}

# ------------------------------------------------------------- fixtures -----
# Every statement in this stage is DISPOSABLE.  It drops and rebuilds schema
# bl in a throwaway database and is not meant for a database anyone cares
# about.  The two session GUCs it sets are PGC_USERSET: session scope, no
# reload and no restart.
stage_fixtures() {
  say "build the fixture suite in $DB (rows=$ROWS)"
  cat > "$SQLD/fixtures.sql" <<'SQL'
-- DISPOSABLE fixture suite for the pgstatindex bloat report.
-- Every object lives in schema bl of a throwaway database.
SET statement_timeout = '30min';   -- PGC_USERSET, session scope
SET lock_timeout      = '30s';     -- PGC_USERSET, session scope
SET client_min_messages = warning;

DROP SCHEMA IF EXISTS bl CASCADE;
CREATE SCHEMA bl;

-- 1. scattered deletes at five fractions, all vacuumed afterwards ----------
CREATE TABLE bl.t_del90 (id int);
INSERT INTO bl.t_del90 SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_del90 ON bl.t_del90 (id);
DELETE FROM bl.t_del90 WHERE id % 10 <> 0;
VACUUM bl.t_del90;

CREATE TABLE bl.t_del50 (id int);
INSERT INTO bl.t_del50 SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_del50 ON bl.t_del50 (id);
DELETE FROM bl.t_del50 WHERE id % 2 <> 0;
VACUUM bl.t_del50;

-- 2. the same shape, deliberately not vacuumed -----------------------------
CREATE TABLE bl.t_novac (id int);
INSERT INTO bl.t_novac SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_novac ON bl.t_novac (id);
DELETE FROM bl.t_novac WHERE id % 10 <> 0;

-- 3. a contiguous head deleted: whole pages empty out ----------------------
CREATE TABLE bl.t_delhead (id int);
INSERT INTO bl.t_delhead SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_delhead ON bl.t_delhead (id);
DELETE FROM bl.t_delhead WHERE id <= (:rows * 7) / 10;
VACUUM bl.t_delhead;

-- 4. partial, unique, expression, INCLUDE, multicolumn, text, wide ---------
CREATE TABLE bl.t_partial (id int, flag boolean);
INSERT INTO bl.t_partial SELECT g, true FROM generate_series(1, :rows) g;
CREATE INDEX i_partial ON bl.t_partial (id) WHERE flag;
DELETE FROM bl.t_partial WHERE flag AND id % 10 <> 0;
VACUUM bl.t_partial;

CREATE TABLE bl.t_uniq (id int);
INSERT INTO bl.t_uniq SELECT g FROM generate_series(1, :rows) g;
CREATE UNIQUE INDEX i_uniq ON bl.t_uniq (id);
DELETE FROM bl.t_uniq WHERE id % 8 <> 0;
VACUUM bl.t_uniq;

CREATE TABLE bl.t_expr (id int);
INSERT INTO bl.t_expr SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_expr ON bl.t_expr ((id * 2));
DELETE FROM bl.t_expr WHERE id % 5 <> 0;
VACUUM bl.t_expr;

CREATE TABLE bl.t_incl (id int, v int);
INSERT INTO bl.t_incl SELECT g, g FROM generate_series(1, :rows) g;
CREATE INDEX i_incl ON bl.t_incl (id) INCLUDE (v);
DELETE FROM bl.t_incl WHERE id % 4 <> 0;
VACUUM bl.t_incl;

CREATE TABLE bl.t_multi (a int, b int, c int);
INSERT INTO bl.t_multi SELECT g, g, g FROM generate_series(1, :rows) g;
CREATE INDEX i_multi ON bl.t_multi (a, b, c);
DELETE FROM bl.t_multi WHERE a % 3 <> 0;
VACUUM bl.t_multi;

CREATE TABLE bl.t_text_del (id int, v text);
INSERT INTO bl.t_text_del SELECT g, md5(g::text) FROM generate_series(1, :rows) g;
CREATE INDEX i_text_del ON bl.t_text_del (v);
DELETE FROM bl.t_text_del WHERE id % 5 <> 0;
VACUUM bl.t_text_del;

CREATE TABLE bl.t_wide (id int, v text);
INSERT INTO bl.t_wide SELECT g, rpad(md5(g::text), 400, 'w') FROM generate_series(1, :wide_rows) g;
CREATE INDEX i_wide ON bl.t_wide (v);
DELETE FROM bl.t_wide WHERE id % 10 >= 3;
VACUUM bl.t_wide;

-- 5. unlogged, partitioned, TOAST-owning, empty ----------------------------
CREATE UNLOGGED TABLE bl.t_unlogged (id int);
INSERT INTO bl.t_unlogged SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_unlogged ON bl.t_unlogged (id);
DELETE FROM bl.t_unlogged WHERE id % 6 <> 0;
VACUUM bl.t_unlogged;

CREATE TABLE bl.t_part (id int) PARTITION BY RANGE (id);
CREATE TABLE bl.t_part_1 PARTITION OF bl.t_part FOR VALUES FROM (1) TO (500001);
CREATE TABLE bl.t_part_2 PARTITION OF bl.t_part FOR VALUES FROM (500001) TO (1000001);
INSERT INTO bl.t_part SELECT g FROM generate_series(1, 1000000) g;
CREATE INDEX i_part ON bl.t_part (id);
DELETE FROM bl.t_part WHERE id % 7 <> 0;
VACUUM bl.t_part_1;
VACUUM bl.t_part_2;

CREATE TABLE bl.t_toast (id int PRIMARY KEY, v text);
ALTER TABLE bl.t_toast ALTER COLUMN v SET STORAGE EXTERNAL;
INSERT INTO bl.t_toast SELECT g, rpad(md5(g::text), 4000, 'p') FROM generate_series(1, 20000) g;
DELETE FROM bl.t_toast WHERE id % 3 <> 0;
VACUUM bl.t_toast;

CREATE TABLE bl.t_empty (id int PRIMARY KEY);

-- 6. churn and reverse-order inserts ---------------------------------------
CREATE TABLE bl.t_churn (id int);
INSERT INTO bl.t_churn SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_churn ON bl.t_churn (id);
UPDATE bl.t_churn SET id = id + :rows;
VACUUM bl.t_churn;

CREATE TABLE bl.t_frag (id int);
CREATE INDEX i_frag ON bl.t_frag (id);
INSERT INTO bl.t_frag SELECT g FROM generate_series(:rows, 1, -1) g;

-- 7. fresh builds at four fillfactors, plus a fresh text index -------------
CREATE TABLE bl.t_fresh (id int);
INSERT INTO bl.t_fresh SELECT g FROM generate_series(1, :rows) g;
CREATE INDEX i_fresh  ON bl.t_fresh (id);
CREATE INDEX i_ff100  ON bl.t_fresh (id) WITH (fillfactor = 100);
CREATE INDEX i_ff50   ON bl.t_fresh (id) WITH (fillfactor = 50);
CREATE INDEX i_ff10   ON bl.t_fresh (id) WITH (fillfactor = 10);

CREATE TABLE bl.t_text (v text);
INSERT INTO bl.t_text SELECT md5(g::text) FROM generate_series(1, :rows) g;
CREATE INDEX i_text ON bl.t_text (v);

-- 8. duplicates: built by CREATE INDEX, and built by inserts ---------------
CREATE TABLE bl.t_dup (v int);
INSERT INTO bl.t_dup SELECT g % 10 FROM generate_series(1, :rows) g;
CREATE INDEX i_dup ON bl.t_dup (v);

CREATE TABLE bl.t_dup_ins (v int);
CREATE INDEX i_dup_ins ON bl.t_dup_ins (v);
INSERT INTO bl.t_dup_ins SELECT g % 10 FROM generate_series(1, :rows) g;

-- 9. deduplication turned off at build time, then turned back on.
-- The reloption does not exist before PostgreSQL 13, so this fixture is
-- built only where the server accepts it; the 12 leg records its absence.
DO $dd$
BEGIN
  IF current_setting('server_version_num')::int >= 130000 THEN
    EXECUTE 'CREATE TABLE bl.t_dedup (v int)';
    EXECUTE 'INSERT INTO bl.t_dedup SELECT g % 10 FROM generate_series(1, ' ||
            current_setting('bl.rows') || ') g';
    EXECUTE 'CREATE INDEX i_dedup_off ON bl.t_dedup (v) WITH (deduplicate_items = off)';
    EXECUTE 'ALTER INDEX bl.i_dedup_off SET (deduplicate_items = on)';
  END IF;
END
$dd$;

-- 10. the four fillfactor fixtures: built at a stated fillfactor, then nine
-- tenths of the rows deleted and the table vacuumed ------------------------
CREATE TABLE bl.t_ffdel (id int);
INSERT INTO bl.t_ffdel SELECT g FROM generate_series(1, :ff_rows) g;
CREATE INDEX i_ff100_del90 ON bl.t_ffdel (id) WITH (fillfactor = 100);
CREATE INDEX i_ff50_del90  ON bl.t_ffdel (id) WITH (fillfactor = 50);
CREATE INDEX i_ff10_del90  ON bl.t_ffdel (id) WITH (fillfactor = 10);
DELETE FROM bl.t_ffdel WHERE id % 10 <> 0;
VACUUM bl.t_ffdel;

CREATE TABLE bl.t_ffhead (id int);
INSERT INTO bl.t_ffhead SELECT g FROM generate_series(1, :ff_rows) g;
CREATE INDEX i_ff50_delhead ON bl.t_ffhead (id) WITH (fillfactor = 50);
DELETE FROM bl.t_ffhead WHERE id <= (:ff_rows * 9) / 10;
VACUUM bl.t_ffhead;

-- 11. two known-content pages, for the implied leaf capacity ---------------
CREATE TABLE bl.c_one (id int);
INSERT INTO bl.c_one VALUES (1);
CREATE INDEX c_one_idx ON bl.c_one (id);

CREATE TABLE bl.c_zero (id int);
INSERT INTO bl.c_zero VALUES (1);
CREATE INDEX c_zero_idx ON bl.c_zero (id);
DELETE FROM bl.c_zero;
VACUUM bl.c_zero;

-- 12. shapes pgstatindex refuses -------------------------------------------
CREATE TABLE bl.s_other (id int, v text, g_point point, r int4range);
INSERT INTO bl.s_other SELECT g, md5(g::text), point(g, g), int4range(g, g + 10)
  FROM generate_series(1, 20000) g;
CREATE INDEX s_hash   ON bl.s_other USING hash   (id);
CREATE INDEX s_gin    ON bl.s_other USING gin    (to_tsvector('simple', v));
CREATE INDEX s_gist   ON bl.s_other USING gist   (g_point);
CREATE INDEX s_spgist ON bl.s_other USING spgist (g_point);
CREATE INDEX s_brin   ON bl.s_other USING brin   (id);
CREATE VIEW  bl.s_view AS SELECT 1 AS one;
CREATE SEQUENCE bl.s_seq;

-- 13. an index the catalog says is not valid.  Only a disposable cluster
-- may have its catalog written to by hand like this.
CREATE TABLE bl.t_invalid (id int);
INSERT INTO bl.t_invalid SELECT g FROM generate_series(1, 300000) g;
CREATE INDEX i_invalid ON bl.t_invalid (id);
UPDATE pg_index SET indisvalid = false WHERE indexrelid = 'bl.i_invalid'::regclass;
SQL
  local ffrows=$((ROWS / 5)) widerows=$((ROWS / 10))
  # psql does not substitute :variables inside a dollar-quoted body, so the
  # version-guarded fixture reads its row count from a database-level GUC.
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" \
    -c "ALTER DATABASE $DB SET bl.rows = '$ROWS'" > /dev/null || die "cannot set bl.rows"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" \
    -v rows="$ROWS" -v ff_rows="$ffrows" -v wide_rows="$widerows" \
    -f "$SQLD/fixtures.sql" || die "fixtures failed"
  note "indexes in bl: $(s "SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace WHERE n.nspname = 'bl' AND c.relkind IN ('i','I')")"
  note "database size: $(s "SELECT pg_size_pretty(pg_database_size(current_database()))")"
}

# --------------------------------------------------------------- report -----
# Runs the filed text exactly as filed, including its two SET lines.
stage_report() {
  say "run the filed statement"
  [ -f "$SQLD/report.sql" ] || die "run the texts stage first"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -P footer=off -A -F '|' -d "$DB" \
    -f "$SQLD/report.sql" > "$OUT/report12.txt" 2> "$OUT/report12.err"
  local rc=$?
  if [ "$rc" != 0 ]; then
    printf 'exact_text=refused\n' > "$OUT/exact12.txt"
    head -5 "$OUT/report12.err" >> "$OUT/exact12.txt"
    cat "$OUT/exact12.txt" >&2
    die "the filed text did not run; see $OUT/report12.err"
  fi
  printf 'exact_text=executes\n' > "$OUT/exact12.txt"
  # footer=off leaves one header line and one line per row.
  local rows cols
  rows=$(grep -c '^' "$OUT/report12.txt")
  rows=$((rows - 1))
  cols=$(head -1 "$OUT/report12.txt" | tr '|' '\n' | grep -c '^')
  {
    printf 'report_rows=%s\n' "$rows"
    printf 'report_columns=%s\n' "$cols"
    printf 'report_bytes=%s\n' "$(wc -c < "$OUT/report12.txt")"
  } >> "$OUT/exact12.txt"
  cat "$OUT/exact12.txt" >&2
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" \
    -f "$SQLD/report.sql" > "$OUT/report12_pretty.txt" 2>&1
}

# ---------------------------------------------------------------- facts -----
stage_facts() {
  say "version-local facts, refusals and page arithmetic"
  fl "$SQLD/view.sql" > /dev/null || die "harness view failed"
  : > "$OUT/facts12.txt"
  local f="$OUT/facts12.txt"
  {
    printf 'server_version_num=%s\n' "$(s 'SHOW server_version_num')"
    printf 'block_size=%s\n' "$(s 'SHOW block_size')"
    printf 'candidates=%s\n' "$(s 'SELECT count(*) FROM bloat_final')"
    printf 'index_size_equals_relation_size=%s of %s\n' \
      "$(s 'SELECT count(*) FROM bloat_final f WHERE f.index_size = pg_relation_size(f.idx_oid)')" \
      "$(s 'SELECT count(*) FROM bloat_final')"
    printf 'nan_density_indexes=%s\n' "$(s 'SELECT count(*) FROM bloat_final WHERE leaf_pages = 0')"
    printf 'nan_float8_gt_20=%s\n' "$(s "SELECT ('NaN'::float8 > 20)::text")"
    printf 'nan_numeric_gt_20=%s\n' "$(s "SELECT ('NaN'::numeric > 20)::text")"
    printf 'dedup_fixture_present=%s\n' "$(s "SELECT (to_regclass('bl.i_dedup_off') IS NOT NULL)::text")"
  } >> "$f"

  # Every refusal the cand filters exist for.
  {
    printf 'err_hash=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_hash'::regclass)")"
    printf 'err_gin=%s\n'       "$(err "SELECT * FROM pgstatindex('bl.s_gin'::regclass)")"
    printf 'err_gist=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_gist'::regclass)")"
    printf 'err_spgist=%s\n'    "$(err "SELECT * FROM pgstatindex('bl.s_spgist'::regclass)")"
    printf 'err_brin=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_brin'::regclass)")"
    printf 'err_partitioned=%s\n' "$(err "SELECT * FROM pgstatindex('bl.i_part'::regclass)")"
    printf 'err_table=%s\n'     "$(err "SELECT * FROM pgstatindex('bl.t_del90'::regclass)")"
    printf 'err_view=%s\n'      "$(err "SELECT * FROM pgstatindex('bl.s_view'::regclass)")"
    printf 'err_sequence=%s\n'  "$(err "SELECT * FROM pgstatindex('bl.s_seq'::regclass)")"
    printf 'err_stale_oid=%s\n' "$(err "SELECT * FROM pgstatindex(2147483647::oid::regclass)")"
  } >> "$f"

  # The invalid index: an error on 17, a row on 12.
  local inv
  inv=$(err "SELECT * FROM pgstatindex('bl.i_invalid'::regclass)")
  if [ -n "$inv" ]; then
    printf 'invalid_index=refused %s\n' "$inv" >> "$f"
  else
    printf 'invalid_index=row %s\n' \
      "$(s "SELECT version || '|' || tree_level || '|' || index_size || '|' || root_block_no || '|' || internal_pages || '|' || leaf_pages || '|' || empty_pages || '|' || deleted_pages || '|' || avg_leaf_density || '|' || leaf_fragmentation FROM pgstatindex('bl.i_invalid'::regclass)")" >> "$f"
  fi

  # Fresh-build density at four fillfactors, and the reported estimate.
  printf 'fresh_builds fillfactor|target_density|avg_leaf_density|est_reclaimable_pct\n' >> "$f"
  s "SELECT f.fillfactor || '|' || round(100 * f.target_density, 2) || '|' ||
            round(f.avg_leaf_density::numeric, 2) || '|' ||
            round(100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size, 1)
       FROM bloat_final f
      WHERE f.index_name IN ('i_ff100', 'i_fresh', 'i_ff50', 'i_ff10')
      ORDER BY f.fillfactor DESC" >> "$f"

  # Leaf capacity implied by two pages of known contents.  A root leaf page
  # holding one int4 entry leaves 8192 - 16 special - 24 header - 16 tuple
  # - 4 line pointer = 8128 free; an empty leaf page leaves 8148.
  printf 'implied_max_avail one_tuple|empty_leaf\n' >> "$f"
  s "SELECT round(8128 / (1 - one.d / 100)::numeric, 1) || '|' ||
            round(8148 / (1 - zero.d / 100)::numeric, 1)
       FROM (SELECT avg_leaf_density::numeric AS d FROM pgstatindex('bl.c_one_idx'::regclass)) one,
            (SELECT avg_leaf_density::numeric AS d FROM pgstatindex('bl.c_zero_idx'::regclass)) zero" >> "$f"
  printf 'known_page_densities one|zero\n' >> "$f"
  s "SELECT (SELECT round(avg_leaf_density::numeric, 2) FROM pgstatindex('bl.c_one_idx'::regclass)) || '|' ||
            (SELECT round(avg_leaf_density::numeric, 2) FROM pgstatindex('bl.c_zero_idx'::regclass))" >> "$f"

  # VACUUM does not give index pages back; REINDEX does.
  printf 'delhead_after_vacuum=%s dead_pages=%s\n' \
    "$(s "SELECT index_size FROM pgstatindex('bl.i_delhead'::regclass)")" \
    "$(s "SELECT empty_pages + deleted_pages FROM pgstatindex('bl.i_delhead'::regclass)")" >> "$f"

  # Another session's temp index, with and without the filter.  Each -c is
  # its own transaction, so the temp relation is committed and visible to
  # this session while the sleeping one still owns it; a single multi-
  # statement -c would keep the catalog rows uncommitted and invisible.
  "$BIN/psql" -X -q -d "$DB" \
    -c "CREATE TEMP TABLE tmp_other(id int)" \
    -c "INSERT INTO tmp_other SELECT g FROM generate_series(1, 300000) g" \
    -c "CREATE INDEX tmp_other_idx ON tmp_other(id)" \
    -c "SELECT pg_sleep(25)" > /dev/null 2>&1 &
  local other=$!
  sleep 8
  {
    printf 'other_temp_candidates_filtered=%s\n' "$(s 'SELECT count(*) FROM bloat_final')"
    printf 'other_temp_candidates_unfiltered=%s\n' \
      "$(s "SELECT count(*) FROM pg_class c JOIN pg_index x ON x.indexrelid = c.oid
              JOIN pg_am a ON a.oid = c.relam
             WHERE a.amname = 'btree' AND c.relkind = 'i'
               AND x.indisvalid AND x.indisready AND x.indislive")"
    printf 'other_temp_index_size=%s\n' \
      "$(s "SELECT coalesce(pg_size_pretty(max(pg_relation_size(c.oid))), 'none')
              FROM pg_class c WHERE c.relname = 'tmp_other_idx' AND c.relkind = 'i'")"
    printf 'other_temp_error=%s\n' \
      "$(err "SELECT * FROM pgstatindex((SELECT c.oid FROM pg_class c WHERE c.relname = 'tmp_other_idx' AND c.relkind = 'i' LIMIT 1)::regclass)")"
    printf 'report_rows_with_other_session=%s\n' \
      "$(( $("$BIN/psql" -X -q -A -F '|' -P footer=off -v ON_ERROR_STOP=1 -d "$DB" -f "$SQLD/report.sql" 2>/dev/null | grep -c '^') - 1 ))"
  } >> "$f"
  wait "$other" 2>/dev/null
  cat "$f" >&2
}

# ---------------------------------------------------------------- score -----
# Ground truth is pg_relation_size before and after REINDEX INDEX, over every
# index the harness view can see.  This stage is destructive: it rebuilds
# every index in the database, so it must run after report, facts and cost.
stage_score() {
  say "score every estimate against a measured REINDEX INDEX"
  fl "$SQLD/view.sql" > /dev/null || die "harness view failed"
  q "DROP VIEW IF EXISTS bl_scored;
     DROP TABLE IF EXISTS bl_before, bl_after, bl_residual;
     CREATE TABLE bl_before AS
       SELECT /* wiki_pgsi_score_before */
              f.idx_oid, f.schema_name, f.index_name, f.index_size,
              f.est_rebuilt_bytes, f.wasted_vs_fillfactor, f.avg_leaf_density,
              f.leaf_pages, f.dead_pages, f.fillfactor, f.target_density,
              f.leaf_bytes, f.live_leaf_bytes, f.dead_bytes,
              pg_relation_size(f.idx_oid) AS before_bytes
         FROM bloat_final f;" > /dev/null || die "bl_before failed"
  note "indexes to score: $(s 'SELECT count(*) FROM bl_before')"
  s "SELECT 'REINDEX INDEX ' || idx_oid::regclass || ';' FROM bl_before ORDER BY idx_oid" \
    > "$SQLD/reindex.sql"
  fl "$SQLD/reindex.sql" > /dev/null || die "REINDEX pass failed"
  q "CREATE TABLE bl_after AS
       SELECT /* wiki_pgsi_score_after */
              b.idx_oid, pg_relation_size(b.idx_oid) AS after_bytes
         FROM bl_before b;
     CREATE VIEW bl_scored AS
       SELECT b.*, a.after_bytes,
              100 * (b.index_size - b.est_rebuilt_bytes) / b.index_size AS est_pct,
              100 * (b.before_bytes - a.after_bytes)::numeric / b.before_bytes AS actual_pct,
              100 * (b.index_size - b.est_rebuilt_bytes) / b.index_size
              - 100 * (b.before_bytes - a.after_bytes)::numeric / b.before_bytes AS delta
         FROM bl_before b JOIN bl_after a USING (idx_oid);" > /dev/null \
    || die "post-reindex pass failed"

  {
    printf 'index|size|est_pct|actual_pct|delta|density|dead\n'
    s "SELECT index_name || '|' || index_size || '|' || round(est_pct, 1) || '|' ||
              round(actual_pct, 1) || '|' || round(delta, 1) || '|' ||
              coalesce(round(avg_leaf_density::numeric, 2)::text, 'NaN') || '|' || dead_pages
         FROM bl_scored WHERE schema_name = 'bl' ORDER BY actual_pct DESC, index_name"
    # Three populations, because the scoring pass rewrites pg_class while it
    # runs: a catalog index can grow between the snapshot and its own
    # REINDEX, which is a property of the harness and not of the estimator.
    printf '\n-- totals, by population\n'
    s "SELECT 'all: scored=' || count(*) ||
              ' within_1.0=' || count(*) FILTER (WHERE abs(delta) <= 1.0) ||
              ' within_2.0=' || count(*) FILTER (WHERE abs(delta) <= 2.0) ||
              ' max_over=' || round(max(delta), 1) || ' max_under=' || round(min(delta), 1)
         FROM bl_scored
        UNION ALL
       SELECT 'fixtures (schema bl): scored=' || count(*) ||
              ' within_1.0=' || count(*) FILTER (WHERE abs(delta) <= 1.0) ||
              ' within_2.0=' || count(*) FILTER (WHERE abs(delta) <= 2.0) ||
              ' max_over=' || round(max(delta), 1) || ' max_under=' || round(min(delta), 1)
         FROM bl_scored WHERE schema_name = 'bl'
        UNION ALL
       SELECT 'catalog and TOAST: scored=' || count(*) ||
              ' within_1.0=' || count(*) FILTER (WHERE abs(delta) <= 1.0) ||
              ' max_over=' || round(max(delta), 1)
         FROM bl_scored WHERE schema_name <> 'bl'
        UNION ALL
       SELECT 'no leaf pages: ' || count(*) || ', all at est ' ||
              round(max(abs(est_pct)), 1) || ' and actual ' || round(max(abs(actual_pct)), 1)
         FROM bl_scored WHERE leaf_pages = 0"
    printf -- '-- the five largest over-estimates\n'
    s "SELECT schema_name || '.' || index_name || ' ' || pg_size_pretty(index_size) ||
              ' +' || round(delta, 1)
         FROM bl_scored ORDER BY delta DESC LIMIT 5"
    printf -- '-- the three largest under-estimates\n'
    s "SELECT schema_name || '.' || index_name || ' ' || pg_size_pretty(index_size) ||
              ' ' || round(delta, 1)
         FROM bl_scored ORDER BY delta LIMIT 3"
  } > "$OUT/scores12.txt"
  tail -16 "$OUT/scores12.txt" >&2
}

# ------------------------------------------------------------- residual -----
# A rebuilt index must report no waste at its own fillfactor.  Reads the
# tables the score stage left behind.
stage_residual() {
  say "post-REINDEX residual of wasted_vs_fillfactor"
  s "SELECT 1 FROM bl_before LIMIT 1" > /dev/null 2>&1 || die "run the score stage first"
  # Read the same population the score stage scored, not whatever the view
  # sees now: creating bl_before itself adds a TOAST index to the database.
  q "DROP TABLE IF EXISTS bl_residual;
     CREATE TABLE bl_residual AS
       SELECT /* wiki_pgsi_score_residual */
              f.idx_oid, f.wasted_vs_fillfactor AS residual_bytes, f.index_size,
              f.avg_leaf_density, f.leaf_pages
         FROM bloat_final f
        WHERE f.idx_oid IN (SELECT idx_oid FROM bl_before);" > /dev/null \
    || die "residual pass failed"
  {
    printf -- '-- post-REINDEX residual, over the population the score stage scored\n'
    s "SELECT 'scored=' || count(*) ||
              ' exactly_zero=' || count(*) FILTER (WHERE residual_bytes = 0) ||
              ' at_or_below_0.1pct=' || count(*) FILTER (WHERE 100 * residual_bytes / index_size <= 0.1) ||
              ' worst_pct=' || round(max(100 * residual_bytes / index_size), 1)
         FROM bl_residual"
    printf -- '-- worst residual, and the worst among indexes the report prints\n'
    s "SELECT r.idx_oid::regclass || ' ' || round(100 * r.residual_bytes / r.index_size, 1) ||
              '% ' || r.residual_bytes || ' bytes, leaf_pages ' || r.leaf_pages ||
              ', density ' || coalesce(round(r.avg_leaf_density::numeric, 2)::text, 'NaN')
         FROM bl_residual r ORDER BY 100 * r.residual_bytes / r.index_size DESC LIMIT 3"
    s "SELECT 'worst at or above 1 MB: ' || r.idx_oid::regclass || ' ' ||
              round(100 * r.residual_bytes / r.index_size, 1) || '% ' || r.residual_bytes || ' bytes'
         FROM bl_residual r WHERE r.index_size >= 1024 * 1024
        ORDER BY 100 * r.residual_bytes / r.index_size DESC LIMIT 1"
    printf -- '-- how many one-leaf-page indexes carry the worst residual\n'
    s "SELECT 'at_44.6pct=' || count(*) || ', all with leaf_pages ' || max(leaf_pages)
         FROM bl_residual WHERE round(100 * residual_bytes / index_size, 1) = 44.6"
    printf -- '-- the ratio between the two percentage columns, before the rebuild\n'
    s "SELECT b.index_name || ' ' ||
              round(b.wasted_vs_fillfactor / (b.index_size - b.est_rebuilt_bytes), 4) ||
              ' dead_pages ' || b.dead_pages
         FROM bl_before b
        WHERE b.index_size - b.est_rebuilt_bytes > 1024 * 1024
        ORDER BY 1"
    printf -- '-- indexes where the two definitions coincide: fillfactor 100, or no leaf pages\n'
    s "SELECT 'coincide=' || count(*) FILTER (WHERE target_density = 1 OR leaf_pages = 0) ||
              ' of ' || count(*) || ' (fillfactor 100: ' ||
              count(*) FILTER (WHERE target_density = 1) || ', no leaf pages: ' ||
              count(*) FILTER (WHERE leaf_pages = 0) || ')'
         FROM bl_before"
    printf -- '-- clamped rows: leaves denser than their own target, so the leaf term is 0\n'
    s "SELECT 'clamped=' || count(*) FILTER (WHERE leaf_pages > 0
                AND round(leaf_bytes * target_density) <= live_leaf_bytes) ||
              ' of ' || count(*) FILTER (WHERE leaf_pages > 0) || ' with leaf pages'
         FROM bl_before"
  } > "$OUT/residual12.txt"
  cat "$OUT/residual12.txt" >&2
}

# ----------------------------------------------------------------- cost -----
stage_cost() {
  say "what the statement costs to run"
  : > "$OUT/cost12.txt"
  local f="$OUT/cost12.txt"
  printf 'EXPLAIN (ANALYZE, BUFFERS) of the filed text\n' >> "$f"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -P pager=off -d "$DB" \
    -c "SET statement_timeout = '15min'; SET lock_timeout = '5s';" \
    -c "EXPLAIN (ANALYZE, BUFFERS) $(cat "$SQLD/bare.sql")" >> "$f" 2>&1
  printf 'plan_lines=%s\n' "$(grep -c '^' "$f")" >> "$f"
  printf 'cte_scans=%s\n' "$(grep -c 'CTE Scan' "$f")" >> "$f"
  local i a b
  printf 'six interleaved end-to-end runs, filed text then superseded text (ms)\n' >> "$f"
  for i in 1 2 3 4 5 6; do
    a=$(date +%s%N)
    "$BIN/psql" -X -q -o /dev/null -d "$DB" -f "$SQLD/report.sql" > /dev/null 2>&1
    b=$(date +%s%N)
    printf 'filed=%s.%s ' $(( (b - a) / 1000000 )) $(( ((b - a) / 100000) % 10 )) >> "$f"
    if [ -f "$SQLD/prev_bare.sql" ]; then
      a=$(date +%s%N)
      "$BIN/psql" -X -q -o /dev/null -d "$DB" -f "$SQLD/prev.sql" > /dev/null 2>&1
      b=$(date +%s%N)
      printf 'prev=%s.%s' $(( (b - a) / 1000000 )) $(( ((b - a) / 100000) % 10 )) >> "$f"
    fi
    printf '\n' >> "$f"
  done
  printf 'database_size=%s\n' "$(s 'SELECT pg_size_pretty(pg_database_size(current_database()))')" >> "$f"
  grep -E 'Execution Time|plan_lines|cte_scans|filed=|Buffers: shared' "$f" | head -20 >&2
}

# ----------------------------------------------------------------- priv -----
stage_priv() {
  say "who may run it"
  : > "$OUT/priv12.txt"
  local f="$OUT/priv12.txt"
  q "DROP ROLE IF EXISTS mon; DROP ROLE IF EXISTS nomon;
     CREATE ROLE mon LOGIN; CREATE ROLE nomon LOGIN;
     GRANT pg_stat_scan_tables TO mon;" > /dev/null 2>&1 || die "role setup failed"
  # The statement passes an OID column cast to regclass, which resolves no
  # name.  Writing 'bl.i_del90'::regclass in a test would resolve one, so the
  # OID is read here, as a number, and substituted.
  local oid
  oid=$(s "SELECT 'bl.i_del90'::regclass::oid")
  {
    printf 'index_oid=%s\n' "$oid"
    printf 'mon_schema_usage=%s\n' \
      "$(s "SELECT has_schema_privilege('mon', 'bl', 'USAGE')::text")"
    printf 'mon_rows_from_statement=%s\n' \
      "$("$BIN/psql" -X -At -q -U mon -d "$DB" -f "$SQLD/report.sql" 2>&1 | grep -c '|')"
    printf 'mon_by_name=%s\n' \
      "$("$BIN/psql" -X -At -q -U mon -d "$DB" -c "SELECT * FROM pgstatindex('bl.i_del90')" 2>&1 | grep -E '^(ERROR|FATAL)' | head -1)"
    printf 'mon_by_oid_leaf_pages=%s\n' \
      "$("$BIN/psql" -X -At -q -U mon -d "$DB" -c "SELECT leaf_pages FROM pgstatindex($oid::regclass)" 2>&1 | head -1)"
    printf 'nomon_by_oid=%s\n' \
      "$("$BIN/psql" -X -At -q -U nomon -d "$DB" -c "SELECT leaf_pages FROM pgstatindex($oid::regclass)" 2>&1 | head -1)"
  } >> "$f"
  q "DROP ROLE IF EXISTS mon; DROP ROLE IF EXISTS nomon;" > /dev/null
  cat "$f" >&2
}

# ----------------------------------------------------------------- race -----
# Two shapes of interference, both driven from a second session:
#   lock_timeout   an uncommitted DROP INDEX holds AccessExclusiveLock and the
#                  call is cancelled at the timeout
#   committed drop the same drop commits while the report waits, so
#                  relation_open finds nothing behind the lock
stage_race() {
  say "locking and the concurrent-drop race"
  : > "$OUT/race12.txt"
  local f="$OUT/race12.txt" a b msg
  "$BIN/psql" -X -q -d "$DB" -c \
    "BEGIN; DROP INDEX bl.i_del50; SELECT pg_sleep(8); ROLLBACK;" > /dev/null 2>&1 &
  local holder=$!
  sleep 2
  a=$(date +%s%N)
  msg=$(err "SET lock_timeout = '2s'; SELECT * FROM pgstatindex('bl.i_del50'::regclass)")
  b=$(date +%s%N)
  printf 'lock_timeout_error=%s\n' "$msg" >> "$f"
  printf 'lock_timeout_ms=%s.%s\n' $(( (b - a) / 1000000 )) $(( ((b - a) / 100000) % 10 )) >> "$f"
  wait "$holder" 2>/dev/null

  # Case 1: the drop commits while cand is still being materialized.  cand
  # sizes each candidate with pg_relation_size, which opens the relation with
  # try_relation_open and returns NULL when it has gone, so the row is
  # filtered out and the report survives, one row shorter.
  local before after
  before=$(( $("$BIN/psql" -X -q -A -F '|' -P footer=off -d "$DB" -f "$SQLD/report.sql" 2>/dev/null | grep -c '^') - 1 ))
  "$BIN/psql" -X -q -d "$DB" -c \
    "BEGIN; DROP INDEX bl.i_uniq; SELECT pg_sleep(3); COMMIT;" > /dev/null 2>&1 &
  local dropper=$!
  sleep 1
  a=$(date +%s%N)
  # psql prefixes an error from -f with "psql:<file>:<line>: ", so the match
  # cannot be anchored at the start of the line; the prefix is then cut.
  "$BIN/psql" -X -q -A -F '|' -P footer=off -d "$DB" -f "$SQLD/report.sql" \
    > "$OUT/race12_during.txt" 2>&1
  b=$(date +%s%N)
  wait "$dropper" 2>/dev/null
  msg=$(grep -E '(ERROR|FATAL):' "$OUT/race12_during.txt" | head -1 | sed 's/^psql:[^ ]* //')
  after=$(( $(grep -c '^' "$OUT/race12_during.txt") - 1 ))
  printf 'drop_during_cand_error=%s\n' "${msg:-none}" >> "$f"
  printf 'drop_during_cand_rows=%s (was %s)\n' "$after" "$before" >> "$f"
  printf 'drop_during_cand_ms=%s\n' $(( (b - a) / 1000000 )) >> "$f"
  q "CREATE UNIQUE INDEX IF NOT EXISTS i_uniq ON bl.t_uniq (id);" > /dev/null 2>&1

  # Case 2: the drop lands after cand has sized that index and released its
  # lock, but before pgstatindex opens it.  That window is short, so the
  # delay is swept until the abort appears.
  # The target is the fixture index cand materializes last, so the window
  # between its size check and its pgstatindex call is the whole loop rather
  # than a few milliseconds.  The dropping session connects first and holds
  # the delay inside the server, as pg_sleep in the same command, so psql's
  # start-up cost is out of the critical path.
  local target defn d attempts=0 hit=
  target=$(s "SELECT index_name FROM (SELECT index_name, row_number() OVER () AS rn
                                        FROM bloat_final
                                       WHERE schema_name = 'bl'
                                         AND index_size >= 1024 * 1024) q
               ORDER BY rn DESC LIMIT 1")
  defn=$(s "SELECT pg_get_indexdef('bl.$target'::regclass)")
  # The reachable window is (end of cand, pgstatindex on the target).  On a
  # warm cache the whole report is under 100 ms, so that window sits inside
  # psql's own start-up skew; the sweep is recorded whether or not it lands.
  printf 'drop_after_cand_target=%s\n' "$target" >> "$f"
  for d in 0.005 0.02 0.04 0.06 0.07 0.075 0.08 0.09 0.10 0.12; do
    a=$(date +%s%N)
    "$BIN/psql" -X -q -d "$DB" \
      -c "SELECT pg_sleep($d)" \
      -c "BEGIN; DROP INDEX bl.$target; SELECT pg_sleep(2); COMMIT;" > /dev/null 2>&1 &
    local dp=$!
    "$BIN/psql" -X -q -d "$DB" -f "$SQLD/report.sql" > "$OUT/race12_after.txt" 2>&1
    b=$(date +%s%N)
    wait "$dp" 2>/dev/null
    attempts=$((attempts + 1))
    msg=$(grep -E '(ERROR|FATAL):' "$OUT/race12_after.txt" | head -1 | sed 's/^psql:[^ ]* //')
    q "$defn" > /dev/null 2>&1
    case $msg in
      *"could not open relation"*)
        hit="$msg after $(( (b - a) / 1000000 )) ms, drop taken ${d}s in"; break ;;
    esac
  done
  printf 'drop_after_cand_error=%s\n' "${hit:-not reproduced in $attempts attempts}" >> "$f"
  printf 'drop_after_cand_attempts=%s\n' "$attempts" >> "$f"
  q "CREATE UNIQUE INDEX IF NOT EXISTS i_uniq ON bl.t_uniq (id);" > /dev/null 2>&1
  cat "$f" >&2
}

# -------------------------------------------------------------- summary -----
stage_summary() {
  say "what landed in $OUT"
  ls -la "$OUT" >&2
  local x
  for x in platform12 exact12 facts12 cost12 priv12 race12; do
    [ -f "$OUT/$x.txt" ] && { printf '\n---- %s\n' "$x" >&2; cat "$OUT/$x.txt" >&2; }
  done
  [ -f "$OUT/scores12.txt" ] && { printf '\n---- scores12 (tail)\n' >&2; tail -14 "$OUT/scores12.txt" >&2; }
  [ -f "$OUT/residual12.txt" ] && { printf '\n---- residual12\n' >&2; cat "$OUT/residual12.txt" >&2; }
}

stage_stop() {
  say "stop the 12.2 cluster"
  [ -d "$DATA" ] || { note "no data directory"; return 0; }
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  fi
  [ -f "$DATA/postmaster.pid" ] && die "postmaster.pid still present in $DATA"
  pgrep -f "postgres.*-D $DATA" > /dev/null 2>&1 && die "a postgres process still runs on $DATA"
  [ -n "$(ls -A "$SOCK" 2>/dev/null)" ] && die "socket directory $SOCK is not empty"
  note "stopped: no postmaster.pid, no process, empty socket directory"
}

# This leg deletes only its own four directories; the 17 leg owns out/ and
# sql/ and removes the sandbox itself.
stage_clean() {
  stage_stop || exit 1
  case "$SANDBOX" in
    "$WIKI_ROOT"/.wiki-runtime/tmp/?*) ;;
    *) die "refusing to delete under $SANDBOX: not under $WIKI_ROOT/.wiki-runtime/tmp/" ;;
  esac
  say "delete this leg's directories under $SANDBOX"
  rm -rf "$BUILD" "$INST" "$DATA" "$SOCK" "$SQLD"
}

STAGES_DEFAULT="build check cluster texts fixtures report facts cost priv score residual race summary"
run_stage() {
  case "$1" in
    build|check|cluster|texts|fixtures|report|facts|score|residual|cost|priv|race|summary|stop|clean)
      "stage_$1" ;;
    *) die "unknown stage: $1" ;;
  esac
}
main() {
  local st
  if [ "$#" -eq 0 ]; then set -- $STAGES_DEFAULT; fi
  for st in "$@"; do run_stage "$st" || die "stage $st failed"; done
  say "done: $*"
}
main "$@"
```

### The last run

| Fact | Value |
|---|---|
| Date | 2026-09-10 |
| Host | Linux x86_64, `uname -sm` recorded into `out/platform17.txt` |
| 17 leg | 17.11 built from `786db8dcf168bd9df8f55047337525ac19118b1c`, `--enable-debug --with-icu --with-readline --with-zlib`, `make check` All 225 tests passed, `contrib/pgstattuple` All 1 |
| 12 leg | 12.2 built from `45b88269a353ad93744772791feb6d01bc7e1e42`, the same without `--with-icu`, `make check` All 192 tests passed, `contrib/pgstattuple` All 1 |
| Platform facts the numbers depend on | `block_size` 8192, `max_data_alignment` 8, `initdb --locale=C --encoding=UTF8` |
| Fixture scale | `ROWS = 1000000`; 39 indexes in schema `bl` and a 1828 MB database on 17.11, 38 and 1801 MB on 12.2 |
| Statement text | `sql` block 1, SHA-256 `9d2e3a2c73c81f3efb848b61f4bf307365ce0da56d780baee08fa9a9606f6c91`, 126 lines, 6,154 bytes |
| Stages run | `fixtures report facts cost priv score residual race summary`, 86 s per leg from a built tree |
| Teardown | both servers stopped with `pg_ctl -m fast -w stop` and the sandbox deleted by the `clean` stages |

The numbers in every section written before this date came from the harness
these scripts replace, not from the scripts themselves; that is recorded under
[Open Questions](#open-questions) and is why `verified_by_agent` is still
`not yet`.

## Context Reviewed

- `contrib/pgstattuple/pgstatindex.c` in both pinned checkouts, function by
  function: the four `pgstatindex` entry points, `pgstatindex_impl`, and the
  `pgstatginindex`/`pgstathashindex` neighbours that share its guards.
- `contrib/pgstattuple/pgstattuple--1.4.sql`, `pgstattuple--1.4--1.5.sql` and
  `pgstattuple.control` in both checkouts; the three files are byte-identical
  between them, which is why one statement can target both.
- `contrib/pgstattuple/sql/pgstattuple.sql` and `expected/pgstattuple.out`, for
  the upstream expectations on empty indexes, wrong access methods, partitioned
  indexes, views, foreign tables and sequences.
- `doc/src/sgml/pgstattuple.sgml`, for the documented column meanings and the
  `pg_stat_scan_tables` access rule.
- `src/backend/access/nbtree/nbtsort.c` and `src/include/access/nbtree.h`, for
  what a rebuild targets: `_bt_pagestate`, `_bt_buildadd`, `BTGetFillFactor`,
  `BTGetTargetPageFreeSpace`, the fillfactor constants, and the page-opaque
  layout.
- `src/backend/access/nbtree/nbtree.c`, for what a vacuum does with deleted pages,
  and the absence of any truncation call under `src/backend/access/nbtree/`.
- `src/include/storage/bufpage.h` and `src/backend/storage/page/bufpage.c`, for
  `SizeOfPageHeaderData` and the line-pointer deduction in `PageGetFreeSpace`.
- `src/include/catalog/pg_index.h`, `pg_class.h` and `src/include/utils/rel.h`,
  for the catalog columns and the temp-relation test the filters mirror.
- `src/backend/utils/misc/guc_tables.c` for the two timeout GUCs' contexts, and
  `src/backend/optimizer/util/plancat.c` for the unlogged-during-recovery rule
  that does not apply to function calls.
- The v17 checkout's own history for `13503eb5905` and its containing release
  tags.
- The ten-column result tuple `pgstatindex` builds
  ([pgstatindex.c#result-tuple](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L339-L378)),
  re-read when the threshold was removed: the statement consumes eight of the ten
  (`version`, `index_size`, `internal_pages`, `leaf_pages`, `empty_pages`,
  `deleted_pages`, `avg_leaf_density`, `leaf_fragmentation`), leaves `tree_level`
  and `root_block_no` unused, and none of them was reached through the removed
  `alert_pct`.
- `src/backend/access/nbtree/nbtsplitloc.c`, re-read when wasted space was
  rebased on the fillfactor: `_bt_findsplitloc` reads `BTGetFillFactor` but
  applies it as `fillfactormult` only on a rightmost page, uses 0.50 for an
  ordinary leaf split, and switches to `BTREE_SINGLEVAL_FILLFACTOR` for a page
  full of one value. That is why the new column is defined against what a
  *rebuild* targets and not against how a growing index packs itself.
- `src/backend/utils/adt/dbsize.c`, read during the 2026-09-10 review for the
  one function in the statement that is not `pgstatindex`: `pg_relation_size`
  opens with `try_relation_open` and returns NULL rather than raising when the
  relation has gone, and its comment says why. That is what makes the
  size prefilter absorb a concurrent drop, and it is the reason this page's
  concurrent-drop claim changed.
- The full `guc_tables.c` entries for every setting the measurement scripts
  write, so each one's apply scope can be named: `listen_addresses`, `port`,
  `unix_socket_directories`, `shared_buffers`, `logging_collector`, `fsync`,
  `autovacuum`, `maintenance_work_mem` and
  `max_parallel_maintenance_workers`.
- `doc/src/sgml/installation.sgml`, for the VPATH build the scripts use to keep
  `raw/postgres-17/` read-only.
- This page's own git history, for the two superseded statement texts and their
  sizes, and the v17 checkout's history for the back-patch note on
  `13503eb5905`.
- All 116 source citations on this page were re-read against the pinned
  checkout on 2026-09-10: 50 distinct ranges over 21 files, every one resolving
  and in bounds, none pointing outside `raw/postgres-17/`.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstatindex` accepts only a B-tree index relation | [pgstatindex.c#IS_BTREE](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228); measured errors for hash, GIN, GiST, SP-GiST, BRIN and a partitioned index on both servers |
| It refuses another session's temp index | [pgstatindex.c#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L230-L238), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669); reproduced on both servers with a second session holding a 4.4 MB temp index |
| 17 refuses an invalid index, 12.2 returns a row | [pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250); commit `13503eb5905`, earliest containing tag `REL_17_0` in this checkout; measured both ways |
| `index_size` is the whole file | [pgstatindex.c#index_size](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L357); equal to `pg_relation_size` for 218/218 and 212/212 candidates |
| `avg_leaf_density` ignores empty and deleted pages | [pgstatindex.c#leaf-accounting](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L324); `i_delhead` at 89.94% density and 69.9% reclaimable, confirmed by `REINDEX` |
| No leaf pages gives `NaN`, and `NaN` outranks every threshold | [pgstatindex.c#NaN](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372), [pgstattuple.out#empty-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52); measured `NaN > 20` true for `float8` and `numeric` on both servers |
| A rebuild's leaf density is `(leaf_capacity - BLCKSZ*(100-ff)/100) / leaf_capacity` | [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L860), [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145); four fillfactors measured within 0.18 points on both servers |
| Leaf capacity is `block_size - 24 - 16` | [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214), [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71), [pgstatindex.c#max_avail](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L314); implied 8151.6 and 8152.1 from two known-content pages on both servers |
| `PageGetFreeSpace` deducts one line pointer | [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923); an empty leaf page reads 0.05% density, not 0.00% |
| VACUUM never returns index pages to the filesystem | no `RelationTruncate`/`smgrtruncate` under `src/backend/access/nbtree/`, [nbtree.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1170); 1,918 dead pages survived VACUUM and disappeared on REINDEX |
| Access is `pg_stat_scan_tables`, and the OID form needs no schema `USAGE` | [pgstattuple--1.4--1.5.sql#grants](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92), [pgstattuple.sgml#access](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L15-L24); measured with two non-superuser roles on both servers |
| The call waits on `AccessExclusiveLock`; `lock_timeout` bounds the wait | [pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631); cancelled at 2000.9 ms and 2001.0 ms |
| A concurrent drop that commits while `cand` runs costs one row, not the report | [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371) opens with `try_relation_open` and returns NULL; measured on both servers on 2026-09-10, the report waited out a three-second `DROP INDEX` and returned 24 rows against 25 on 17.11 and 23 against 24 on 12.2, with no error |
| A concurrent drop that commits between the size check and the per-index call aborts the whole report | [pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213) uses `relation_open`, which raises; measured `could not open relation with OID 16897` (17.11) and `17173` (12.2) after 6.0 s in the original run, **not reproduced** in the 2026-09-10 re-run's ten-delay sweep on either server |
| The scan uses a 256 kB bulk-read ring | [pgstatindex.c#BAS_BULKREAD](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L222), [freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L540-L574) |
| Deduplication is the only thing that made the two servers' numbers differ | [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1151); 24 of the report's rows identical, `i_dup`/`i_dup_ins` 21 MB against 6800 kB and 20 MB against 6368 kB |
| Dropping `alert_pct` and `status` changes nothing else the statement returns | measured on both restarted servers: the amended output equals the filed output with field 14 cut, byte for byte (2,448 and 2,531 bytes); one view per text over the internal `final` stage exposes 29 columns against 28 with `alert_pct` the only loss, and `EXCEPT` in both directions over the 28 shared columns returns 0 rows across 214 and 220 indexes |
| Removing the column costs nothing to run | measured: identical plan shape (4 `CTE Scan` nodes; 68 and 60 plan lines), `EXPLAIN (ANALYZE, BUFFERS)` execution 136.3 against 135.7 ms on 17.11 and 134.7 against 127.9 ms on 12.2, over six interleaved end-to-end runs of each text per server |
| Fillfactor is a build and rightmost-split target, not a property a growing index holds | [nbtsplitloc.c#fillfactormult](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L279-L335), [nbtsplitloc.c#SPLIT_SINGLE_VALUE](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416), [nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202) |
| Rebasing wasted space on the fillfactor moves those two fields and nothing else | measured on both restarted servers: the 12 untouched presentation fields identical byte for byte (2,109 and 2,180 bytes); the internal `final` stage exposes 28 columns either way with `wasted_space` swapped one-for-one for `wasted_vs_fillfactor`; `EXCEPT` in both directions over the 27 shared columns returns 0 rows across 214 and 220 indexes |
| At fillfactor 100 the two definitions coincide exactly | `target_free` is `BLCKSZ * 0 / 100`, so `target_density` is 1 ([nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)); measured equal to the byte on `i_ff100` and `probe_ff_100`, and equal for 110 of 214 and 110 of 220 indexes once the no-leaf-page indexes are included |
| A rebuilt index reports no waste at its own fillfactor | measured after `REINDEX INDEX` over every scored index: exactly 0 bytes for 81 of 97 and 76 of 96, at or below 0.1% for 87 and 85, and at or below 0.4% for every index the report actually prints |
| The new column over-reports on indexes too small to fill a page | measured: `c_one_idx`, one tuple on one leaf page at 0.29% density, reports 7,309 bytes and 44.6% after a rebuild on both servers; the old baseline read 8,128 bytes and 49.6% on the same index |
| For in-page waste the new column is `(leaf_capacity - target_free) / block_size` of `est_reclaimable` | predicted 0.8951 at 8192/90; measured 0.8868-0.8921 over the 13 dead-page-free indexes with more than 1 MB of estimated reclaim, `1.0001` on the dead-page fixture `i_delhead`, and `0.8321` on `i_wide`, identically on both servers |
| Rebasing costs nothing to run | measured: identical plan shape (4 `CTE Scan` nodes; 68 and 60 plan lines) and identical total buffers, 108,021 on 17.11 and 108,327 on 12.2, differing only in the hit/read split; execution 131.1 against 120.9 ms on 17.11 and 123.6 against 117.5 ms on 12.2 |
| `BTPageOpaqueData` is 16 bytes across five fields | [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71), [nbtree.h:29](../../../../raw/postgres-17/src/include/access/nbtree.h#L29): two `BlockNumber`, one `uint32`, `btpo_flags` as `uint16` and `btpo_cycleid` as `BTCycleId`, itself a `uint16` |
| The invalid-index check separates these two minors, not the two majors | commit `13503eb5905` in this checkout's history ends "Back-patch to v11 (all supported versions)"; the pinned 12.2 predates it and returns a row |
| Every number this page takes from a running server has a published script | [Measurement Script](#measurement-script); both legs run end to end and were last run on 2026-09-10 |

## Open Questions

- **Most of the numbers on this page predate the script that is now filed for
  them.** The sections written before 2026-09-10 were measured by a harness
  that was never published, in a sandbox that has since been deleted, so the
  fixture DDL behind figures such as the 27-and-28-row reports, the 94-and-93
  index accuracy totals and the `EXCEPT` attribution counts cannot be
  recovered. [Measurement Script](#measurement-script) is a reconstruction
  built to the same recipe from the page's own descriptions, and
  [Re-measured from a published script](#re-measured-from-a-published-script)
  reports how close it lands: identical on the refusals, the invalid-index row,
  the implied leaf capacity, the dead-page fixture and eighteen of the accuracy
  rows, and different where the reconstruction's row counts differ. Until every
  older figure is re-derived from the filed scripts, or deleted,
  `verified_by_agent` stays `not yet`.
- **One of the four fillfactor fixtures does not scale like the other three.**
  The filed table gives `i_ff10_del90` 1,579 leaf pages, which is a third of
  what a fillfactor-10 index over the same row count as `i_ff100_del90` (493
  pages) and `i_ff50_del90` (991) would hold; the reconstruction, built at one
  row count for all four, reads 5,264. Every percentage in that row is
  scale-invariant and reproduced exactly, so the discrepancy is a fixture-size
  difference and not an estimator defect, but the original row counts cannot be
  confirmed.
- **The concurrent-drop abort was not reproduced.** The source says it is
  reachable — `pgstatindexbyid_v1_5` uses `relation_open`, which raises — and
  the original run recorded it on both servers, but the 2026-09-10 sweep of ten
  delays per server never landed in the window between `cand`'s size check and
  the per-index call. On this fixture suite the whole report runs in about
  80 ms, so that window is smaller than the difference between two `psql`
  start-ups. A reproduction needs either a report slow enough to widen the
  window or an injection point the filed text does not have.
- **The page reports three different scored populations and only the newest one
  is defined.** 218 and 212 candidates, 94 and 93 scored indexes, 97 and 96 in
  the residual pass, 214 and 220 in the `EXCEPT` attribution. Part of that is
  explained — creating a helper table adds a TOAST index between passes, which
  the re-run reproduced — but the older figures cannot be reconciled against
  each other now that their harness is gone. The re-run states its population
  exactly: 201 and 195, of which 32 and 31 are fixtures.
- **Which 12 minor first refuses an invalid index is not readable here.** The
  commit that added the check was back-patched to every then-supported branch,
  so some 12.x behaves like 17.11, but naming it would need a v12 checkout at
  another pin, which this page may not cite.
- **The 12 leg's cluster settings have no citable apply scope on this page.**
  The contexts in
  [What the scripts read from the environment](#what-the-scripts-read-from-the-environment)
  come from the v17 GUC table, because a v17 page may not cite the v12
  checkout. Nothing in the run depends on them differing, and the 12 leg
  accepted every setting, but the 12.2 contexts are asserted, not cited.
- **Nothing was run on a standby.** The `relpersistence <> 'u' OR NOT
  pg_is_in_recovery()` filter is reasoning from
  [plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)
  and from the absence of any recovery guard in `pgstatindex_impl`, not from a
  measurement. What `pgstatindex` actually returns for an unlogged index on a hot
  standby is untested, and so is the whole report's behaviour there.
- **Only two minor versions were tested**, 12.2 and 17.11, both from this repo's
  pins, twice: once by the original harness and once by the filed scripts. The
  invalid-index check has no release tag before `REL_17_0` in the v17 checkout
  and its commit says it was back-patched, so at least one later 12.x minor
  differs from the pinned 12.2; what else may have changed in a later 12.x
  cannot be checked from this page's evidence base.
- **One block size.** Every measurement is at `block_size` 8192. The statement
  reads `block_size` from the server, but the 24 and 16 constants, the whole
  target-density model, and the `(leaf_capacity - target_free) / block_size`
  ratio between the two percentage columns are unverified at 4 kB, 16 kB or
  32 kB.
- **The internal-page term is a proportional guess.** `round(internal_pages *
  est_leaf / leaf_pages)` was never tested against a case where the rebuilt tree
  loses a level; internal pages were under 0.5% of every fixture, so the suite
  cannot distinguish a good model from a lucky one.
- **The `+1.7` over-estimate is unexplained in detail.** It reproduced exactly on
  both servers on the same 456 kB TOAST primary key, which suggests per-page
  rounding rather than noise, but no per-page accounting was done to confirm it,
  and no fixture was built to find the worst case for small indexes.
- **`i_novac` and `i_dedup_off` have no in-statement warning.** Both come back
  with an empty `notes` string, a near-zero estimate (`−0.1%` and `−0.3%`) and,
  since wasted space was rebased on the fillfactor, `0.0` wasted as well, on an
  index a rebuild would shrink by 90% and 69%. Neither condition is visible in
  any `pgstatindex` column, so closing them would require a second tool and would
  break the "pgstatindex only" constraint; the page documents them instead.
- **The fillfactor-relative column over-reports on indexes too small to fill a
  page**, and nothing in the statement says so. A one-row index measured 44.6%
  wasted immediately after `REINDEX` on both servers, because one tuple cannot
  fill 89.95% of a page. The 1 MB `min_index_bytes` prefilter keeps every such
  index out of the report, but a caller who lowers that threshold gets the
  over-report with no note attached, and no term was designed to catch it. The
  same effect leaves a 1.5% residual on a freshly rebuilt 416 kB `fillfactor =
  100` index, whose rightmost page holds the remainder.
- **The clamp hides how far above target an index sits.** Every index denser than
  its fillfactor target reports the same `0`, whether it is 0.11 points over like
  `i_fresh` at 90.06% or 6 points over like `i_dup_ins` at 95.94%, both against a
  89.95% target. The `denser than a rebuild would leave it` note does not close
  the gap, because it fires on `est_reclaimable_pct <= -1` rather than on the
  clamp: measured, `i_dup_ins` carries the note and `i_fresh` clamps to `0` with
  an empty `notes` string. Thirteen indexes per server clamped in this run.
- **Whether both percentage columns should still exist was not settled by
  measurement.** For in-page waste the new column is a fixed multiple of
  `est_reclaimable_pct`, so on a default-fillfactor database it carries little
  independent information; at fillfactor 10 it carries a great deal, since the
  same index reads 8.3% and 89.9%. No reader other than the author has judged
  whether two near-proportional columns help or confuse.
- **The `−2.8` under-estimate on `i_ff10_del90` is explained but not proven.**
  The per-page high-key and line-pointer overhead that `avg_leaf_density` counts
  as payload is the plausible cause and the arithmetic is consistent with it, but
  no per-page accounting was done, and doing it needs a tool this page excludes.
  It is also the first vacuumed fixture on this page to miss by more than a
  point, which means the accuracy figures in
  [Accuracy against REINDEX INDEX](#accuracy-against-reindex-index) are a
  fillfactor-90 result, not a general one.
- **Removing the verdict column moves the judgement off the page.** The statement
  now returns numbers only, and nothing in this repository measures what
  threshold is right for a given environment. The 20% that the removed `status`
  column used was never derived from anything but convention, which is part of
  why it is gone, but no replacement rule was measured either.
- **The ordering has no tie-break.** `ORDER BY index_size - est_rebuilt_bytes
  DESC` left two equal-sized 12.2 rows in a different order on the two runs
  recorded here. It never changed a value, but a caller diffing two reports will
  see tied rows move.
- **The churn fixture is not deterministic**, so `i_churn` is the one row that
  cannot be used as a cross-version identity check.
- **No test of a partitioned table with hundreds of partitions**, where the report
  returns one row per leaf index and the aggregate reading a DBA wants is per
  parent. The statement has no roll-up.

## Source References

- [pgstatindex.c#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L144-L160)
- [pgstatindex.c#pgstatindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L162-L180)
- [pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213)
- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L381)
- [pgstatindex.c#result-tuple](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L339-L378)
- [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L30)
- [pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)
- [pgstattuple.control:1-5](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml#access](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L15-L24)
- [pgstattuple.sgml#pgstatindex](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L161-L281)
- [pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37)
- [pgstattuple.out#empty-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)
- [pgstattuple.out#wrong-relkinds](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L140-L171)
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71)
- [nbtree.h#MaxTIDsPerBTreePage](../../../../raw/postgres-17/src/include/access/nbtree.h#L185-L187)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)
- [nbtree.h#P_ISLEAF](../../../../raw/postgres-17/src/include/access/nbtree.h#L212-L227)
- [nbtree.h#P_HIKEY](../../../../raw/postgres-17/src/include/access/nbtree.h#L348-L369)
- [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)
- [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1151)
- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671)
- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L860)
- [nbtsplitloc.c#fillfactormult](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L279-L335)
- [nbtsplitloc.c#SPLIT_SINGLE_VALUE](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416)
- [nbtree.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059)
- [nbtree.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1170)
- [reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)
- [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214)
- [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)
- [freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L540-L574)
- [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669)
- [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)
- [pg_class.h#RELKIND_PARTITIONED_INDEX](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L165-L173)
- [pg_proc.dat#pg_is_other_temp_schema](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L6448-L6450)
- [plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)
- [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)
- [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)
- [nbtree.h:29](../../../../raw/postgres-17/src/include/access/nbtree.h#L29)
- [pgstatindex.c:70](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371)
- [installation.sgml#VPATH](../../../../raw/postgres-17/doc/src/sgml/installation.sgml#L425-L436)
- [guc_tables.c#listen_addresses](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4436-L4445)
- [guc_tables.c#port](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2393-L2401)
- [guc_tables.c#unix_socket_directories](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4425-L4434)
- [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2270)
- [guc_tables.c#logging_collector](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1640-L1648)
- [guc_tables.c#fsync](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1096-L1107)
- [guc_tables.c#autovacuum](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)
- [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)
- [guc_tables.c#max_parallel_maintenance_workers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)

## Navigation

- [v17 index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [v17: Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17](btree-index-bloat-core-sql-only.md)
- [v17: Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade](btree-deduplication-after-pg-upgrade.md)
- [v17: How CREATE INDEX CONCURRENTLY Is Implemented](create-index-concurrently.md)
- [v17: How REINDEX INDEX CONCURRENTLY Is Implemented](reindex-index-concurrently.md)
- [v17: Planner Penalties for Bloated Indexes](../query-planning/bloated-indexes-query-planner.md)
- [v12: How pgstatindex Calculates B-Tree Index Statistics](../../../v12/questions/indexing/how-pgstatindex-calculates-information.md)
- [v12: Leaf Density Versus Fragmentation for Index-Scan I/O](../../../v12/questions/indexing/leaf-density-vs-fragmentation-index-scan-io.md)
