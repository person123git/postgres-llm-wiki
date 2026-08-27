---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: claude-opus-5-max 2026-08-27T15:31:08Z
---

# A COMMENT-Stored Baseline and Normalized Index Growth for Finding GIN Indexes That Need REINDEX CONCURRENTLY in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
  - [Prompt corrections](#prompt-corrections)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [Why a GIN index's physical size only ever goes up](#why-a-gin-indexs-physical-size-only-ever-goes-up)
  - [Why the baseline must not use the GIN index's own reltuples](#why-the-baseline-must-not-use-the-gin-indexs-own-reltuples)
  - [What the COMMENT stores](#what-the-comment-stores)
  - [The comment format](#the-comment-format)
  - [SQL 1: record the baseline](#sql-1-record-the-baseline)
  - [SQL 2: read the baseline back](#sql-2-read-the-baseline-back)
  - [SQL 3 and 4: the ratios and the verdict](#sql-3-and-4-the-ratios-and-the-verdict)
  - [End-to-end run of the published statements](#end-to-end-run-of-the-published-statements)
  - [Re-verification on a second 17.11 build](#re-verification-on-a-second-1711-build)
  - [The verdict ladder, in order](#the-verdict-ladder-in-order)
  - [Why the shrinkage rule can never fire on its own](#why-the-shrinkage-rule-can-never-fire-on-its-own)
  - [Test methodology](#test-methodology)
  - [The 12 cells and their results](#the-12-cells-and-their-results)
  - [How well normalized_index_growth predicts reclaimable bytes](#how-well-normalized_index_growth-predicts-reclaimable-bytes)
  - [First failure: a fresh GIN build is not linear in heap tuples](#first-failure-a-fresh-gin-build-is-not-linear-in-heap-tuples)
  - [Second failure: the pending-list high-water mark](#second-failure-the-pending-list-high-water-mark)
  - [The baseline itself depends on maintenance_work_mem](#the-baseline-itself-depends-on-maintenance_work_mem)
  - [The statistics counter you must not build the guard on](#the-statistics-counter-you-must-not-build-the-guard-on)
  - [Edge cases proven on the server](#edge-cases-proven-on-the-server)
  - [Operational notes: locks, privileges, timeouts, GUC scopes](#operational-notes-locks-privileges-timeouts-guc-scopes)
  - [Recommended thresholds](#recommended-thresholds)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

### Prompt corrections

Filed after prompt-hygiene correction, at the asker's request. The original prompt wrote
`agents.md` for `AGENTS.md` and lowercase `postgresql` for `PostgreSQL`, put a space
before the comma after the final requirement, spliced an operational instruction
(`before anything perform a cleanup of .wiki-runtime`) into the question text, and wrote
the threshold range `75–80%` with an en dash. The asker chose "correct and restate". The
corrected text is below; the `.wiki-runtime` cleanup instruction is carried out but is
not part of the question.

Follow `AGENTS.md`, in PostgreSQL 17. Question:

Design a PostgreSQL heuristic to identify GIN indexes that may need
`REINDEX CONCURRENTLY`, using only catalog/metadata information and storing the baseline
in the index `COMMENT`. Do not create any new tables.

At index creation or immediately after a successful reindex, store:

- Baseline physical index size
- Baseline heap `reltuples`

During evaluation, collect:

- Current physical index size
- Current heap `reltuples`

Calculate:

```text
index_size_ratio =
    current_index_size / baseline_index_size

heap_tuple_ratio =
    current_heap_reltuples / baseline_heap_reltuples

normalized_index_growth =
    index_size_ratio / heap_tuple_ratio
```

Use the normalized value to distinguish legitimate index growth caused by table growth
from disproportionate GIN growth.

Example:

```text
Baseline:
index size = 10 GB
heap tuples = 10M

Current:
index size = 15 GB
heap tuples = 10M

normalized_index_growth = 1.50
```

This should be considered a potential `REINDEX` candidate.

If both the index and heap grow by 50%:

```text
index size: 10 GB -> 15 GB
heap tuples: 10M -> 15M

normalized_index_growth = 1.0
```

Do not recommend `REINDEX` based on size growth alone.

Also detect large table shrinkage. For example:

```text
heap tuples <= 50% of baseline
AND
index size remains >= 75-80% of baseline
```

This should be considered a strong `REINDEX` candidate because the indexed population has
fallen substantially while the physical GIN index has not shrunk proportionally.

Requirements:

- Do not use GIN index `pg_class.reltuples` as the tuple-count baseline.
- Use the heap/table `reltuples` instead.
- Treat thresholds as heuristics for identifying candidates, not proof of bloat.
- Prefer evaluating only after meaningful table activity/churn.
- Store all baseline metadata inside the index `COMMENT`.
- After a successful `REINDEX`, replace the `COMMENT` baseline with the new index size and
  heap tuple count.
- Provide SQL examples for recording the baseline, reading it, calculating the ratios, and
  determining whether the index is a `REINDEX` candidate.

## Answer

### Verdict

The design works, and it is buildable exactly as specified: four plain SQL statements, no
new table, no extension, and the whole baseline living in an `@ginbase:` JSON payload
appended to the index's own `COMMENT ON INDEX`. On a 12-cell matrix run three times end to
end against a 17.11 server built from the pin, with `REINDEX INDEX` as ground truth, it
made the right call on **10 of 12 cells**, and `1 - 1/normalized_index_growth` predicted
the actually-reclaimed fraction to **within 0.05 percentage points on 5 of the 9 evaluable
cells** (83.82 against 83.82, 54.63 against 54.63, 57.18 against 57.14, 11.91 against
11.95, 0.00 against 0.00).

The three statements published below were then extracted from this page and re-run **verbatim**
in a fourth database against a differently named index; the whole
capture -> churn -> evaluate -> rebuild -> re-capture lifecycle worked, and the prediction came
out at **55.66% against a measured 55.66%**. See
[End-to-end run of the published statements](#end-to-end-run-of-the-published-statements).

Four results contradict the brief and are the reason to read the rest of this page.

1. **The `index size remains >= 75-80% of baseline` clause is vacuous for GIN.** A GIN
   index's file never shrinks without a rebuild, so `index_size_ratio >= 1` always holds
   on an un-rebuilt index. The shrinkage rule therefore reduces to `heap_tuple_ratio <= 0.5`,
   which by itself forces `normalized_index_growth >= 2.0`. The shrinkage rule is a strict
   subset of the growth rule and can never fire independently. It is worth keeping only as
   a severity label.
2. **"Both grew by 50%, so normalized = 1.0, so do nothing" does not happen for GIN.**
   Measured: growing the table 50% by ordinary `INSERT` grew the index **120%** (20,594,688
   -> 45,391,872 bytes), giving `normalized_index_growth` 1.4694 — and `REINDEX` then
   reclaimed **39.51%**. Incremental GIN insertion is much less space-efficient than a bulk
   build, so proportional table growth still leaves a real reclaim opportunity that a 1.50
   threshold classes only as `watch`.
3. **A fresh GIN build's size is not a linear function of heap tuples, so `heap_tuple_ratio`
   is a poor normalizer.** Rebuilt over the same 10,000-key universe the same index measured
   7,495,680 / 13,729,792 / 20,594,688 / 82,329,600 / 82,329,600 / 246,169,600 bytes at
   250k / 500k / 1M / 2M / 4M / 8M rows. It **quadruples** between 1M and 2M rows, when every
   posting list converts to a posting tree, then adds **zero bytes** from 2M to 4M.
4. **The heuristic has a systematic blind spot on `fastupdate = on` indexes**, and its size
   is `gin_pending_list_limit`, not a fraction of anything. Adding 150,000 rows (+25%) to a
   600,000-row table grew the `fastupdate = on` index by 515 pages (4,218,880 bytes) and the
   `fastupdate = off` index by **nothing at all**; after `VACUUM` flushed the pending list
   those 515 pages went to the free space map and stayed in the file, and `REINDEX` returned
   both indexes to exactly 16,474,112 bytes. `normalized_index_growth` read 1.0049 on that
   cell while 20.39% was reclaimable.

Treat the output as a ranked shortlist, never as proof of bloat — which is what the brief
asks for, and what the same-version documentation asks for too: "The potential for bloat in
non-B-tree indexes has not been well researched. It is a good idea to periodically monitor
the index's physical size when using any non-B-tree index type."
([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046))

### Why a GIN index's physical size only ever goes up

This is the premise that makes a stored size baseline meaningful, and it is decided in one
function. `ginvacuumcleanup` walks every block, hands each recyclable page to the free space
map, updates the metapage counters, and vacuums the FSM
([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L803)).
A page is recyclable when it is new, or deleted with a delete-xid no longer visible to any
backend
([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)).
Pending-list pages are recycled the same way, by `ginInsertCleanup` and its FSM vacuum
([ginfast.c:1015-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1015-L1020)).

Nothing in `src/backend/access/gin/` calls `RelationTruncate`. Freed GIN pages are returned
to the index for reuse and never to the filesystem, so `pg_relation_size` on a GIN index is a
high-water mark. `pg_relation_size(regclass)` is the `main` fork only
([system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)),
and it is computed by stat-ing the segment files rather than read from a catalog counter
([dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L302-L343)),
so it reflects the file, not an estimate.

`REINDEX` is what shrinks it. Plain `REINDEX INDEX` keeps the index's OID and gives it a new
relfilenode
([index.c:3781-3789](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)),
which is both why the comment survives and how a rebuild is detected.

Two consequences, both measured below: `index_size_ratio` cannot fall below 1 on an
un-rebuilt index, and the only `index_size_ratio` values below 1 seen anywhere in this
work (0.5334, twice) appeared *after* the ground-truth `REINDEX`, where the rebuild detector
correctly refused to score them.

### Why the baseline must not use the GIN index's own reltuples

The brief forbids it. The source says why: three commands write an index's
`pg_class.reltuples` and they mean three different things. `reltuples` is documented as
"# of tuples (not always up-to-date; -1 means \"unknown\")"
([pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)).

- **`CREATE INDEX` / `REINDEX`** write the access method's own `index_tuples` through
  `index_update_stats`
  ([index.c:3126-3135](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135)).
  For GIN that count is extracted entries, not rows: `ginHeapTupleBulkInsert` does
  `buildstate->indtuples += nentries`
  ([gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L252-L274)),
  and `ginBuildCallback` calls it once per indexed column per heap tuple
  ([gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L288)),
  so the total is summed over rows *and* over columns. `ginbuild` returns it as
  `result->index_tuples`
  ([gininsert.c:418-428](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L418-L428)).
- **`ANALYZE`** overwrites every index unconditionally with `ceil(tupleFract * totalrows)`,
  and `tupleFract` is initialised to `1.0` for a plain non-partial index
  ([analyze.c:439-449](../../../../raw/postgres-17/src/backend/commands/analyze.c#L439-L449),
  [analyze.c:647-663](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)),
  so the index inherits the *table's* row estimate.
- **`VACUUM`** *may* write the AM's `num_index_tuples`. The AM produces it in
  `vac_cleanup_one_index`
  ([vacuum.c#vac_cleanup_one_index](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2564-L2583)),
  and GIN sets it to the heap tuple count with an explicit `XXX` admitting the value is
  wrong: "we always report the heap tuple count as the number of index entries. This is
  bogus if the index is partial, but it's real hard to tell how many distinct heap entries
  are referenced by a GIN index."
  ([ginvacuum.c:733-739](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L733-L739))
  The `pg_class` write itself happens later, in `update_relstats_all_indexes`
  ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3072-L3099)),
  and it is **skipped entirely** when the AM's count is flagged as an estimate:
  `if (istat == NULL || istat->estimated_count) continue;`
  ([vacuumlazy.c:3086-3087](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3086-L3087)).

Measured on one 200,000-row table with 5 tags per row, the same GIN index's own `reltuples`
takes four values in four commands, a 10x swing, while the table's stays coherent:

| stage | table `reltuples` | GIN index `reltuples` | entries actually indexed |
|---|---|---|---|
| after `CREATE INDEX` | 200000 | **1000000** | 1,000,000 |
| after `ANALYZE` | 200000 | **200000** | 1,000,000 |
| after `DELETE` 50% + `VACUUM` | 100000 | **100000** | 500,000 |
| after `REINDEX` | 100000 | **500000** | 500,000 |

The table's `reltuples` is the only one of the two that means the same thing after all four
commands, which is exactly why the brief pins the denominator to it.

There is a fifth value the index can hold, which is **the one it held before**. Because the
`pg_class` write is guarded by `estimated_count`, and `estimated_count` is true whenever the
heap scan skipped a page — `vacrel->scanned_pages < vacrel->rel_pages`
([vacuumlazy.c:2356](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2356)),
handed to the AM through `ivinfo`
([vacuumlazy.c:2481](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2481)) and
copied straight back out by GIN
([ginvacuum.c:733-739](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L733-L739)) —
VACUUM is a *conditional* writer, and the steady state does not write at all. Once a table's
pages are all-visible an ordinary `VACUUM` skips them and leaves the index's `reltuples`
frozen at whatever the previous writer left. Measured by carrying the same 100,000-row table
one step further, from the `500000` its `REINDEX` had just written: a second plain `VACUUM`
left it at **500000**, and only `VACUUM (DISABLE_PAGE_SKIPPING)`, which forces every page to
be scanned, wrote the heap count **100000**. So the row above labelled
`after DELETE 50% + VACUUM` depends on that `VACUUM` having had dead tuples to chase across
the whole heap; a routine one on a quiet table writes nothing.

### What the COMMENT stores

Nine fields, all obtainable from catalogs and `pg_stat_all_tables`. The brief mandates the
first two; each of the rest exists because a measured failure needed it.

| key | meaning | why it is there |
|---|---|---|
| `v` | payload format version | lets a later ladder change re-read old payloads |
| `ts` | UTC capture time | human triage only; never used in a comparison |
| `bis` | baseline index size, bytes | the brief's baseline physical index size |
| `bhr` | baseline table `reltuples` | the brief's baseline heap `reltuples` |
| `bfn` | baseline index `relfilenode` | detects a rebuild that did not refresh the baseline |
| `bti` | baseline `n_tup_ins` | churn gate |
| `btu` | baseline `n_tup_upd` | churn gate |
| `btd` | baseline `n_tup_del` | churn gate |
| `bac` | baseline `analyze_count + autoanalyze_count` | proves an `ANALYZE` ran since capture, so `bhr` and the current `reltuples` are comparable |

The churn and analyze counters come from `pg_stat_all_tables`
([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703)),
the sizes from `pg_relation_size`, and `bfn` from `pg_class.relfilenode`.

`bac` is not decoration. It is the only guard that caught the one cell where the ratios were
silently meaningless: a table 60% deleted with no `ANALYZE`, whose `reltuples` still read the
pre-delete 1000000, giving `heap_tuple_ratio` 1.0000 and `normalized_index_growth` 1.0000
while `REINDEX` reclaimed 46.66%.

Deliberately **not** stored: the baseline `n_live_tup` / `n_dead_tup` (no decision power that
`reltuples` does not already carry), and `n_mod_since_analyze` — see
[The statistics counter you must not build the guard on](#the-statistics-counter-you-must-not-build-the-guard-on).

### The comment format

The payload is appended to whatever a human already wrote, on its own line, behind an
`@ginbase:` tag:

```text
owner: search-team
ticket: OPS-1234 {do not drop} @ 100%
@ginbase:{"v": 1, "ts": "2026-08-27T14:29:23Z", "bac": 1, "bfn": "16900", "bhr": 300000, "bis": 8241152, "btd": 0, "bti": 300000, "btu": 0}
```

Measured sizes: the JSON payload is **133 bytes** on the 11 `int[]` cells, where `bhr` and
`bti` are both `1000000`, and **130 bytes** on the `tsvector` cell and on the example printed
above, whose counters are one digit shorter. The `@ginbase:` tag adds 9 bytes and the
separating newline 1, and the two-line human note above is 56 bytes, so the whole comment is
**142 bytes** with a 133-byte payload and no human note, and **199 bytes** with one — or 139
and 196 for a 130-byte payload, which is the 196 that edge case E1 reports below.
`pg_description.description` is `text` and TOAST-able
([pg_description.h:48-66](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L66)),
so size is not a constraint.

Two format rules matter:

- **The JSON must stay flat.** Both the reader and the stripper match `\{[^}]*\}`, which
  stops at the first `}`. A nested object would truncate the payload. Nine scalar fields is
  the whole design budget.
- **Anchoring on `@ginbase:` is what makes a human comment safe.** The two-line note above
  contains both `{do not drop}` and `@ 100%`, and survived capture, `REINDEX`,
  `REINDEX CONCURRENTLY` and re-capture byte for byte.

`bfn` is rendered as a JSON *string* (`"16900"`) because `jsonb_build_object` renders `oid`
that way; the reader casts it back with `::oid`.

### SQL 1: record the baseline

Run at index creation, and again immediately after every successful rebuild. This is a
`SELECT` that *returns* the `COMMENT` statement, executed with psql's `\gexec`; when the
index is not GIN, or the table has no usable `reltuples`, it returns a `DO` block that
raises instead, so a refusal is loud rather than silent.

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

SELECT /* wiki_gin_capture_baseline */
       CASE
         WHEN am.amname <> 'gin' THEN
           format('DO $$ BEGIN RAISE EXCEPTION %L; END $$',
                  'not a GIN index: ' || i.indexrelid::regclass::text)
         WHEN ct.reltuples <= 0 THEN
           format('DO $$ BEGIN RAISE EXCEPTION %L; END $$',
                  'refusing baseline for ' || i.indexrelid::regclass::text ||
                  ': table reltuples is ' || ct.reltuples || ' (run ANALYZE first)')
         ELSE
           format('COMMENT ON INDEX %s IS %L',
                  i.indexrelid::regclass::text,
                  btrim(
                    btrim(regexp_replace(coalesce(obj_description(i.indexrelid, 'pg_class'), ''),
                                         '\s*@ginbase:\{[^}]*\}', '', 'g'))
                    || E'\n@ginbase:' ||
                    jsonb_build_object(
                      'v',   1,
                      'ts',  to_char(now() AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"'),
                      'bis', pg_relation_size(i.indexrelid),
                      'bhr', ct.reltuples::bigint,
                      'bfn', ci.relfilenode,
                      'bti', coalesce(s.n_tup_ins, 0),
                      'btu', coalesce(s.n_tup_upd, 0),
                      'btd', coalesce(s.n_tup_del, 0),
                      'bac', coalesce(s.analyze_count, 0) + coalesce(s.autoanalyze_count, 0)
                    )::text))
       END
  FROM pg_index i
  JOIN pg_class ci ON ci.oid = i.indexrelid
  JOIN pg_class ct ON ct.oid = i.indrelid
  JOIN pg_am    am ON am.oid = ci.relam
  LEFT JOIN pg_stat_all_tables s ON s.relid = i.indrelid
 WHERE i.indexrelid = 'public.orders_tags_gin'::regclass
\gexec

COMMIT;
```

The re-capture after a rebuild is the same statement. It is idempotent because the
`regexp_replace` strips any previous `@ginbase:` payload before appending the new one, which
is what satisfies the brief's "replace the `COMMENT` baseline" requirement without losing the
human text.

No `ANALYZE` is needed at `t0`: `CREATE INDEX` writes the *table's* `reltuples` itself, via
the same `index_update_stats` call
([index.c:3126-3131](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3131)).
Measured on an 80,000-row table: `reltuples` reads `-1` before any index exists and `80000`
straight after `CREATE INDEX`, so a baseline captured at index creation is already valid.

### SQL 2: read the baseline back

```sql
SELECT /* wiki_gin_read_baseline */
       i.indexrelid::regclass                                  AS index_name,
       b.payload->>'ts'                                        AS baseline_taken,
       (b.payload->>'bis')::bigint                             AS baseline_index_size,
       (b.payload->>'bhr')::bigint                             AS baseline_heap_reltuples,
       (b.payload->>'bfn')::oid                                AS baseline_filenode,
       (b.payload->>'bti')::bigint                             AS baseline_n_tup_ins,
       (b.payload->>'btu')::bigint                             AS baseline_n_tup_upd,
       (b.payload->>'btd')::bigint                             AS baseline_n_tup_del,
       (b.payload->>'bac')::bigint                             AS baseline_analyze_count,
       btrim(regexp_replace(coalesce(obj_description(i.indexrelid, 'pg_class'), ''),
                            '\s*@ginbase:\{[^}]*\}', '', 'g')) AS human_comment
  FROM pg_index i
  CROSS JOIN LATERAL (
       SELECT substring(coalesce(obj_description(i.indexrelid, 'pg_class'), '')
                        from '@ginbase:(\{[^}]*\})')::jsonb AS payload
  ) b
 WHERE i.indexrelid = 'public.orders_tags_gin'::regclass;
```

`obj_description(oid, 'pg_class')` reads `pg_description` for `objsubid = 0`
([system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301)).

### SQL 3 and 4: the ratios and the verdict

One statement, because the verdict is a function of the ratios and splitting them means
computing the payload twice. The three ratios the brief specifies are in the `r` lateral; the
verdict ladder is in the `v` lateral. `stats_lag` is reported but deliberately not used as a
veto.

```sql
BEGIN;
SET LOCAL statement_timeout = '60s';
SET LOCAL lock_timeout = '5s';

SELECT /* wiki_gin_reindex_candidates */
       i.indexrelid::regclass                       AS index_name,
       pg_size_pretty(m.cur_index_size)             AS cur_size,
       pg_size_pretty(m.base_index_size)            AS base_size,
       m.cur_heap_reltuples::bigint                 AS cur_heap_reltuples,
       m.base_heap_reltuples,
       round(r.index_size_ratio::numeric, 4)        AS index_size_ratio,
       round(r.heap_tuple_ratio::numeric, 4)        AS heap_tuple_ratio,
       round(r.normalized_index_growth::numeric, 4) AS normalized_index_growth,
       round(r.churn_ratio::numeric, 4)             AS churn_ratio,
       round(r.stats_lag::numeric, 3)               AS stats_lag,
       v.verdict,
       CASE WHEN r.normalized_index_growth > 1
            THEN pg_size_pretty((m.cur_index_size
                                 * (1 - 1 / r.normalized_index_growth))::bigint)
       END                                          AS est_reclaimable
  FROM pg_index i
  JOIN pg_class ci ON ci.oid = i.indexrelid
  JOIN pg_class ct ON ct.oid = i.indrelid
  JOIN pg_am    am ON am.oid = ci.relam
  LEFT JOIN pg_stat_all_tables s ON s.relid = i.indrelid
  CROSS JOIN LATERAL (
        SELECT substring(coalesce(obj_description(i.indexrelid, 'pg_class'), '')
                         from '@ginbase:(\{[^}]*\})')::jsonb AS p
  ) b
  CROSS JOIN LATERAL (
        SELECT pg_relation_size(i.indexrelid)     AS cur_index_size,
               ct.reltuples                       AS cur_heap_reltuples,
               (b.p->>'bis')::bigint              AS base_index_size,
               (b.p->>'bhr')::bigint              AS base_heap_reltuples,
               (b.p->>'bfn')::oid                 AS base_filenode,
               (b.p->>'bac')::bigint              AS base_analyze_count,
               coalesce(s.analyze_count, 0)
                 + coalesce(s.autoanalyze_count, 0)             AS cur_analyze_count,
               coalesce(s.n_tup_ins, 0) - (b.p->>'bti')::bigint AS d_ins,
               coalesce(s.n_tup_upd, 0) - (b.p->>'btu')::bigint AS d_upd,
               coalesce(s.n_tup_del, 0) - (b.p->>'btd')::bigint AS d_del
  ) m
  CROSS JOIN LATERAL (
        SELECT m.cur_index_size::float8 / nullif(m.base_index_size, 0)         AS index_size_ratio,
               m.cur_heap_reltuples::float8 / nullif(m.base_heap_reltuples, 0) AS heap_tuple_ratio,
               (m.cur_index_size::float8 / nullif(m.base_index_size, 0))
                 / nullif(m.cur_heap_reltuples::float8
                          / nullif(m.base_heap_reltuples, 0), 0)               AS normalized_index_growth,
               (m.d_ins + m.d_upd + m.d_del)::float8
                 / nullif(m.base_heap_reltuples, 0)                            AS churn_ratio,
               coalesce(s.n_mod_since_analyze, 0)::float8
                 / nullif(m.cur_heap_reltuples, 0)                             AS stats_lag
  ) r
  CROSS JOIN LATERAL (
        SELECT CASE
                 WHEN b.p IS NULL                        THEN 'no baseline: capture one'
                 WHEN NOT i.indisvalid                   THEN 'invalid index: rebuild for validity, not for size'
                 WHEN ci.relfilenode <> m.base_filenode  THEN 'rebuilt since baseline: re-capture'
                 WHEN m.d_ins < 0 OR m.d_upd < 0 OR m.d_del < 0
                                                         THEN 'counters reset: re-capture'
                 WHEN m.cur_heap_reltuples <= 0          THEN 'no table statistics: ANALYZE first'
                 WHEN r.churn_ratio < 0.20               THEN 'insufficient churn: not evaluated'
                 WHEN m.cur_analyze_count <= m.base_analyze_count
                                                         THEN 'no ANALYZE since baseline: ANALYZE first'
                 WHEN r.heap_tuple_ratio <= 0.50
                      AND r.index_size_ratio >= 0.75     THEN 'strong candidate: indexed population collapsed'
                 WHEN r.normalized_index_growth >= 1.50  THEN 'candidate: disproportionate growth'
                 WHEN r.normalized_index_growth >= 1.20  THEN 'watch'
                 ELSE 'no action'
               END AS verdict
  ) v
 WHERE am.amname = 'gin'
   AND ci.relpersistence <> 't'
 ORDER BY r.normalized_index_growth DESC NULLS LAST;

COMMIT;
```

`indisvalid` is the "valid for use by queries" flag
([pg_index.h:42](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42)); an invalid
GIN index left by a failed `CREATE INDEX CONCURRENTLY` must be rebuilt for correctness, and
scoring its size is beside the point.

### End-to-end run of the published statements

The three blocks above were extracted from this page programmatically and run **verbatim** —
41, 18 and 76 lines — in a fourth, freshly created database, against a differently named index
(`orders_tags_gin` on a 600,000-row `orders` table) carrying the same two-line human comment.
The whole lifecycle, in order:

| step | result |
|---|---|
| capture (statement 1) | `bis` 16474112, `bhr` 600000, `bfn` 17273, `bti` 600000, `bac` 0; human comment preserved |
| read (statement 2) | all nine fields round-trip; `human_comment` returns the two original lines only |
| churn: two full-table `UPDATE`s + `VACUUM (ANALYZE)` | index 16,474,112 -> 37,150,720 bytes |
| evaluate (statement 3) | `index_size_ratio` 2.2551, `heap_tuple_ratio` 1.0000, `normalized_index_growth` 2.2551, `churn_ratio` 2.0000, verdict `candidate: disproportionate growth`, `est_reclaimable` 20 MB |
| ground truth: `REINDEX INDEX` | 37,150,720 -> 16,474,112 = **55.66% reclaimed**, against a predicted **55.66%** |
| re-capture (statement 1 again) | `bfn` 17273 -> 17278, `bis` back to 16474112, `btu` now 1200000, `bac` 2, human comment still intact |
| re-evaluate | `insufficient churn: not evaluated` — the baseline is clean again |

`1 - 1/2.2551 = 55.66%` against a measured 55.66% is the model at its best: the population never
changed, so `heap_tuple_ratio` is exactly 1 and the rebuild returned to the baseline byte for
byte (16,474,112 both times).

### Re-verification on a second 17.11 build

Everything above was first measured on a server that was deleted after filing. On 2026-08-27 the
page was reviewed against a **second** 17.11 built out of tree from the same pin, with the same
`configure` line and the same cluster settings. The published statements were re-extracted from
this page — the extractor still finds exactly three `wiki_gin` blocks, at 41, 18 and 76 lines —
and re-run verbatim against a new `orders_tags_gin`.

**The fixture SQL was never published, so the absolute byte sizes did not come back.** The
re-derived 600,000-row table built a 14,737,408-byte index where the original built 16,474,112,
and every figure downstream of that base moved with it. What did come back is everything that
does not depend on the fixture's size:

| claim | filed | re-measured | verdict |
|---|---|---|---|
| published blocks found, and their line counts | 3, at 41 / 18 / 76 | 3, at 41 / 18 / 76 | identical |
| all three statements run verbatim | yes | yes | identical |
| capture writes 9 fields, human note preserved | `bac` 0, `bti` 600000, `btu` 0, `btd` 0 | `bac` 0, `bti` 600000, `btu` 0, `btd` 0 | identical |
| read round-trips, `human_comment` is the two lines only | yes | yes | identical |
| `churn_ratio` after two full-table `UPDATE`s | 2.0000 | 2.0000 | identical |
| `heap_tuple_ratio` | 1.0000 | 1.0000 | identical |
| verdict | `candidate: disproportionate growth` | `candidate: disproportionate growth` | identical |
| `btu` after re-capture | 1200000 | 1200000 | identical |
| re-evaluate after re-capture | `insufficient churn: not evaluated` | `insufficient churn: not evaluated` | identical |
| payload count after re-capture | one | one | identical |
| `index_size_ratio` = `normalized_index_growth` | 2.2551 | 2.5575 | fixture-dependent |
| predicted against measured reclaim | 55.66 / 55.66 | **60.90 / 60.88** | within 0.02 points |

The last row is the claim that matters, and it survives the fixture change: with the population
unchanged the prediction is an identity, and it landed 0.02 points off. It missed by 0.02 rather
than 0.00 because the rebuild came back **one page larger** than the baseline (14,745,600 against
14,737,408) instead of exactly equal — the same one-page effect the filed run saw on c10, rather
than the exact return it saw on c02 and c12.

Re-measured at the same time, and **identical to the filed figures**: the four-value `reltuples`
swing (1000000 / 200000 / 100000 / 500000), and the whole `fastupdate` pairing — 310 pending
pages, **515** free pages in the FSM, **4,218,880 bytes** of growth on the `fastupdate = on`
index and **not one byte** on the `fastupdate = off` one. Those three numbers are set by
`gin_pending_list_limit`, not by the fixture, which is exactly what the page claims about them.
Six edge cases reproduced with byte-identical output, including both refusal strings
(`not a GIN index: fresh_btree`, `refusing baseline for t0_gin: table reltuples is -1 (run
ANALYZE first)`), the single `ShareUpdateExclusiveLock` on the index and none on the table, and
`REINDEX INDEX CONCURRENTLY` moving the comment to a new index OID with a stale `bfn` so the
rebuild detector fires ahead of the churn gate.

Two corrections came out of the review and are already applied above: the comment-size
arithmetic under [The comment format](#the-comment-format), and the `VACUUM` writer under
[Why the baseline must not use the GIN index's own reltuples](#why-the-baseline-must-not-use-the-gin-indexs-own-reltuples).
A third finding is unresolved and is filed as open question 14.

### The verdict ladder, in order

Order is load-bearing. Two orderings were measured and one was wrong.

1. `no baseline: capture one`
2. `invalid index: rebuild for validity, not for size`
3. `rebuilt since baseline: re-capture` — `relfilenode <> bfn`
4. `counters reset: re-capture` — any of the three churn deltas is negative
5. `no table statistics: ANALYZE first` — `reltuples <= 0`
6. `insufficient churn: not evaluated` — `churn_ratio < 0.20`
7. `no ANALYZE since baseline: ANALYZE first`
8. `strong candidate: indexed population collapsed`
9. `candidate: disproportionate growth` — `normalized_index_growth >= 1.50`
10. `watch` — `normalized_index_growth >= 1.20`
11. `no action`

Steps 6 and 7 were originally the other way round, and the no-churn control cell then
reported `no ANALYZE since baseline: ANALYZE first`, which is true but useless advice for a
table nothing has happened to. With the churn gate first it reports
`insufficient churn: not evaluated`, and the cell that genuinely needs an `ANALYZE` (60%
deleted, `churn_ratio` 0.6000) still reaches step 7. This is the one ladder change between the
second and third matrix runs, and it is the only difference in their output.

`churn_ratio` is normalized by the baseline `reltuples`, not by the current one, so a table
that has shrunk still shows the churn that shrank it.

### Why the shrinkage rule can never fire on its own

The brief's strong-candidate rule is `heap_tuple_ratio <= 0.50 AND index_size_ratio >= 0.75`.
On GIN the second clause is always true, because the file never shrinks without a rebuild
([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L803)),
and a rebuild is caught one rung higher by the `relfilenode` test. So the rule collapses to
`heap_tuple_ratio <= 0.50`, and with `index_size_ratio >= 1`:

```text
normalized_index_growth = index_size_ratio / heap_tuple_ratio >= 1 / 0.50 = 2.00
```

Every input that satisfies the shrinkage rule already satisfies `normalized_index_growth >= 1.50`
with margin to spare. Using the 80% variant of the threshold changes nothing:
`0.80 / 0.50 = 1.60`, still above 1.50.

Measured confirmation: the mass-delete cell landed at `index_size_ratio` 1.0000,
`heap_tuple_ratio` 0.4000, `normalized_index_growth` 2.5000 — flagged by both rules — and the
only sub-1 `index_size_ratio` anywhere in the run (0.5334) occurred after the ground-truth
`REINDEX`, on rows the ladder had already diverted to `rebuilt since baseline: re-capture`.

Keep the rule, because "the indexed population collapsed" is more actionable than
"disproportionate growth" and it costs one `AND`. Do not expect it to detect anything the
growth rule misses.

### Test methodology

- PostgreSQL **17.11** built out of tree from `raw/postgres-17/` at pin
  `786db8dcf168bd9df8f55047337525ac19118b1c`, `configure --without-icu --without-readline
  --with-zlib --enable-debug`, contrib included.
- Isolated cluster: `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`,
  `work_mem = 64MB`, `autovacuum = off` (so that every `VACUUM` and `ANALYZE` in a cell is
  the one the script ran), `listen_addresses = ''`, unix socket only.
- 12 cells, one table and one GIN index each. Base population 1,000,000 rows with 5 `int`
  tags drawn from a 10,000-value universe by a deterministic hash, except the `tsvector` cell
  at 400,000 rows. All 12 baseline indexes measured **20,594,688 bytes** (the `tsvector` one
  8,495,104), identical across all three runs.
- Sequence per run: snapshot -> capture baselines -> churn -> snapshot -> **evaluate** ->
  `REINDEX INDEX` every cell -> snapshot. Ground truth is
  `1 - post_reindex_bytes / post_churn_bytes`. `VACUUM FULL` was never used.
- `pageinspect` (`gin_metapage_info`, `get_raw_page`), `pgstattuple` and
  `pg_freespacemap` were used for mechanism probes only; the heuristic itself reads nothing
  but catalogs, `pg_stat_all_tables` and `pg_relation_size`.
- **The matrix ran three times, in three freshly created databases.** Runs 2 and 3 produced
  byte-identical ground-truth tables; the baseline payloads differ only in `ts` and `bfn`
  (24 lines of diff, all timestamp or filenode). Run 1 differed from runs 2 and 3 in nothing
  but the two ladder fixes described above.
- **The published statements were then re-derived from this page and run verbatim** in a fourth
  database, against an index with a different name and a different base size, as described under
  [End-to-end run of the published statements](#end-to-end-run-of-the-published-statements). The
  extraction script pulls every fenced `sql` block that carries a `wiki_gin` tag and asserts there
  are exactly three, so the statements in this page are the statements that were executed.
- **On 2026-08-27 the whole page was reviewed against a second 17.11 built from the same pin**,
  with the same `configure` line and cluster settings. The fixture SQL is not published, so the
  base sizes differ and every absolute byte figure below belongs to the original run; the
  behavioural claims, the ratios that do not depend on the base, and the `gin_pending_list_limit`
  figures all reproduced. See
  [Re-verification on a second 17.11 build](#re-verification-on-a-second-1711-build).

### The 12 cells and their results

Sizes in bytes. `isr` = `index_size_ratio`, `htr` = `heap_tuple_ratio`, `norm` =
`normalized_index_growth`, `pred` = `100 * (1 - 1/norm)`.

| cell | churn applied | post-churn | after `REINDEX` | reclaimed | isr | htr | norm | pred | verdict |
|---|---|---|---|---|---|---|---|---|---|
| c01 | `INSERT` +50% rows, same keys | 45,391,872 | 27,459,584 | **39.51%** | 2.2041 | 1.5000 | 1.4694 | 31.94 | `watch` |
| c02 | two full-table `UPDATE`s | 127,311,872 | 20,594,688 | 83.82% | 6.1818 | 1.0000 | 6.1818 | 83.82 | `candidate` |
| c03 | `DELETE` 60%, `VACUUM (ANALYZE)` | 20,594,688 | 10,985,472 | 46.66% | 1.0000 | 0.4000 | 2.5000 | 60.00 | `strong candidate` |
| c04 | none | 20,594,688 | 20,594,688 | 0.00% | 1.0000 | 1.0000 | 1.0000 | 0.00 | `insufficient churn` |
| c05 | `INSERT` +100% rows, same keys | 127,311,872 | 82,329,600 | 35.33% | 6.1818 | 2.0000 | 3.0909 | 67.65 | `candidate` |
| c06 | `INSERT` +100% rows, all-new keys | 46,759,936 | 41,172,992 | 11.95% | 2.2705 | 2.0000 | 1.1352 | 11.91 | `no action` |
| c07 | `fastupdate = on`, `UPDATE` 30%, no `VACUUM` | 50,937,856 | 22,290,432 | 56.24% | 2.4733 | 1.0000 | 2.4733 | 59.57 | `candidate` |
| c08 | `DELETE` 60%, no `VACUUM`, no `ANALYZE` | 20,594,688 | 10,985,472 | **46.66%** | 1.0000 | 1.0000 | 1.0000 | 0.00 | `no ANALYZE since baseline` |
| c09 | full `UPDATE`, then out-of-band `REINDEX` | 20,594,688 | 20,594,688 | 0.00% | 1.0000 | 1.0000 | 1.0000 | — | `rebuilt since baseline` |
| c10 | `tsvector`, rewrite every document | 19,841,024 | 8,503,296 | 57.14% | 2.3356 | 1.0000 | 2.3356 | 57.18 | `candidate` |
| c11 | full `UPDATE`, then reset table counters | 127,311,872 | 20,594,688 | 83.82% | 6.1818 | 1.0000 | 6.1818 | 83.82 | `counters reset` |
| c12 | `DELETE` 50% then `INSERT` 50% fresh rows | 45,391,872 | 20,594,688 | 54.63% | 2.2041 | 1.0000 | 2.2041 | 54.63 | `candidate` |

Scoring against ground truth:

- **6 true positives.** c02, c03, c07, c10, c12 flagged, reclaiming 83.82 / 46.66 / 56.24 /
  57.14 / 54.63%; c05 flagged, reclaiming 35.33%.
- **2 true negatives.** c04 (0.00% reclaimable) gated on churn; c06 (11.95%) left at
  `no action` at `norm` 1.1352.
- **3 correct refusals.** c08 refused for want of an `ANALYZE`, and refusing was the right
  answer — its `norm` of 1.0000 would otherwise have hidden a 46.66% reclaim. c09 detected the
  out-of-band rebuild from the stored `relfilenode`. c11 detected the counter reset from a
  negative churn delta, and would have been a correct `candidate` had the baseline been fresh.
- **1 under-ranked true positive.** c01 reclaimed 39.51% but scored `norm` 1.4694, landing in
  `watch` rather than `candidate` — 0.03 below the brief's 1.50 threshold. See
  [Second failure](#second-failure-the-pending-list-high-water-mark) for the mechanism, and
  [Recommended thresholds](#recommended-thresholds) for what to do about it.

### How well normalized_index_growth predicts reclaimable bytes

`1 - 1/normalized_index_growth` is the fraction of the current file a rebuild would not need,
*if* a fresh build's size were proportional to heap tuples. Over the nine cells the ladder
actually scores:

| accuracy band | cells |
|---|---|
| within 0.05 points | c02 (83.82/83.82), c12 (54.63/54.63), c04 (0.00/0.00), c10 (57.18/57.14), c06 (11.91/11.95) |
| within 3.5 points | c07 (59.57/56.24) |
| over-predicts by 13.3 | c03 (60.00/46.66) |
| under-predicts by 7.6 | c01 (31.94/39.51) |
| over-predicts by 32.3 | c05 (67.65/35.33) |

**5 of 9 within 0.05 points is not luck, and the reason is instructive.** In c02, c10 and c12
the population is unchanged (`heap_tuple_ratio` 1.0000), so `norm` collapses to
`index_size_ratio`, and the rebuild returned to **exactly** the baseline size — 20,594,688 for
both c02 and c12, and 8,503,296 against a baseline of 8,495,104 for c10, one page apart. When
the row count and key distribution are unchanged, a fresh GIN build is byte-reproducible, and
the model is then not an estimate but an identity.

The three misses are all cells where the population changed, and they are exactly the two
failure modes below. The estimate is therefore trustworthy for pure churn and unreliable
across growth or shrinkage — which is the opposite of what a normalizer is supposed to buy
you, and is the most important caveat on this page.

### First failure: a fresh GIN build is not linear in heap tuples

Rebuilding the same index over a growing table with the same 10,000-key universe, at
`maintenance_work_mem = 256MB`:

| rows | fresh build bytes | `n_entry_pages` | `n_data_pages` |
|---|---|---|---|
| 250,000 | 7,495,680 | 914 | 0 |
| 500,000 | 13,729,792 | 1,675 | 0 |
| 1,000,000 | 20,594,688 | 2,513 | 0 |
| 2,000,000 | **82,329,600** | 49 | **10,000** |
| 4,000,000 | **82,329,600** | 49 | 10,000 |
| 8,000,000 | 246,169,600 | 49 | 30,000 |

Between 1M and 2M rows the fresh build **quadruples** while `n_data_pages` goes from 0 to
10,000 — one posting tree per key — and `n_entry_pages` collapses from 2,513 to 49. Then 2M ->
4M rows adds **zero bytes**: the same 10,000 posting-tree pages absorb twice the TIDs.

The mechanism is in the build path. GIN keeps a key's TIDs as a compressed posting list inside
the entry tuple while it fits in `GinMaxItemSize`, and converts to a posting tree when it does
not
([gininsert.c#buildFreshLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L166),
[gininsert.c#addItemPointersToLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L45-L111)).
The conversion is per key and is a step, not a slope; and the same key universe over more rows
means longer lists, so every key crosses the threshold at roughly the same table size.

Consequence for the heuristic: dividing by `heap_tuple_ratio` assumes the denominator tracks
what a fresh build would cost. Across a posting-tree conversion it does not, in either
direction. c05 is this failure — the fresh 2M-row build is 4.00x the 1M-row baseline, not 2.00x,
so `norm` 3.0909 predicted 67.65% against an actual 35.33%. c06 escapes it, and predicts to
0.04 points, precisely because doubling the *key* universe alongside the rows kept every
posting list inline (`n_data_pages` 0, `n_entry_pages` 5,025) and the fresh build came out at
41,172,992 — 2.00x the baseline, exactly linear.

### Second failure: the pending-list high-water mark

A six-point proportional-growth sweep on a 600,000-row base, growing by ordinary `INSERT` with
GIN's default `fastupdate = on` — the default is `true` both in the access method
([gin_private.h:33](../../../../raw/postgres-17/src/include/access/gin_private.h#L33)) and in the
reloption table
([reloptions.c:123-131](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)):

| heap growth | post-growth bytes | fresh build | isr | htr | norm | reclaimed | pred |
|---|---|---|---|---|---|---|---|
| +10% | 20,692,992 | 16,474,112 | 1.2561 | 1.10 | 1.1419 | **20.39%** | 12.43 |
| +25% | 20,692,992 | 16,474,112 | 1.2561 | 1.25 | 1.0049 | **20.39%** | 0.48 |
| +50% | 20,692,992 | 20,594,688 | 1.2561 | 1.50 | 0.8374 | 0.48% | -19.42 |
| +100% | 37,150,720 | 27,451,392 | 2.2551 | 2.00 | 1.1275 | **26.11%** | 11.31 |
| +200% | 118,317,056 | 82,329,600 | 7.1820 | 3.00 | 2.3940 | 30.42% | 58.23 |
| +400% | 119,070,720 | 82,329,600 | 7.2277 | 5.00 | 1.4455 | 30.86% | 30.82 |

At +10%, +25% and +100% the heuristic returns `no action` or `watch` while 20-26% of the file
is reclaimable. Note also that the post-growth size is *identical* (20,692,992) for +10%, +25%
and +50%: adding 60,000, 150,000 or 300,000 rows grew the file by exactly the same 515 pages.

A paired probe isolates the cause. Two identical 600,000-row tables, one index with
`fastupdate = on` and one with `fastupdate = off`, then 150,000 rows inserted into both:

| stage | `fastupdate = on` | pending pages | `fastupdate = off` | pending pages |
|---|---|---|---|---|
| base | 16,474,112 | — | 16,474,112 | — |
| after +150,000 inserts, before `VACUUM` | **20,692,992** | 310 | **16,474,112** | 0 |
| after `VACUUM (ANALYZE)` | 20,692,992 | 0 | 16,474,112 | 0 |
| free pages in the FSM | **515** | — | **0** | — |
| after `REINDEX` | 16,474,112 | — | 16,474,112 | — |

With `fastupdate = off`, 150,000 new rows fit in existing page slack and the file did not grow
by one byte. With `fastupdate = on` the file grew 515 pages (4,218,880 bytes, against a 4 MB
default `gin_pending_list_limit`), `VACUUM` flushed the pending list to the main structure, and
all 515 pages went to the free space map and stayed in the file. `REINDEX` returned both to the
same 16,474,112 bytes.

So the pending list contributes a roughly fixed high-water mark, bounded by
`gin_pending_list_limit`, that scales with neither the heap nor the index. On a 16 MB index that
is 25% of the file and the heuristic misses it; on a 1 GB index it is 0.4% and does not matter.
GIN's own documentation describes the pending list and its limit
([gin.sgml#gin-fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537),
[gin.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L595-L616));
the recycling of flushed pending pages into the FSM is
[ginfast.c:1015-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1015-L1020).

This also explains c01's under-ranking, and the +50% row of the sweep, where `norm` fell to
0.8374 — *below* 1 — while 0.48% was reclaimable. The verdict happened to be right; the ranking
was not.

### The baseline itself depends on maintenance_work_mem

The same 2,000,000-row index, rebuilt three times at three settings:

| `maintenance_work_mem` | fresh build bytes | `n_entry_pages` | `n_data_pages` |
|---|---|---|---|
| 64MB | **109,371,392** | 3,350 | 10,000 |
| 256MB | 82,329,600 | 49 | 10,000 |
| 1GB | 82,329,600 | 49 | 10,000 |

The 64MB row's three figures do not agree with each other by exactly one page; see open
question 14. The 32.8% conclusion below does not depend on which of them is off.

A build at 64MB is **32.8% larger** than the same build at 256MB, and 256MB and 1GB agree to the
byte. The setting is not incidental to the build: `ginBuildCallback` dumps its accumulator to the
index whenever `buildstate->accum.allocatedMemory >= (Size) maintenance_work_mem * 1024L`
([gininsert.c:290-291](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L291)),
so the budget decides how many flush rounds a build takes, and each round appends to the entry
and posting structures independently. The 64MB build shows it in the shape as well as the size:
3,350 entry pages against 49 for the same 10,000 keys.

Since `bis` is captured right after a build, the baseline inherits whatever
`maintenance_work_mem` that build ran with, and a later rebuild under a different setting will
not return to it. The same-version documentation warns that "Build time for a GIN index is very
sensitive to the `maintenance_work_mem` setting"
([gin.sgml#guc-maintenance-work-mem](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L584-L593));
the measured *size* sensitivity is the part that matters here.

Practical rule: capture the baseline in the same session, and with the same
`maintenance_work_mem`, as the build that produced it, and use that same setting for the
rebuild. `maintenance_work_mem` is `PGC_USERSET`
([guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474)),
so this is session/transaction scope — no reload, no restart.

### The statistics counter you must not build the guard on

The first ladder used `n_mod_since_analyze > 0.10 * reltuples` as a hard stale-statistics veto.
It misfired on the mass-delete cell in **every** matrix run: `stats_lag` read 1.500
(`n_mod_since_analyze` 600000 over `reltuples` 400000) even though the cell's
`VACUUM (ANALYZE)` had run and had set `reltuples` correctly to 400000, and the cell was refused
instead of being flagged as the 46.66%-reclaimable true positive it is.

The cause is a flush-ordering race, not a bug in the guard's arithmetic. `pgstat_report_analyze`
zeroes `mod_since_analyze` in the shared entry
([pgstat_relation.c:331-337](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337)),
but a backend's own pending counts are flushed separately and *additively*:
`mod_since_analyze += changed_tuples`, `live_tuples += delta_live_tuples`, then
`live_tuples = Max(live_tuples, 0)`
([pgstat_relation.c:849-867](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L849-L867)).
A non-forced flush is rate-limited to once per `PGSTAT_MIN_INTERVAL`, 1000 ms
([pgstat.c:117-122](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122),
[pgstat.c:636-655](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L655)).
A `DELETE` followed within that window by `VACUUM (ANALYZE)` therefore has its counts applied
*after* the reset, adding the whole delete back onto a counter the `ANALYZE` had just zeroed.

Reproduced independently, in a second fixture, on 1 of 4 back-to-back `DELETE` +
`VACUUM (ANALYZE)` pairs — and the corrupted reading is unmistakable:

| attempt | `n_live_tup` | `n_dead_tup` | `n_mod_since_analyze` | `pg_class.reltuples` |
|---|---|---|---|---|
| 1 | 120000 | 0 | 0 | 120000 |
| 2 | **0** | **210000** | **210000** | 90000 |
| 3 | 60000 | 0 | 0 | 60000 |
| 4 | 30000 | 0 | 0 | 30000 |

`live_tuples` went to 0 through the `Max(..., 0)` clamp while `pg_class.reltuples` — written by
`vac_update_relstats`, not by pgstat — stayed correct at 90000. Inserting a 2 s pause before the
`VACUUM (ANALYZE)` produced a clean reading every time.

The error is one-directional: the counter over-reports changes the `ANALYZE` did in fact see, so
it raises false staleness alarms and never false all-clears. That makes it safe as the advisory
`stats_lag` column and unsafe as a veto. The veto that replaced it, `cur_analyze_count <= bac`,
is immune, because `analyze_count` is incremented in the same locked section as the reset
([pgstat_relation.c:339-348](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L339-L348)).

The same race affects **baseline capture**. On an 80,000-row table indexed in the same session,
the capture recorded `bti` = 0 while 80,000 rows had been inserted, because the insert's counts
were still pending; the effect is a permanently over-stated churn delta, so the churn gate opens
earlier than intended. `SELECT pg_stat_force_next_flush();`
([pg_proc.dat:5916-5920](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5916-L5920))
in the capturing session fixes it — measured, `n_tup_ins` went 0 -> 80000 and `n_live_tup`
0 -> 80000 — but only for that backend's own pending stats. A monitoring session cannot force
another backend's flush, so baseline churn counters can lag a busy writer by up to
`PGSTAT_MIN_INTERVAL`, or `PGSTAT_MAX_INTERVAL` (60 s) in the worst case
([pgstat.c:117-122](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122)).

### Edge cases proven on the server

| # | case | result |
|---|---|---|
| E1 | two-line human comment containing `{do not drop}` and `@ 100%` | preserved through capture; comment 196 bytes, payload 130 |
| E2 | plain `REINDEX INDEX` | index OID stays 16900, relfilenode 16900 -> 16903, comment intact |
| E3 | `REINDEX INDEX CONCURRENTLY` | index OID moves 16900 -> 16904 and the comment follows, payload and human text unchanged |
| E4 | evaluate after E3 | `rebuilt since baseline: re-capture`; re-capture writes `bfn` 16904 and the next evaluation reports `insufficient churn` |
| E5 | `ALTER INDEX ... RENAME` | comment survives (same OID) |
| E6 | `COMMENT ON INDEX` lock footprint | one row in `pg_locks`: `ShareUpdateExclusiveLock` on the index, and none on the table |
| E7 | non-owner with `SELECT` and `pg_read_all_stats` | reads the baseline fine; write refused, `SQLSTATE 42501`, `must be owner of index edge_gin`; baseline intact |
| E13 | 80,000-row table, no `ANALYZE` | `reltuples` `-1` before any index, `80000` after `CREATE INDEX` (the GIN index itself reads `400000`), so capture at `t0` succeeds |
| E14 | `TRUNCATE` then capture | both `reltuples` reset to `-1`; capture refuses: `refusing baseline for t0_gin: table reltuples is -1 (run ANALYZE first)` |
| E15 | `pg_dump -t ... --section=post-data` | emits `COMMENT ON INDEX ... @ginbase:{...}`, so the baseline survives dump and restore |
| E11 | capture aimed at a B-tree index | refuses: `not a GIN index: fresh_btree` |

E3 is the case the brief's `REINDEX CONCURRENTLY` target depends on, and it works because
`index_concurrently_swap` explicitly moves the `pg_description` row from the old index OID to
the new one
([index.c:1740-1784](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784)).
The moved payload still carries the *old* `bfn`, which is why the rebuild detector fires and the
operator is told to re-capture rather than being handed a bogus ratio.

E7 follows from `CommentObject` calling `check_object_ownership` after taking
`ShareUpdateExclusiveLock`
([comment.c:66-77](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L77)),
and E6 from the same call. E15 works because a comment is an ordinary catalog row keyed on
`objoid`/`classoid`/`objsubid`
([pg_description.h:48-66](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L66)).

### Operational notes: locks, privileges, timeouts, GUC scopes

- **Capture takes `ShareUpdateExclusiveLock` on the index.** Documented
  ([comment.sgml:95-98](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L98))
  and measured (E6). That mode self-conflicts and conflicts with `VACUUM`, `ANALYZE`,
  `CREATE INDEX CONCURRENTLY` and `REINDEX CONCURRENTLY`, so a capture can be blocked by, or
  block, routine maintenance. Always set `lock_timeout`.
- **Only the index's owner can write the baseline**
  ([comment.sgml:100-109](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L100-L109)),
  so the capture job needs table ownership, not just `pg_monitor`.
- **Anyone connected to the database can read it.** "There is presently no security mechanism
  for viewing comments: any user connected to a database can see all the comments for objects in
  that database."
  ([comment.sgml:292-298](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L292-L298))
  Row counts and index sizes are not secrets in most shops, but the payload is world-readable —
  do not extend it with anything sensitive.
- **A comment is dropped with its object** and replaced wholesale by the next `COMMENT`
  ([comment.sgml:87-93](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L87-L93)),
  which is why the capture statement re-reads and re-writes the human text rather than appending
  blindly.
- **Timeouts.** Both `statement_timeout` and `lock_timeout` are `PGC_USERSET`
  ([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620),
  [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)),
  so the `SET LOCAL` values in the statements above are session/transaction scope: no reload, no
  restart.
- **`gin_pending_list_limit` is `PGC_USERSET`** too
  ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3577-L3585)),
  session/transaction scope, and can also be set per index as a storage parameter
  ([gin.sgml:610-616](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L610-L616)).
- **The evaluation is not lock-free.** `pg_relation_size` opens each relation with
  `AccessShareLock`
  ([dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371)),
  so a scan over many GIN indexes touches many relations. Keep `lock_timeout` set there too.
- **`REINDEX` is the action, not `VACUUM`.** The same-version `REINDEX` documentation lists a
  bloated index — "it contains many empty or nearly-empty pages" — as a reason to rebuild
  ([reindex.sgml:54-64](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)).

### Recommended thresholds

Starting values, to be re-calibrated per installation. These are heuristics for building a
shortlist, not a bloat measurement.

| knob | value | basis |
|---|---|---|
| `churn_ratio` gate | 0.20 | the only cell below it (c04, 0.0000) was the one with nothing to reclaim; the lowest scored cell was c07 at 0.3000 |
| `candidate` | `norm >= 1.50` | the brief's value; separates c05 (3.0909), c03 (2.5000), c07 (2.4733), c10 (2.3356), c12 (2.2041) from c06 (1.1352) |
| `watch` | `norm >= 1.20` | catches c01 at 1.4694, which reclaimed 39.51% |
| strong-candidate shrink test | `htr <= 0.50` | the `isr >= 0.75` clause is vacuous on GIN; keep it only as documentation |

If you would rather not lose c01's 39.51% to `watch`, lowering `candidate` to **1.25** flags it
while still leaving c06 (1.1352, 11.95% reclaimable) alone. That is a two-cell margin on twelve
cells and one growth sweep, so it is a suggestion, not a calibration — and the sweep shows the
1.25 threshold would still miss the +25% row at `norm` 1.0049 with 20.39% reclaimable. On
`fastupdate = on` indexes smaller than a few hundred megabytes, no threshold on this statistic
recovers the pending-list high-water mark; only a periodic rebuild, or `fastupdate = off`, does —
and note that flipping that reloption takes an `AccessExclusiveLock` on the index
([reloptions.c:123-131](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)),
so it is not an online change.

## Context Reviewed

- **GIN access method**: `ginvacuum.c` (`ginvacuumcleanup`, `ginvacuumcleanup`'s
  `num_index_tuples` assignment, `GinPageIsRecyclable`), `gininsert.c` (`ginbuild`,
  `ginBuildCallback`, `buildFreshLeafTuple`, `addItemPointersToLeafTuple`,
  `ginEntryInsert`), `ginfast.c` (`ginInsertCleanup`, pending-page FSM recycling),
  `ginutil.c` (`ginhandler` AM routine table). Confirmed by grep that no file under
  `src/backend/access/gin/` calls `RelationTruncate`.
- **Catalog and index maintenance**: `index.c` (`index_build`, `index_update_stats`,
  `reindex_index`, `index_concurrently_swap`), `pg_class.h`, `pg_index.h`,
  `pg_description.h`.
- **Statistics writers**: `analyze.c` (`do_analyze_rel`, `compute_index_stats`, `tupleFract`),
  `vacuum.c` (`vac_cleanup_one_index`), `vacuumlazy.c` (`update_relstats_all_indexes`,
  `lazy_cleanup_one_index`, the `estimated_count` guard), `pgstat_relation.c` (`pgstat_report_analyze`,
  `pgstat_report_vacuum`, `pgstat_relation_flush_cb`, `AtEOXact_PgStat_Relations`),
  `pgstat.c` (`pgstat_report_stat`, `PGSTAT_MIN_INTERVAL`, `PGSTAT_MAX_INTERVAL`).
- **Comment plumbing**: `comment.c` (`CommentObject`, `CreateComments`, `DeleteComments`),
  `system_functions.sql` (`obj_description`, `pg_relation_size`), `system_views.sql`
  (`pg_stat_all_tables`), `dbsize.c` (`calculate_relation_size`, `pg_relation_size`,
  `pg_indexes_size`), `pg_proc.dat` (`pg_stat_force_next_flush`).
- **GUCs**: `guc_tables.c` for `maintenance_work_mem`, `gin_pending_list_limit`,
  `statement_timeout`, `lock_timeout`.
- **Documentation, same checkout**: `gin.sgml` (fast update, tips), `maintenance.sgml`
  (`routine-reindex`), `ref/reindex.sgml`, `ref/comment.sgml`.
- **Server work**: 17.11 built from the pin; three full matrix runs in three fresh databases
  (`gin1`, `gin2`, `gin3`), a verbatim replay of the published statements in a fourth
  (`pubsql`), a four-stage `reltuples` probe, a six-point fresh-build linearity probe, a
  three-point `maintenance_work_mem` probe, a six-point proportional-growth sweep, a paired
  `fastupdate = on`/`off` probe with FSM readings, two pgstat-race probes, and 11 edge cases.
- **Review, 2026-08-27**: a second 17.11 built from the same pin; the three published
  statements re-extracted from this page and replayed end to end, the `reltuples` swing, the
  `fastupdate` pairing, the `estimated_count` VACUUM probe, a six-build metapage-invariant
  check, and six edge cases re-run. See
  [Re-verification on a second 17.11 build](#re-verification-on-a-second-1711-build).

## Evidence Map

| Claim | Source |
|---|---|
| A GIN index's file never shrinks; freed pages go to the FSM | [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L803), [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829), [ginfast.c:1015-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1015-L1020) |
| `pg_relation_size` is the main fork, stat-ed from the files | [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289), [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L302-L343) |
| `reltuples` is `-1` when unknown | [pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66) |
| `CREATE INDEX`/`REINDEX` write the AM's `index_tuples` into the index row and the heap count into the table row | [index.c:3126-3135](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2788-L2842) |
| For GIN that count is extracted entries, summed over rows and over indexed columns | [gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L252-L274), [gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L288), [gininsert.c:418-428](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L418-L428) |
| A GIN build flushes its accumulator when it reaches `maintenance_work_mem` | [gininsert.c:290-291](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L291) |
| `ANALYZE` overwrites every index with `ceil(tupleFract * totalrows)`, `tupleFract` = 1.0 for plain indexes | [analyze.c:439-449](../../../../raw/postgres-17/src/backend/commands/analyze.c#L439-L449), [analyze.c:647-663](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663) |
| `VACUUM` produces `num_index_tuples`, and GIN sets it to the heap tuple count | [vacuum.c#vac_cleanup_one_index](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2564-L2583), [ginvacuum.c:733-739](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L733-L739) |
| The `pg_class` write happens in `update_relstats_all_indexes`, and is skipped when the count is an estimate | [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3072-L3099), [vacuumlazy.c:3086-3087](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3086-L3087) |
| `estimated_count` is true whenever the heap scan skipped a page, and is passed to the AM | [vacuumlazy.c:2356](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2356), [vacuumlazy.c:2481](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2481) |
| Plain `REINDEX` keeps the index OID and changes the relfilenode | [index.c:3781-3789](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789) |
| `REINDEX CONCURRENTLY` moves the `pg_description` row to the new index OID | [index.c:1740-1784](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784) |
| `obj_description(oid, 'pg_class')` reads `pg_description` at `objsubid = 0` | [system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301) |
| A comment is a `text` catalog row keyed on objoid/classoid/objsubid | [pg_description.h:48-66](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L66) |
| Churn, analyze and staleness counters come from `pg_stat_all_tables` | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703) |
| `indisvalid` means "valid for use by queries" | [pg_index.h:42](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42) |
| Posting lists convert to posting trees when they outgrow `GinMaxItemSize` | [gininsert.c#buildFreshLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L166), [gininsert.c#addItemPointersToLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L45-L111) |
| The pending list exists, is bounded by `gin_pending_list_limit`, and is flushed by VACUUM/autoanalyze | [gin.sgml#gin-fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537), [gin.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L595-L616) |
| GIN build size is sensitive to `maintenance_work_mem` | [gin.sgml#guc-maintenance-work-mem](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L584-L593) |
| `ANALYZE` zeroes `mod_since_analyze` in the shared entry | [pgstat_relation.c:331-337](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337) |
| Pending relation stats are applied additively, with a clamp on live/dead | [pgstat_relation.c:849-867](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L849-L867) |
| `analyze_count` is incremented in the same locked section as the reset | [pgstat_relation.c:339-348](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L339-L348) |
| Non-forced flushes are rate-limited to 1000 ms, forced at 60000 ms | [pgstat.c:117-122](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122), [pgstat.c:636-655](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L655) |
| `pg_stat_force_next_flush()` exists and forces the next flush | [pg_proc.dat:5916-5920](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5916-L5920) |
| `COMMENT` takes `ShareUpdateExclusiveLock` and requires ownership | [comment.sgml:95-98](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L98), [comment.sgml:100-109](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L100-L109), [comment.c:66-77](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L77) |
| Comments are replaced wholesale and dropped with the object | [comment.sgml:87-93](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L87-L93) |
| Any user in the database can read any comment | [comment.sgml:292-298](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L292-L298) |
| `pg_relation_size` opens the relation with `AccessShareLock` | [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371) |
| `maintenance_work_mem`, `gin_pending_list_limit`, `statement_timeout`, `lock_timeout` are all `PGC_USERSET` | [guc_tables.c#maintenance_work_mem](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474), [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3577-L3585), [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631) |
| `fastupdate` defaults to on, and changing it needs `AccessExclusiveLock` | [gin_private.h:33](../../../../raw/postgres-17/src/include/access/gin_private.h#L33), [reloptions.c:123-131](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131) |
| Non-B-tree bloat is not well researched; monitor the physical size | [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046) |
| A bloated index is a documented reason to `REINDEX` | [reindex.sgml:54-64](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64) |

Every number in the tables above is a measurement from the 17.11 server described under
[Test methodology](#test-methodology), not a source claim.

## Open Questions

1. **The thresholds are calibrated on 12 cells, one growth sweep and one scale.** Base
   populations were 1,000,000 rows (400,000 for the `tsvector` cell) with a 10,000-value key
   universe. Nothing here says the 1.50/1.20 boundaries, or the 0.20 churn gate, transfer to a
   different key cardinality, a different tuple width, or a 100x larger table.
2. **`normalized_index_growth` is not monotone in reclaimable space.** Over the nine scored
   cells it ranks c05 (`norm` 3.0909, 35.33% reclaimed) above c03 (2.5000, 46.66%) and c07
   (2.4733, 56.24%). Whether a better normalizer exists inside the catalog-only constraint is
   untested; the obvious candidate, an entry-count term, is exactly what the brief rules out.
3. **The pending-list blind spot is characterised but not corrected.** `pgstatginindex` reports
   `pending_pages`, which would let the evaluation subtract the live pending list, but it is
   contrib and so outside the brief's catalog-only rule, and it does not report the *high-water
   mark* left behind after a flush. No catalog-only substitute was found.
4. **`fastupdate = off` was measured only in the paired probe.** The matrix used the default
   `fastupdate = on` for 11 cells and an explicit `on` for c07. How the heuristic scores a fleet
   of `fastupdate = off` GIN indexes is untested.
5. **The pgstat race is intermittent and its rate is not characterised.** It fired on 1 of 4
   deliberate attempts in one probe and on the same cell in 3 of 3 matrix runs. The rate depends
   on statement timing relative to a 1000 ms window, so the 3-of-3 figure is a property of this
   harness, not of PostgreSQL.
6. **The `counters reset` and `no baseline` arms are proven to fire, not proven correct.** c11
   shows the reset arm firing, and would have been a correct `candidate` with a fresh baseline;
   whether "re-capture" is the right advice, versus scoring on sizes alone and reporting the
   churn as unknown, is a design choice this page does not test.
7. **The `invalid index` arm was never exercised on a genuine failed `CREATE INDEX
   CONCURRENTLY`.** The intended fixture was abandoned when a simpler probe errored out, so that
   ladder rung is source-justified from `indisvalid` only.
8. **Partial and multi-column GIN indexes were not tested at all.** `ginvacuumcleanup`'s own
   comment says its entry count is "bogus if the index is partial"
   ([ginvacuum.c:733-739](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L733-L739)),
   and `ANALYZE` computes a real `tupleFract` for a partial index
   ([analyze.c:948-953](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)) —
   which means the table `reltuples` the heuristic divides by is *not* the indexed population for
   a partial index. Expect false positives there; none were measured.
9. **Expression GIN indexes, `jsonb_path_ops`, `gin_trgm_ops` and `btree_gin` were not tested.**
   Only default `array_ops` on `int[]` and default `tsvector_ops`.
10. **Everything is one cluster, one build, one filesystem.** `block_size` 8192,
    `shared_buffers` 512MB, `maintenance_work_mem` 256MB throughout except in the explicit
    `maintenance_work_mem` probe. The byte-exact reproducibility across three runs is
    repeatability on one machine, not portability.
11. **Autovacuum was off for the whole matrix.** In production, autoanalyze both refreshes
    `reltuples` (helping) and flushes GIN pending lists (moving the size reading) on its own
    schedule; the interaction between the churn gate, the `bac` gate and autovacuum's timing is
    untested.
12. **The `est_reclaimable` column can be wildly wrong and is still printed.** It read 82 MB
    against a true 45 MB on c05. It is kept because it was right to within 0.05 points on 5 of 9
    cells, but nothing in the statement warns the reader which regime they are in.
13. **The sandbox is gone, and only the three published statements survived it.**
    `.wiki-runtime/tmp/ginorm/` was deleted on 2026-08-27 after this page was filed, taking the
    cluster, the exact-pin 17.11 install and the whole harness (`matrix.sh`, `sweep.sh`,
    `verify_published_sql.sh`, `check_citations.py` and 12 SQL files) with it. The
    2026-08-27 review then rebuilt 17.11 from the pin a second time, and its sandbox
    (`.wiki-runtime/tmp/ginrev/`) was deleted the same way once the review finished, so nothing
    executable remains again. Re-verification means rebuilding 17.11 out of tree from
    `raw/postgres-17/` and re-deriving the fixtures and churn from
    [Test methodology](#test-methodology) and
    [The 12 cells and their results](#the-12-cells-and-their-results), which describe them in
    prose but do not publish their SQL. Only the three statements under
    [SQL 1](#sql-1-record-the-baseline), [SQL 2](#sql-2-read-the-baseline-back) and
    [SQL 3 and 4](#sql-3-and-4-the-ratios-and-the-verdict) are recoverable verbatim — which is
    why the review could reproduce every behavioural claim and none of the absolute byte sizes.
    Publishing the fixture builder would fix this and has not been done.
14. **One row of the `maintenance_work_mem` table is internally inconsistent by one page, and
    the review could not decide which figure is wrong.** A fresh GIN build satisfies
    `total_pages = n_entry_pages + n_data_pages + n_pending_pages + 1`, the `+ 1` being the
    metapage that `ginvacuumcleanup` skips by starting its loop at `GIN_ROOT_BLKNO`
    ([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L803)).
    Every other measured table on this page satisfies it — 7,495,680 = (914 + 0 + 1) x 8192,
    82,329,600 = (49 + 10,000 + 1) x 8192, 246,169,600 = (49 + 30,000 + 1) x 8192 — and the
    review confirmed it holds with residual 0 on six fresh builds, in both the all-inline and
    the posting-tree regime and at both 64MB and 256MB. The 64MB row does not: 109,371,392
    bytes is 13,350 pages, while 3,350 + 10,000 + 1 is 13,351. One of those three numbers is
    off by one — most likely `n_entry_pages` should read 3,349, since 109,379,584 bytes would
    also satisfy the invariant — and the original fixture no longer exists to settle it.

## Source References

- [ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c) - `ginvacuumcleanup`, `GinPageIsRecyclable`, the `num_index_tuples` assignment.
- [gininsert.c](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c) - `ginbuild`, `ginBuildCallback`, `buildFreshLeafTuple`, `addItemPointersToLeafTuple`, `ginEntryInsert`.
- [ginfast.c](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c) - pending list, `ginInsertCleanup`, FSM recycling.
- [ginutil.c](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c) - `ginhandler` AM routine table, `ginoptions`.
- [gin_private.h](../../../../raw/postgres-17/src/include/access/gin_private.h), [reloptions.c](../../../../raw/postgres-17/src/backend/access/common/reloptions.c) - `GinOptions`, `fastupdate` and `gin_pending_list_limit` defaults and lock levels.
- [index.c](../../../../raw/postgres-17/src/backend/catalog/index.c) - `index_build`, `index_update_stats`, `reindex_index`, `index_concurrently_swap`.
- [analyze.c](../../../../raw/postgres-17/src/backend/commands/analyze.c) - `do_analyze_rel`, `compute_index_stats`, `tupleFract`.
- [vacuum.c](../../../../raw/postgres-17/src/backend/commands/vacuum.c) - `vac_cleanup_one_index`.
- [vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c) - `update_relstats_all_indexes`, `lazy_cleanup_one_index`, the `estimated_count` guard.
- [pgstat_relation.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c) - `pgstat_report_analyze`, `pgstat_report_vacuum`, `pgstat_relation_flush_cb`.
- [pgstat.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c) - `pgstat_report_stat`, `PGSTAT_MIN_INTERVAL`, `PGSTAT_MAX_INTERVAL`.
- [comment.c](../../../../raw/postgres-17/src/backend/commands/comment.c) - `CommentObject`, `CreateComments`, `DeleteComments`.
- [dbsize.c](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c) - `calculate_relation_size`, `pg_relation_size`, `pg_indexes_size`.
- [system_functions.sql](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql) - `obj_description`, `pg_relation_size`.
- [system_views.sql](../../../../raw/postgres-17/src/backend/catalog/system_views.sql) - `pg_stat_all_tables`.
- [guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c) - `maintenance_work_mem`, `gin_pending_list_limit`, `statement_timeout`, `lock_timeout`.
- [pg_class.h](../../../../raw/postgres-17/src/include/catalog/pg_class.h), [pg_index.h](../../../../raw/postgres-17/src/include/catalog/pg_index.h), [pg_description.h](../../../../raw/postgres-17/src/include/catalog/pg_description.h), [pg_proc.dat](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat).
- [gin.sgml](../../../../raw/postgres-17/doc/src/sgml/gin.sgml) - fast update technique, tips and tricks.
- [maintenance.sgml](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml) - `routine-reindex`.
- [ref/reindex.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml), [ref/comment.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml).

## Navigation

- [v17/index](../../index.md) - PostgreSQL 17 landing page.
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17 (unverified)](non-btree-index-inflation-comment-baseline.md) - the five-access-method sibling, which normalizes by a per-AM population term instead of the table `reltuples`.
- [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17 (unverified)](gin-index-wasted-space-contrib.md) - what a `pageinspect` page census can measure that catalogs cannot.
- [Reading an Index's Entry Count From the Catalogs, for Every Index Type, in PostgreSQL 17 (unverified)](index-entry-count-from-catalogs.md) - the full three-writer story behind an index's own `reltuples`.
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md) - the rebuild path this heuristic recommends.
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md) - what the planner does and does not see about a bloated index.
- [versions](../../../versions.md) - source pin manifest.
- [index](../../../index.md) - global wiki catalog.
