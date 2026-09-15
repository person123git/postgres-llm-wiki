---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# A COMMENT-Stored Baseline and Normalized Index Growth for Finding GIN Indexes That Need REINDEX CONCURRENTLY in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
  - [Prompt corrections](#prompt-corrections)
  - [The first prompt](#the-first-prompt)
  - [The second prompt](#the-second-prompt)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What conformance to the protocol changed](#what-conformance-to-the-protocol-changed)
  - [Why a GIN index's physical size only ever goes up](#why-a-gin-indexs-physical-size-only-ever-goes-up)
  - [Why the baseline must not use the GIN index's own reltuples](#why-the-baseline-must-not-use-the-gin-indexs-own-reltuples)
  - [What the COMMENT stores](#what-the-comment-stores)
  - [The comment format](#the-comment-format)
  - [SQL 1: record the baseline](#sql-1-record-the-baseline)
  - [SQL 2: read the baseline back](#sql-2-read-the-baseline-back)
  - [SQL 3 and 4: the ratios and the verdict](#sql-3-and-4-the-ratios-and-the-verdict)
  - [The declared kind of every published column](#the-declared-kind-of-every-published-column)
  - [The fixture corpus and its recipes](#the-fixture-corpus-and-its-recipes)
  - [The phases every fixture ran](#the-phases-every-fixture-ran)
  - [The 24 scored fixtures and their results](#the-24-scored-fixtures-and-their-results)
  - [The declared upper bound failed, so est_reclaimable is a level](#the-declared-upper-bound-failed-so-est_reclaimable-is-a-level)
  - [How close the prediction came](#how-close-the-prediction-came)
  - [First failure: a fresh GIN build is not linear in heap tuples](#first-failure-a-fresh-gin-build-is-not-linear-in-heap-tuples)
  - [Second failure: the pending-list high-water mark](#second-failure-the-pending-list-high-water-mark)
  - [The baseline itself depends on maintenance_work_mem](#the-baseline-itself-depends-on-maintenance_work_mem)
  - [The maintenance pair: what the settle step and the maintenance step move](#the-maintenance-pair-what-the-settle-step-and-the-maintenance-step-move)
  - [The auto-analyze window, and why the method almost never sees it](#the-auto-analyze-window-and-why-the-method-almost-never-sees-it)
  - [A snapshot held across the settling VACUUM](#a-snapshot-held-across-the-settling-vacuum)
  - [A VACUUM whose index cleanup did not run](#a-vacuum-whose-index-cleanup-did-not-run)
  - [Keys that no longer occur](#keys-that-no-longer-occur)
  - [The verdict ladder, in order](#the-verdict-ladder-in-order)
  - [Why the shrinkage rule can never fire on its own](#why-the-shrinkage-rule-can-never-fire-on-its-own)
  - [The statistics counter you must not build the guard on](#the-statistics-counter-you-must-not-build-the-guard-on)
  - [The four cross-checks and the six invariants](#the-four-cross-checks-and-the-six-invariants)
  - [The simulated auto-analyze census](#the-simulated-auto-analyze-census)
  - [End-to-end run of the published statements](#end-to-end-run-of-the-published-statements)
  - [Edge cases proven on the server](#edge-cases-proven-on-the-server)
  - [Operational notes: locks, privileges, timeouts, GUC scopes](#operational-notes-locks-privileges-timeouts-guc-scopes)
  - [Recommended thresholds](#recommended-thresholds)
  - [Coverage the protocol requires, and what this page skipped](#coverage-the-protocol-requires-and-what-this-page-skipped)
  - [What left the page with its fixtures](#what-left-the-page-with-its-fixtures)
- [Measurement Script](#measurement-script)
  - [How to use it](#how-to-use-it)
  - [Prerequisites](#prerequisites)
  - [Where the results land](#where-the-results-land)
  - [The last run](#the-last-run)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

### Prompt corrections

Two prompts drove this page, and both were filed after prompt-hygiene correction at the
asker's request.

The first prompt wrote `agents.md` for `AGENTS.md` and lowercase `postgresql` for
`PostgreSQL`, put a space before the comma after the final requirement, spliced an
operational instruction (`before anything perform a cleanup of .wiki-runtime`) into the
question text, and wrote the threshold range `75–80%` with an en dash. The
`.wiki-runtime` cleanup instruction was carried out but is not part of the question.

The second prompt read:

```text
follow agents.md, in postgresql 17, review question: # A COMMENT-Stored Baseline and
Normalized Index Growth for Finding GIN Indexes That Need REINDEX CONCURRENTLY in
PostgreSQL 17 (unverified), update tests based on the changes from common-concept,
update or remove all tests that aren't following # Mandatory GIN Bloat Tests (unverified)
```

Its defects were `agents.md` for `AGENTS.md`, lowercase `postgresql`, `review question:`
without an article, a pasted `# ` heading marker before each of the two page titles, the
`(unverified)` hint treated as part of both titles, `from common-concept` for "from the
common concept page", the contraction `aren't following` for "do not follow", and no
terminal period. The asker chose **correct and restate**, and then settled the scope in
three further answers: a **full re-run with a published measurement script** rather than a
paper re-port; tests that cannot conform are **removed with the claims they backed**, not
relabelled; and **every coverage behavior the method can reach** is added.

### The first prompt

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

### The second prompt

Follow `AGENTS.md`, in PostgreSQL 17. Review the question
"A COMMENT-Stored Baseline and Normalized Index Growth for Finding GIN Indexes That Need
REINDEX CONCURRENTLY in PostgreSQL 17". Update the tests based on the changes from the
common concept page, and update or remove all tests that do not follow
"Mandatory GIN Bloat Tests".

## Answer

### Verdict

The design works and is buildable exactly as specified: four plain SQL statements, no new
table, no extension, and the whole baseline living in an `@ginbase:` JSON payload appended
to the index's own `COMMENT ON INDEX`. Everything below was re-measured on 2026-09-15
under [Mandatory GIN Bloat Tests](../../common-concepts/mandatory-gin-bloat-tests.md),
from the single script filed under [Measurement Script](#measurement-script), on a 17.11
server built out of tree from this repository's pin.

Over **24 scored fixtures**, each run through the protocol's five phases and each measured
against one `REINDEX INDEX` oracle, the method's decision was right **19 times**, with
**1 false positive**, **0 false negatives**, and **4 refusals** of which one hid a real
73.95 % candidate on purpose.

The declared columns fared worse than the decision, and that is the headline result of
the re-run:

1. **The one bound this page was willing to declare failed.** `est_reclaimable` was filed
   as an **upper bound** — the bytes it names are never fewer than the bytes a rebuild
   returns — before any fixture existed. It **held on 14 of 21** fixtures that published
   it and was **violated on 7**, by as little as 0.01 points (`c02`) and as much as 10.43
   (`c01`). Under the protocol a violated bound is corrected or demoted, and a
   catalogs-only method cannot be corrected here, so the column is **demoted to a level**:
   it may be read as a ranking hint and must not be read as reclaimable space.
2. **A rebuild can make a GIN index bigger.** `p02`, a `fastupdate = off` index that grew
   by no bytes at all under 150,000 new rows, rebuilt from 14,778,368 to **17,883,136**
   bytes — a **−21.01 %** oracle. Incremental insertion packed it denser than its own
   bulk build at `maintenance_work_mem = 256MB`. "Reclaimable" is therefore not always a
   non-negative quantity, and nothing the method reads can tell.
3. **"Both grew by 50 %, so normalized = 1.0, so do nothing" is still the wrong
   instinct, but for a smaller amount than the page used to claim.** `c01` grew the table
   50 % by ordinary `INSERT` and the index by 51.7 % (22,339,584 -> 33,890,304), giving
   `normalized_index_growth` 1.0114 — and the rebuild still returned **11.55 %**.
4. **A fresh GIN build is not linear in heap tuples, so `heap_tuple_ratio` is a poor
   normalizer.** Rebuilt over the same 10,000-key universe the same index measured
   6,938,624 / 12,599,296 / 22,339,584 / 59,613,184 / 82,264,064 / 245,768,192 bytes at
   250k / 500k / 1M / 2M / 4M / 8M rows: **2.67x** between 1M and 2M rows as posting lists
   convert to posting trees, then only **1.38x** between 2M and 4M.
5. **The pending-list blind spot is real and is sized by `gin_pending_list_limit`, not by
   the index.** `p01` (`fastupdate = on`) grew 515 pages — 4,218,880 bytes against a 4 MB
   default limit — where its `fastupdate = off` twin grew nothing, and `p01` scored
   `no action` at `normalized_index_growth` 1.0284.

Treat the output as a ranked shortlist, never as proof of bloat — which is what the brief
asks for, and what the same-version documentation asks for too: "The potential for bloat in
non-B-tree indexes has not been well researched. It is a good idea to periodically monitor
the index's physical size when using any non-B-tree index type."
([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046))

### What conformance to the protocol changed

The protocol is defined on
[Mandatory GIN Bloat Tests](../../common-concepts/mandatory-gin-bloat-tests.md) and is not
restated here. What it changed on this page:

| Protocol rule | What this page had before | What it has now |
|---|---|---|
| five phases per fixture, churn ending in settle then maintenance | a 12-cell matrix whose recipes were one-line prose; three cells deliberately skipped `VACUUM` or `ANALYZE` | 24 scored fixtures, every one through build -> baseline -> churn (writes, settle, maintenance) -> decide -> oracle, with per-phase readings filed |
| the settle step must be proven | not asserted | `n_pending_pages` read at every phase boundary: **0 on 24 of 24** after the settle step, and the `VACUUM VERBOSE` index line recorded for each |
| the maintenance step is mandatory | absent | `VACUUM ANALYZE` after every churn, except the two auto-analyze stand-ins, which declare `ANALYZE` plus `gin_clean_pending_list()` instead |
| the simulated auto-analyze census | absent | 28 tables walked, 1 analyzed, one declined **exactly on** its threshold, one short-circuited by `autovacuum_enabled = false` |
| `SHARE ROW EXCLUSIVE` measurement lock | the evaluation ran unlocked | one transaction locks all 25 fixture tables and runs the published statement inside it |
| one measured `REINDEX INDEX` oracle, bracketed by `pg_relation_size(index, 'main')` | ground truth was `1 - post/pre`, unbracketed | every fixture's oracle is two size readings around one rebuild at a recorded `maintenance_work_mem` |
| a declared kind per published column, filed before the run | absent | 10 columns declared at 17:24:53Z, three seconds before the first fixture's baseline payload |
| four mandatory cross-checks | two caveats stated in prose | four checks plus six declared invariants, scored per fixture |
| a published measurement script | none; the sandbox and harness were deleted | one 1,596-line Bash-and-SQL script, filed in full |

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

Measured: declared invariant **I1 — `index_size_ratio >= 1` on every fixture not rebuilt
since its baseline — held on 24 of 24**. The only value below 1 anywhere in the run is
`c09`'s 0.9996, and `c09` is the fixture that *was* rebuilt out of band, which the ladder
caught one rung above the ratios.

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

Measured on one 200,000-row table with 5 tags per row from a 2,000-key universe, the same
GIN index's own `reltuples` takes five values in six commands, a near 10x swing, while the
table's stays coherent:

| stage | table `reltuples` | GIN index `reltuples` |
|---|---|---|
| after `CREATE INDEX` | 200000 | **994200** |
| after `ANALYZE` | 200000 | **200000** |
| after `DELETE` 50% + `VACUUM` | 100000 | **100000** |
| after `REINDEX` | 100000 | **500000** |
| after a second plain `VACUUM` | 100000 | **500000** |
| after `VACUUM (DISABLE_PAGE_SKIPPING)` | 100000 | **100000** |

The build's 994,200 is the extracted-entry count, below 200,000 x 5 because
`ginExtractEntries` de-duplicates the keys of one value; the `ANALYZE` row is the table's
estimate; the `VACUUM` rows are the heap count.

The fifth and sixth rows are the important ones, and they are the value the index
**held before**. Because the `pg_class` write is guarded by `estimated_count`, and
`estimated_count` is true whenever the heap scan skipped a page —
`vacrel->scanned_pages < vacrel->rel_pages`
([vacuumlazy.c:2356](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2356)),
handed to the AM through `ivinfo`
([vacuumlazy.c:2481](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2481)) and
copied straight back out by GIN
([ginvacuum.c:733-739](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L733-L739)) —
VACUUM is a *conditional* writer, and on a quiet table it does not write at all: the second
plain `VACUUM` left the index at the 500000 its `REINDEX` had written, and only
`VACUUM (DISABLE_PAGE_SKIPPING)`, which forces every page to be scanned, wrote the heap
count 100000.

The table's `reltuples` is the only one of the two that means the same thing after all six
commands, which is exactly why the brief pins the denominator to it.

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

`bfn` earned its place twice in this run: `c09` (an out-of-band `REINDEX INDEX`) and edge
case E3 (`REINDEX INDEX CONCURRENTLY`, which moves the comment to a new index OID) were both
diverted to `rebuilt since baseline: re-capture` ahead of every ratio.

`bac` is the guard with the least to do under this protocol, and that is itself a finding:
the maintenance step analyzes every churned table, so `cur_analyze_count > bac` holds by
construction on every fixture here, and the rung can only fire on a server where the
assumption fails. Deliberately **not** stored: the baseline `n_live_tup` / `n_dead_tup` (no
decision power that `reltuples` does not already carry), and `n_mod_since_analyze` — see
[The statistics counter you must not build the guard on](#the-statistics-counter-you-must-not-build-the-guard-on).

### The comment format

The payload is appended to whatever a human already wrote, on its own line, behind an
`@ginbase:` tag:

```text
owner: search-team
ticket: OPS-1234 {do not drop} @ 100%
@ginbase:{"v": 1, "ts": "2026-09-15T17:28:27Z", "bac": 1, "bfn": "21290", "bhr": 600000, "bis": 14778368, "btd": 0, "bti": 600000, "btu": 0}
```

Measured sizes on this run: the `@ginbase:{...}` payload is **139 bytes** on the edge-case
index, whose comment as a whole is **196 bytes** with the two-line human note above; and
**137 bytes** with a **138-byte** whole comment where there is no human note (the extra byte
is the separating newline). `pg_description.description` is `text` and TOAST-able
([pg_description.h:48-66](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L66)),
so size is not a constraint.

Two format rules matter:

- **The JSON must stay flat.** Both the reader and the stripper match `\{[^}]*\}`, which
  stops at the first `}`. A nested object would truncate the payload. Nine scalar fields is
  the whole design budget.
- **Anchoring on `@ginbase:` is what makes a human comment safe.** The two-line note above
  contains both `{do not drop}` and `@ 100%`, and survived capture, `REINDEX`,
  `REINDEX CONCURRENTLY`, `ALTER INDEX ... RENAME` and re-capture byte for byte, with
  exactly one payload left in the comment afterwards.

`bfn` is rendered as a JSON *string* (`"21290"`) because `jsonb_build_object` renders `oid`
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
human text; measured, the comment carries **exactly one** payload after a capture,
a rebuild and a re-capture.

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

All three statements above are the statements that ran. The script's `verify` stage
re-extracts them from this page and diffs them against the files it executed: **3 of 3
identical, at 41, 18 and 76 lines.**

### The declared kind of every published column

The protocol requires every published column to be filed as a lower bound, an upper bound
or a level **before** the run, and forbids rewriting the declaration afterwards. The
script's `declare` stage writes them into their own database and refuses to run once the
fixture database exists; on the filed run they were written at **2026-09-15T17:24:53Z**,
three seconds before the first fixture's baseline payload at **17:24:56Z**.

| Published column | Declared kind | The claim that was filed |
|---|---|---|
| `est_reclaimable` | **upper bound** | the bytes it names are never fewer than the bytes `REINDEX INDEX` returns |
| `normalized_index_growth` | level | a ranking statistic; no claim against the oracle |
| `index_size_ratio` | level | current over baseline file size |
| `heap_tuple_ratio` | level | current over baseline table `reltuples` |
| `churn_ratio` | level | write volume over baseline `reltuples` |
| `stats_lag` | level | advisory staleness reading |
| `cur_size`, `base_size` | level | the two file sizes themselves |
| `cur_heap_reltuples`, `base_heap_reltuples` | level | the two row estimates themselves |

The method also makes a **decision**, so it declares its own thresholds, and one more
number the oracle side needs:

| Knob | Filed value |
|---|---|
| `candidate` | `normalized_index_growth >= 1.50` (the brief's value) |
| `strong candidate` | `heap_tuple_ratio <= 0.50 AND index_size_ratio >= 0.75` |
| `watch` | `normalized_index_growth >= 1.20`, and **not** a flag to rebuild |
| churn gate | `churn_ratio >= 0.20`, below which the method refuses to evaluate |
| oracle justification | `truth_pct >= 33.33`, which is `100 * (1 - 1/1.50)`: the method's own candidate threshold carried to the oracle side |

Six invariants were filed with them, so that a cross-check failure is a scored outcome and
not a discovery:

| # | Filed claim | Result |
|---|---|---|
| I1 | `index_size_ratio >= 1` on every fixture not rebuilt since its baseline | **24 of 24** |
| I2 | a freshly built index satisfies `n_total_pages = n_entry_pages + n_data_pages + n_pending_pages + 1` | **25 of 25** |
| I3 | the FSM free-page count never exceeds the census's new-plus-deleted count | **23 of 23** |
| I4 | `pg_relation_size` re-read after the census equals the blocks the census scanned | **23 of 23** |
| I5 | the fourth number of the `VACUUM VERBOSE` index line equals the census's new-plus-deleted count | **21 of 21** |
| I6 | `n_total_pages = 1 + census entry + data + list + new-plus-deleted` | **21 of 23**, and both exceptions are the auto-analyze stand-ins |

### The fixture corpus and its recipes

Twenty-four scored fixtures and one unscored refusal fixture, all in one database, all
disposable. Every recipe is a line of the published script, not a prose description, so a
reviewer can re-run any one of them. The int-array fixtures carry five keys per row drawn
from a 10,000-value universe by a deterministic hash.

| id | rows | index | churn recipe |
|---|---|---|---|
| c01 | 1,000,000 | `gin (tags)` | `INSERT` +50 % rows, same key universe |
| c02 | 1,000,000 | `gin (tags)` | two full-table `UPDATE`s, new keys from the same universe |
| c03 | 1,000,000 | `gin (tags)` | `DELETE` 60 % (`id % 5 < 3`) |
| c04 | 1,000,000 | `gin (tags)` | none: the control |
| c05 | 1,000,000 | `gin (tags)` | `INSERT` +100 % rows, same key universe |
| c06 | 1,000,000 | `gin (tags)` | `INSERT` +100 % rows, all-new keys |
| c09 | 1,000,000 | `gin (tags)` | full `UPDATE`, then an out-of-band `REINDEX INDEX` |
| c10 | 300,000 | `gin (doc)`, `tsvector` | rewrite every document |
| c11 | 1,000,000 | `gin (tags)` | full `UPDATE`, then `pg_stat_reset_single_table_counters()` |
| c12 | 1,000,000 | `gin (tags)` | `DELETE` the lower half of the ids, then `INSERT` 50 % fresh rows |
| p01 | 600,000 | `gin (tags) WITH (fastupdate = on)` | `INSERT` +25 % rows |
| p02 | 600,000 | `gin (tags) WITH (fastupdate = off)` | `INSERT` +25 % rows |
| m01 | 300,000 | `gin (tags)` | full `UPDATE`, measured before and after the settle and maintenance steps |
| a01 | 300,000 | `gin (tags) WITH (fastupdate = on)` | `INSERT` 40,000 rows; maintained by the auto-analyze stand-in |
| a02 | 300,000 | `gin (tags) WITH (fastupdate = on)` | `INSERT` 60,500 rows; same stand-in |
| s01 | 1,000,000 | `gin (tags)`, one key per 100,000 rows | `DELETE` the lower half of the ids, with a repeatable-read snapshot held across the settling `VACUUM` |
| i01 | 300,000 | `gin (tags)` | `DELETE` 30 %, then `VACUUM (INDEX_CLEANUP OFF)` before the settle step |
| o01 | 300,000 | `gin (j jsonb_path_ops)` | rewrite every document |
| o02 | 100,000 | `gin (txt gin_trgm_ops)` | rewrite every document |
| o03 | 300,000 | `gin (n)`, `btree_gin` `int4_ops` | `UPDATE` every row's key |
| x01 | 300,000 | partial `gin (tags) WHERE id % 4 = 0` | full `UPDATE` |
| x02 | 300,000 | two-column `gin (tags, tags2)` | full `UPDATE` of `tags` |
| x03 | 300,000 | expression `gin ((tags[1:3]))` | full `UPDATE` |
| k01 | 300,000 | `gin (tags)`, one key per 100 rows | `DELETE` the lower half of the ids, retiring 1,500 keys |
| e01 | 0 | `gin (tags)` | none; unscored, because the capture refuses it |

`e01` is the empty fixture and the only one the capture statement declines:
`refusing baseline for f_e01_gin: table reltuples is 0 (run ANALYZE first)`. Its decide row
is therefore `no baseline: capture one`, with every ratio null.

### The phases every fixture ran

| Phase | What the script does | What it records |
|---|---|---|
| build | create, load, create the index, `VACUUM (VERBOSE, ANALYZE)` | the as-built file size, the metapage row, the catalog row and the statistics counters |
| baseline | run statement 1 against that index | the `@ginbase:` payload, byte for byte |
| churn | the recipe's writes, `pg_stat_force_next_flush()`, the settle `VACUUM (VERBOSE)`, then the maintenance `VACUUM (VERBOSE, ANALYZE)` | a reading after the writes, after the settle step and after the maintenance step |
| decide | statement 3, verbatim, in one transaction holding `SHARE ROW EXCLUSIVE` on all 25 fixture tables | every published column, per fixture |
| oracle | `REINDEX INDEX` at a recorded `maintenance_work_mem`, bracketed by `pg_relation_size(index, 'main')` | the two sizes and `truth_pct` |

The settle step is proven rather than assumed: `n_pending_pages` read **0 on all 24 scored
fixtures** at the decide phase — after the settle step on the 22 that have one, and after
the stand-in's own flush on `a01` and `a02` — and the `VACUUM VERBOSE` index line is filed
for each. The two auto-analyze stand-ins do not claim the settle step; see
[The auto-analyze window, and why the method almost never sees it](#the-auto-analyze-window-and-why-the-method-almost-never-sees-it).

### The 24 scored fixtures and their results

`decide` is the file size the method saw, `rebuilt` the size after the oracle's
`REINDEX INDEX`, `truth` its returned percentage, `norm` the unrounded
`normalized_index_growth`, `pred` the `100 * (1 - 1/norm)` the method's `est_reclaimable`
is built from, and `err` the prediction's error in points. `bound` is the verdict on the
declared upper bound.

| id | verdict | decide | rebuilt | truth | norm | pred | err | bound | score |
|---|---|---|---|---|---|---|---|---|---|
| a01 | insufficient churn: not evaluated | 12,197,888 | 9,084,928 | 25.52 | 1.3191 | 24.19 | −1.33 | VIOLATED | refused, nothing to reclaim |
| a02 | watch | 12,378,112 | 9,568,256 | 22.70 | 1.2625 | 20.79 | −1.91 | VIOLATED | PASS |
| c01 | no action | 33,890,304 | 29,974,528 | 11.55 | 1.0114 | 1.12 | −10.43 | VIOLATED | PASS |
| c02 | candidate | 123,592,704 | 22,331,392 | 81.93 | 5.5361 | 81.94 | +0.01 | HELD | PASS |
| c03 | strong candidate | 22,339,584 | 9,691,136 | 56.62 | 2.5000 | 60.00 | +3.38 | HELD | PASS |
| c04 | insufficient churn: not evaluated | 22,339,584 | 22,339,584 | 0.00 | 1.0000 | — | — | not published | refused, nothing to reclaim |
| c05 | candidate | 85,712,896 | 59,613,184 | 30.45 | 1.9184 | 47.87 | +17.42 | HELD | **FALSE POSITIVE** |
| c06 | no action | 47,808,512 | 44,654,592 | 6.60 | 1.0700 | 6.55 | −0.05 | VIOLATED | PASS |
| c09 | rebuilt since baseline: re-capture | 22,331,392 | 22,331,392 | 0.00 | 0.9996 | — | — | not published | refused, nothing to reclaim |
| c10 | candidate | 18,997,248 | 8,437,760 | 55.58 | 2.2515 | 55.58 | 0.00 | HELD | PASS |
| c11 | counters reset: re-capture | 85,712,896 | 22,331,392 | 73.95 | 3.8368 | 73.94 | −0.01 | VIOLATED | refused, **a real candidate** |
| c12 | candidate | 33,890,304 | 22,339,584 | 34.08 | 1.5171 | 34.08 | 0.00 | HELD | PASS |
| i01 | watch | 8,159,232 | 5,988,352 | 26.61 | 1.4286 | 30.00 | +3.39 | HELD | PASS |
| k01 | strong candidate | 778,240 | 393,216 | 49.47 | 2.0000 | 50.00 | +0.53 | HELD | PASS |
| m01 | candidate | 19,505,152 | 8,159,232 | 58.17 | 2.3906 | 58.17 | 0.00 | HELD | PASS |
| o01 | candidate | 20,135,936 | 8,257,536 | 58.99 | 2.4385 | 58.99 | 0.00 | HELD | PASS |
| o02 | candidate | 20,701,184 | 4,825,088 | 76.69 | 4.2903 | 76.69 | 0.00 | HELD | PASS |
| o03 | candidate | 6,512,640 | 2,285,568 | 64.91 | 2.8495 | 64.91 | 0.00 | HELD | PASS |
| p01 | no action | 18,997,248 | 17,883,136 | 5.86 | 1.0284 | 2.76 | −3.10 | VIOLATED | PASS |
| p02 | no action | 14,778,368 | 17,883,136 | **−21.01** | 0.8000 | — | — | not published | PASS |
| s01 | strong candidate | 1,245,184 | 630,784 | 49.34 | 2.0000 | 50.00 | +0.66 | HELD | PASS |
| x01 | candidate | 7,471,104 | 2,334,720 | 68.75 | 3.6480 | 72.59 | +3.84 | HELD | PASS |
| x02 | candidate | 21,446,656 | 10,534,912 | 50.88 | 2.0358 | 50.88 | 0.00 | HELD | PASS |
| x03 | candidate | 9,994,240 | 5,464,064 | 45.33 | 1.8263 | 45.25 | −0.08 | VIOLATED | PASS |

Totals: **19 `PASS`, 1 `FALSE POSITIVE`, 0 `FALSE NEGATIVE`, 4 refusals** (three with
nothing worth reclaiming, one — `c11` — hiding a genuine 73.95 % candidate behind a
deliberately conservative counter-reset rung).

Two rows deserve reading twice:

- **`c05` is the only false positive**, and it is a threshold artefact rather than a wrong
  reading: the method flagged an index whose rebuild returned 30.45 %, against the 33.33 %
  this page declared as the point where a rebuild pays. Its `norm` of 1.9184 predicted
  47.87 %, a 17.42-point over-prediction, and the reason is
  [the posting-tree conversion](#first-failure-a-fresh-gin-build-is-not-linear-in-heap-tuples):
  `c05` doubled a 1,000,000-row table, and the fresh 2,000,000-row build is 2.67x the
  1,000,000-row baseline rather than 2x.
- **`p02` returned negative bytes.** Its file never grew, and its rebuild is 3,104,768
  bytes *larger* than the churned index. The method read `norm` 0.8000, printed no
  `est_reclaimable` at all because the column is gated on `norm > 1`, and said
  `no action` — the right answer, reached without any way of knowing why.

### The declared upper bound failed, so est_reclaimable is a level

`est_reclaimable` was declared an upper bound. Scored against the oracle on the 21
fixtures that published it:

| Verdict | Count | Fixtures |
|---|---|---|
| HELD | **14** | c02, c03, c05, c10, c12, i01, k01, m01, o01, o02, o03, s01, x01, x02 |
| VIOLATED | **7** | a01 (−1.33), a02 (−1.91), c01 (−10.43), c06 (−0.05), c11 (−0.01), p01 (−3.10), x03 (−0.08) |
| not published | 3 | c04, c09, p02, where `norm <= 1` |

The violations are all under-predictions, they range from 0.01 to 10.43 points, and four of
the seven are under 2 points. That pattern matters: the bound does not fail because the
model is wildly wrong, it fails because the model is an *estimate* that lands on both sides
of the truth, and an estimate cannot be a bound. The protocol's rule is that a violated
bound is corrected in the method or demoted to a level, and a catalogs-only method has
nothing to correct with — the missing quantity is the size of a rebuild, which is what the
oracle exists to measure.

**So this page demotes `est_reclaimable` to a level.** The statement still prints it,
because deleting a column would change the text that was scored, and the reading rule is
now explicit:

- `est_reclaimable` is a **ranking hint**, not reclaimable space, and must not be quoted to
  anyone as bytes that will come back.
- The only number on this page that *is* reclaimable space is the oracle's own
  `truth`, measured by rebuilding.
- The recomputation used for scoring matched the statement's own `pg_size_pretty` output on
  **24 of 24** fixtures, so the demotion is about the model, not about a reporting bug.

### How close the prediction came

Over the 21 fixtures whose `norm` exceeded 1:

| Accuracy band | Count | Fixtures |
|---|---|---|
| within 0.05 points | **10** | c02, c06, c10, c11, c12, m01, o01, o02, o03, x02 |
| within 3.5 points | **18** | the ten above plus a01, a02, c03, i01, k01, p01, s01, x03 |
| over-predicts by more than 3.5 | 2 | c05 (+17.42), x01 (+3.84) |
| under-predicts by more than 3.5 | 1 | c01 (−10.43) |

Ten of 21 within 0.05 points is not luck, and the reason is instructive: in c02, c10, c11,
c12, m01, o01, o02, o03 and x02 the population is unchanged, so `norm` collapses to
`index_size_ratio`, and a rebuild over an unchanged row set returns the index to almost
exactly its baseline — c10 to 8,437,760 against a baseline of 8,437,760, m01 to 8,159,232
against 8,159,232, o01 to 8,257,536 against 8,257,536, x02 to 10,534,912 against
10,534,912. When the row count and the key distribution are unchanged, a fresh GIN build is
byte-reproducible, and the model is then not an estimate but an identity. The tenth,
`c06`, is the one case where the population *doubled* and the prediction still landed
within 0.05 points, because doubling the key universe alongside the rows kept the build
proportional — which is exactly the condition `heap_tuple_ratio` silently assumes.

The misses are all fixtures where the population changed or the pending list moved, which
is the opposite of what a normalizer is supposed to buy you, and is the most important
caveat on this page.

### First failure: a fresh GIN build is not linear in heap tuples

Rebuilding the same index over a growing table with the same 10,000-key universe, at
`maintenance_work_mem = 256MB`:

| rows | fresh build bytes | pages | `n_entry_pages` | `n_data_pages` | `n_entries` |
|---|---|---|---|---|---|
| 250,000 | 6,938,624 | 847 | 846 | 0 | 10,000 |
| 500,000 | 12,599,296 | 1,538 | 1,537 | 0 | 10,000 |
| 1,000,000 | 22,339,584 | 2,727 | 2,726 | 0 | 10,000 |
| 2,000,000 | **59,613,184** | 7,277 | 1,904 | **5,372** | 10,000 |
| 4,000,000 | 82,264,064 | 10,042 | 53 | 9,988 | 10,000 |
| 8,000,000 | 245,768,192 | 30,001 | 50 | 29,950 | 10,000 |

Doubling the rows multiplies the build by 1.82x, 1.77x, **2.67x**, **1.38x** and 2.99x in
turn. The 2.67x step is the posting-tree conversion arriving: at 1,000,000 rows every key's
TIDs still fit inside its entry tuple (0 data pages, 2,726 entry pages), at 2,000,000 rows
5,372 data pages exist beside 1,904 entry pages, and at 4,000,000 rows the entry tree has
collapsed to 53 pages against 9,988 data pages — one posting tree per key, with `n_entries`
pinned at 10,000 throughout.

The mechanism is in the build path. GIN keeps a key's TIDs as a compressed posting list inside
the entry tuple while it fits in `GinMaxItemSize`, and converts to a posting tree when it does
not
([gininsert.c#buildFreshLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L166),
[gininsert.c#addItemPointersToLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L45-L111)).
The conversion is per key and is a step, not a slope.

Consequence for the heuristic: dividing by `heap_tuple_ratio` assumes the denominator tracks
what a fresh build would cost. Across a posting-tree conversion it does not, in either
direction. `c05` is this failure — `norm` 1.9184 predicted 47.87 % against a measured
30.45 %. `c06` escapes it, and predicts to within 0.05 points, precisely because doubling the
*key* universe alongside the rows kept the growth proportional.

### Second failure: the pending-list high-water mark

Two identical 600,000-row tables, one index with `fastupdate = on` and one with
`fastupdate = off` — the default is `true` both in the access method
([gin_private.h:33](../../../../raw/postgres-17/src/include/access/gin_private.h#L33)) and in the
reloption table
([reloptions.c:123-131](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131))
— then 150,000 rows inserted into both:

| stage | `p01`, `fastupdate = on` | `p02`, `fastupdate = off` |
|---|---|---|
| baseline | 14,778,368 | 14,778,368 |
| after +150,000 inserts, before the settle step | **18,997,248** (307 pending pages) | **14,778,368** (0 pending pages) |
| after the settle and maintenance steps | 18,997,248 (0 pending pages) | 14,778,368 |
| census: deleted pages / FSM free pages | **515 / 515** | 0 / 0 |
| after `REINDEX INDEX` | 17,883,136 | **17,883,136** |
| the method said | `no action`, `norm` 1.0284 | `no action`, `norm` 0.8000 |
| the oracle returned | 5.86 % | **−21.01 %** |

With `fastupdate = off`, 150,000 new rows fit in existing page slack and the file did not grow
by one byte. With `fastupdate = on` the file grew 515 pages — 4,218,880 bytes, against a 4 MB
default `gin_pending_list_limit` — the settle step merged the list into the main structure, and
all 515 pages went to the free space map and stayed in the file, where the census and the FSM
agree on them exactly.

Two things follow that the old version of this page had wrong:

- **The pending list's high-water mark is bounded by `gin_pending_list_limit`, and the bound
  is enforced during the inserts, not at the end.** `ginfast.c` compares
  `metadata->nPendingPages * GIN_PAGE_FREESIZE` against `GinGetPendingListCleanupSize(index) * 1024`
  and, when it is over, calls `ginInsertCleanup` in the foreground
  ([ginfast.c:458-471](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L458-L471)),
  where `GIN_PAGE_FREESIZE` is `BLCKSZ - MAXALIGN(SizeOfPageHeaderData) - MAXALIGN(sizeof(GinPageOpaqueData))`
  ([ginfast.c:41-42](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L41-L42)),
  which is 8,160 bytes at `block_size` 8192. So the flush fires at the first page count above
  `4096 * 1024 / 8160 = 513.9`, i.e. at 514 pending pages — and `p01` ends with 307 pending
  pages *after* such a flush and 515 deleted ones, while `a02` ends with 514 deleted pages
  from the same mechanism.
- **A rebuild does not always return those bytes; sometimes it costs more.** Both twins
  rebuild to the same 17,883,136 bytes, which is larger than either churned file, because a
  bulk build at `maintenance_work_mem = 256MB` flushes its accumulator in rounds and each
  round appends
  ([gininsert.c:290-291](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L291)),
  where incremental insertion had packed the same entries denser.

On a 14 MB index the 4 MB limit is a quarter of the file and the heuristic cannot see it; on a
1 GB index it is 0.4 % and does not matter. GIN's own documentation describes the pending list
and its limit
([gin.sgml#gin-fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537),
[gin.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L595-L616));
the recycling of flushed pending pages into the FSM is
[ginfast.c:1015-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1015-L1020).

### The baseline itself depends on maintenance_work_mem

The same 2,000,000-row index, rebuilt three times at three settings:

| `maintenance_work_mem` | fresh build bytes | pages | `n_entry_pages` | `n_data_pages` | identity |
|---|---|---|---|---|---|
| 64MB | **71,737,344** | 8,757 | 3,384 | 5,372 | exact |
| 256MB | 59,613,184 | 7,277 | 1,904 | 5,372 | exact |
| 1GB | 59,613,184 | 7,277 | 1,904 | 5,372 | exact |

A build at 64MB is **20.3 % larger** than the same build at 256MB, and 256MB and 1GB agree to
the byte. The setting is not incidental to the build: `ginBuildCallback` dumps its accumulator
to the index whenever `buildstate->accum.allocatedMemory >= (Size) maintenance_work_mem * 1024L`
([gininsert.c:290-291](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L291)),
so the budget decides how many flush rounds a build takes, and each round appends to the entry
and posting structures independently. The 64MB build shows it in the shape as well as the size:
3,384 entry pages against 1,904 for the same 10,000 keys and the identical 5,372 data pages.

The declared invariant I2 — `n_total_pages = n_entry_pages + n_data_pages + n_pending_pages + 1`,
the `+ 1` being the metapage that `ginvacuumcleanup` skips by starting its loop at
`GIN_ROOT_BLKNO`
([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L803))
— is **exact on all nine probe builds and all 25 fixture baselines**, including the 64MB row.
That settles the internal inconsistency this page used to carry as an open question: the
earlier 64MB reading was off by one page, and it does not reproduce.

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

### The maintenance pair: what the settle step and the maintenance step move

`m01` is the protocol's maintenance pair: one fixture, one full-table `UPDATE`, measured
after the writes and again after the settle and maintenance steps.

| reading | file size | metapage total / entry / pending | census entry / list / deleted | FSM free | index `relpages` |
|---|---|---|---|---|---|
| baseline | 8,159,232 | 996 / 995 / 0 | — | — | 996 |
| after the writes, before the settle step | 19,505,152 | **996 / 995 / 99** | 1,865 / 99 / 416 | 416 | **996** |
| after the settle step | 19,505,152 | **2,381 / 1,946 / 0** | — | — | **2,381** |
| after the maintenance step | 19,505,152 | 2,381 / 1,946 / 0 | 1,946 / 0 / 434 | 434 | 2,381 |
| after the oracle's rebuild | 8,159,232 | 996 / 995 / 0 | — | — | 996 |

Three readings come out of that table:

- **The settle step moved no bytes and changed every count.** The file is 19,505,152 bytes at
  all three points, but before it the metapage still described a 996-page index while the file
  held 2,381 blocks, and 99 live `list` pages existed that the metapage's page counts do not
  bucket at all. This is why the protocol forbids handing a method the pre-settle state: a
  size-only method reads the same number, but anything reading the metapage reads the
  as-built index.
- **The `ANALYZE` half of the maintenance step moved no page**, exactly as the protocol says
  it cannot: `vacuum()` vacuums each relation and then analyzes it
  ([vacuum.c#vacuum-then-analyze](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650)),
  and `analyze_rel` enters the index AM's ANALYZE-only cleanup only when the command is not a
  `VACUUM ANALYZE`
  ([analyze.c#analyze-only-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L694-L721)).
- **What did move in the catalog is the measured index's own row.** `relpages` went 996 ->
  2,381 across the settle step, written by `update_relstats_all_indexes`
  ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3072-L3099)),
  and the rebuild wrote it back to 996 while setting the index's `reltuples` to 1,498,260 —
  the extracted-entry count, not a row count.

The 416 deleted pages already present before the settle step are the foreground pending-list
flush described above; the settle step turned the remaining 99 into 18 more.

### The auto-analyze window, and why the method almost never sees it

On a GIN index an auto-analyze is not a statistics-only event: a worker's bare `ANALYZE`
reaches `ginvacuumcleanup` with `analyze_only` set and, because the caller is an autovacuum
worker, flushes the pending list and returns before the page census
([ginvacuum.c#analyze_only-worker](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L709-L717)),
while the same call from any other backend returns at once. The protocol's stand-in for that
state is `ANALYZE` **plus** `gin_clean_pending_list()`, and `a01` and `a02` are built on it.
`gin_clean_pending_list` returns its own count of deleted pending pages
([ginfast.c:1090](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1090)).

| | `a01` | `a02` |
|---|---|---|
| rows inserted onto a 300,000-row table | 40,000 | 60,500 |
| `n_mod_since_analyze` at the end of the writes | 40,000 | 60,500 |
| the launcher's analyze threshold, `50 + 0.1 * reltuples` | 30,050 — crossed | 30,050 — crossed |
| the launcher's insert-vacuum threshold, `1000 + 0.2 * reltuples` | 61,000 — not crossed | 61,000 — not crossed |
| pending pages the stand-in flushed | **493** | **232** |
| census deleted pages afterwards | 493 | **514** |
| metapage `n_total_pages` against the real file | 996 against 1,489 blocks | 996 against 1,511 blocks |
| the method's `churn_ratio` | 0.1333 | **0.2017** |
| the method said | `insufficient churn: not evaluated` | `watch`, `norm` 1.2625 |
| the oracle returned | 25.52 % | 22.70 % |

Two findings, and the second is the sharper one:

- **A stand-in fixture carries stale metapage counts by construction**, which the protocol
  states in advance and this run measures: neither the worker's flush nor
  `gin_clean_pending_list` reaches `ginUpdateStats`, so `a01` and `a02` are the only two
  fixtures that fail invariant I6, and `a02` is additionally off by one *entry* page (996
  recorded, 996 counted, 514 deleted and one entry page the flush allocated that the metapage
  never learned about). Both are excluded from the settle-step claim, as the protocol requires.
- **The window in which an auto-analyze-only maintained table can be evaluated at all is
  1,001 rows wide.** To be maintained by auto-analyze alone, inserts `I` must satisfy
  `I > 50 + 0.1 N` (the analyze verdict) and `I <= 1000 + 0.2 N` (staying under the
  insert-vacuum verdict), and both tests are strictly-greater comparisons against effective
  per-table values
  ([autovacuum.c#vacthresh-anlthresh](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076),
  [autovacuum.c#verdicts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095)).
  The method's own churn gate needs `I >= 0.2 N`. The overlap is
  `0.2 N <= I <= 0.2 N + 1000` — one thousand rows, whatever `N` is. `a01` sits below it and is
  refused; `a02` sits inside it by 500 rows and is evaluated. On any real table, an
  insert-only workload maintained by auto-analyze alone is therefore almost always refused by
  this method's churn gate, and the refusal is not wrong so much as structural.

### A snapshot held across the settling VACUUM

`s01` holds a repeatable-read snapshot open across its settle step. Its ten keys of 100,000
rows each make every key a posting tree, so deleting the lower half of the ids empties whole
data pages. The engine's own `VACUUM VERBOSE` index lines, in order:

```text
index "f_s01_gin": pages: 152 in total, 0 newly deleted, 0 currently deleted, 0 reusable
index "f_s01_gin": pages: 152 in total, 60 newly deleted, 60 currently deleted, 0 reusable
index "f_s01_gin": pages: 152 in total, 0 newly deleted, 0 currently deleted, 60 reusable
```

Three `VACUUM`s, three different answers, and none of them is the one a single-pass fixture
would have recorded:

1. **With the snapshot held, the settling `VACUUM` deleted nothing at all.** It could not:
   `vacuum_get_cutoffs` takes `OldestXmin` from `GetOldestNonRemovableTransactionId(rel)`
   ([vacuum.c:1120](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1120)), the
   holder's snapshot keeps the deleted heap tuples non-removable, so no index TID is deletable
   and no posting-tree page empties. The page census taken inside that state reads 140 data
   leaves, 0 deleted pages and 0 FSM-free pages.
2. **With the holder gone, the second `VACUUM` deleted 60 pages and freed none of them.** The
   census reads 80 data leaves and 60 deleted pages, all 60 carrying a delete xid, and the FSM
   still holds nothing: `GinPageIsRecyclable` accepts a deleted page only once
   `GlobalVisCheckRemovableXid` has moved past its delete xid
   ([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)).
3. **The maintenance `VACUUM` recycled them**: 60 reusable pages, the FSM count moves to 60,
   and the metapage's `n_data_pages` falls 150 -> 90.

The file never changed size through any of it — 1,245,184 bytes at every phase — and the
rebuild returned 49.34 %. A method that reads only `pg_relation_size` is blind to all three
states, which is the honest limit of this heuristic, and the protocol's rule that more than
one `VACUUM` is normal is here measured rather than assumed.

### A VACUUM whose index cleanup did not run

`i01` runs `VACUUM (INDEX_CLEANUP OFF)` between its writes and its settle step. The
command succeeded, reported the table vacuumed, and printed **no index line at all**:

```text
INFO:  vacuuming "ginnorm.public.f_i01"
INFO:  finished vacuuming "ginnorm.public.f_i01": index scans: 0
index scan bypassed: 3093 pages from table (100.00% of total) have 90000 dead item identifiers
```

`INDEX_CLEANUP` off forces `do_index_cleanup` false
([vacuumlazy.c#do_index_cleanup-init](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397)),
the cleanup call is made only while it is set
([vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066)),
and `VERBOSE` then prints "index scan bypassed" instead of a per-index line because
`do_index_vacuuming` is false and `indstats[i]` is NULL
([vacuumlazy.c#verbose-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L731)).
Measured consequence: the metapage read 996 / 10,000 entries before and after that command,
`n_dead_tup` went to 0 while 90,000 dead item identifiers stayed in the heap, and the index
file did not move. Only the settle step that followed touched the index. This is why the
protocol makes a run assert the settle step from the index's own state rather than from the
fact that a `VACUUM` returned.

### Keys that no longer occur

`k01` stores one key per 100 contiguous rows and then deletes the lower half of the ids,
retiring 1,500 of its 3,001 keys outright.

| reading | file size | `n_entries` | `n_entry_pages` |
|---|---|---|---|
| baseline | 778,240 | 3,001 | 94 |
| after the writes | 778,240 | 3,001 | 94 |
| after the settle step | 778,240 | 3,001 | 94 |
| after the maintenance step | 778,240 | 3,001 | 94 |
| after the oracle's rebuild | **393,216** | **1,501** | **47** |

The entry count does not move, because VACUUM removes deletable TIDs from posting lists and
deletes emptied posting-tree pages but never deletes a tuple or a page from the entry tree
([README#page-deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)).
Half the entry tree is dead weight that only a rebuild removes, and the method — which sees a
file that never changed size and a table whose `reltuples` halved — reached
`strong candidate: indexed population collapsed` and was right: the rebuild returned 49.47 %.
`i01` shows the same effect at smaller scale, 10,000 entries before the rebuild and 9,000
after.

### The verdict ladder, in order

Order is load-bearing.

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

Which rungs this run exercised, and which it cannot:

| Rung | Exercised by | Note |
|---|---|---|
| 1 | `e01` | the empty fixture, whose capture the statement refuses |
| 2 | nothing | still source-justified from `indisvalid` only |
| 3 | `c09`, and edge case E3 | both times ahead of the churn gate |
| 4 | `c11` | would have been a correct `candidate` on a fresh baseline |
| 5 | nothing | `e01` reaches rung 1 first, and every other fixture is analyzed |
| 6 | `a01`, `c04` | 0.1333 and 0.0000 |
| 7 | **nothing, and it cannot be reached** | the maintenance step analyzes every churned table, so `cur_analyze_count > bac` always holds here |
| 8 | `c03`, `k01`, `s01` | all three correct, 56.62 / 49.47 / 49.34 % |
| 9 | 11 fixtures | one of them, `c05`, is the run's only false positive |
| 10 | `a02`, `i01` | 22.70 % and 26.61 %, both below the declared payoff threshold |
| 11 | `c01`, `c06`, `p01`, `p02` | 11.55 / 6.60 / 5.86 / −21.01 % |

Rung 7 is the interesting absence. It exists to catch a table whose `reltuples` predates the
churn, and under the protocol's maintenance assumption that state cannot occur — so the guard
is now insurance against a server that does not maintain its tables, not something this page
can score. The same is true of `stats_lag`, which read **0.000 on all 24 fixtures** because the
maintenance `ANALYZE` zeroes `mod_since_analyze`
([pgstat_relation.c:331-337](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337)).

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

Measured confirmation on the three fixtures that reached the rung: `c03` at
`index_size_ratio` 1.0000, `heap_tuple_ratio` 0.4000, `norm` 2.5000; `k01` and `s01` both at
1.0000 / 0.5000 / 2.0000 — all three flagged by the growth rule too, and invariant I1 shows
that no un-rebuilt fixture in the run had `index_size_ratio` below 1.

Keep the rule, because "the indexed population collapsed" is more actionable than
"disproportionate growth" and it costs one `AND`. Do not expect it to detect anything the
growth rule misses.

### The statistics counter you must not build the guard on

The first ladder used `n_mod_since_analyze > 0.10 * reltuples` as a hard stale-statistics veto.
It is unusable, and this run reproduces why on **4 of 4 deliberate attempts**: a `DELETE`
followed immediately by `VACUUM (ANALYZE)` in the same session leaves the statistics view
describing changes the `ANALYZE` already saw.

| attempt | `n_live_tup` | `n_dead_tup` | `n_mod_since_analyze` | `pg_class.reltuples` |
|---|---|---|---|---|
| 1 | 120000 | **15000** | **15000** | 135000 |
| 2 | 105000 | **15000** | **15000** | 120000 |
| 3 | 90000 | **15000** | **15000** | 105000 |
| 4 | 75000 | **15000** | **15000** | 90000 |
| 5, with a 2 s pause before the `VACUUM (ANALYZE)` | 75000 | 0 | 0 | 75000 |

Each pair deleted exactly 15,000 rows and then analyzed; the true dead count afterwards is 0
and the true `mod_since_analyze` is 0, which is what attempt 5 reads once the pause lets the
flush land first. `pg_class.reltuples` — written by `vac_update_relstats`, not by pgstat —
is correct in every row.

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

The error is one-directional: the counter over-reports changes the `ANALYZE` did in fact see, so
it raises false staleness alarms and never false all-clears. That makes it safe as the advisory
`stats_lag` column and unsafe as a veto. The veto that replaced it, `cur_analyze_count <= bac`,
is immune, because `analyze_count` is incremented in the same locked section as the reset
([pgstat_relation.c:339-348](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L339-L348)).

The same race affects **baseline capture**, and the measured form is sharper than the old text
claimed. A capture that runs in a *different session* from the loader reads the full counts,
because the loader's backend published them when it exited. A capture in the **same
transaction** as the load cannot: measured on an 80,000-row table built and captured in one
transaction, the payload recorded `bti = 0` while `n_tup_ins` reached 80,000 immediately
afterwards, and repeating the capture after `SELECT pg_stat_force_next_flush();`
([pg_proc.dat:5916-5920](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5916-L5920))
recorded `bti = 80000`. The effect of the stale value is a permanently over-stated churn delta,
so the churn gate opens earlier than intended. A monitoring session cannot force another
backend's flush, so baseline churn counters can still lag a busy writer by up to
`PGSTAT_MIN_INTERVAL`, or `PGSTAT_MAX_INTERVAL` (60 s) in the worst case
([pgstat.c:117-122](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122)).

### The four cross-checks and the six invariants

The method reads only catalogs, `pg_stat_all_tables` and `pg_relation_size`. The protocol's
cross-checks are the *run's* obligation, not the method's, so the script takes
`SHARE ROW EXCLUSIVE` on each fixture's table and censuses every page of its index with
`pageinspect`, beside the FSM and the metapage.

| Check | How it was run | Result |
|---|---|---|
| FSM free pages against the census's recyclable count | `pg_freespace` per block against the census's new-plus-deleted count | **23 of 23**, and the two counts are **equal** on every fixture, not merely bounded |
| size bracket | `pg_relation_size(index, 'main')` re-read after the census, against the blocks it scanned | **23 of 23** |
| `VACUUM VERBOSE` | the fourth field of the index line — `pages_free` ([vacuumlazy.c#verbose-index-line](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L720-L731)) — from the maintenance step, against the census | **21 of 21** |
| metapage page-type counts | `n_entry_pages` and `n_data_pages` against the census's live pages | **22 of 23** exact; the exception is `a02`, off by one entry page |

The FSM equality is worth a sentence: an index FSM records only free-versus-in-use, writing a
free page as `BLCKSZ - 1`
([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)),
and the value read back is the category floor
([freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435)).
Every fixture with free pages read `max(avail) = 8160` from `pg_freespace`, which is
`MaxFSMRequestSize` on this build, derived from the running server rather than typed in.

Invariant I6 is the one that failed, and it failed exactly where the protocol says it must:
`a01` and `a02`, the two auto-analyze stand-ins, whose metapage page counts were never
rewritten because neither the worker's flush nor `gin_clean_pending_list` reaches
`ginUpdateStats`
([ginutil.c#ginUpdateStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650)).
`c09` is excluded from I6 rather than failing it: its census describes the out-of-band rebuild,
and its metapage row the churned index before it.

### The simulated auto-analyze census

After the maintenance step and before the decide phase, the run recomputes the launcher's
analyze verdict for every table in the fixture database from the effective reloption-or-GUC
values, and analyzes the tables it names.

| Table | `reltuples` | `n_mod_since_analyze` | threshold | `autovacuum_enabled` | verdict |
|---|---|---|---|---|---|
| `tc_past` | 10,000 | 2,000 | 1,050.00 | true | **analyze** |
| `tc_exact` | 10,000 | **1,050** | **1,050.00** | true | declined |
| `tc_off` | 10,000 | 2,000 | 1,050.00 | **false** | declined |
| the 25 fixture tables | 0 to 2,000,000 | **0** | 50 to 200,050 | true | declined |

**28 tables walked, 1 analyzed.** `tc_exact` lands exactly on its threshold and is left
alone, which is the engine's strictly-greater comparison measured
([autovacuum.c#verdicts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095));
`tc_off` is short-circuited by its reloption
([autovacuum.c#av_enabled-return](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054));
and every fixture table reads `n_mod_since_analyze = 0` because the maintenance step analyzed
it already.

That last column is a limit, not a success: the census's decision power on this page is
exercised only by the two synthetic tables, because the protocol's own maintenance step has
already analyzed everything the fixtures touched.

One defect in my first attempt is worth recording, because the protocol names it: the
census tables' build-phase `ANALYZE` originally ran **before** `pg_stat_force_next_flush()`,
so the load's own pending counts landed on the freshly zeroed counter and `tc_exact` came out
at 11,050 modifications instead of 1,050 — the first publication point, failing exactly as
documented. Publishing first and analyzing second fixed it.

### End-to-end run of the published statements

The three statements were extracted from this page and run **verbatim** — 41, 18 and 76 lines,
with no substitution at all — in their own database against `orders_tags_gin` on a
600,000-row `orders` table carrying the same two-line human comment, through the protocol's
phases:

| step | result |
|---|---|
| build + settle | index 14,778,368 bytes, table `reltuples` 600,000 |
| capture (statement 1) | `bis` 14778368, `bhr` 600000, `bfn` 21290, `bti` 600000, `bac` 1; human comment preserved |
| read (statement 2) | all nine fields round-trip; `human_comment` returns the two original lines only |
| churn: two full-table `UPDATE`s, then settle, then maintenance | 14,778,368 -> 36,978,688 bytes, 0 pending pages |
| decide (statement 3, under `SHARE ROW EXCLUSIVE`) | `index_size_ratio` 2.5022, `heap_tuple_ratio` 1.0000, `normalized_index_growth` 2.5022, `churn_ratio` 2.0000, `stats_lag` 0.000, verdict `candidate: disproportionate growth`, `est_reclaimable` 21 MB |
| oracle: `REINDEX INDEX` | 36,978,688 -> 14,770,176 = **60.06 %** returned, against a predicted **60.04 %** |
| re-capture (statement 1 again) | `bfn` 21290 -> 21295, `bis` 14770176, `btu` 1200000, `bac` 2, human comment intact, exactly one payload |
| re-evaluate | `insufficient churn: not evaluated` — the baseline is clean again |

The 0.02-point gap is the model at its best and still not a bound: the population never
changed, so `heap_tuple_ratio` is exactly 1, and the rebuild came back **one page smaller**
than the baseline (14,770,176 against 14,778,368) rather than exactly equal.

### Edge cases proven on the server

| # | case | result |
|---|---|---|
| E1 | two-line human comment containing `{do not drop}` and `@ 100%` | preserved through capture; comment 196 bytes, payload 139 |
| E2 | plain `REINDEX INDEX` | index OID stays 21221, relfilenode 21221 -> 21225, comment intact |
| E3 | `REINDEX INDEX CONCURRENTLY` | index OID moves 21221 -> 21226 and the comment follows, payload and human text unchanged |
| E4 | evaluate after E3 | `rebuilt since baseline: re-capture` |
| E5 | `ALTER INDEX ... RENAME` | comment survives (same OID) |
| E6 | `COMMENT ON INDEX` lock footprint | one row in `pg_locks`: `ShareUpdateExclusiveLock` on the index, and none on the table |
| E7 | non-owner with `SELECT` and `pg_read_all_stats` | reads the baseline fine (`bis` 1064960); write refused with `must be owner of index edge_gin` |
| E11 | capture aimed at a B-tree | refuses: `not a GIN index: fresh_btree` |
| E13 | 80,000-row table, no `ANALYZE` | `reltuples` `-1` before any index, `80000` after `CREATE INDEX`, so capture at `t0` succeeds |
| E14 | `TRUNCATE` then capture | `reltuples` resets to `-1`; capture refuses: `refusing baseline for t0_gin: table reltuples is -1 (run ANALYZE first)` |
| E15 | `pg_dump -t ... --section=post-data` | emits one `COMMENT ON INDEX ... @ginbase:{...}` line, so the baseline survives dump and restore |

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
  The protocol's own measurement lock is stronger still — `SHARE ROW EXCLUSIVE` on the table
  blocks every writer for the duration — and is a property of the *run*, not advice for
  production.
- **`REINDEX` is the action, not `VACUUM`.** The same-version `REINDEX` documentation lists a
  bloated index — "it contains many empty or nearly-empty pages" — as a reason to rebuild
  ([reindex.sgml:54-64](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64)).

### Recommended thresholds

Starting values, to be re-calibrated per installation. These are heuristics for building a
shortlist, not a bloat measurement.

| knob | value | basis on this run |
|---|---|---|
| `churn_ratio` gate | 0.20 | the two fixtures below it returned 0.00 % (`c04`) and 25.52 % (`a01`), so the gate costs one real 25 % opportunity and saves one pointless evaluation |
| `candidate` | `norm >= 1.50` | separates the eleven flagged fixtures, which returned 30.45 % to 81.93 %, from `a02`/`i01` at 22.70 % and 26.61 % |
| `watch` | `norm >= 1.20` | catches `a02` and `i01`; neither is worth a rebuild at this page's declared 33.33 % payoff threshold |
| strong-candidate shrink test | `htr <= 0.50` | fired on `c03`, `k01`, `s01`, all correct; the `isr >= 0.75` clause is vacuous on GIN and is kept as documentation |
| `est_reclaimable` | **do not use as bytes** | declared an upper bound, violated on 7 of 21, demoted to a level |

Lowering `candidate` to 1.25 would add `a02` (22.70 %) and `i01` (26.61 %) to the flagged set
and would not have avoided the one false positive, which sits at 1.9184. On
`fastupdate = on` indexes smaller than a few hundred megabytes, no threshold on this statistic
recovers the pending-list high-water mark: `p01` reads 1.0284 with 5.86 % returned while
`p02`, which never grew at all, would *lose* 21 % to a rebuild. Only a periodic rebuild, or
`fastupdate = off`, addresses that — and note that flipping the reloption takes an
`AccessExclusiveLock` on the index
([reloptions.c:123-131](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)),
so it is not an online change.

### Coverage the protocol requires, and what this page skipped

| Behavior the protocol requires | Reached by | |
|---|---|---|
| keys that no longer occur after churn | `k01` (3,001 -> 1,501 entries on rebuild), `i01` | yes |
| emptied posting-tree pages | `s01` (60 pages deleted, then recycled) | yes |
| half-empty posting-tree leaves with nothing deletable | `c02`, `c05`, `c11` (9,988 and 5,372 data pages after churn) | yes |
| a populated pending list, and the same index after a flush | `p01`, `m01`, `a01`, `a02` | yes |
| an untouched index and an empty index | `c04`, `e01` | yes |
| a snapshot held across the settling `VACUUM` | `s01` | yes |
| a `VACUUM` whose index cleanup did not run | `i01` | yes |
| more than one operator class | `array_ops`, `tsvector_ops`, `jsonb_path_ops`, `gin_trgm_ops`, `btree_gin int4_ops`, plus partial, two-column and expression indexes | yes |
| one rebuild at more than one `maintenance_work_mem` | the 64MB / 256MB / 1GB probe | yes |
| a churned index measured before and after the maintenance step | `m01` | yes |
| an index maintained by the auto-analyze stand-in | `a01`, `a02` | yes |
| a table the census analyzed, and one it declined | `tc_past`, `tc_exact`, `tc_off` | yes |
| an undecodable or unclassifiable page, and an all-zero page | **skipped** | the method reads no index page, so it cannot be scored on one; the run's census would raise on such a page rather than classify it |
| a concurrent `VACUUM`, a concurrent rebuild and a writer stream | **partly skipped** | `c09` covers an out-of-band rebuild between the maintenance step and the decide pass; no concurrent writer or `VACUUM` ran against a decide pass, because the measurement lock excludes both |

### What left the page with its fixtures

The asker's instruction was to remove what cannot conform, with the claims it backed. What
went, and why:

| Removed | Why | What replaced it |
|---|---|---|
| cell `c07`: `fastupdate = on`, `UPDATE` 30 %, **no `VACUUM`** | unmaintained churn, which the maintenance assumption forbids | `p01`/`p02`, the same mechanism measured in a maintained state |
| cell `c08`: `DELETE` 60 %, no `VACUUM`, no `ANALYZE` | unmaintained churn; and its point — the `bac` gate catching a missing `ANALYZE` — is unreachable once the maintenance step always analyzes | the ladder table now states that rung 7 cannot fire under the protocol |
| the six-point proportional-growth sweep's +10 %, +25 % and +50 % rows, and the 20,692,992-byte plateau they shared | all three were read before any `VACUUM` | `p01`/`p02`, plus the foreground-flush arithmetic that explains the plateau |
| the `fastupdate` probe's "after +150,000 inserts, before `VACUUM`" line as a *scored* reading | pre-settle state | the same reading kept as churn-phase evidence, explicitly unscored |
| the whole 12-cell matrix and its byte values | prose-only recipes; nothing on the page could re-run them | 24 fixtures whose recipes are lines of the published script |
| the `Test methodology` section | superseded | [Measurement Script](#measurement-script) |
| the `Re-verification on a second 17.11 build` section | its entire subject was that the fixture SQL had never been published | the script is published, so any run is a re-run |
| the claim that a fresh build "quadruples between 1M and 2M rows, then adds zero bytes from 2M to 4M" | did not reproduce | 2.67x then 1.38x, with the entry/data page split for each point |
| the claim that a 64MB build is "32.8 % larger" than a 256MB one, and the one-page inconsistency filed against it | did not reproduce | 20.3 % larger, with the metapage identity exact on 9 of 9 builds |
| the claim that the pgstat race "fired on 1 of 4" attempts | did not reproduce | 4 of 4, with a 2 s pause reading clean |
| the claim that `norm` 1.0049 hid 20.39 % reclaimable, and that the +50 % row's 0.8374 hid 0.48 % | fixtures gone | `p01` at 1.0284 with 5.86 %, and `p02` at 0.8000 with −21.01 % |
| the claim that `REINDEX` "returned both indexes to exactly 16,474,112 bytes" | the twins do rebuild to the same size, but it is **larger** than either churned file | the `p01`/`p02` table, including the negative oracle |
| open question 13, that nothing executable survived the sandbox | resolved | the script |
| open question 14, the internally inconsistent 64MB row | resolved | I2 exact on 25 baselines and 9 probe builds |

## Measurement Script

Every number on this page comes from one script, `gin_norm_protocol.sh`, filed in full under
[The script](#the-script). It is Bash and SQL only: a reviewer needs a C toolchain, a shell
and this page.

### How to use it

| Item | What to give |
|---|---|
| Purpose | Builds PostgreSQL 17.11 out of tree from this repository's pinned checkout and runs this page's whole programme under [Mandatory GIN Bloat Tests](../../common-concepts/mandatory-gin-bloat-tests.md): the declared kinds filed before any fixture exists, 24 scored fixtures plus one refusal fixture through build -> baseline -> churn (writes, settle, maintenance) -> decide -> oracle, the simulated auto-analyze census, the four cross-checks and six invariants, the five mechanism probes, eleven edge cases, and a verbatim end-to-end replay of the three published statements |
| Invocation | `bash .wiki-runtime/tmp/ginnorm/gin_norm_protocol.sh [stage ...]`, run from the repository root. With no arguments it runs every stage except `reset` and `clean`. Extract the fenced script below to that path first |
| Stages | Default order: `build declare fixtures churn analyze_census crosscheck decide oracle score probes edge pubsql verify`. `build` configures, builds and installs out of tree (skipped when the binary is already there), runs `make check` plus the five contrib suites this page reads, `initdb`s and starts the cluster; `declare` files the declared kind of every published column, the decision thresholds and the six invariants, and **refuses to run once the fixture database exists**; `fixtures` runs the build phase and the baseline capture for all 25 fixtures; `churn` runs each recipe's writes, then the settle `VACUUM`, then the maintenance `VACUUM ANALYZE`, with the auto-analyze stand-in and the held-snapshot, no-cleanup, out-of-band-rebuild and counter-reset cases in line; `analyze_census` recomputes the launcher's analyze verdict for every table and analyzes the ones it names; `crosscheck` censuses every page of every fixture index under `SHARE ROW EXCLUSIVE` and reads the FSM; `decide` runs statement 3 verbatim inside one locked transaction, then stores the same rows for scoring; `oracle` rebuilds each index between two `pg_relation_size` readings; `score` checks the filed declarations against the ones the scoring assumes and prints the bound verdicts, the decision scores, the accuracy bands, the six invariants and every phase size; `probes` runs the `reltuples` swing, the six-point linearity sweep, the three-budget rebuild, the pgstat race and the capture race; `edge` runs the eleven edge cases in their own database; `pubsql` replays the three statements verbatim through a whole lifecycle; `verify` re-extracts them from this page and diffs them against the files that ran. `reset` drops the databases so a re-run starts clean, `start` starts an already-built cluster, and `clean` is not in the default order and must be run last |
| Environment | `REPO` (`$PWD`), `SRC` (`$REPO/raw/postgres-17`), `SANDBOX` (`$REPO/.wiki-runtime/tmp/ginnorm`), `PAGE` (this file), `PORT` (`55417`), `JOBS` (`20`), `BASE_ROWS` (`1000000`), `PEND_ROWS` (`600000`), `SMALL_ROWS` (`300000`), `TRGM_ROWS` (`100000`), `A01_INSERTS` (`40000`), `A02_INSERTS` (`60500`), `KEYS` (`10000`), `MWM` (`256MB`) |
| Prerequisites | See [Prerequisites](#prerequisites) |
| Output | Everything lands under `$SANDBOX/out/`; see [Where the results land](#where-the-results-land). Read `score-table.txt` and `score-summary.txt` first |
| Runtime | About 7 minutes from a built tree on the recorded host, and about 10 from an empty sandbox: 2 min 30 s for `make check` and the five contrib suites, 38 s for the 25 fixtures, 1 min 50 s for the churn phase (45 s of which is `s01`'s held snapshot), under 2 s each for the census, cross-check, decide, oracle and score stages, 40 s for the probes, and 20 s for the edge and pubsql stages together |
| Cleanup | `bash gin_norm_protocol.sh clean` stops the cluster with `pg_ctl -m fast -w stop`, reports whether any `postmaster.pid` or matching `postgres` process survived and whether the port is free, and deletes the whole sandbox |

### Prerequisites

- A C toolchain, `make`, `flex`, `bison` and `perl`. The recorded run used gcc 13.3.0 on
  `Linux x86_64`. The script builds its own server; no installed PostgreSQL is used, and it
  never touches a cluster it did not create.
- No ICU or readline development headers are needed: the build configures
  `--without-icu --without-readline --with-zlib --enable-debug`, and `initdb` runs with
  `--locale=C --encoding=UTF8`, which is what makes the `pg_trgm` and `tsvector` fixtures
  deterministic.
- The pinned checkout present at `raw/postgres-17`, read-only. The script builds out of tree
  and writes nothing inside it.
- Port 55417 free, and about 8 GB under `.wiki-runtime/tmp/`.
- The `pageinspect`, `pgstattuple`, `pg_freespacemap`, `pg_trgm` and `btree_gin` contrib
  modules, which `make -C contrib install` provides from the same tree.
- Every `psql` call is `psql -X -v ON_ERROR_STOP=1` against the sandbox's own socket
  directory, so a stray `~/.psqlrc` cannot change a result and no error passes silently. The
  few statements whose *error text* is the result being measured are read from the stage log
  instead.

### Where the results land

| File | What is in it |
|---|---|
| `declared_kind.txt`, `declared_decision.txt`, `declared_invariant.txt`, `declared_at.txt` | what was declared, and the timestamp it was filed at |
| `check-summary.txt`, `check-*.log` | `make check` and the five contrib suites |
| `baseline-sizes.txt` | the as-built size of every fixture index |
| `build-*.log`, `churn-*.log`, `settle-*.log`, `maint-*.log`, `nocleanup-i01.log`, `standin-a0*.log`, `settle2-s01.log`, `snapshot-s01.log`, `outofband-c09.log` | every phase's own output, including every `VACUUM VERBOSE` index line |
| `census-verdicts.txt`, `census-analyzed.txt` | the simulated analyze census: one line per table, and the `ANALYZE` statements it generated |
| `census-summary.txt` | the page census per fixture: scanned, entry, data, list, deleted, new, FSM |
| `decide.txt` | the published statement's own output, under the measurement lock |
| `oracle-summary.txt` | the two size readings and the returned percentage per fixture |
| `score-table.txt`, `score-summary.txt` | the scored table and its totals |
| `invariants.txt`, `phase-sizes.txt` | the six invariants, and every phase size with the pending-page transition |
| `probe-p1.txt`, `probe-p2.txt`, `probe-p4.txt`, `probe-p5.txt` | the `reltuples` swing, the linearity and budget sweeps, the pgstat race, the capture race |
| `edge.txt`, `pubsql.txt`, `verify.txt` | the eleven edge cases, the verbatim lifecycle, and the three-way diff against this page |
| `settings.txt`, `version.txt`, `pin.txt` | the settings the run fixes with their contexts, the server version, the commit the source is parked on |

### The last run

| Fact | Value |
|---|---|
| Date | 2026-09-15 |
| Server | PostgreSQL 17.11, built from `786db8dcf168bd9df8f55047337525ac19118b1c` (`REL_17_11-7-g786db8dcf16`) |
| Platform | `Linux x86_64`, Ubuntu 24.04 on a WSL2 kernel, gcc 13.3.0, 22 cores, `JOBS=20` |
| `block_size` | 8192 |
| `max_data_alignment` | 8 |
| Regression suites | core **All 225**, `pageinspect` **All 8**, `pgstattuple` **All 1**, `pg_freespacemap` **All 1**, `btree_gin` **All 30**, `pg_trgm` **All 4** |
| Cluster settings, with contexts | `autovacuum = off` (`PGC_SIGHUP`, reload), `shared_buffers = 512MB` (`PGC_POSTMASTER`, restart), `maintenance_work_mem = 256MB`, `work_mem = 64MB`, `gin_pending_list_limit = 4096 kB`, `statement_timeout = 900s`, `lock_timeout = 15s`, `stats_fetch_consistency = cache` (all `PGC_USERSET`, session/transaction scope) |
| Declarations filed | 2026-09-15T17:24:53Z, three seconds before the first fixture's baseline payload at 17:24:56Z |
| Published statements | 3 of 3 byte-identical to this page, at 41 / 18 / 76 lines |
| Filed text | the fenced script below was diffed against the file that ran: identical |

### The script

```bash
#!/usr/bin/env bash
#
# gin_norm_protocol.sh - the measurement programme behind
# wiki/v17/questions/indexing/gin-reindex-normalized-growth-comment-baseline.md
#
# Runs the COMMENT-baseline / normalized-index-growth heuristic for GIN indexes
# under the wiki's Mandatory GIN Bloat Tests protocol: five phases per fixture
# (build, baseline, churn ending in settle + maintenance + census, decide under
# the measurement lock, and a measured REINDEX INDEX oracle), a declared kind per
# published column filed before the first fixture exists, and the four mandatory
# cross-checks.
#
# Every object this script creates is DISPOSABLE. It builds its own PostgreSQL
# out of tree, runs its own cluster on a non-default port with its own socket
# directory, and the `clean` stage stops that cluster and deletes the sandbox.
# It never touches a cluster it did not start, and it treats raw/postgres-17 as
# read-only.
#
# Usage:  bash gin_norm_protocol.sh [stage ...]      (run from the repo root)
#         bash gin_norm_protocol.sh                  (same as `all`)
#
set -uo pipefail

REPO="${REPO:-$PWD}"
SRC="${SRC:-$REPO/raw/postgres-17}"
SANDBOX="${SANDBOX:-$REPO/.wiki-runtime/tmp/ginnorm}"
PAGE="${PAGE:-$REPO/wiki/v17/questions/indexing/gin-reindex-normalized-growth-comment-baseline.md}"
PORT="${PORT:-55417}"
JOBS="${JOBS:-20}"
BASE_ROWS="${BASE_ROWS:-1000000}"
PEND_ROWS="${PEND_ROWS:-600000}"
SMALL_ROWS="${SMALL_ROWS:-300000}"
TRGM_ROWS="${TRGM_ROWS:-100000}"
# a01 sits above the analyze threshold and below the method's churn gate;
# a02 sits in the 1000-row window where both are satisfied at once
A01_INSERTS="${A01_INSERTS:-40000}"
A02_INSERTS="${A02_INSERTS:-60500}"
KEYS="${KEYS:-10000}"
MWM="${MWM:-256MB}"

BUILD="$SANDBOX/build"
INST="$SANDBOX/install"
BIN="$INST/bin"
PGDATA="$SANDBOX/data"
SOCK="$SANDBOX/sock"
OUT="$SANDBOX/out"
SQLD="$SANDBOX/sql"
LOG="$SANDBOX/server.log"
DB=ginnorm
PDB=protocol

export PGOPTIONS="-c statement_timeout=900s -c lock_timeout=15s"

mkdir -p "$SANDBOX" "$SOCK" "$OUT" "$SQLD"

say() { printf '\n=== %s\n' "$*"; }
die() { printf 'FATAL: %s\n' "$*" >&2; exit 1; }

q()  { "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$1" -At -c "$2"; }
Q()  { "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$1" -c "$2"; }
qf() { "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$1" -f "$2"; }
qin(){ "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$1"; }
qat(){ "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$1" -At -f "$2"; }

# ---------------------------------------------------------------- the method
# The three statements below are the page's published SQL, byte for byte. The
# `verify` stage re-extracts them from the page and diffs them against these
# files, so the text that ran is provably the text that is published.
write_published_sql() {
  cat > "$SQLD/capture.sql" <<'WIKISQL'
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
WIKISQL

  cat > "$SQLD/read.sql" <<'WIKISQL'
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
WIKISQL

  cat > "$SQLD/evaluate.sql" <<'WIKISQL'
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
WIKISQL
}

# capture the baseline for one index, using the published text with only the
# target line substituted
capture_for() { # $1 = index name
  sed "s/public\.orders_tags_gin/$1/" "$SQLD/capture.sql" > "$SQLD/.capture_run.sql"
  qf "$DB" "$SQLD/.capture_run.sql" > /dev/null || die "capture failed for $1"
}

# ---------------------------------------------------------------- stage: build
stage_build() {
  say "build: 17.11 out of tree from $SRC"
  [ -d "$SRC" ] || die "no source checkout at $SRC"
  if [ ! -x "$BIN/postgres" ]; then
    mkdir -p "$BUILD"
    ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --without-icu --without-readline \
        --with-zlib --enable-debug > "$SANDBOX/configure.log" 2>&1 ) || die "configure failed"
    ( cd "$BUILD" && make -j"$JOBS" > "$SANDBOX/make.log" 2>&1 ) || die "make failed"
    ( cd "$BUILD" && make -C contrib -j"$JOBS" > "$SANDBOX/make-contrib.log" 2>&1 ) || die "contrib make failed"
    ( cd "$BUILD" && make install > "$SANDBOX/install.log" 2>&1 ) || die "install failed"
    ( cd "$BUILD" && make -C contrib install > "$SANDBOX/install-contrib.log" 2>&1 ) || die "contrib install failed"
  fi
  "$BIN/postgres" --version | tee "$OUT/version.txt"
  ( cd "$SRC" && git log -1 --format='%H %D' ) | tee "$OUT/pin.txt"

  say "build: make check plus the five contrib suites this run reads"
  ( cd "$BUILD" && make check > "$OUT/check-core.log" 2>&1 )
  for c in pageinspect pgstattuple pg_freespacemap btree_gin pg_trgm; do
    ( cd "$BUILD/contrib/$c" && make check > "$OUT/check-$c.log" 2>&1 )
  done
  : > "$OUT/check-summary.txt"
  for f in "$OUT"/check-*.log; do
    printf '%-28s %s\n' "$(basename "$f")" \
      "$(grep -Eo '(All [0-9]+ tests? passed|[0-9]+ of [0-9]+ tests failed)' "$f" | tail -1)" \
      >> "$OUT/check-summary.txt"
  done
  cat "$OUT/check-summary.txt"

  if [ ! -d "$PGDATA" ]; then
    "$BIN/initdb" -D "$PGDATA" --locale=C --encoding=UTF8 > "$SANDBOX/initdb.log" 2>&1 \
      || die "initdb failed"
    cat >> "$PGDATA/postgresql.conf" <<CONF
# every setting below is recorded on the page with its context and apply scope
listen_addresses = ''
unix_socket_directories = '$SOCK'
port = $PORT
shared_buffers = 512MB
maintenance_work_mem = $MWM
work_mem = 64MB
autovacuum = off
log_line_prefix = '%m [%p] '
CONF
  fi
  stage_start
}

stage_start() {
  if "$BIN/pg_ctl" -D "$PGDATA" status > /dev/null 2>&1; then
    say "start: already running on port $PORT"
  else
    say "start: postmaster on port $PORT, socket $SOCK"
    "$BIN/pg_ctl" -D "$PGDATA" -l "$LOG" -w start > /dev/null || die "pg_ctl start failed"
  fi
  q postgres "SELECT /* wiki_gin_norm_hello */ version()" | tee -a "$OUT/version.txt"
  q postgres "SELECT /* wiki_gin_norm_settings */ name || ' = ' || setting || ' [' || context || ']'
                FROM pg_settings
               WHERE name IN ('autovacuum','block_size','gin_pending_list_limit','maintenance_work_mem',
                              'shared_buffers','stats_fetch_consistency','autovacuum_analyze_threshold',
                              'autovacuum_analyze_scale_factor','autovacuum_vacuum_threshold',
                              'autovacuum_vacuum_scale_factor','autovacuum_vacuum_insert_threshold',
                              'autovacuum_vacuum_insert_scale_factor','autovacuum_naptime',
                              'statement_timeout','lock_timeout')
               ORDER BY name" | tee "$OUT/settings.txt"
}

# ------------------------------------------------------------- stage: declare
# Files the declared kind of every published column BEFORE any fixture exists.
# The protocol forbids rewriting a declaration after the run, so this stage
# refuses to run once the fixture database is there.
stage_declare() {
  say "declare: the kind of every published column, filed before the first fixture"
  if q postgres "SELECT /* wiki_gin_norm_guard */ count(*) FROM pg_database WHERE datname = '$DB'" \
     | grep -q '^1$'; then
    die "refusing to re-file declarations: database $DB already exists (run reset first)"
  fi
  q postgres "DROP /* wiki_gin_norm_declare */ DATABASE IF EXISTS $PDB" > /dev/null
  q postgres "CREATE /* wiki_gin_norm_declare */ DATABASE $PDB" > /dev/null
  qin "$PDB" <<'SQL'
-- DISPOSABLE bookkeeping objects.
CREATE /* wiki_gin_norm_declare */ TABLE declared_kind (
  column_name text primary key,
  declared_kind text not null check (declared_kind in ('lower bound','upper bound','level')),
  claim text not null,
  filed_at timestamptz not null default now()
);
INSERT /* wiki_gin_norm_declare */ INTO declared_kind (column_name, declared_kind, claim) VALUES
 ('est_reclaimable',         'upper bound', 'the bytes it names are never fewer than the bytes REINDEX INDEX returns'),
 ('normalized_index_growth', 'level',       'a ranking statistic; no claim against the oracle'),
 ('index_size_ratio',        'level',       'current over baseline file size; no claim against the oracle'),
 ('heap_tuple_ratio',        'level',       'current over baseline table reltuples; no claim against the oracle'),
 ('churn_ratio',             'level',       'write volume over baseline reltuples; no claim against the oracle'),
 ('stats_lag',               'level',       'advisory staleness reading; no claim against the oracle'),
 ('cur_size',                'level',       'the current file size itself'),
 ('base_size',               'level',       'the stored baseline file size itself'),
 ('cur_heap_reltuples',      'level',       'the table row estimate itself'),
 ('base_heap_reltuples',     'level',       'the stored baseline row estimate itself');

CREATE /* wiki_gin_norm_declare */ TABLE declared_decision (
  knob text primary key, value text not null, meaning text not null,
  filed_at timestamptz not null default now()
);
INSERT /* wiki_gin_norm_declare */ INTO declared_decision (knob, value, meaning) VALUES
 ('candidate',            'normalized_index_growth >= 1.50', 'the method flags a rebuild'),
 ('strong candidate',     'heap_tuple_ratio <= 0.50 AND index_size_ratio >= 0.75', 'the method flags a rebuild'),
 ('watch',                'normalized_index_growth >= 1.20', 'the method does NOT flag a rebuild'),
 ('churn gate',           'churn_ratio >= 0.20', 'below this the method refuses to evaluate'),
 ('oracle justification', 'truth_pct >= 33.33', 'a rebuild is worth it, = 100 * (1 - 1/1.50), the method''s own threshold carried to the oracle side');

CREATE /* wiki_gin_norm_declare */ TABLE declared_invariant (
  id text primary key, claim text not null, filed_at timestamptz not null default now()
);
INSERT /* wiki_gin_norm_declare */ INTO declared_invariant (id, claim) VALUES
 ('I1', 'index_size_ratio >= 1 on every fixture that was not rebuilt since its baseline'),
 ('I2', 'a freshly built index satisfies n_total_pages = n_entry_pages + n_data_pages + n_pending_pages + 1'),
 ('I3', 'the FSM free-page count never exceeds the census new-plus-deleted page count'),
 ('I4', 'pg_relation_size re-read after the page census equals the blocks the census scanned'),
 ('I5', 'the fourth number of the VACUUM VERBOSE index line equals the census new-plus-deleted count'),
 ('I6', 'n_total_pages = 1 + census entry + census data + census list + census new-plus-deleted');
SQL
  q "$PDB" "SELECT /* wiki_gin_norm_declare */ column_name || ' -> ' || declared_kind || ' @ ' ||
              to_char(filed_at, 'YYYY-MM-DD HH24:MI:SS') FROM declared_kind ORDER BY 1" \
    | tee "$OUT/declared_kind.txt"
  q "$PDB" "SELECT /* wiki_gin_norm_declare */ knob || ': ' || value FROM declared_decision ORDER BY 1" \
    | tee "$OUT/declared_decision.txt"
  q "$PDB" "SELECT /* wiki_gin_norm_declare */ id || ': ' || claim FROM declared_invariant ORDER BY 1" \
    | tee "$OUT/declared_invariant.txt"
  date -u +'declarations filed at %Y-%m-%dT%H:%M:%SZ' | tee "$OUT/declared_at.txt"
}

stage_reset() {
  say "reset: drop both databases so a re-run starts clean"
  if "$BIN/pg_ctl" -D "$PGDATA" status > /dev/null 2>&1; then
    q postgres "DROP /* wiki_gin_norm_reset */ DATABASE IF EXISTS $DB" > /dev/null
    q postgres "DROP /* wiki_gin_norm_reset */ DATABASE IF EXISTS $PDB" > /dev/null
    q postgres "DROP /* wiki_gin_norm_reset */ DATABASE IF EXISTS pubsql" > /dev/null
  fi
}

# ------------------------------------------------------- fixture bookkeeping
# Every fixture object below is DISPOSABLE: created by this script, dropped
# with the sandbox.
SCORED="c01 c02 c03 c04 c05 c06 c09 c10 c11 c12 p01 p02 m01 a01 a02 s01 i01 o01 o02 o03 x01 x02 x03 k01"
UNSCORED="e01"
tbl() { printf 'f_%s' "$1"; }
idx() { printf 'f_%s_gin' "$1"; }

proto_ddl() {
  cat <<'SQL'
CREATE /* wiki_gin_norm_proto */ SCHEMA proto;
CREATE /* wiki_gin_norm_proto */ EXTENSION pageinspect;
CREATE /* wiki_gin_norm_proto */ EXTENSION pgstattuple;
CREATE /* wiki_gin_norm_proto */ EXTENSION pg_freespacemap;
CREATE /* wiki_gin_norm_proto */ EXTENSION pg_trgm;
CREATE /* wiki_gin_norm_proto */ EXTENSION btree_gin;

CREATE TABLE proto.meas (
  fixture text not null, phase text not null, metric text not null,
  num numeric, txt text, at timestamptz not null default clock_timestamp()
);

CREATE FUNCTION proto.note(p_fix text, p_phase text, p_metric text,
                           p_num numeric DEFAULT NULL, p_txt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS $fn$
  INSERT INTO proto.meas(fixture,phase,metric,num,txt)
  VALUES (p_fix,p_phase,p_metric,p_num,p_txt);
$fn$;

-- five keys per row, drawn from a p_keys-wide universe by a deterministic hash
CREATE FUNCTION proto.mk_tags(p_tab text, p_lo bigint, p_hi bigint,
                              p_keybase int, p_keys int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$INSERT INTO %I (id, tags) SELECT i, ARRAY[
      $1 + ((i *     7919) %% $2)::int, $1 + ((i *   104729) %% $2)::int,
      $1 + ((i *  1299709) %% $2)::int, $1 + ((i * 15485863) %% $2)::int,
      $1 + ((i * 32452843) %% $2)::int] FROM generate_series($3, $4) i$q$, p_tab)
    USING p_keybase, p_keys, p_lo, p_hi;
END $fn$;

-- rewrite every matching row's keys, from the same universe, under a new salt
CREATE FUNCTION proto.churn_tags(p_tab text, p_salt int, p_where text,
                                 p_keybase int, p_keys int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$UPDATE %I SET tags = ARRAY[
      $1 + (((id + $2) *     7919) %% $3)::int, $1 + (((id + $2) *   104729) %% $3)::int,
      $1 + (((id + $2) *  1299709) %% $3)::int, $1 + (((id + $2) * 15485863) %% $3)::int,
      $1 + (((id + $2) * 32452843) %% $3)::int] WHERE %s$q$, p_tab, p_where)
    USING p_keybase, p_salt, p_keys;
END $fn$;

-- one key per p_per_key contiguous rows, so deleting an id range retires whole keys
CREATE FUNCTION proto.mk_keyband(p_tab text, p_lo bigint, p_hi bigint, p_per_key int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$INSERT INTO %I (id, tags)
      SELECT i, ARRAY[(i / $1)::int] FROM generate_series($2, $3) i$q$, p_tab)
    USING p_per_key, p_lo, p_hi;
END $fn$;

-- five words per row from a p_words-wide vocabulary, plus a tsvector
CREATE FUNCTION proto.mk_docs(p_tab text, p_lo bigint, p_hi bigint, p_words int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$INSERT INTO %I (id, txt, doc)
      SELECT i, t, to_tsvector('simple', t) FROM (
        SELECT i, concat_ws(' ',
          'w' || ((i *     7919) %% $1), 'w' || ((i *   104729) %% $1),
          'w' || ((i *  1299709) %% $1), 'w' || ((i * 15485863) %% $1),
          'w' || ((i * 32452843) %% $1)) AS t
          FROM generate_series($2, $3) i) s$q$, p_tab)
    USING p_words, p_lo, p_hi;
END $fn$;

CREATE FUNCTION proto.churn_docs(p_tab text, p_salt int, p_where text, p_words int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$UPDATE %I SET txt = s.t, doc = to_tsvector('simple', s.t)
      FROM (SELECT id AS sid, concat_ws(' ',
          'w' || (((id + $1) *     7919) %% $2), 'w' || (((id + $1) *   104729) %% $2),
          'w' || (((id + $1) *  1299709) %% $2), 'w' || (((id + $1) * 15485863) %% $2),
          'w' || (((id + $1) * 32452843) %% $2)) AS t FROM %I WHERE %s) s
      WHERE id = s.sid$q$, p_tab, p_tab, p_where)
    USING p_salt, p_words;
END $fn$;

CREATE FUNCTION proto.mk_json(p_tab text, p_lo bigint, p_hi bigint, p_words int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$INSERT INTO %I (id, j)
      SELECT i, jsonb_build_object('tags', to_jsonb(ARRAY[
          'w' || ((i *     7919) %% $1), 'w' || ((i *   104729) %% $1),
          'w' || ((i *  1299709) %% $1), 'w' || ((i * 15485863) %% $1),
          'w' || ((i * 32452843) %% $1)]))
        FROM generate_series($2, $3) i$q$, p_tab)
    USING p_words, p_lo, p_hi;
END $fn$;

CREATE FUNCTION proto.churn_json(p_tab text, p_salt int, p_where text, p_words int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$UPDATE %I SET j = jsonb_build_object('tags', to_jsonb(ARRAY[
          'w' || (((id + $1) *     7919) %% $2), 'w' || (((id + $1) *   104729) %% $2),
          'w' || (((id + $1) *  1299709) %% $2), 'w' || (((id + $1) * 15485863) %% $2),
          'w' || (((id + $1) * 32452843) %% $2)])) WHERE %s$q$, p_tab, p_where)
    USING p_salt, p_words;
END $fn$;

-- one reading of everything the method and the cross-checks can see
CREATE FUNCTION proto.record(p_fix text, p_phase text, p_tab text, p_idx text)
RETURNS void LANGUAGE plpgsql AS $fn$
DECLARE
  m record; s record; ct record; ci record;
  blk int := current_setting('block_size')::int;
BEGIN
  SELECT * INTO m FROM gin_metapage_info(get_raw_page(p_idx, 0));
  SELECT reltuples, relpages INTO ct FROM pg_class WHERE oid = p_tab::regclass;
  SELECT reltuples, relpages, relfilenode INTO ci FROM pg_class WHERE oid = p_idx::regclass;
  SELECT n_tup_ins, n_tup_upd, n_tup_del, n_live_tup, n_dead_tup, n_mod_since_analyze,
         n_ins_since_vacuum, analyze_count, autoanalyze_count, vacuum_count
    INTO s FROM pg_stat_all_tables WHERE relid = p_tab::regclass;
  INSERT INTO proto.meas(fixture,phase,metric,num) VALUES
    (p_fix,p_phase,'index_size',          pg_relation_size(p_idx::regclass,'main')),
    (p_fix,p_phase,'index_blocks',        pg_relation_size(p_idx::regclass,'main')/blk),
    (p_fix,p_phase,'table_size',          pg_relation_size(p_tab::regclass,'main')),
    (p_fix,p_phase,'meta_total',          m.n_total_pages),
    (p_fix,p_phase,'meta_entry',          m.n_entry_pages),
    (p_fix,p_phase,'meta_data',           m.n_data_pages),
    (p_fix,p_phase,'meta_pending',        m.n_pending_pages),
    (p_fix,p_phase,'meta_pending_tuples', m.n_pending_tuples),
    (p_fix,p_phase,'meta_entries',        m.n_entries),
    (p_fix,p_phase,'meta_version',        m.version),
    (p_fix,p_phase,'tab_reltuples',       ct.reltuples),
    (p_fix,p_phase,'tab_relpages',        ct.relpages),
    (p_fix,p_phase,'idx_reltuples',       ci.reltuples),
    (p_fix,p_phase,'idx_relpages',        ci.relpages),
    (p_fix,p_phase,'idx_relfilenode',     ci.relfilenode::bigint),
    (p_fix,p_phase,'n_tup_ins',           coalesce(s.n_tup_ins,0)),
    (p_fix,p_phase,'n_tup_upd',           coalesce(s.n_tup_upd,0)),
    (p_fix,p_phase,'n_tup_del',           coalesce(s.n_tup_del,0)),
    (p_fix,p_phase,'n_live_tup',          coalesce(s.n_live_tup,0)),
    (p_fix,p_phase,'n_dead_tup',          coalesce(s.n_dead_tup,0)),
    (p_fix,p_phase,'n_mod_since_analyze', coalesce(s.n_mod_since_analyze,0)),
    (p_fix,p_phase,'n_ins_since_vacuum',  coalesce(s.n_ins_since_vacuum,0)),
    (p_fix,p_phase,'analyze_count',       coalesce(s.analyze_count,0)+coalesce(s.autoanalyze_count,0)),
    (p_fix,p_phase,'vacuum_count',        coalesce(s.vacuum_count,0));
END $fn$;

-- the page census and the two cross-checks that need it, under the caller's lock
CREATE FUNCTION proto.pagecensus(p_fix text, p_phase text, p_idx text)
RETURNS void LANGUAGE plpgsql AS $fn$
DECLARE
  blocks bigint; blk int := current_setting('block_size')::int;
  r record; fsm record; after bigint;
BEGIN
  blocks := pg_relation_size(p_idx::regclass,'main') / blk;
  SELECT
    count(*) FILTER (WHERE fl IS NULL)                              AS new_pages,
    count(*) FILTER (WHERE fl @> ARRAY['deleted'])                  AS deleted_pages,
    count(*) FILTER (WHERE fl @> ARRAY['deleted'] AND px = 0)       AS deleted_noxid,
    count(*) FILTER (WHERE fl IS NOT NULL AND NOT fl @> ARRAY['deleted']
                       AND fl @> ARRAY['list'])                     AS list_pages,
    count(*) FILTER (WHERE fl IS NOT NULL AND NOT fl @> ARRAY['deleted']
                       AND fl @> ARRAY['data'] AND fl @> ARRAY['leaf'])     AS data_leaf,
    count(*) FILTER (WHERE fl IS NOT NULL AND NOT fl @> ARRAY['deleted']
                       AND fl @> ARRAY['data'] AND NOT fl @> ARRAY['leaf']) AS data_inner,
    count(*) FILTER (WHERE fl IS NOT NULL AND NOT fl @> ARRAY['deleted']
                       AND NOT fl @> ARRAY['data'] AND NOT fl @> ARRAY['list']) AS entry_pages,
    count(*) AS scanned
    INTO r
    FROM (SELECT (gin_page_opaque_info(get_raw_page(p_idx, b::int))).flags AS fl,
                 (page_header(get_raw_page(p_idx, b::int))).prune_xid      AS px
            FROM generate_series(1, blocks - 1) b) p;
  SELECT count(*) FILTER (WHERE avail > 0) AS free_pages, max(avail) AS max_avail
    INTO fsm FROM pg_freespace(p_idx::regclass);
  after := pg_relation_size(p_idx::regclass,'main') / blk;
  INSERT INTO proto.meas(fixture,phase,metric,num) VALUES
    (p_fix,p_phase,'census_scanned',     r.scanned),
    (p_fix,p_phase,'census_new',         r.new_pages),
    (p_fix,p_phase,'census_deleted',     r.deleted_pages),
    (p_fix,p_phase,'census_deleted_noxid', r.deleted_noxid),
    (p_fix,p_phase,'census_list',        r.list_pages),
    (p_fix,p_phase,'census_data_leaf',   r.data_leaf),
    (p_fix,p_phase,'census_data_inner',  r.data_inner),
    (p_fix,p_phase,'census_entry',       r.entry_pages),
    (p_fix,p_phase,'census_recyclable',  r.new_pages + r.deleted_pages),
    (p_fix,p_phase,'fsm_free_pages',     fsm.free_pages),
    (p_fix,p_phase,'fsm_max_avail',      coalesce(fsm.max_avail,0)),
    (p_fix,p_phase,'bracket_blocks_before', blocks),
    (p_fix,p_phase,'bracket_blocks_after',  after);
END $fn$;
SQL
}

build_sql() { # $1 = fixture id; prints the whole build phase for that fixture
  local f=$1 t; t=$(tbl "$f")
  case "$f" in
    c01|c02|c03|c04|c05|c09|c11|c12|m01|i01)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$(rows_for "$f")" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags);\n" "$(idx "$f")" "$t" ;;
    s01)
      # few keys, so every key's posting list becomes a posting tree: deleting a
      # contiguous id range then empties whole data pages
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_keyband('%s', 1, %s, 100000);\n" "$t" "$(rows_for "$f")"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags);\n" "$(idx "$f")" "$t" ;;
    c06)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$BASE_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags);\n" "$(idx "$f")" "$t" ;;
    p01|p02)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$PEND_ROWS" "$KEYS"
      if [ "$f" = p01 ]; then
        printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags) WITH (fastupdate = on);\n" "$(idx "$f")" "$t"
      else
        printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags) WITH (fastupdate = off);\n" "$(idx "$f")" "$t"
      fi ;;
    a01|a02)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$SMALL_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags) WITH (fastupdate = on);\n" "$(idx "$f")" "$t" ;;
    c10)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, txt text, doc tsvector);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_docs('%s', 1, %s, %s);\n" "$t" "$SMALL_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (doc);\n" "$(idx "$f")" "$t" ;;
    o01)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, j jsonb);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_json('%s', 1, %s, %s);\n" "$t" "$SMALL_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (j jsonb_path_ops);\n" "$(idx "$f")" "$t" ;;
    o02)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, txt text, doc tsvector);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_docs('%s', 1, %s, %s);\n" "$t" "$TRGM_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (txt gin_trgm_ops);\n" "$(idx "$f")" "$t" ;;
    o03)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, n int);\n" "$t"
      printf "INSERT /* wiki_gin_norm_fixture */ INTO %s (id, n) SELECT i, (i %% %s)::int FROM generate_series(1, %s) i;\n" "$t" "$KEYS" "$SMALL_ROWS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (n);\n" "$(idx "$f")" "$t" ;;
    x01)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$SMALL_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags) WHERE id %% 4 = 0;\n" "$(idx "$f")" "$t" ;;
    x02)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[], tags2 int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$SMALL_ROWS" "$KEYS"
      printf "UPDATE /* wiki_gin_norm_fixture */ %s SET tags2 = ARRAY[(id %% %s)::int];\n" "$t" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags, tags2);\n" "$(idx "$f")" "$t" ;;
    x03)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_tags('%s', 1, %s, 0, %s);\n" "$t" "$SMALL_ROWS" "$KEYS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin ((tags[1:3]));\n" "$(idx "$f")" "$t" ;;
    k01)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "SELECT /* wiki_gin_norm_fixture */ proto.mk_keyband('%s', 1, %s, 100);\n" "$t" "$SMALL_ROWS"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags);\n" "$(idx "$f")" "$t" ;;
    e01)
      printf "CREATE /* wiki_gin_norm_fixture */ TABLE %s (id bigint, tags int[]);\n" "$t"
      printf "CREATE /* wiki_gin_norm_fixture */ INDEX %s ON %s USING gin (tags);\n" "$(idx "$f")" "$t" ;;
  esac
}

rows_for() {
  case "$1" in
    c01|c02|c03|c04|c05|c06|c09|c11|c12|s01) printf '%s' "$BASE_ROWS" ;;
    p01|p02)                             printf '%s' "$PEND_ROWS" ;;
    o02)                                 printf '%s' "$TRGM_ROWS" ;;
    e01)                                 printf '0' ;;
    *)                                   printf '%s' "$SMALL_ROWS" ;;
  esac
}

churn_sql() { # $1 = fixture id; prints the recipe's own writes, nothing else
  local f=$1 t half; t=$(tbl "$f")
  case "$f" in
    c01) printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, 0, %s);\n" \
           "$t" "$((BASE_ROWS + 1))" "$((BASE_ROWS * 3 / 2))" "$KEYS" ;;
    c02) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_tags('%s', 1, 'true', 0, %s);\n" "$t" "$KEYS"
         printf "SELECT /* wiki_gin_norm_churn */ proto.churn_tags('%s', 2, 'true', 0, %s);\n" "$t" "$KEYS" ;;
    c03) printf "DELETE /* wiki_gin_norm_churn */ FROM %s WHERE id %% 5 < 3;\n" "$t" ;;
    c04) printf "SELECT /* wiki_gin_norm_churn */ 'no churn: control cell';\n" ;;
    c05) printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, 0, %s);\n" \
           "$t" "$((BASE_ROWS + 1))" "$((BASE_ROWS * 2))" "$KEYS" ;;
    c06) printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, %s, %s);\n" \
           "$t" "$((BASE_ROWS + 1))" "$((BASE_ROWS * 2))" "$KEYS" "$KEYS" ;;
    c09|c11) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_tags('%s', 1, 'true', 0, %s);\n" "$t" "$KEYS" ;;
    c10) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_docs('%s', 1, 'true', %s);\n" "$t" "$KEYS" ;;
    c12) half=$((BASE_ROWS / 2))
         printf "DELETE /* wiki_gin_norm_churn */ FROM %s WHERE id <= %s;\n" "$t" "$half"
         printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, 0, %s);\n" \
           "$t" "$((BASE_ROWS + 1))" "$((BASE_ROWS + half))" "$KEYS" ;;
    p01|p02) printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, 0, %s);\n" \
           "$t" "$((PEND_ROWS + 1))" "$((PEND_ROWS + PEND_ROWS / 4))" "$KEYS" ;;
    a01) printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, 0, %s);\n" \
           "$t" "$((SMALL_ROWS + 1))" "$((SMALL_ROWS + A01_INSERTS))" "$KEYS" ;;
    a02) printf "SELECT /* wiki_gin_norm_churn */ proto.mk_tags('%s', %s, %s, 0, %s);\n" \
           "$t" "$((SMALL_ROWS + 1))" "$((SMALL_ROWS + A02_INSERTS))" "$KEYS" ;;
    m01) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_tags('%s', 1, 'true', 0, %s);\n" "$t" "$KEYS" ;;
    s01) printf "DELETE /* wiki_gin_norm_churn */ FROM %s WHERE id <= %s;\n" "$t" "$((BASE_ROWS / 2))" ;;
    i01) printf "DELETE /* wiki_gin_norm_churn */ FROM %s WHERE id %% 10 < 3;\n" "$t" ;;
    o01) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_json('%s', 1, 'true', %s);\n" "$t" "$KEYS" ;;
    o02) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_docs('%s', 1, 'true', %s);\n" "$t" "$KEYS" ;;
    o03) printf "UPDATE /* wiki_gin_norm_churn */ %s SET n = ((id + 1) %% %s)::int;\n" "$t" "$KEYS" ;;
    x01|x03) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_tags('%s', 1, 'true', 0, %s);\n" "$t" "$KEYS" ;;
    x02) printf "SELECT /* wiki_gin_norm_churn */ proto.churn_tags('%s', 1, 'true', 0, %s);\n" "$t" "$KEYS" ;;
    k01) printf "DELETE /* wiki_gin_norm_churn */ FROM %s WHERE id <= %s;\n" "$t" "$((SMALL_ROWS / 2))" ;;
    e01) printf "SELECT /* wiki_gin_norm_churn */ 'no churn: empty fixture';\n" ;;
  esac
}

# ----------------------------------------------------------- stage: fixtures
# build phase (create, load, index, settle) and baseline phase (record + capture)
stage_fixtures() {
  say "fixtures: build and baseline phases"
  [ -f "$OUT/declared_kind.txt" ] || die "declare must run before fixtures"
  q postgres "DROP /* wiki_gin_norm_fixture */ DATABASE IF EXISTS $DB" > /dev/null
  q postgres "CREATE /* wiki_gin_norm_fixture */ DATABASE $DB" > /dev/null
  proto_ddl | qin "$DB" > "$OUT/proto-ddl.log" 2>&1 || die "proto DDL failed"
  write_published_sql

  for f in $SCORED $UNSCORED; do
    local t i
    t=$(tbl "$f"); i=$(idx "$f")
    printf '  build %s (%s rows)\n' "$f" "$(rows_for "$f")"
    { build_sql "$f"; } | qin "$DB" > "$OUT/build-$f.log" 2>&1 || die "build failed for $f"
    # build-phase settle: VACUUM settles the index, ANALYZE gives the table a row estimate
    Q "$DB" "VACUUM /* wiki_gin_norm_settle_build */ (VERBOSE, ANALYZE) $t;" \
      > "$OUT/settle-build-$f.log" 2>&1
    q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('$f','baseline','$t','$i')" > /dev/null \
      || die "baseline record failed for $f"
    if [ "$f" = e01 ]; then
      # the empty fixture is expected to be refused: record the refusal text
      sed "s/public\.orders_tags_gin/$i/" "$SQLD/capture.sql" > "$SQLD/.capture_run.sql"
      qf "$DB" "$SQLD/.capture_run.sql" > "$OUT/capture-$f.log" 2>&1
      printf '    capture refused as expected: %s\n' \
        "$(grep -o 'refusing baseline for [^"]*' "$OUT/capture-$f.log" | head -1)"
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.note('$f','baseline','capture_refused',NULL,
                 \$\$$(grep -o 'refusing baseline for .*' "$OUT/capture-$f.log" | head -1)\$\$)" > /dev/null
    else
      capture_for "$i"
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.note('$f','baseline','payload',NULL,
                 substring(coalesce(obj_description('$i'::regclass,'pg_class'),'') from '@ginbase:(\{[^}]*\})'))" > /dev/null
    fi
  done
  q "$DB" "SELECT /* wiki_gin_norm_report */ fixture || ' ' || num FROM proto.meas
             WHERE phase='baseline' AND metric='index_size' ORDER BY fixture" \
    | tee "$OUT/baseline-sizes.txt"
}

# -------------------------------------------------------------- stage: churn
# recipe writes -> settle step -> maintenance step. The census is the next stage.
stage_churn() {
  say "churn: writes, then the settle step, then the maintenance step"
  for f in $SCORED $UNSCORED; do
    local t i pf
    t=$(tbl "$f"); i=$(idx "$f")
    printf '  churn %s\n' "$f"
    if [ "$f" = s01 ]; then
      # a snapshot is held across the settling VACUUM, so deleted pages stay
      # un-recyclable: the writer below keeps a repeatable-read snapshot open
      ( q "$DB" "BEGIN /* wiki_gin_norm_snapshot */ ISOLATION LEVEL REPEATABLE READ;
                 SELECT /* wiki_gin_norm_snapshot */ count(*) FROM $t;
                 SELECT /* wiki_gin_norm_snapshot */ pg_sleep(45);
                 COMMIT;" > "$OUT/snapshot-s01.log" 2>&1 ) &
      sleep 3
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.note('s01','churn','holder_backend_xmin',NULL,
                 (SELECT count(*)::text FROM pg_stat_activity
                   WHERE backend_xmin IS NOT NULL AND datname = '$DB'))" > /dev/null
    fi
    { churn_sql "$f"; printf "SELECT /* wiki_gin_norm_flush */ pg_stat_force_next_flush();\n"; } \
      | qin "$DB" > "$OUT/churn-$f.log" 2>&1 || die "churn failed for $f"
    q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('$f','churn_written','$t','$i')" > /dev/null

    if [ "$f" = i01 ]; then
      # a VACUUM that succeeded and left the index alone: the settle step did not run
      Q "$DB" "VACUUM /* wiki_gin_norm_nocleanup */ (VERBOSE, INDEX_CLEANUP OFF) $t;" \
        > "$OUT/nocleanup-i01.log" 2>&1
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('i01','churn_nocleanup','$t','$i')" > /dev/null
    fi

    if [ "$f" = a01 ] || [ "$f" = a02 ]; then
      # the auto-analyze stand-in: ANALYZE plus the flush only a worker would do.
      # Such a fixture carries stale metapage page counts by construction.
      Q "$DB" "ANALYZE /* wiki_gin_norm_standin */ $t;" > "$OUT/standin-$f.log" 2>&1
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.note('$f','churn','pending_pages_flushed',
                 (SELECT gin_clean_pending_list('$i')))" > /dev/null
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('$f','churn_maintained','$t','$i')" > /dev/null
      continue
    fi

    if [ "$f" = m01 ]; then
      # half A of the maintenance pair: the same churn, measured before the
      # settle step. Evidence only: the protocol does not score this state.
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.pagecensus('m01','churn_written','$i')" > /dev/null
    fi
    Q "$DB" "VACUUM /* wiki_gin_norm_settle */ (VERBOSE) $t;" > "$OUT/settle-$f.log" 2>&1
    q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('$f','churn_settled','$t','$i')" > /dev/null
    if [ "$f" = s01 ]; then
      # census while the holder's snapshot is still open: pages are deleted but
      # GinPageIsRecyclable refuses them, so the FSM does not have them yet
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.pagecensus('s01','churn_settled','$i')" > /dev/null
      wait
      Q "$DB" "VACUUM /* wiki_gin_norm_settle2 */ (VERBOSE) $t;" > "$OUT/settle2-s01.log" 2>&1
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('s01','churn_settled2','$t','$i')" > /dev/null
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.pagecensus('s01','churn_settled2','$i')" > /dev/null
    fi
    Q "$DB" "VACUUM /* wiki_gin_norm_maintenance */ (VERBOSE, ANALYZE) $t;" > "$OUT/maint-$f.log" 2>&1
    q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('$f','churn_maintained','$t','$i')" > /dev/null

    # cross-check 3: the fourth number of the VACUUM VERBOSE index line
    pf=$(grep -o "index \"$i\": pages: [0-9]* in total, [0-9]* newly deleted, [0-9]* currently deleted, [0-9]* reusable" \
           "$OUT/maint-$f.log" | tail -1 | sed 's/.*, \([0-9]*\) reusable/\1/')
    if [ -n "${pf:-}" ]; then
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.note('$f','churn_maintained','verbose_pages_free',$pf)" > /dev/null
    fi
    if [ "$f" = c09 ]; then
      # out-of-band rebuild after the maintenance step: the ladder must notice
      Q "$DB" "REINDEX /* wiki_gin_norm_outofband */ INDEX $i;" > "$OUT/outofband-c09.log" 2>&1
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('c09','churn_rebuilt','$t','$i')" > /dev/null
    fi
    if [ "$f" = c11 ]; then
      q "$DB" "SELECT /* wiki_gin_norm_reset_counters */ pg_stat_reset_single_table_counters('$t'::regclass)" > /dev/null
      q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('c11','churn_counters_reset','$t','$i')" > /dev/null
    fi
  done
}

# --------------------------------------------- stage: simulated analyze census
# Recomputes relation_needs_vacanalyze's analyze verdict per table from the
# effective reloption-or-GUC values, and analyzes the tables it names.
stage_analyze_census() {
  say "analyze census: the launcher's analyze verdict, recomputed per table"
  # two tables the census must decide: one past its threshold, one exactly on it
  for t in tc_past tc_exact tc_off; do
    q "$DB" "DROP /* wiki_gin_norm_census */ TABLE IF EXISTS $t" > /dev/null
  done
  q "$DB" "CREATE /* wiki_gin_norm_census */ TABLE tc_past (id bigint, tags int[])" > /dev/null
  q "$DB" "CREATE /* wiki_gin_norm_census */ TABLE tc_exact (id bigint, tags int[])" > /dev/null
  q "$DB" "CREATE /* wiki_gin_norm_census */ TABLE tc_off (id bigint, tags int[])
             WITH (autovacuum_enabled = false)" > /dev/null
  # build each to a known reltuples. The flush comes BEFORE the ANALYZE: a reset
  # counter that is then handed the build's own pending counts reads as churn
  # that never happened, which is the protocol's first publication point.
  qin "$DB" <<SQL > "$OUT/census-build.log" 2>&1
SELECT /* wiki_gin_norm_census */ proto.mk_tags('tc_past',  1, 10000, 0, $KEYS);
SELECT /* wiki_gin_norm_census */ proto.mk_tags('tc_exact', 1, 10000, 0, $KEYS);
SELECT /* wiki_gin_norm_census */ proto.mk_tags('tc_off',   1, 10000, 0, $KEYS);
SELECT /* wiki_gin_norm_census */ pg_stat_force_next_flush();
ANALYZE /* wiki_gin_norm_census */ tc_past, tc_exact, tc_off;
SELECT /* wiki_gin_norm_census */ pg_stat_force_next_flush();
SQL
  # thresholds at the shipped defaults are 50 + 0.1 * reltuples = 1050 for 10000 rows;
  # tc_past crosses it, tc_exact lands exactly on it, tc_off is short-circuited
  qin "$DB" <<SQL > "$OUT/census-push.log" 2>&1
UPDATE /* wiki_gin_norm_census */ tc_past  SET tags = tags WHERE id <= 2000;
UPDATE /* wiki_gin_norm_census */ tc_exact SET tags = tags WHERE id <= 1050;
UPDATE /* wiki_gin_norm_census */ tc_off   SET tags = tags WHERE id <= 2000;
SELECT /* wiki_gin_norm_census */ pg_stat_force_next_flush();
SQL
  q "$DB" "SELECT /* wiki_gin_norm_census */ pg_stat_clear_snapshot()" > /dev/null
  qat "$DB" /dev/stdin > "$OUT/census-verdicts.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_census */
       c.relname
       || ' reltuples=' || greatest(c.reltuples, 0)::bigint
       || ' mod=' || coalesce(s.n_mod_since_analyze, 0)
       || ' thresh=' || round((CASE WHEN o.analyze_threshold >= 0 THEN o.analyze_threshold
                                    ELSE current_setting('autovacuum_analyze_threshold')::numeric END)
                              + (CASE WHEN o.analyze_scale_factor >= 0 THEN o.analyze_scale_factor
                                      ELSE current_setting('autovacuum_analyze_scale_factor')::numeric END)
                                * greatest(c.reltuples, 0)::numeric, 2)
       || ' av_enabled=' || o.av_enabled
       || ' doanalyze=' || (o.av_enabled AND coalesce(s.n_mod_since_analyze, 0) >
              (CASE WHEN o.analyze_threshold >= 0 THEN o.analyze_threshold
                    ELSE current_setting('autovacuum_analyze_threshold')::numeric END)
            + (CASE WHEN o.analyze_scale_factor >= 0 THEN o.analyze_scale_factor
                    ELSE current_setting('autovacuum_analyze_scale_factor')::numeric END)
              * greatest(c.reltuples, 0)::numeric)
  FROM pg_class c
  LEFT JOIN pg_stat_all_tables s ON s.relid = c.oid
  CROSS JOIN LATERAL (
       SELECT coalesce((SELECT (regexp_match(opt, '^autovacuum_analyze_threshold=(.*)$'))[1]::numeric
                          FROM unnest(coalesce(c.reloptions, '{}')) opt
                         WHERE opt LIKE 'autovacuum_analyze_threshold=%'), -1)     AS analyze_threshold,
              coalesce((SELECT (regexp_match(opt, '^autovacuum_analyze_scale_factor=(.*)$'))[1]::numeric
                          FROM unnest(coalesce(c.reloptions, '{}')) opt
                         WHERE opt LIKE 'autovacuum_analyze_scale_factor=%'), -1)  AS analyze_scale_factor,
              NOT coalesce((SELECT (regexp_match(opt, '^autovacuum_enabled=(.*)$'))[1] = 'false'
                          FROM unnest(coalesce(c.reloptions, '{}')) opt
                         WHERE opt LIKE 'autovacuum_enabled=%'), false)            AS av_enabled
  ) o
 WHERE c.relkind = 'r' AND c.relnamespace = 'public'::regnamespace
 ORDER BY c.relname;
SQL
  # analyze exactly the tables the census named
  qat "$DB" /dev/stdin > "$OUT/census-analyzed.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_census */ 'ANALYZE ' || quote_ident(c.relname) || ';'
  FROM pg_class c
  LEFT JOIN pg_stat_all_tables s ON s.relid = c.oid
  CROSS JOIN LATERAL (
       SELECT coalesce((SELECT (regexp_match(opt, '^autovacuum_analyze_threshold=(.*)$'))[1]::numeric
                          FROM unnest(coalesce(c.reloptions, '{}')) opt
                         WHERE opt LIKE 'autovacuum_analyze_threshold=%'), -1)     AS analyze_threshold,
              coalesce((SELECT (regexp_match(opt, '^autovacuum_analyze_scale_factor=(.*)$'))[1]::numeric
                          FROM unnest(coalesce(c.reloptions, '{}')) opt
                         WHERE opt LIKE 'autovacuum_analyze_scale_factor=%'), -1)  AS analyze_scale_factor,
              NOT coalesce((SELECT (regexp_match(opt, '^autovacuum_enabled=(.*)$'))[1] = 'false'
                          FROM unnest(coalesce(c.reloptions, '{}')) opt
                         WHERE opt LIKE 'autovacuum_enabled=%'), false)            AS av_enabled
  ) o
 WHERE c.relkind = 'r' AND c.relnamespace = 'public'::regnamespace
   AND o.av_enabled
   AND coalesce(s.n_mod_since_analyze, 0) >
         (CASE WHEN o.analyze_threshold >= 0 THEN o.analyze_threshold
               ELSE current_setting('autovacuum_analyze_threshold')::numeric END)
       + (CASE WHEN o.analyze_scale_factor >= 0 THEN o.analyze_scale_factor
               ELSE current_setting('autovacuum_analyze_scale_factor')::numeric END)
         * greatest(c.reltuples, 0)::numeric
 ORDER BY c.relname;
SQL
  printf 'census named %s table(s) for ANALYZE\n' "$(grep -c 'ANALYZE' "$OUT/census-analyzed.txt")"
  grep 'ANALYZE' "$OUT/census-analyzed.txt" | qin "$DB" > "$OUT/census-run.log" 2>&1
  q "$DB" "SELECT /* wiki_gin_norm_census */ pg_stat_clear_snapshot()" > /dev/null
  grep -E 'tc_past|tc_exact|tc_off' "$OUT/census-verdicts.txt"
}

# ------------------------------------------------- stage: the four cross-checks
stage_crosscheck() {
  say "cross-checks: page census under SHARE ROW EXCLUSIVE, FSM, bracket, metapage"
  for f in $SCORED; do
    local t i ph
    t=$(tbl "$f"); i=$(idx "$f")
    # c09 was rebuilt out of band after its maintenance step, so the state this
    # census reads is that rebuild, not the churned file
    ph=churn_maintained
    [ "$f" = c09 ] && ph=churn_rebuilt
    printf '  census %s (%s)\n' "$f" "$ph"
    qin "$DB" > "$OUT/crosscheck-$f.log" 2>&1 <<SQL
BEGIN;
SET LOCAL statement_timeout = '600s';
SET LOCAL lock_timeout = '15s';
LOCK /* wiki_gin_norm_crosscheck */ TABLE $t IN SHARE ROW EXCLUSIVE MODE;
SELECT /* wiki_gin_norm_crosscheck */ proto.pagecensus('$f','$ph','$i');
COMMIT;
SQL
  done
  q "$DB" "SELECT /* wiki_gin_norm_report */ fixture
             || ' scanned=' || max(num) FILTER (WHERE metric='census_scanned')
             || ' entry=' || max(num) FILTER (WHERE metric='census_entry')
             || ' data=' || (max(num) FILTER (WHERE metric='census_data_leaf')
                           + max(num) FILTER (WHERE metric='census_data_inner'))
             || ' list=' || max(num) FILTER (WHERE metric='census_list')
             || ' del=' || max(num) FILTER (WHERE metric='census_deleted')
             || ' new=' || max(num) FILTER (WHERE metric='census_new')
             || ' fsm=' || max(num) FILTER (WHERE metric='fsm_free_pages')
             || ' fsm_avail=' || max(num) FILTER (WHERE metric='fsm_max_avail')
             FROM proto.meas WHERE phase IN ('churn_maintained','churn_rebuilt')
             GROUP BY fixture HAVING count(*) FILTER (WHERE metric='census_scanned') > 0
             ORDER BY fixture" \
    | tee "$OUT/census-summary.txt"
}

# -------------------------------------------------------------- stage: decide
# One pass of the published statement, verbatim, inside one transaction holding
# SHARE ROW EXCLUSIVE on every fixture table. Nothing else runs in this phase.
stage_decide() {
  say "decide: the published statement under the measurement lock"
  local locks=""
  for f in $SCORED $UNSCORED; do locks="$locks$(tbl "$f"), "; done
  locks="${locks%, }"
  { printf "SET /* wiki_gin_norm_decide */ lock_timeout = '30s';\n"
    printf "BEGIN /* wiki_gin_norm_decide */;\n"
    printf "LOCK /* wiki_gin_norm_decide */ TABLE %s IN SHARE ROW EXCLUSIVE MODE;\n" "$locks"
    cat "$SQLD/evaluate.sql"
  } | "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$DB" \
      > "$OUT/decide.txt" 2>&1 || die "decide pass failed"
  # the same pass again, in expanded form, recorded per fixture for scoring
  { printf "SET /* wiki_gin_norm_decide */ lock_timeout = '30s';\n"
    printf "BEGIN /* wiki_gin_norm_decide */;\n"
    printf "LOCK /* wiki_gin_norm_decide */ TABLE %s IN SHARE ROW EXCLUSIVE MODE;\n" "$locks"
    printf "CREATE /* wiki_gin_norm_decide */ TABLE proto.decided AS\n"
    sed -e '1,3d' -e 's/^COMMIT;$//' "$SQLD/evaluate.sql"
    printf "COMMIT;\n"
  } | "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d "$DB" \
      > "$OUT/decide-store.log" 2>&1 || die "decide store failed"
  cat "$OUT/decide.txt"
}

# -------------------------------------------------------------- stage: oracle
stage_oracle() {
  say "oracle: measured REINDEX INDEX at maintenance_work_mem = $MWM"
  for f in $SCORED; do
    local i; i=$(idx "$f")
    printf '  reindex %s\n' "$f"
    qin "$DB" > "$OUT/oracle-$f.log" 2>&1 <<SQL
SET /* wiki_gin_norm_oracle */ maintenance_work_mem = '$MWM';
SELECT /* wiki_gin_norm_oracle */ proto.note('$f','oracle','size_before',
         pg_relation_size('$i'::regclass,'main'));
REINDEX /* wiki_gin_norm_oracle */ INDEX $i;
SELECT /* wiki_gin_norm_oracle */ proto.note('$f','oracle','size_after',
         pg_relation_size('$i'::regclass,'main'));
SELECT /* wiki_gin_norm_oracle */ proto.note('$f','oracle','mwm',NULL,
         current_setting('maintenance_work_mem'));
SQL
    q "$DB" "SELECT /* wiki_gin_norm_record */ proto.record('$f','oracle_after','$(tbl "$f")','$i')" > /dev/null
  done
  q "$DB" "SELECT /* wiki_gin_norm_report */ fixture || ' ' ||
             max(num) FILTER (WHERE metric='size_before') || ' -> ' ||
             max(num) FILTER (WHERE metric='size_after')  || ' = ' ||
             round(100.0 * (1 - max(num) FILTER (WHERE metric='size_after')
                              / max(num) FILTER (WHERE metric='size_before')), 2) || '%'
             FROM proto.meas WHERE phase='oracle' AND metric IN ('size_before','size_after')
             GROUP BY fixture ORDER BY fixture" | tee "$OUT/oracle-summary.txt"
}

# --------------------------------------------------------------- stage: score
stage_score() {
  say "score: declared kinds against the oracle, and the decision against it"
  # the declarations the scoring below assumes, checked against what was filed
  q "$PDB" "SELECT /* wiki_gin_norm_score */ column_name || '=' || declared_kind
              FROM declared_kind ORDER BY 1" > "$OUT/.filed_kinds.txt"
  cat > "$OUT/.assumed_kinds.txt" <<'KINDS'
base_heap_reltuples=level
base_size=level
churn_ratio=level
cur_heap_reltuples=level
cur_size=level
est_reclaimable=upper bound
heap_tuple_ratio=level
index_size_ratio=level
normalized_index_growth=level
stats_lag=level
KINDS
  if ! diff -u "$OUT/.filed_kinds.txt" "$OUT/.assumed_kinds.txt" > "$OUT/kinds-diff.txt" 2>&1; then
    cat "$OUT/kinds-diff.txt"; die "the scoring assumes kinds that were not the filed ones"
  fi
  printf 'declared kinds: the scoring matches the declaration filed at %s\n' \
    "$(cat "$OUT/declared_at.txt")"

  qin "$DB" > "$OUT/score-build.log" 2>&1 <<'SQL'
DROP /* wiki_gin_norm_score */ TABLE IF EXISTS proto.scored;
CREATE /* wiki_gin_norm_score */ TABLE proto.scored AS
WITH p AS (
  SELECT fixture,
    max(num) FILTER (WHERE phase='baseline'         AND metric='index_size')    AS base_size_rec,
    max(num) FILTER (WHERE phase='churn_written'    AND metric='index_size')    AS written_size,
    max(num) FILTER (WHERE phase='churn_settled'    AND metric='index_size')    AS settled_size,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='index_size')    AS maint_size,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='tab_reltuples') AS cur_heap,
    max(num) FILTER (WHERE phase='oracle'           AND metric='size_before')   AS ora_before,
    max(num) FILTER (WHERE phase='oracle'           AND metric='size_after')    AS ora_after,
    max(txt) FILTER (WHERE phase='baseline'         AND metric='payload')       AS payload
  FROM proto.meas GROUP BY fixture
), b AS (
  SELECT p.*,
         (payload::jsonb->>'bis')::numeric AS bis,
         (payload::jsonb->>'bhr')::numeric AS bhr
    FROM p WHERE payload IS NOT NULL
), r AS (
  SELECT b.*,
         (ora_before / nullif(bis,0)) / nullif(cur_heap / nullif(bhr,0), 0) AS norm_exact,
         ora_before - ora_after                                            AS returned_bytes,
         100.0 * (1 - ora_after / nullif(ora_before,0))                    AS truth_pct
    FROM b
)
SELECT r.fixture,
       d.verdict,
       r.bis::bigint                                     AS base_size_bytes,
       r.ora_before::bigint                              AS decide_size_bytes,
       r.ora_after::bigint                               AS rebuilt_size_bytes,
       r.returned_bytes::bigint                          AS returned_bytes,
       round(r.truth_pct, 2)                             AS truth_pct,
       round(r.norm_exact, 4)                            AS norm_exact,
       d.normalized_index_growth                         AS norm_published,
       d.index_size_ratio, d.heap_tuple_ratio, d.churn_ratio, d.stats_lag,
       CASE WHEN r.norm_exact > 1
            THEN (r.ora_before * (1 - 1/r.norm_exact))::bigint END AS est_bytes,
       CASE WHEN r.norm_exact > 1
            THEN round(100.0 * (1 - 1/r.norm_exact), 2) END        AS pred_pct,
       d.est_reclaimable                                 AS est_published,
       CASE WHEN r.norm_exact > 1
            THEN pg_size_pretty((r.ora_before * (1 - 1/r.norm_exact))::bigint) END
                                                         AS est_recomputed_pretty,
       (d.verdict LIKE 'candidate%' OR d.verdict LIKE 'strong candidate%') AS flagged,
       (d.verdict IN ('no baseline: capture one',
                      'invalid index: rebuild for validity, not for size',
                      'rebuilt since baseline: re-capture',
                      'counters reset: re-capture',
                      'no table statistics: ANALYZE first',
                      'insufficient churn: not evaluated',
                      'no ANALYZE since baseline: ANALYZE first'))         AS refusal
  FROM r
  JOIN proto.decided d ON d.index_name::text = 'f_' || r.fixture || '_gin';

DROP /* wiki_gin_norm_score */ TABLE IF EXISTS proto.report;
CREATE /* wiki_gin_norm_score */ TABLE proto.report AS
SELECT s.*,
       CASE WHEN est_bytes IS NULL             THEN 'not published'
            WHEN est_bytes >= returned_bytes   THEN 'HELD'
            ELSE 'VIOLATED' END                          AS upper_bound_verdict,
       CASE WHEN refusal AND truth_pct >= 33.33           THEN 'REFUSED (a real candidate)'
            WHEN refusal                                  THEN 'REFUSED (nothing to reclaim)'
            WHEN flagged AND truth_pct >= 33.33           THEN 'PASS'
            WHEN flagged                                  THEN 'FALSE POSITIVE'
            WHEN NOT flagged AND truth_pct >= 33.33       THEN 'FALSE NEGATIVE'
            ELSE 'PASS' END                              AS decision_score,
       CASE WHEN est_published IS DISTINCT FROM est_recomputed_pretty
            THEN 'recomputation differs' ELSE 'recomputation matches' END AS est_check,
       round(pred_pct - truth_pct, 2)                    AS pred_error
  FROM proto.scored s;
SQL
  qat "$DB" /dev/stdin > "$OUT/score-table.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_report */
       rpad(fixture,4) || ' | ' || rpad(verdict,52) || ' | ' ||
       lpad(decide_size_bytes::text,11) || ' | ' || lpad(rebuilt_size_bytes::text,11) || ' | ' ||
       lpad(truth_pct::text,6) || ' | ' || lpad(norm_exact::text,8) || ' | ' ||
       lpad(coalesce(pred_pct::text,'-'),7) || ' | ' || lpad(coalesce(pred_error::text,'-'),8) || ' | ' ||
       rpad(upper_bound_verdict,13) || ' | ' || rpad(decision_score,26) || ' | ' || est_check
  FROM proto.report ORDER BY fixture;
SQL
  cat "$OUT/score-table.txt"
  qat "$DB" /dev/stdin > "$OUT/score-summary.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_report */ 'scored fixtures: ' || count(*) FROM proto.report
UNION ALL SELECT 'upper bound HELD: ' || count(*) FROM proto.report WHERE upper_bound_verdict='HELD'
UNION ALL SELECT 'upper bound VIOLATED: ' || count(*) FROM proto.report WHERE upper_bound_verdict='VIOLATED'
UNION ALL SELECT 'upper bound not published: ' || count(*) FROM proto.report WHERE upper_bound_verdict='not published'
UNION ALL SELECT 'decision PASS: ' || count(*) FROM proto.report WHERE decision_score='PASS'
UNION ALL SELECT 'decision FALSE POSITIVE: ' || count(*) FROM proto.report WHERE decision_score='FALSE POSITIVE'
UNION ALL SELECT 'decision FALSE NEGATIVE: ' || count(*) FROM proto.report WHERE decision_score='FALSE NEGATIVE'
UNION ALL SELECT 'refused a real candidate: ' || count(*) FROM proto.report WHERE decision_score='REFUSED (a real candidate)'
UNION ALL SELECT 'refused nothing to reclaim: ' || count(*) FROM proto.report WHERE decision_score='REFUSED (nothing to reclaim)'
UNION ALL SELECT 'est recomputation matches the published text: ' || count(*) FROM proto.report WHERE est_check='recomputation matches'
UNION ALL SELECT 'prediction within 0.05 points: ' || count(*) FROM proto.report WHERE abs(pred_error) <= 0.05
UNION ALL SELECT 'prediction within 3.5 points: ' || count(*) FROM proto.report WHERE abs(pred_error) <= 3.5
UNION ALL SELECT 'worst over-prediction: ' || coalesce(max(pred_error)::text,'-') FROM proto.report
UNION ALL SELECT 'worst under-prediction: ' || coalesce(min(pred_error)::text,'-') FROM proto.report;
SQL
  cat "$OUT/score-summary.txt"

  say "score: the six declared invariants"
  qat "$DB" /dev/stdin > "$OUT/invariants.txt" 2>&1 <<'SQL'
WITH p AS (
  SELECT fixture,
    max(num) FILTER (WHERE phase='baseline' AND metric='index_size')             AS base_size,
    max(num) FILTER (WHERE phase='baseline' AND metric='meta_total')             AS b_total,
    max(num) FILTER (WHERE phase='baseline' AND metric='meta_entry')             AS b_entry,
    max(num) FILTER (WHERE phase='baseline' AND metric='meta_data')              AS b_data,
    max(num) FILTER (WHERE phase='baseline' AND metric='meta_pending')           AS b_pending,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='index_size')     AS m_size,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='meta_total')     AS m_total,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='meta_entry')     AS m_entry,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='meta_data')      AS m_data,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='meta_pending')   AS m_pending,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='census_entry')   AS c_entry,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='census_data_leaf')  AS c_dleaf,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='census_data_inner') AS c_dinner,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='census_list')    AS c_list,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='census_recyclable') AS c_recyc,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='fsm_free_pages') AS fsm,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='bracket_blocks_before') AS br_before,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='bracket_blocks_after')  AS br_after,
    max(num) FILTER (WHERE phase='churn_maintained' AND metric='verbose_pages_free')    AS verb
  FROM proto.meas GROUP BY fixture
)
SELECT 'I1 index_size_ratio >= 1 (un-rebuilt fixtures): '
       || count(*) FILTER (WHERE m_size >= base_size) || ' of ' || count(*)
  FROM p WHERE base_size IS NOT NULL AND m_size IS NOT NULL AND fixture <> 'c09'
UNION ALL
SELECT 'I2 fresh build total = entry + data + pending + 1: '
       || count(*) FILTER (WHERE b_total = b_entry + b_data + b_pending + 1) || ' of ' || count(*)
  FROM p WHERE b_total IS NOT NULL
UNION ALL
SELECT 'I3 FSM free <= census new+deleted: '
       || count(*) FILTER (WHERE fsm <= c_recyc) || ' of ' || count(*)
  FROM p WHERE fsm IS NOT NULL AND c_recyc IS NOT NULL
UNION ALL
SELECT 'I4 size bracket held: '
       || count(*) FILTER (WHERE br_before = br_after) || ' of ' || count(*)
  FROM p WHERE br_before IS NOT NULL
UNION ALL
SELECT 'I5 VACUUM VERBOSE reusable = census new+deleted: '
       || count(*) FILTER (WHERE verb = c_recyc) || ' of ' || count(*)
  FROM p WHERE verb IS NOT NULL AND c_recyc IS NOT NULL
UNION ALL
SELECT 'I6 meta_total = 1 + census entry + data + list + new/deleted: '
       || count(*) FILTER (WHERE m_total = 1 + c_entry + c_dleaf + c_dinner + c_list + c_recyc)
       || ' of ' || count(*)
  FROM p WHERE m_total IS NOT NULL AND c_entry IS NOT NULL
UNION ALL
SELECT 'metapage entry+data vs census (exact rows): ' || string_agg(fixture, ' ')
  FROM p WHERE m_total IS NOT NULL AND c_entry IS NOT NULL
    AND m_entry = c_entry AND m_data = c_dleaf + c_dinner;
SQL
  cat "$OUT/invariants.txt"

  say "score: the maintenance pair, and every phase size per fixture"
  qat "$DB" /dev/stdin > "$OUT/phase-sizes.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_report */ rpad(fixture,4) || ' | ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='baseline' AND metric='index_size')::text,'-'),11) || ' | ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='churn_written' AND metric='index_size')::text,'-'),11) || ' | ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='churn_settled' AND metric='index_size')::text,'-'),11) || ' | ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='churn_maintained' AND metric='index_size')::text,'-'),11) || ' | ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='oracle_after' AND metric='index_size')::text,'-'),11) || ' | meta ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='churn_written' AND metric='meta_pending')::text,'-'),5) || ' -> ' ||
       lpad(coalesce(max(num) FILTER (WHERE phase='churn_maintained' AND metric='meta_pending')::text,'-'),5)
  FROM proto.meas GROUP BY fixture ORDER BY fixture;
SQL
  cat "$OUT/phase-sizes.txt"
}

# -------------------------------------------------------------- stage: probes
# The mechanism probes behind the page's two failure sections, its reltuples
# table and its statistics-counter warning. Own database, own disposable tables.
stage_probes() {
  say "probes: reltuples writers, build linearity, maintenance_work_mem, pgstat timing"
  q postgres "DROP /* wiki_gin_norm_probe */ DATABASE IF EXISTS probes" > /dev/null
  q postgres "CREATE /* wiki_gin_norm_probe */ DATABASE probes" > /dev/null
  qin probes <<'SQL' > "$OUT/probe-ddl.log" 2>&1
CREATE /* wiki_gin_norm_probe */ EXTENSION pageinspect;
CREATE /* wiki_gin_norm_probe */ SCHEMA proto;
CREATE TABLE proto.meas (fixture text, phase text, metric text, num numeric, txt text,
                         at timestamptz default clock_timestamp());
CREATE FUNCTION proto.note(p_fix text, p_phase text, p_metric text,
                           p_num numeric DEFAULT NULL, p_txt text DEFAULT NULL)
RETURNS void LANGUAGE sql AS $fn$
  INSERT INTO proto.meas(fixture,phase,metric,num,txt) VALUES (p_fix,p_phase,p_metric,p_num,p_txt);
$fn$;
CREATE FUNCTION proto.mk_tags(p_tab text, p_lo bigint, p_hi bigint, p_keybase int, p_keys int)
RETURNS void LANGUAGE plpgsql AS $fn$
BEGIN
  EXECUTE format($q$INSERT INTO %I (id, tags) SELECT i, ARRAY[
      $1 + ((i *     7919) %% $2)::int, $1 + ((i *   104729) %% $2)::int,
      $1 + ((i *  1299709) %% $2)::int, $1 + ((i * 15485863) %% $2)::int,
      $1 + ((i * 32452843) %% $2)::int] FROM generate_series($3, $4) i$q$, p_tab)
    USING p_keybase, p_keys, p_lo, p_hi;
END $fn$;
CREATE FUNCTION proto.rt(p_stage text, p_tab text, p_idx text) RETURNS void
LANGUAGE plpgsql AS $fn$
BEGIN
  INSERT INTO proto.meas(fixture,phase,metric,num)
  SELECT 'P1', p_stage, 'tab_reltuples', reltuples FROM pg_class WHERE oid = p_tab::regclass;
  INSERT INTO proto.meas(fixture,phase,metric,num)
  SELECT 'P1', p_stage, 'idx_reltuples', reltuples FROM pg_class WHERE oid = p_idx::regclass;
  INSERT INTO proto.meas(fixture,phase,metric,num)
  SELECT 'P1', p_stage, 'index_size', pg_relation_size(p_idx::regclass,'main');
END $fn$;
SQL

  say "P1: the four writers of an index's own reltuples, and the fifth value"
  qin probes > "$OUT/probe-p1.log" 2>&1 <<SQL
CREATE /* wiki_gin_norm_probe */ TABLE p1 (id bigint, tags int[]);
SELECT /* wiki_gin_norm_probe */ proto.mk_tags('p1', 1, 200000, 0, 2000);
CREATE /* wiki_gin_norm_probe */ INDEX p1_gin ON p1 USING gin (tags);
SELECT /* wiki_gin_norm_probe */ proto.rt('after CREATE INDEX','p1','p1_gin');
ANALYZE /* wiki_gin_norm_probe */ p1;
SELECT /* wiki_gin_norm_probe */ proto.rt('after ANALYZE','p1','p1_gin');
DELETE /* wiki_gin_norm_probe */ FROM p1 WHERE id % 2 = 0;
VACUUM /* wiki_gin_norm_probe */ p1;
SELECT /* wiki_gin_norm_probe */ proto.rt('after DELETE 50% + VACUUM','p1','p1_gin');
REINDEX /* wiki_gin_norm_probe */ INDEX p1_gin;
SELECT /* wiki_gin_norm_probe */ proto.rt('after REINDEX','p1','p1_gin');
VACUUM /* wiki_gin_norm_probe */ p1;
SELECT /* wiki_gin_norm_probe */ proto.rt('after a second plain VACUUM','p1','p1_gin');
VACUUM /* wiki_gin_norm_probe */ (DISABLE_PAGE_SKIPPING) p1;
SELECT /* wiki_gin_norm_probe */ proto.rt('after VACUUM (DISABLE_PAGE_SKIPPING)','p1','p1_gin');
SQL
  qat probes /dev/stdin > "$OUT/probe-p1.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_report */ rpad(phase,40) || ' | table ' ||
       lpad(max(num) FILTER (WHERE metric='tab_reltuples')::bigint::text,8) || ' | index ' ||
       lpad(max(num) FILTER (WHERE metric='idx_reltuples')::bigint::text,8)
  FROM proto.meas WHERE fixture='P1' GROUP BY phase ORDER BY min(at);
SQL
  cat "$OUT/probe-p1.txt"

  say "P2/P3: fresh-build size against heap tuples, and against maintenance_work_mem"
  q probes "CREATE /* wiki_gin_norm_probe */ TABLE lin (id bigint, tags int[])" > /dev/null
  local prev=0
  for n in 250000 500000 1000000 2000000 4000000 8000000; do
    printf '  linearity point %s\n' "$n"
    qin probes > "$OUT/probe-lin-$n.log" 2>&1 <<SQL
SELECT /* wiki_gin_norm_probe */ proto.mk_tags('lin', $((prev + 1)), $n, 0, $KEYS);
VACUUM /* wiki_gin_norm_probe */ (ANALYZE) lin;
SET /* wiki_gin_norm_probe */ maintenance_work_mem = '$MWM';
CREATE /* wiki_gin_norm_probe */ INDEX lin_gin ON lin USING gin (tags);
SELECT /* wiki_gin_norm_probe */ proto.note('P2','$n','index_size', pg_relation_size('lin_gin','main'));
INSERT /* wiki_gin_norm_probe */ INTO proto.meas(fixture,phase,metric,num)
SELECT 'P2','$n',k,v FROM (SELECT * FROM gin_metapage_info(get_raw_page('lin_gin',0))) m,
  LATERAL (VALUES ('meta_total',m.n_total_pages),('meta_entry',m.n_entry_pages),
                  ('meta_data',m.n_data_pages),('meta_pending',m.n_pending_pages),
                  ('meta_entries',m.n_entries)) AS t(k,v);
DROP /* wiki_gin_norm_probe */ INDEX lin_gin;
SQL
    if [ "$n" = 2000000 ]; then
      for mwm in 64MB 256MB 1GB; do
        printf '  budget %s at %s rows\n' "$mwm" "$n"
        qin probes > "$OUT/probe-mwm-$mwm.log" 2>&1 <<SQL
SET /* wiki_gin_norm_probe */ maintenance_work_mem = '$mwm';
CREATE /* wiki_gin_norm_probe */ INDEX lin_gin ON lin USING gin (tags);
SELECT /* wiki_gin_norm_probe */ proto.note('P3','$mwm','index_size', pg_relation_size('lin_gin','main'));
INSERT /* wiki_gin_norm_probe */ INTO proto.meas(fixture,phase,metric,num)
SELECT 'P3','$mwm',k,v FROM (SELECT * FROM gin_metapage_info(get_raw_page('lin_gin',0))) m,
  LATERAL (VALUES ('meta_total',m.n_total_pages),('meta_entry',m.n_entry_pages),
                  ('meta_data',m.n_data_pages),('meta_pending',m.n_pending_pages),
                  ('meta_entries',m.n_entries)) AS t(k,v);
DROP /* wiki_gin_norm_probe */ INDEX lin_gin;
SQL
      done
    fi
    prev=$n
  done
  qat probes /dev/stdin > "$OUT/probe-p2.txt" 2>&1 <<'SQL'
SELECT /* wiki_gin_norm_report */ rpad(fixture,3) || ' | ' || lpad(phase,9) || ' | bytes ' ||
       lpad(max(num) FILTER (WHERE metric='index_size')::bigint::text,10) || ' | pages ' ||
       lpad((max(num) FILTER (WHERE metric='index_size')/8192)::bigint::text,7) || ' | entry ' ||
       lpad(max(num) FILTER (WHERE metric='meta_entry')::bigint::text,6) || ' | data ' ||
       lpad(max(num) FILTER (WHERE metric='meta_data')::bigint::text,6) || ' | pending ' ||
       lpad(max(num) FILTER (WHERE metric='meta_pending')::bigint::text,3) || ' | entries ' ||
       lpad(max(num) FILTER (WHERE metric='meta_entries')::bigint::text,7) || ' | identity ' ||
       CASE WHEN max(num) FILTER (WHERE metric='meta_total')
               = max(num) FILTER (WHERE metric='meta_entry')
               + max(num) FILTER (WHERE metric='meta_data')
               + max(num) FILTER (WHERE metric='meta_pending') + 1
            THEN 'exact' ELSE 'OFF BY ' ||
              (max(num) FILTER (WHERE metric='meta_total')
                 - max(num) FILTER (WHERE metric='meta_entry')
                 - max(num) FILTER (WHERE metric='meta_data')
                 - max(num) FILTER (WHERE metric='meta_pending') - 1)::text END
  FROM proto.meas WHERE fixture IN ('P2','P3')
 GROUP BY fixture, phase ORDER BY fixture, (CASE WHEN phase ~ '^[0-9]+$' THEN phase::bigint ELSE 0 END), phase;
SQL
  cat "$OUT/probe-p2.txt"

  say "P4: the statistics counter the guard must not use"
  qin probes > "$OUT/probe-p4.log" 2>&1 <<SQL
CREATE /* wiki_gin_norm_probe */ TABLE p4 (id bigint, tags int[]);
SELECT /* wiki_gin_norm_probe */ proto.mk_tags('p4', 1, 150000, 0, 2000);
CREATE /* wiki_gin_norm_probe */ INDEX p4_gin ON p4 USING gin (tags);
VACUUM /* wiki_gin_norm_probe */ (ANALYZE) p4;
SQL
  : > "$OUT/probe-p4.txt"
  for attempt in 1 2 3 4; do
    qin probes >> "$OUT/probe-p4.log" 2>&1 <<SQL
DELETE /* wiki_gin_norm_probe */ FROM p4 WHERE id % 10 = $attempt;
VACUUM /* wiki_gin_norm_probe */ (ANALYZE) p4;
SELECT /* wiki_gin_norm_probe */ pg_stat_force_next_flush();
SQL
    q probes "SELECT /* wiki_gin_norm_report */ 'attempt $attempt | live ' || n_live_tup ||
                ' | dead ' || n_dead_tup || ' | mod ' || n_mod_since_analyze ||
                ' | reltuples ' || (SELECT reltuples::bigint FROM pg_class WHERE oid='p4'::regclass)
                FROM pg_stat_all_tables WHERE relid = 'p4'::regclass" >> "$OUT/probe-p4.txt"
  done
  qin probes >> "$OUT/probe-p4.log" 2>&1 <<SQL
DELETE /* wiki_gin_norm_probe */ FROM p4 WHERE id % 10 = 5;
SELECT /* wiki_gin_norm_probe */ pg_sleep(2);
VACUUM /* wiki_gin_norm_probe */ (ANALYZE) p4;
SELECT /* wiki_gin_norm_probe */ pg_stat_force_next_flush();
SQL
  q probes "SELECT /* wiki_gin_norm_report */ 'attempt 5, 2 s pause | live ' || n_live_tup ||
              ' | dead ' || n_dead_tup || ' | mod ' || n_mod_since_analyze ||
              ' | reltuples ' || (SELECT reltuples::bigint FROM pg_class WHERE oid='p4'::regclass)
              FROM pg_stat_all_tables WHERE relid = 'p4'::regclass" >> "$OUT/probe-p4.txt"
  cat "$OUT/probe-p4.txt"

  say "P5: the same race at baseline capture, with and without a forced flush"
  write_published_sql
  sed "s/public\.orders_tags_gin/orders_tags_gin/" "$SQLD/capture.sql" > "$SQLD/.p5.sql"
  # the load, the build and the capture in ONE transaction, so the loader's own
  # counts cannot have been published when the baseline is written
  { cat <<'SQL'
BEGIN /* wiki_gin_norm_probe */;
CREATE /* wiki_gin_norm_probe */ TABLE orders_tags (id bigint, tags int[]);
SELECT /* wiki_gin_norm_probe */ proto.mk_tags('orders_tags', 1, 80000, 0, 2000);
CREATE /* wiki_gin_norm_probe */ INDEX orders_tags_gin ON orders_tags USING gin (tags);
SELECT /* wiki_gin_norm_probe */ 'reltuples right after CREATE INDEX: ' ||
       (SELECT reltuples::bigint FROM pg_class WHERE oid = 'orders_tags'::regclass);
SQL
    cat "$SQLD/.p5.sql"
  } | qin probes > "$OUT/probe-p5.log" 2>&1
  grep -o 'reltuples right after CREATE INDEX: [0-9-]*' "$OUT/probe-p5.log" | head -1 \
    | tee "$OUT/probe-p5.txt"
  q probes "SELECT /* wiki_gin_norm_report */ 'capture in the loading transaction: bti=' ||
              (substring(obj_description('orders_tags_gin'::regclass,'pg_class') from '@ginbase:(\{[^}]*\})')::jsonb->>'bti')
              || ' n_tup_ins=' || (SELECT coalesce(n_tup_ins,0) FROM pg_stat_all_tables WHERE relid='orders_tags'::regclass)" \
    | tee -a "$OUT/probe-p5.txt"
  q probes "SELECT /* wiki_gin_norm_probe */ pg_stat_force_next_flush()" > /dev/null
  qf probes "$SQLD/.p5.sql" > /dev/null 2>&1
  q probes "SELECT /* wiki_gin_norm_report */ 'capture after pg_stat_force_next_flush(): bti=' ||
              (substring(obj_description('orders_tags_gin'::regclass,'pg_class') from '@ginbase:(\{[^}]*\})')::jsonb->>'bti')
              || ' n_tup_ins=' || (SELECT coalesce(n_tup_ins,0) FROM pg_stat_all_tables WHERE relid='orders_tags'::regclass)" \
    | tee -a "$OUT/probe-p5.txt"
  q probes "SELECT /* wiki_gin_norm_report */ 'payload bytes: ' || length(
              substring(obj_description('orders_tags_gin'::regclass,'pg_class') from '@ginbase:\{[^}]*\}'))
              || ' | whole comment bytes: ' || length(obj_description('orders_tags_gin'::regclass,'pg_class'))" \
    | tee -a "$OUT/probe-p5.txt"
}

# ---------------------------------------------------------------- stage: edge
stage_edge() {
  say "edge cases: comment survival, rebuild detection, locks, privileges, dump"
  write_published_sql
  q postgres "DROP /* wiki_gin_norm_edge */ DATABASE IF EXISTS edge" > /dev/null
  q postgres "DROP /* wiki_gin_norm_edge */ ROLE IF EXISTS wiki_reader" > /dev/null
  q postgres "CREATE /* wiki_gin_norm_edge */ DATABASE edge" > /dev/null
  : > "$OUT/edge.txt"
  qin edge > "$OUT/edge-build.log" 2>&1 <<SQL
CREATE /* wiki_gin_norm_edge */ TABLE edge_t (id bigint, tags int[]);
INSERT /* wiki_gin_norm_edge */ INTO edge_t (id, tags)
SELECT i, ARRAY[(i % 2000)::int, ((i * 7919) % 2000)::int] FROM generate_series(1, 100000) i;
CREATE /* wiki_gin_norm_edge */ INDEX edge_gin ON edge_t USING gin (tags);
CREATE /* wiki_gin_norm_edge */ INDEX fresh_btree ON edge_t (id);
VACUUM /* wiki_gin_norm_edge */ (ANALYZE) edge_t;
COMMENT /* wiki_gin_norm_edge */ ON INDEX edge_gin IS
  E'owner: search-team\nticket: OPS-1234 {do not drop} @ 100%';
SQL
  sed "s/public\.orders_tags_gin/edge_gin/" "$SQLD/capture.sql" > "$SQLD/.edge_cap.sql"
  qf edge "$SQLD/.edge_cap.sql" > /dev/null
  {
    printf 'E1 human comment kept: %s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ 'payload=' || length(substring(obj_description('edge_gin'::regclass,'pg_class') from '@ginbase:\{[^}]*\}')) || 'B comment=' || length(obj_description('edge_gin'::regclass,'pg_class')) || 'B human=' || btrim(regexp_replace(obj_description('edge_gin'::regclass,'pg_class'), '\s*@ginbase:\{[^}]*\}', '', 'g'))")"
    printf 'E2 plain REINDEX: %s\n' "$(
      q edge "SELECT /* wiki_gin_norm_edge */ 'oid=' || 'edge_gin'::regclass::oid || ' relfilenode=' || relfilenode FROM pg_class WHERE oid='edge_gin'::regclass")"
    q edge "REINDEX /* wiki_gin_norm_edge */ INDEX edge_gin" > /dev/null
    printf '   after:              %s comment_intact=%s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ 'oid=' || 'edge_gin'::regclass::oid || ' relfilenode=' || relfilenode FROM pg_class WHERE oid='edge_gin'::regclass")" \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ obj_description('edge_gin'::regclass,'pg_class') LIKE '%@ginbase:%'")"
    printf 'E3 REINDEX CONCURRENTLY before: %s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ 'oid=' || 'edge_gin'::regclass::oid")"
    q edge "REINDEX /* wiki_gin_norm_edge */ INDEX CONCURRENTLY edge_gin" > /dev/null
    printf '   after: %s payload_moved=%s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ 'oid=' || 'edge_gin'::regclass::oid")" \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ obj_description('edge_gin'::regclass,'pg_class') LIKE '%@ginbase:%'")"
    printf 'E4 evaluate after E3: %s\n' \
      "$(qat edge "$SQLD/evaluate.sql" 2>&1 | grep -o 'rebuilt since baseline: re-capture' | head -1)"
    q edge "ALTER /* wiki_gin_norm_edge */ INDEX edge_gin RENAME TO edge_gin2" > /dev/null
    printf 'E5 ALTER INDEX RENAME: payload_survives=%s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ obj_description('edge_gin2'::regclass,'pg_class') LIKE '%@ginbase:%'")"
    q edge "ALTER /* wiki_gin_norm_edge */ INDEX edge_gin2 RENAME TO edge_gin" > /dev/null
    printf 'E6 COMMENT lock footprint:\n'
    qin edge <<'SQL' 2>&1 | sed 's/^/   /'
BEGIN;
COMMENT /* wiki_gin_norm_edge */ ON INDEX edge_gin IS 'probe';
SELECT /* wiki_gin_norm_edge */ l.relation::regclass::text AS rel, l.mode
  FROM pg_locks l WHERE l.pid = pg_backend_pid() AND l.relation IS NOT NULL
   AND l.relation IN ('edge_gin'::regclass, 'edge_t'::regclass) ORDER BY 1;
ROLLBACK;
SQL
    printf 'E7 non-owner: '
    q postgres "CREATE /* wiki_gin_norm_edge */ ROLE wiki_reader LOGIN" > /dev/null
    q postgres "GRANT /* wiki_gin_norm_edge */ pg_read_all_stats TO wiki_reader" > /dev/null
    q edge "GRANT /* wiki_gin_norm_edge */ SELECT ON edge_t TO wiki_reader" > /dev/null
    q edge "GRANT /* wiki_gin_norm_edge */ CONNECT ON DATABASE edge TO wiki_reader" > /dev/null
    printf 'reads=%s ' "$("$BIN/psql" -X -U wiki_reader -h "$SOCK" -p "$PORT" -d edge -At \
      -c "SELECT /* wiki_gin_norm_edge */ (substring(obj_description('edge_gin'::regclass,'pg_class') from '@ginbase:(\{[^}]*\})')::jsonb->>'bis')" 2>&1 | head -1)"
    printf 'write=%s\n' "$("$BIN/psql" -X -U wiki_reader -h "$SOCK" -p "$PORT" -d edge -At \
      -c "COMMENT /* wiki_gin_norm_edge */ ON INDEX edge_gin IS 'nope'" 2>&1 | tr '\n' ' ')"
    printf 'E11 capture aimed at a B-tree: %s\n' \
      "$(sed "s/public\.orders_tags_gin/fresh_btree/" "$SQLD/capture.sql" > "$SQLD/.edge_bt.sql";
         qf edge "$SQLD/.edge_bt.sql" 2>&1 | grep -o 'not a GIN index: [a-z_]*' | head -1)"
    qin edge > /dev/null 2>&1 <<'SQL'
CREATE /* wiki_gin_norm_edge */ TABLE t0 (id bigint, tags int[]);
INSERT /* wiki_gin_norm_edge */ INTO t0 (id, tags)
SELECT i, ARRAY[(i % 2000)::int] FROM generate_series(1, 80000) i;
SQL
    printf 'E13 reltuples before any index: %s' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ reltuples FROM pg_class WHERE oid='t0'::regclass")"
    q edge "CREATE /* wiki_gin_norm_edge */ INDEX t0_gin ON t0 USING gin (tags)" > /dev/null
    printf ', after CREATE INDEX: table=%s index=%s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ reltuples::bigint FROM pg_class WHERE oid='t0'::regclass")" \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ reltuples::bigint FROM pg_class WHERE oid='t0_gin'::regclass")"
    q edge "TRUNCATE /* wiki_gin_norm_edge */ t0" > /dev/null
    printf 'E14 after TRUNCATE, reltuples=%s, capture says: %s\n' \
      "$(q edge "SELECT /* wiki_gin_norm_edge */ reltuples FROM pg_class WHERE oid='t0'::regclass")" \
      "$(sed "s/public\.orders_tags_gin/t0_gin/" "$SQLD/capture.sql" > "$SQLD/.edge_t0.sql";
         qf edge "$SQLD/.edge_t0.sql" 2>&1 | grep -o 'refusing baseline for .*' | head -1)"
    printf 'E15 pg_dump --section=post-data carries the payload: %s line(s)\n' \
      "$("$BIN/pg_dump" -h "$SOCK" -p "$PORT" -d edge -t edge_t --section=post-data 2>/dev/null \
         | grep -c '@ginbase:')"
  } | tee "$OUT/edge.txt"
}

# -------------------------------------------------------------- stage: pubsql
# The published statements, verbatim and unsubstituted, through one whole
# lifecycle under the protocol's phases.
stage_pubsql() {
  say "pubsql: capture -> churn -> settle -> maintain -> decide -> oracle -> re-capture"
  write_published_sql
  q postgres "DROP /* wiki_gin_norm_pub */ DATABASE IF EXISTS pubsql" > /dev/null
  q postgres "CREATE /* wiki_gin_norm_pub */ DATABASE pubsql" > /dev/null
  : > "$OUT/pubsql.txt"
  qin pubsql > "$OUT/pubsql-build.log" 2>&1 <<SQL
CREATE /* wiki_gin_norm_pub */ EXTENSION pageinspect;
CREATE /* wiki_gin_norm_pub */ TABLE orders (id bigint, tags int[]);
INSERT /* wiki_gin_norm_pub */ INTO orders (id, tags)
SELECT i, ARRAY[((i * 7919) % $KEYS)::int, ((i * 104729) % $KEYS)::int,
                ((i * 1299709) % $KEYS)::int, ((i * 15485863) % $KEYS)::int,
                ((i * 32452843) % $KEYS)::int]
  FROM generate_series(1::bigint, $PEND_ROWS) i;
CREATE /* wiki_gin_norm_pub */ INDEX orders_tags_gin ON orders USING gin (tags);
VACUUM /* wiki_gin_norm_pub */ (ANALYZE) orders;
COMMENT /* wiki_gin_norm_pub */ ON INDEX orders_tags_gin IS
  E'owner: search-team\nticket: OPS-1234 {do not drop} @ 100%';
SELECT /* wiki_gin_norm_pub */ pg_stat_force_next_flush();
SQL
  {
    printf 'build: index %s bytes, table reltuples %s\n' \
      "$(q pubsql "SELECT /* wiki_gin_norm_pub */ pg_relation_size('orders_tags_gin','main')")" \
      "$(q pubsql "SELECT /* wiki_gin_norm_pub */ reltuples::bigint FROM pg_class WHERE oid='orders'::regclass")"
    printf '\ncapture (statement 1, verbatim):\n'
    qf pubsql "$SQLD/capture.sql" 2>&1 | sed 's/^/   /'
    q pubsql "SELECT /* wiki_gin_norm_pub */ '   payload: ' ||
                substring(obj_description('orders_tags_gin'::regclass,'pg_class') from '@ginbase:(\{[^}]*\})')"
    printf '\nread (statement 2, verbatim):\n'
    "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d pubsql -x -f "$SQLD/read.sql" 2>&1 | sed 's/^/   /'
  } >> "$OUT/pubsql.txt"
  qin pubsql > "$OUT/pubsql-churn.log" 2>&1 <<SQL
UPDATE /* wiki_gin_norm_pub */ orders SET tags = ARRAY[((id + 1) % $KEYS)::int, ((id * 7919 + 1) % $KEYS)::int,
       ((id * 104729 + 1) % $KEYS)::int, ((id * 1299709 + 1) % $KEYS)::int, ((id * 15485863 + 1) % $KEYS)::int];
UPDATE /* wiki_gin_norm_pub */ orders SET tags = ARRAY[((id + 2) % $KEYS)::int, ((id * 7919 + 2) % $KEYS)::int,
       ((id * 104729 + 2) % $KEYS)::int, ((id * 1299709 + 2) % $KEYS)::int, ((id * 15485863 + 2) % $KEYS)::int];
SELECT /* wiki_gin_norm_pub */ pg_stat_force_next_flush();
SQL
  Q pubsql "VACUUM /* wiki_gin_norm_pub_settle */ (VERBOSE) orders;" > "$OUT/pubsql-settle.log" 2>&1
  Q pubsql "VACUUM /* wiki_gin_norm_pub_maint */ (VERBOSE, ANALYZE) orders;" > "$OUT/pubsql-maint.log" 2>&1
  {
    printf '\nchurn + settle + maintenance: index %s bytes, pending pages %s\n' \
      "$(q pubsql "SELECT /* wiki_gin_norm_pub */ pg_relation_size('orders_tags_gin','main')")" \
      "$(q pubsql "SELECT /* wiki_gin_norm_pub */ n_pending_pages FROM gin_metapage_info(get_raw_page('orders_tags_gin',0))" 2>/dev/null || echo 'n/a')"
    printf '\ndecide (statement 3, verbatim, under SHARE ROW EXCLUSIVE):\n'
    { printf "SET /* wiki_gin_norm_pub */ lock_timeout = '30s';\nBEGIN /* wiki_gin_norm_pub */;\n"
      printf "LOCK /* wiki_gin_norm_pub */ TABLE orders IN SHARE ROW EXCLUSIVE MODE;\n"
      cat "$SQLD/evaluate.sql"
    } | "$BIN/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$PORT" -d pubsql -x 2>&1 | sed 's/^/   /'
    printf '\noracle:\n'
    printf '   before %s\n' "$(q pubsql "SELECT /* wiki_gin_norm_pub */ pg_relation_size('orders_tags_gin','main')")"
    q pubsql "REINDEX /* wiki_gin_norm_pub */ INDEX orders_tags_gin" > /dev/null
    printf '   after  %s\n' "$(q pubsql "SELECT /* wiki_gin_norm_pub */ pg_relation_size('orders_tags_gin','main')")"
    printf '\nre-capture (statement 1 again):\n'
    qf pubsql "$SQLD/capture.sql" 2>&1 | sed 's/^/   /'
    q pubsql "SELECT /* wiki_gin_norm_pub */ '   payload: ' ||
                substring(obj_description('orders_tags_gin'::regclass,'pg_class') from '@ginbase:(\{[^}]*\})')"
    q pubsql "SELECT /* wiki_gin_norm_pub */ '   human comment: ' ||
                btrim(regexp_replace(obj_description('orders_tags_gin'::regclass,'pg_class'),
                                     '\s*@ginbase:\{[^}]*\}', '', 'g'))"
    q pubsql "SELECT /* wiki_gin_norm_pub */ '   payload count: ' ||
                (length(obj_description('orders_tags_gin'::regclass,'pg_class'))
                 - length(replace(obj_description('orders_tags_gin'::regclass,'pg_class'), '@ginbase:', '')))/9"
    printf '\nre-evaluate:\n'
    qat pubsql "$SQLD/evaluate.sql" 2>&1 | sed 's/^/   /'
  } >> "$OUT/pubsql.txt"
  cat "$OUT/pubsql.txt"
}

# -------------------------------------------------------------- stage: verify
# Re-extract the three published statements from the page and diff them against
# the files this run executed. No literal fence appears in this script.
stage_verify() {
  say "verify: the published SQL against the SQL that ran"
  write_published_sql
  local fence line buf in_block n=0
  fence=$(printf '\140\140\140')
  in_block=0; buf=""
  rm -f "$OUT/x-capture.sql" "$OUT/x-read.sql" "$OUT/x-evaluate.sql"
  while IFS= read -r line; do
    if [ "$in_block" = 0 ]; then
      [ "$line" = "${fence}sql" ] && { in_block=1; buf=""; }
      continue
    fi
    if [ "$line" = "$fence" ]; then
      in_block=0
      case "$buf" in
        *wiki_gin_capture_baseline*)   printf '%s' "$buf" > "$OUT/x-capture.sql";  n=$((n+1)) ;;
        *wiki_gin_read_baseline*)      printf '%s' "$buf" > "$OUT/x-read.sql";     n=$((n+1)) ;;
        *wiki_gin_reindex_candidates*) printf '%s' "$buf" > "$OUT/x-evaluate.sql"; n=$((n+1)) ;;
      esac
      continue
    fi
    buf="$buf$line
"
  done < "$PAGE"
  printf 'published wiki_gin blocks found: %s\n' "$n"
  local ok=0
  for pair in capture read evaluate; do
    if diff -u "$OUT/x-$pair.sql" "$SQLD/$pair.sql" > "$OUT/diff-$pair.txt" 2>&1; then
      printf '  %-9s identical (%s lines)\n' "$pair" "$(wc -l < "$SQLD/$pair.sql")"
      ok=$((ok+1))
    else
      printf '  %-9s DIFFERS:\n' "$pair"; sed 's/^/     /' "$OUT/diff-$pair.txt"
    fi
  done
  printf 'verbatim: %s of 3\n' "$ok" | tee "$OUT/verify.txt"
}

# --------------------------------------------------------------- stage: clean
stage_clean() {
  say "clean: stop the cluster and delete the sandbox"
  if [ -x "$BIN/pg_ctl" ] && "$BIN/pg_ctl" -D "$PGDATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$PGDATA" -m fast -w stop
  else
    printf 'no cluster of this sandbox is running\n'
  fi
  printf 'postmaster.pid present: %s\n' "$([ -f "$PGDATA/postmaster.pid" ] && echo yes || echo no)"
  printf 'matching postgres processes: %s\n' "$(pgrep -af "$PGDATA" | wc -l)"
  printf 'port %s listeners: %s\n' "$PORT" "$(pgrep -af "port=$PORT" | wc -l)"
  cd /
  rm -rf "$SANDBOX"
  printf 'sandbox deleted: %s\n' "$([ -d "$SANDBOX" ] && echo no || echo yes)"
}

need_server() {
  [ -x "$BIN/postgres" ] || die "no install under $INST: run the build stage first"
  "$BIN/pg_ctl" -D "$PGDATA" status > /dev/null 2>&1 || stage_start
}

ALL="build declare fixtures churn analyze_census crosscheck decide oracle score probes edge pubsql verify"

run_stage() {
  case "$1" in
    build|start|clean|reset) : ;;
    *) need_server ;;
  esac
  "stage_$1"
}

main() {
  local stages="$*"
  [ -z "$stages" ] && stages="$ALL"
  [ "$stages" = "all" ] && stages="$ALL"
  for s in $stages; do
    declare -F "stage_$s" > /dev/null || die "no such stage: $s (have: $ALL reset start clean)"
  done
  for s in $stages; do run_stage "$s"; done
  say "done: $stages"
}

main "$@"
```

## Context Reviewed

- **The protocol**: [Mandatory GIN Bloat Tests](../../common-concepts/mandatory-gin-bloat-tests.md)
  read end to end before this revision — its five phases, the settle step's five proof
  obligations, the maintenance assumption's three parts, the auto-analyze stand-in and its
  three stated differences, the simulated census and its two publication points, the
  measurement lock's three rules, the oracle's two constraints, the declare-then-score rule,
  the four cross-checks, the concurrency and reading rules, the twelve-row coverage table and
  the named limits. The concept page was **not edited** as part of this work.
- **GIN access method**: `ginvacuum.c` (`ginvacuumcleanup` including its `analyze_only`
  branch, the `num_index_tuples` assignment, `GinPageIsRecyclable`, `ginDeletePage`),
  `gininsert.c` (`ginbuild`, `ginBuildCallback` and its `maintenance_work_mem` flush,
  `ginHeapTupleBulkInsert`, `buildFreshLeafTuple`, `addItemPointersToLeafTuple`),
  `ginfast.c` (`ginInsertCleanup`, `GIN_PAGE_FREESIZE`, the foreground cleanup threshold,
  `shiftList`, `gin_clean_pending_list` and its return value, the pending-page FSM
  recycling), `ginutil.c` (`ginhandler`, `ginoptions`, `GinNewBuffer`, `ginUpdateStats`),
  `ginblock.h` (`GinMetaPageData`, the flag bits, `GinPageGetDeleteXid`). Confirmed by grep
  that no file under `src/backend/access/gin/` calls `RelationTruncate`.
- **Catalog and index maintenance**: `index.c` (`index_build`, `index_update_stats`,
  `reindex_index`, `index_concurrently_swap`), `pg_class.h`, `pg_index.h`,
  `pg_description.h`.
- **Statistics writers**: `analyze.c` (`do_analyze_rel`, `tupleFract`, the ANALYZE-only
  cleanup gate, the per-index `vac_update_relstats` call), `vacuum.c`
  (`vac_cleanup_one_index`, `vacuum_get_cutoffs`, the vacuum-then-analyze order),
  `vacuumlazy.c` (`do_index_cleanup`, `lazy_cleanup_all_indexes`,
  `update_relstats_all_indexes`, the `estimated_count` guard, the `VERBOSE` index line and
  the bypass message), `pgstat_relation.c` (`pgstat_report_analyze`, `pgstat_report_vacuum`,
  `pgstat_relation_flush_cb`), `pgstat.c` (`pgstat_report_stat`, `PGSTAT_MIN_INTERVAL`,
  `PGSTAT_MAX_INTERVAL`).
- **Autovacuum**: `autovacuum.c` (`relation_needs_vacanalyze`'s three threshold formulas,
  the reloption-or-GUC selection, the `av_enabled` short circuit, the strictly-greater
  comparisons), `guc_tables.c` for the five threshold and scale-factor GUCs.
- **Comment plumbing**: `comment.c` (`CommentObject`), `system_functions.sql`
  (`obj_description`, `pg_relation_size`), `system_views.sql` (`pg_stat_all_tables`),
  `dbsize.c` (`calculate_relation_size`, `pg_relation_size`), `pg_proc.dat`
  (`pg_stat_force_next_flush`).
- **The free space map**: `indexfsm.c` (`RecordFreeIndexPage`), `freespace.c`
  (`fsm_space_cat_to_avail`), and `pg_freespacemap`'s `pg_freespace`.
- **GUCs**: `guc_tables.c` for `maintenance_work_mem`, `gin_pending_list_limit`,
  `statement_timeout`, `lock_timeout`.
- **Documentation, same checkout**: `gin.sgml` (fast update, tips), `maintenance.sgml`
  (`routine-reindex`), `ref/reindex.sgml`, `ref/comment.sgml`.
- **Server work, 2026-09-15**: one 17.11 build from the pin and one run of the filed script
  end to end — `make check` plus five contrib suites, 25 fixtures through five phases, a
  28-table analyze census, 23 page censuses under the measurement lock, 24 oracle rebuilds,
  five probes, eleven edge cases, a verbatim lifecycle replay, and the three-way diff of the
  published statements against this page.

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
| The entry tree never deletes a tuple or a page, so retired keys survive every VACUUM | [README#page-deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396) |
| The pending list exists, is bounded by `gin_pending_list_limit`, and is flushed by VACUUM/autoanalyze | [gin.sgml#gin-fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537), [gin.sgml#guc-gin-pending-list-limit](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L595-L616) |
| An insert stream flushes the pending list in the foreground once `nPendingPages * GIN_PAGE_FREESIZE` exceeds the limit | [ginfast.c:458-471](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L458-L471), [ginfast.c:41-42](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L41-L42) |
| `gin_clean_pending_list()` returns its own count of deleted pending pages | [ginfast.c:1090](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1090) |
| In an autovacuum worker an `analyze_only` GIN cleanup flushes the pending list and returns before the census; every other caller gets a no-op | [ginvacuum.c#analyze_only-worker](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L709-L717) |
| ANALYZE-only index cleanup is skipped inside a `VACUUM ANALYZE` | [analyze.c#analyze-only-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L694-L721) |
| `vacuum()` vacuums each relation and then analyzes it | [vacuum.c#vacuum-then-analyze](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650) |
| The metapage page counts have one writer, `ginUpdateStats`, reached only from the census inside `ginvacuumcleanup` | [ginutil.c#ginUpdateStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650) |
| `INDEX_CLEANUP` off forces `do_index_cleanup` false, and the cleanup call is gated on it | [vacuumlazy.c#do_index_cleanup-init](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397), [vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066) |
| `VERBOSE` prints "index scan bypassed" and no per-index line when index vacuuming did not run | [vacuumlazy.c#verbose-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L731) |
| The fourth field of the `VACUUM VERBOSE` index line is `pages_free` | [vacuumlazy.c#verbose-index-line](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L720-L731) |
| A held snapshot keeps deleted tuples non-removable, because `OldestXmin` comes from `GetOldestNonRemovableTransactionId` | [vacuum.c:1120](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1120) |
| An index FSM records a free page as `BLCKSZ - 1` and reads it back as a category floor | [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435) |
| The launcher's three verdicts, their formulas and the strictly-greater comparison | [autovacuum.c#vacthresh-anlthresh](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076), [autovacuum.c#verdicts](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095) |
| `autovacuum_enabled = false` short-circuits both verdicts | [autovacuum.c#av_enabled-return](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054) |
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
| GIN build size is sensitive to `maintenance_work_mem` | [gin.sgml#guc-maintenance-work-mem](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L584-L593) |
| Non-B-tree bloat is not well researched; monitor the physical size | [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046) |
| A bloated index is a documented reason to `REINDEX` | [reindex.sgml:54-64](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64) |

Every number in the tables above is a measurement from the 17.11 run described under
[The last run](#the-last-run), not a source claim.

## Open Questions

1. **The one declared bound failed, and the column that failed it is still printed.**
   `est_reclaimable` was violated on 7 of 21 fixtures and is demoted to a level, but the
   statement's text is unchanged, so an operator reading the output sees a byte count with
   nothing in the row to warn them. Whether the right fix is to drop the column, rename it to
   a ranking score, or print the declared kind beside it, is a design choice this page does
   not make — and any of them would need a re-run, because the text that ran is the text that
   is scored.
2. **The payoff threshold is this page's, not the engine's.** `truth_pct >= 33.33` was
   derived from the brief's own 1.50 candidate threshold, and `c05`'s 30.45 % is a false
   positive only against that number. v17 defines no target density for a GIN index
   ([ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)),
   so a `FALSE POSITIVE` here is not comparable with one on a page that chose differently.
3. **`normalized_index_growth` is not monotone in reclaimable space.** It ranks `c05`
   (1.9184, 30.45 % returned) above `c12` (1.5171, 34.08 %) and `x03` (1.8263, 45.33 %).
   Whether a better normalizer exists inside the catalog-only constraint is untested; the
   obvious candidate, an entry-count term, is what the brief rules out.
4. **A rebuild can grow a GIN index, and nothing the method reads can predict it.** `p02`
   returned −21.01 %. The mechanism is understood (a bulk build appends per flush round while
   incremental insertion packed denser), but the method has no input that distinguishes such
   an index from one worth rebuilding, and this run has exactly one example of it.
5. **The pending-list blind spot is characterised, not corrected.** `pgstatginindex` reports
   `pending_pages`, which would let an evaluation subtract the live pending list, but it is
   contrib and so outside the brief's catalog-only rule, and it does not report the
   *high-water mark* left behind after a flush. No catalog-only substitute was found.
6. **An auto-analyze-only maintained table is almost always refused.** The window where the
   analyze verdict is crossed, both vacuum verdicts are not, and the method's churn gate is
   satisfied is 1,001 rows wide at any table size; `a02` sits inside it only because the
   fixture was built to. Whether the churn gate should be lowered to `0.1 N` to match the
   launcher's analyze scale factor is untested, and doing so would evaluate far more indexes
   for far less return.
7. **Two ladder rungs cannot be exercised under the protocol.** Rung 7,
   `no ANALYZE since baseline`, cannot fire because the maintenance step analyzes every
   churned table, and `stats_lag` reads 0.000 on all 24 fixtures for the same reason. Both
   are still the right guards for a server that does not maintain its tables; neither is
   scored here.
8. **Rung 2, `invalid index`, was never exercised.** No fixture produced a failed
   `CREATE INDEX CONCURRENTLY`, so that rung remains source-justified from `indisvalid` only.
9. **The vacuum side of the launcher is never simulated.** The protocol records the
   dead-tuple and insert thresholds and then maintains regardless of them, so no fixture here
   is a GIN index whose dead entries were never vacuumed — which is exactly the state a
   size-only heuristic would most like to catch. The concept page files the same gap as a
   limit of the protocol.
10. **The partial index was measured at one selectivity.** `x01` indexes a quarter of its
    rows, and the method divides by the table's `reltuples`, which is not the indexed
    population; the rebuild returned 68.75 % and the method flagged it, so the mismatch did
    not hurt here. `ginvacuumcleanup`'s own comment calls its entry count "bogus if the index
    is partial"
    ([ginvacuum.c:733-739](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L733-L739)),
    and `ANALYZE` computes a real `tupleFract` for one
    ([analyze.c:948-953](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)).
    Expect false positives at other selectivities; none were measured.
11. **The census's decision power is exercised only by synthetic tables.** All 25 fixture
    tables read `n_mod_since_analyze = 0` when the census ran, because the maintenance step
    had already analyzed them, so `tc_past`, `tc_exact` and `tc_off` are doing all the work.
12. **`a02`'s metapage is off by one entry page against the census.** The stand-in's flush
    allocated an entry page that no `ginUpdateStats` ever recorded, and only a `VACUUM` would
    reconcile it. That is the declared consequence of the stand-in, but it means an
    auto-analyze-maintained GIN index has a metapage that under-reports its own entry tree,
    which no shipped test covers.
13. **One cluster, one block size, one platform, one key universe.** `block_size` 8192,
    `max_data_alignment` 8, `shared_buffers` 512MB, `maintenance_work_mem` 256MB except in
    the explicit budget probe, five keys per row from a 10,000-value universe, and one run.
    Nothing here says the 1.50/1.20 boundaries or the 0.20 churn gate transfer to a different
    key cardinality, a different tuple width, or a 100x larger table.
14. **The scored `est_reclaimable` is a recomputation, not the printed string.** The scoring
    recomputes the bytes from the unrounded inputs and checks them against the statement's own
    `pg_size_pretty` output, which matched on 24 of 24; it does not parse the printed text
    back, so a formatting change in `pg_size_pretty` would go unnoticed.
15. **The two concurrency behaviors the protocol lists are only partly reached.** `c09`
    covers a rebuild between the maintenance step and the decide pass, but no writer stream or
    concurrent `VACUUM` ran against a decide pass, because the measurement lock excludes both
    by construction. The page census's own non-atomicity is therefore unexercised here.

## Source References

- [ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c) - `ginvacuumcleanup`, its `analyze_only` branch, `GinPageIsRecyclable`, `ginDeletePage`, the `num_index_tuples` assignment.
- [gininsert.c](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c) - `ginbuild`, `ginBuildCallback` and its memory-driven flush, `ginHeapTupleBulkInsert`, `buildFreshLeafTuple`, `addItemPointersToLeafTuple`.
- [ginfast.c](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c) - `GIN_PAGE_FREESIZE`, the foreground cleanup threshold, `ginInsertCleanup`, `shiftList`, `gin_clean_pending_list` and its return value, the pending-page FSM recycling.
- [ginutil.c](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c) - `ginhandler`, `ginoptions`, `GinNewBuffer`, `ginUpdateStats`.
- [gin_private.h](../../../../raw/postgres-17/src/include/access/gin_private.h), [ginblock.h](../../../../raw/postgres-17/src/include/access/ginblock.h), [reloptions.c](../../../../raw/postgres-17/src/backend/access/common/reloptions.c) - `GinOptions`, the metapage struct and flag bits, `fastupdate` and `gin_pending_list_limit` defaults and lock levels.
- [README](../../../../raw/postgres-17/src/backend/access/gin/README) - page deletion, and why the entry tree keeps its tuples.
- [index.c](../../../../raw/postgres-17/src/backend/catalog/index.c) - `index_build`, `index_update_stats`, `reindex_index`, `index_concurrently_swap`.
- [analyze.c](../../../../raw/postgres-17/src/backend/commands/analyze.c) - `do_analyze_rel`, `tupleFract`, the ANALYZE-only cleanup gate, the per-index `vac_update_relstats` call.
- [vacuum.c](../../../../raw/postgres-17/src/backend/commands/vacuum.c) - `vac_cleanup_one_index`, `vacuum_get_cutoffs`, the vacuum-then-analyze order.
- [vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c) - `do_index_cleanup`, `lazy_cleanup_all_indexes`, `update_relstats_all_indexes`, the `estimated_count` guard, the `VERBOSE` index line and the bypass message.
- [autovacuum.c](../../../../raw/postgres-17/src/backend/postmaster/autovacuum.c) - `relation_needs_vacanalyze`'s thresholds, the reloption-or-GUC selection, the `av_enabled` short circuit.
- [pgstat_relation.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c) - `pgstat_report_analyze`, `pgstat_report_vacuum`, `pgstat_relation_flush_cb`.
- [pgstat.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c) - `pgstat_report_stat`, `PGSTAT_MIN_INTERVAL`, `PGSTAT_MAX_INTERVAL`.
- [comment.c](../../../../raw/postgres-17/src/backend/commands/comment.c) - `CommentObject`, `check_object_ownership`.
- [dbsize.c](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c) - `calculate_relation_size`, `pg_relation_size`.
- [indexfsm.c](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c), [freespace.c](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c) - what an index FSM stores, and what a free page reads back.
- [system_functions.sql](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql) - `obj_description`, `pg_relation_size`.
- [system_views.sql](../../../../raw/postgres-17/src/backend/catalog/system_views.sql) - `pg_stat_all_tables`.
- [guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c) - `maintenance_work_mem`, `gin_pending_list_limit`, `statement_timeout`, `lock_timeout`, the autovacuum thresholds.
- [pg_class.h](../../../../raw/postgres-17/src/include/catalog/pg_class.h), [pg_index.h](../../../../raw/postgres-17/src/include/catalog/pg_index.h), [pg_description.h](../../../../raw/postgres-17/src/include/catalog/pg_description.h), [pg_proc.dat](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat).
- [gin.sgml](../../../../raw/postgres-17/doc/src/sgml/gin.sgml) - fast update technique, tips and tricks.
- [maintenance.sgml](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml) - `routine-reindex`.
- [ref/reindex.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml), [ref/comment.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml).

## Navigation

- [v17/index](../../index.md) - PostgreSQL 17 landing page.
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Mandatory GIN Bloat Tests (unverified)](../../common-concepts/mandatory-gin-bloat-tests.md) - the measurement protocol every number on this page was produced under.
- [Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17 (unverified)](non-btree-index-inflation-comment-baseline.md) - the five-access-method sibling, which normalizes by a per-AM population term instead of the table `reltuples`.
- [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17 (unverified)](gin-index-wasted-space-contrib.md) - what a `pageinspect` page census can measure that catalogs cannot.
- [Reading an Index's Entry Count From the Catalogs, for Every Index Type, in PostgreSQL 17 (unverified)](index-entry-count-from-catalogs.md) - the full three-writer story behind an index's own `reltuples`.
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md) - the rebuild path this heuristic recommends.
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md) - what the planner does and does not see about a bloated index.
- [versions](../../../versions.md) - source pin manifest.
- [index](../../../index.md) - global wiki catalog.
