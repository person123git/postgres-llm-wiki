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
  - [How this was measured](#how-this-was-measured)
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
| `c.relkind = 'i'` | A partitioned index is `'I'` and fails the same `IS_INDEX` test: `ERROR: relation "i_part" is not a btree index` ([pg_class.h#RELKIND_PARTITIONED_INDEX](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L165-L173), [pgstattuple.out#partitioned](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L155-L171)) |
| `indisvalid AND indisready AND indislive` | On 17.11, `ERROR: index "i_invalid" is not valid`. On 12.2 the same index returns a row. See the next section ([pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250), [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)) |
| `NOT pg_is_other_temp_schema(...)` | `ERROR: cannot access temporary tables of other sessions`. Measured: one other session holding a 4.4 MB temp index moved the candidate count from 27 to 28 on 17.11 and 26 to 27 on 12.2, and the statement still returned every row with that session open ([pgstatindex.c#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L230-L238), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669)) |
| `relpersistence <> 'u' OR NOT pg_is_in_recovery()` | Untested belt and braces. The planner refuses unlogged relations during recovery ([plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)) but a function call never goes through that path, and `pgstatindex_impl` has no such guard, so on a standby it would read whatever is in the file. On a primary, unlogged indexes are read normally and one is in the fixture output |
| `pg_relation_size(c.oid) >= min_index_bytes` | Not a correctness filter, a cost filter. It is the only place a non-`pgstatindex` measurement appears, and it is a `stat()`-level answer, not a page read. Replace it with `c.relpages` for a strictly-catalog prefilter, at the price of trusting a stale number |

The index is passed by OID, not by name (`pgstatindex(c.idx_oid::regclass)`). That
is deliberate: see [Privileges](#privileges).

`cand` is `AS MATERIALIZED` — v12 syntax, and the earliest major this statement
claims — so the candidate list is fixed before the first page is read.

### The one behavioural difference between 12 and 17

**An invalid index is the only shape where the two majors disagree**, and it is
the reason the `indisvalid` filter is not optional.

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
the 16 is `MAXALIGN(sizeof(BTPageOpaqueData))`, whose four fields are two
`BlockNumber`, one `uint32` and two `uint16`
([nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71)).
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

The unfixable hole is a drop that commits while the statement is running. The
candidate list is a snapshot of the catalog; `relation_open` is not. An index
dropped three seconds into a running report killed the whole statement on both
servers:

```text
ERROR:  could not open relation with OID 16897
```

Nothing in a single statement can survive that: one failed index loses the whole
report. On a database with heavy DDL, run the report against a candidate list that
excludes tables under migration, or drive `pgstatindex` from a loop that catches
the error per index and keeps going.

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

### How this was measured

Two isolated servers on one host, each initialised for this run:

- **12.2**, `server_version_num` 120002, built from this repo's v12 pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`, with `contrib/pgstattuple` compiled
  from the same checkout through PGXS into a copy of the source tree, never into
  `raw/`.
- **17.11**, `server_version_num` 170011, from the `--with-icu --enable-debug`
  install of this page's pin `786db8dcf168bd9df8f55047337525ac19118b1c`.

Both at `block_size` 8192, `autovacuum = off`, `fsync = off`, one shared fixture
script of 24 index shapes plus one 17-only deduplication fixture, `REINDEX INDEX`
as ground truth, and `pgstatindex` as the only measurement tool — no
`pageinspect`, no `pgstattuple()`, no `amcheck`. The sandbox is retained at
`.wiki-runtime/tmp/pgsi/`.

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
| A concurrent drop aborts the whole report | measured `could not open relation with OID 16897` (17.11) and `17173` (12.2) after 6.0 s |
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

## Open Questions

- **Nothing was run on a standby.** The `relpersistence <> 'u' OR NOT
  pg_is_in_recovery()` filter is reasoning from
  [plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)
  and from the absence of any recovery guard in `pgstatindex_impl`, not from a
  measurement. What `pgstatindex` actually returns for an unlogged index on a hot
  standby is untested, and so is the whole report's behaviour there.
- **Only two minor versions were tested**, 12.2 and 17.11, both from this repo's
  pins. The invalid-index check has no release tag before `REL_17_0` in the v17
  checkout, but whether some later 12.x minor changed anything relevant was not
  checked and cannot be checked from this page's evidence base.
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
