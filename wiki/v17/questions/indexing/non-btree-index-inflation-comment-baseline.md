---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
  - [Review prompts, 2026-08-25](#review-prompts-2026-08-25)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What the 2026-08-25 review changed](#what-the-2026-08-25-review-changed)
  - [The three-run re-execution](#the-three-run-re-execution)
  - [Why REINDEX is the only thing that shrinks these five access methods](#why-reindex-is-the-only-thing-that-shrinks-these-five-access-methods)
  - [Why a current-over-baseline size ratio is the wrong question](#why-a-current-over-baseline-size-ratio-is-the-wrong-question)
  - [Three catalog facts the design depends on](#three-catalog-facts-the-design-depends-on)
  - [The baseline field set](#the-baseline-field-set)
  - [The comment format](#the-comment-format)
  - [SQL: capture a fresh baseline](#sql-capture-a-fresh-baseline)
  - [SQL: retrieve and parse the baseline](#sql-retrieve-and-parse-the-baseline)
  - [SQL: evaluate](#sql-evaluate)
  - [Access-method-specific normalization](#access-method-specific-normalization)
  - [Handling the four required edge cases](#handling-the-four-required-edge-cases)
  - [Test methodology](#test-methodology)
  - [The 13 test cells](#the-13-test-cells)
  - [Measurement queries](#measurement-queries)
  - [Results](#results)
  - [Inflation predicts reclaimable space, not just rank](#inflation-predicts-reclaimable-space-not-just-rank)
  - [BRIN: desummarize plus summarize made the index bigger](#brin-desummarize-plus-summarize-made-the-index-bigger)
  - [Experimental thresholds](#experimental-thresholds)
  - [Known limitations](#known-limitations)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Filed after prompt-hygiene correction, at the asker's request. The original prompt
wrote `agents.md` for `AGENTS.md`, lowercase `postgresql`, spaces before commas,
`minmax-multi` for the `minmax_multi` opclass family, and `DESUMMARIZE + SUMMARIZE`
for the `brin_desummarize_range()` / `brin_summarize_range()` functions. The asker
chose "correct and restate"; the corrected text is below.

Follow `AGENTS.md`, in PostgreSQL 17. Question:

Design and validate a PostgreSQL heuristic that detects potentially inflated
non-B-tree indexes using only PostgreSQL catalog information, statistics,
table/index metadata, and information stored directly in the index comment. The
heuristic must estimate whether rebuilding an index would materially reduce its
physical size.

The target index access methods are HASH, GIN, GiST, SP-GiST and BRIN. B-tree
indexes are outside the scope of this work.

**Fundamental constraint.** Do not create any PostgreSQL table, metadata table,
extension-specific repository, or external persistent store for the heuristic. The
only persistent information created by this mechanism must be stored with
`COMMENT ON INDEX ...`, so the baseline must live in the existing PostgreSQL index
comment stored in `pg_description`. Existing human-readable comments must be
preserved. Use a recognizable marker such as `@idxmaint:`, and the heuristic
payload may use compact JSON.

**Baseline concept.** Capture the physical and logical characteristics of an index
immediately after it has been freshly built, that is after `CREATE INDEX`,
`REINDEX INDEX`, or `REINDEX INDEX CONCURRENTLY`. The baseline represents what the
index looked like when freshly built for this logical dataset. It must not be
continuously overwritten during normal monitoring; it remains unchanged until the
index is physically rebuilt, and after a successful `REINDEX` the old baseline is
replaced with a new one.

**Suggested baseline information.** Determine the minimum set of values required,
using catalogs and statistics such as `pg_class`, `pg_index`, `pg_am`,
`pg_namespace`, `pg_stat_all_tables`, `pg_stat_user_tables`, `pg_stat_user_indexes`,
`pg_stats`, `pg_description` and the relation-size functions. Consider storing
heuristic version, baseline timestamp, access method, relation filenode, index size,
index relpages, index reltuples, table relpages, table reltuples, `n_live_tup`,
`n_dead_tup`, `n_tup_ins`, `n_tup_upd`, `n_tup_hot_upd`, `n_tup_del`,
`vacuum_count`, `autovacuum_count`, and statistics reset information. Use
abbreviated JSON field names if appropriate to keep the comment small. Review which
fields are actually useful and remove unnecessary ones.

**Core heuristic.** Distinguish legitimate index growth from physical growth caused
by data churn that could be removed by rebuilding. Do not simply compare
`current_index_size / baseline_index_size`. Instead estimate how large a freshly
rebuilt index should be for the current logical dataset:

```text
expected_fresh_size_now = baseline_index_size
                        * current_logical_population
                        / baseline_logical_population

size_inflation = current_index_size / expected_fresh_size_now
```

For a baseline of 100M logical units at 10 GB and a current state of 120M logical
units at 18 GB, the expected fresh size is 12 GB and the size inflation is 1.50.
Interpret this as approximately 50% larger than predicted by the fresh-build
baseline, not as an exact bloat percentage.

**Churn heuristic.** From the table statistics captured at baseline and the current
cumulative statistics, calculate `delta_inserts`, `delta_non_hot_updates` as
`(current_n_tup_upd - baseline_n_tup_upd) - (current_n_tup_hot_upd -
baseline_n_tup_hot_upd)`, and `delta_deletes`; sum them into
`index_affecting_churn`, and normalize as
`churn_ratio = index_affecting_churn / current_logical_population`. Do not
recommend `REINDEX` solely because churn is high. The strongest signal should be a
combination of normalized size inflation, substantial index-affecting churn,
`VACUUM` having occurred, and a low remaining dead tuple count.

**VACUUM consideration.** Differentiate an index that contains dead entries
`VACUUM` has not yet processed from an index that remains physically inflated even
after `VACUUM`, using baseline and current `vacuum_count`, `autovacuum_count`, or
other suitable catalog/statistics information. A strong `REINDEX` candidate should
generally have high normalized size inflation AND high churn AND `VACUUM` occurred
after significant churn AND a relatively low current dead tuple ratio. This is
intended to identify post-`VACUUM` residual physical inflation.

**Access-method-specific normalization.** Do not assume one logical model works
equally well for every access method.

For HASH use approximately `logical_population = indexed tuples`. HASH should be
one of the highest-confidence access methods. Test both overflow-page growth caused
by a highly duplicated key, and high-water growth where the table/index grows
dramatically and is later reduced to its original logical population. Candidate
starting thresholds may be tested around size inflation >= 1.30 AND churn >= 0.50,
with stronger confidence around size inflation >= 1.50 AND churn >= 1.00. Do not
assume these are final production thresholds.

For GiST start with logical population = estimated indexed tuples, potentially
improved using indexed-value statistics such as average width. Test repeated
changes of indexed range values while keeping row count constant, and high-water
population growth followed by deletion back to baseline. Use built-in PostgreSQL
range types if possible so the test does not require PostGIS, for example
`int8range`. Use more conservative thresholds than HASH.

For SP-GiST use approximately `logical population = indexed tuples`, with possible
adjustment for indexed-value size. Test repeated replacement of text values with
different prefix families while row count stays constant, average text width stays
approximately constant, and indexed values change every round. The objective is to
generate substantial SP-GiST structural churn without legitimate population growth.
Also test temporary high-water growth followed by deletion back to baseline.

GIN requires a more conservative model because one heap row may generate many
indexed entries. Investigate whether normalization should incorporate row count
times average indexed-value width, or another catalog-derived approximation of
logical input mass. Test at least three workloads: (1) common-key posting growth,
using arrays where millions of rows temporarily reference the same few keys, then
deleting those temporary rows, running `VACUUM`, and comparing physical size with a
subsequent `REINDEX`; (2) key churn with constant table population, repeatedly
updating the indexed arrays so each round produces entirely different indexed keys
while row count and average indexed-value width stay approximately constant, run
over several rounds without `VACUUM` and followed by `VACUUM (ANALYZE)`; and (3) a
false-positive control for pending-list behavior, using `fastupdate = on` to
generate a significant pending list and measuring the index before and after
pending-list processing / `VACUUM`. The heuristic must not automatically recommend
`REINDEX` solely because of temporary pending-list growth, and the test should
demonstrate why post-`VACUUM` evaluation is important for GIN.

For BRIN do not use table row count as the primary logical unit. Use approximately
`logical_ranges = ceil(table_relpages / pages_per_range)`, then estimate
`expected_size = baseline_size * current_logical_ranges / baseline_logical_ranges`.
BRIN must have separate tests because it behaves fundamentally differently from
tuple-oriented indexes. BRIN test 1 is a negative control using a standard minmax
BRIN index with massive updates of the indexed values while retaining the same row
population; it should demonstrate that large table churn does not automatically
imply large BRIN physical inflation, and the heuristic should avoid recommending
`REINDEX` based solely on churn. BRIN test 2 uses `minmax_multi` with a suitable
`values_per_range`, starting with strongly correlated data, then changing values so
individual heap ranges contain widely scattered values, then returning the logical
values toward the original correlated pattern; compare after churn + `VACUUM`
against after `REINDEX`. Also investigate whether the more appropriate
recommendation for some BRIN cases is `brin_desummarize_range()` followed by
`brin_summarize_range()` rather than `REINDEX`. The final design should allow
different maintenance recommendations for BRIN.

**Partial indexes.** Partial indexes must not be normalized blindly against
whole-table row count. Use available metadata such as `pg_index.indpred` and
index/table `reltuples`. At baseline calculate an approximate fraction
`baseline_partial_fraction = baseline_index_reltuples / baseline_table_reltuples`,
and at evaluation time `current_partial_fraction = current_index_reltuples /
current_table_reltuples`. If the predicate-selected population changes
substantially, reduce confidence or suppress an automatic `REINDEX`
recommendation. Include tests where the partial-index selectivity changes
dramatically but no meaningful index inflation exists; treat this as a
false-positive test.

**Baseline validity.** Store the relation filenode at baseline. At evaluation time
compare it with `pg_relation_filenode(index_oid)`. If the current filenode differs
from the baseline filenode, assume the physical index has been rebuilt and the old
physical baseline is invalid; the mechanism should then establish a fresh baseline
in the index comment. Also detect situations where cumulative PostgreSQL statistics
have been reset. If churn counters cannot safely be compared, report
`CHURN = UNKNOWN` and do not fabricate a churn estimate; physical size evaluation
may continue, but confidence should be reduced.

**Human comments.** Do not destroy an existing index comment. The implementation
must preserve text that appears before the heuristic marker. When refreshing the
maintenance metadata, replace only the `@idxmaint:` payload.

**Test methodology.** For every inflation-producing test follow approximately:
create a disposable table; disable autovacuum on the test table for deterministic
testing; insert the baseline dataset; `CREATE INDEX`; `ANALYZE`; record the
fresh-build baseline; execute the access-method-specific churn workload; return the
logical indexed population close to baseline where applicable; `VACUUM (ANALYZE)`;
capture current measurements; evaluate the heuristic; `REINDEX INDEX`; `ANALYZE` if
required; measure the index again; and compare the heuristic prediction with the
actual reclaimed space. Do not use `VACUUM FULL` as part of the test, because the
objective is to validate index maintenance behavior, not to rewrite the table.

**Ground-truth validation.** For every test define B as the fresh baseline index
size, C as the index size after churn + `VACUUM`, and R as the index size
immediately after `REINDEX`. The heuristic predicts `estimated_fresh_size` and
`size_inflation = C / estimated_fresh_size`. Calculate
`actual_reclaimed_bytes = C - R` and `actual_reclaimed_fraction = (C - R) / C`. The
purpose of the test suite is to establish whether high heuristic inflation
correlates with significant actual space reclaimed by `REINDEX`.

**Required principal test matrix.** At minimum implement and analyze: HASH with a
highly duplicated hot key (strong physical inflation candidate); HASH growing the
population dramatically then shrinking (strong high-water inflation); GIN
common-key posting growth (possible inflation); GIN repeated complete key
replacement (possible persistent inflation); GIN pending-list-only growth
(false-positive control); GiST repeated range relocation (possible persistent
inflation); GiST population grow/shrink (high-water test); SP-GiST repeated prefix
replacement (possible persistent inflation); SP-GiST population grow/shrink
(high-water test); BRIN minmax with massive indexed-value churn (negative control);
BRIN `minmax_multi` summary expansion/degradation (BRIN-specific maintenance case);
and partial indexes with predicate population changes (false-positive control).

**Output requirements.** Produce a technical assessment of the proposed heuristic;
a description of the baseline fields that should be stored in `COMMENT ON INDEX`;
the proposed compact/versioned comment format; SQL to capture a fresh baseline; SQL
to retrieve and parse the baseline; SQL to calculate current metadata; SQL to
calculate normalized size inflation, insert delta, non-HOT update delta, delete
delta, churn ratio, dead tuple ratio, and whether `VACUUM` occurred since baseline;
access-method-specific heuristic logic for HASH, GIN, GiST, SP-GiST and BRIN;
handling for partial indexes, statistics resets, manual `REINDEX` detected through
filenode changes, and existing human comments; a complete reproducible SQL test for
each access method; positive tests that deliberately produce reclaimable index
inflation; negative/false-positive tests where `REINDEX` should not be recommended;
measurement queries to record baseline size, post-churn size, post-`VACUUM` size and
post-`REINDEX` size; a results table showing predicted inflation, actual reclaimed
bytes, actual reclaimed percentage, heuristic recommendation, and whether the
recommendation was correct; recommendations for initial thresholds, clearly
identified as experimental rather than authoritative; and known limitations and
situations where catalog-only metadata cannot reliably distinguish legitimate
structural growth from reclaimable inflation.

**Design principles.** Do not describe this as an exact bloat measurement. Use
terminology such as index inflation, estimated fresh-build size, reclaimable-space
proxy, `REINDEX` candidate, and maintenance heuristic. The primary question the
heuristic must answer is: given the current logical dataset and the characteristics
captured when this index was freshly built, is the index now materially larger than
we would expect a fresh rebuild to be, after accounting for legitimate data growth
and normal `VACUUM` cleanup? The ultimate success criterion is not whether the
heuristic reports a large number, but whether a high heuristic score reliably
predicts that `REINDEX INDEX` or `REINDEX INDEX CONCURRENTLY` will materially
reduce the physical size of the index.

### Review prompts, 2026-08-25

Both filed after prompt-hygiene correction, at the asker's request. Both prompts wrote
`agents.md` for `AGENTS.md`, lowercase `postgresql`, and a space before a comma; the
second also wrote "run again all tests" for "run all tests again". The asker chose
"correct and restate" each time. The corrected texts are:

> Follow `AGENTS.md`, in PostgreSQL 17, review question: # Detecting Inflated
> Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17
> (unverified)

> Follow `AGENTS.md`, in PostgreSQL 17, review question: # Detecting Inflated
> Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17
> (unverified), run all tests again and check the heuristic

The first review re-read every citation and exercised the page's SQL, but deliberately
did not re-run the matrix; defects were fixed in place. Its outcome is
[What the 2026-08-25 review changed](#what-the-2026-08-25-review-changed). The second
answered the gap that left: all tests again, meaning the 13-cell matrix three times
plus every probe, scored by the statement this page now publishes. Its outcome is
[The three-run re-execution](#the-three-run-re-execution).

## Answer

### Verdict

The design works, and on this evidence it does more than rank candidates. Across a
13-cell matrix run on an isolated 17.11 server built from the pin, the reported
`size_inflation` **predicted the fraction of the index that `REINDEX` actually gave
back**, through the closed form `1 - 1/size_inflation`, to within **0.0 percentage
points on 7 of 13 cells and 2.5 points on 12 of 13**. Seven cells were flagged as
`strong REINDEX candidate` and every one of them released between 37.7% and 85.7%
of its file. Four negative and false-positive controls were correctly left alone.

The matrix has since been run **six times in total across two days**, and every run
produced identical sizes, identical reclaimed bytes and identical recommendations on all
13 cells. The only quantity that moves at all is the inflation figure itself, on the
four cells whose population term is an `ANALYZE` sample, and by at most 2.6%.

Four findings changed the design away from what the prompt proposed:

1. **`pg_class.reltuples` on an index means three different things** depending on
   which command last wrote it, and for GIN and BRIN the differences are 2x and
   10,000x. The baseline is therefore only valid if it is captured **after
   `ANALYZE`**, and the evaluation is gated on an `ANALYZE` having happened since.
2. **There is no per-table statistics reset timestamp in v17.** `PgStat_StatTabEntry`
   has no such field. Reset detection has to be a monotonicity check on the stored
   counters, which is why the raw counters — not just their deltas — must be in the
   baseline.
3. **`brin_desummarize_range()` + `brin_summarize_range()` is not a space
   remedy.** Measured, it grew a churned `minmax_multi` index from 114,688 to
   196,608 bytes — 71% *bigger* — where `REINDEX` returned it to 114,688. The
   prompt's hypothesis is not supported, and the BRIN arm recommends against it.
4. **Neither BRIN cell produced any reclaimable space at all** (0 bytes on both),
   so the BRIN arm of the heuristic is unvalidated against a true positive and is
   deliberately the most conservative.

The one place the model is materially wrong is `c01_hash_dup`, where it predicted
57.4% reclaimable and `REINDEX` returned 37.7% — a +19.7 point over-estimate. That
cell is the one where the *key distribution* changed, which is exactly the
assumption the multiplicative model makes and cannot check.

### What the 2026-08-25 review changed

The page was re-reviewed against the same pin and the same isolated 17.11 cluster,
restarted from `.wiki-runtime/tmp/idxm/` with all 13 fixtures and all four recorded
runs intact. The 57 distinct source ranges the page cited before this review were all
re-read in `raw/postgres-17/`, every filed size was checked against the harness's own
CSVs, and every claim whose recorded evidence was missing or stale was re-run. Six
things changed.

1. **Three figures were run-3 values sitting in a run-4 table.** The results table is
   run 4, but the prose quoted `pf_shift` 9.024, the derived 9.02x, `size_inflation`
   0.180 and `c10_brin_minmax`'s churn ratio 5.468 — all run 3. They now read 9.124,
   9.12x, 0.178 and 5.418. No verdict moves; this is the 1-2% `ANALYZE`-sample drift
   the page already documents, showing up as an internal inconsistency.
2. **The `reltuples` table's third column had no recorded evidence.** The probe that
   produced it ran `DELETE ...; VACUUM ...` inside one `psql -c`, which fails with
   `VACUUM cannot run inside a transaction block` and rolls the `DELETE` back, so the
   recorded output repeats the post-`ANALYZE` column. Re-run with the two statements
   separated, the filed column is **confirmed exactly**: 180000 for hash, GIN, GiST and
   SP-GiST, and 22 for BRIN.
3. **The BRIN build-time cell is a parallel-build artifact, and the stated mechanism
   was wrong.** The merge does not "increment per merged range" — it unions duplicates
   with `brin_doinsert` and never recounts. The count is the number of
   (participant, range) pairs. Building the same index on the same table three ways now
   reads 23 serial, 44-45 default, 102 with four workers, against 23 real summary
   tuples. The table cell and the mechanism paragraph were both corrected.
4. **The statistics-reset output block had never been produced by the filed
   statement.** The recorded probe predates the final text and printed the fabricated
   `-200000.000` with a different recommendation. Re-run today with the statement
   exactly as filed, the block **reproduces character for character**.
5. **`brin_summarize_range()` was never actually run.** The filed table measured
   `brin_summarize_new_values()` while the prose, and the prompt, named
   `brin_summarize_range()`. Re-run with 318 explicit `brin_desummarize_range()` calls
   followed by 318 `brin_summarize_range()` calls, the result is **identical**: 114688
   -> 114688 -> 196608, with `REINDEX` returning it to 114688. The conclusion stands on
   the function pair the prompt asked about.
6. **A three-times-bloated B-tree index scored `none`.** The threshold table has no
   `btree` row, so the `LEFT JOIN` left every comparison NULL and the `CASE` fell
   through to its `ELSE` — a refusal that reads as a verdict. Measured on a B-tree at
   `size_inflation` 2.978 and `churn_ratio` 2.000. One arm was added,
   `WHEN t.amname IS NULL THEN 'unsupported access method'`, and it is proven inert on
   everything the page measures: run over all 19 indexes carrying a baseline on that
   cluster, the filed and amended texts differ on **exactly 2 rows, both `btree`**, and
   all 13 cells plus the four non-B-tree review fixtures are byte-identical.

Two further behaviours were measured for the first time and are filed as evidence, not
corrections: a failed concurrent index build leaves `indisvalid = false` and the
statement reports `skip: index not valid` (demonstrated on a B-tree fixture, which the
guard now pre-empts — see [The three-run re-execution](#the-three-run-re-execution) for
the same arm on a hash index); and an index with no baseline at all reports
`churn_state = unknown: counters reset`, which is a mislabel — there are no counters to
compare, and the `recommendation` of `capture new baseline` is right.

Everything else held. The three-run reproducibility claim is confirmed from the stored
CSVs — runs 2, 3 and 4 carry identical `(B, post-churn, C, R)` quadruples on all 13
cells, and run 1 does differ on exactly the two cells the page says were defective. All
13 published cell scripts match the scripts that ran, the 13 filed `size_inflation`
values match run 4's recorded output, and every kB figure in the results table is the
recorded byte count divided by 1024.

### The three-run re-execution

Every test on this page was then run again from scratch: the 13-cell matrix three more
times (**runs 5, 6 and 7**, 10m24s / 10m11s / 10m09s), the six original edge-case
probes, and the 13 review probes — all scored by the statement this page publishes,
extracted from the block above rather than from the harness's copy. The heuristic came
out of it unchanged, and three claims got sharper.

**The physical measurements are exactly reproducible.** All 13 `(B, post-churn, C, R)`
quadruples in runs 5, 6 and 7 are **byte-identical to filed run 4**, and therefore to
runs 2 and 3: six runs, two days, across a server restart. So are all 13
`reclaimed_bytes` and `reclaimed_pct` values. **All 13 recommendations are identical in
all three runs**, which is also the direct proof that the new access-method guard is
inert on matrix data: runs 5-7 were scored by the guarded text and run 4 by the
unguarded one.

**The `ANALYZE` gate held in all three runs.** Every one of the 39 pre-`VACUUM`
evaluations returned `inconclusive: no ANALYZE since baseline`, and `c05_gin_pending`'s
pre-`VACUUM` inflation was **1.372 in all three runs**, the filed value to three
decimals.

**Only four cells' inflation figures move at all.** Across all six runs, nine of the 13
cells report a bit-identical `size_inflation`; the four that drift are exactly the cells
whose population term is an `ANALYZE` sample rather than an exact count:

| Cell | AM | run 2 | run 3 | run 4 | run 5 | run 6 | run 7 | spread |
|---|---|---|---|---|---|---|---|---|
| `c00_control` | hash | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 0.000 |
| `c01_hash_dup` | hash | 2.347 | 2.347 | 2.347 | 2.347 | 2.347 | 2.347 | 0.000 |
| `c02_hash_highwater` | hash | 5.063 | 5.063 | 5.063 | 5.063 | 5.063 | 5.063 | 0.000 |
| `c03_gin_common` | gin | 1.171 | 1.171 | 1.171 | 1.171 | 1.171 | 1.171 | 0.000 |
| `c04_gin_keychurn` | gin | 6.892 | 6.922 | 6.998 | 6.877 | 6.952 | 6.962 | **0.121** |
| `c05_gin_pending` | gin | 1.247 | 1.247 | 1.247 | 1.247 | 1.247 | 1.247 | 0.000 |
| `c06_gist_range` | gist | 4.861 | 4.801 | 4.907 | 4.812 | 4.812 | 4.833 | **0.106** |
| `c07_gist_highwater` | gist | 3.706 | 3.706 | 3.706 | 3.706 | 3.706 | 3.706 | 0.000 |
| `c08_spgist_prefix` | spgist | 7.299 | 7.228 | 7.405 | 7.419 | 7.258 | 7.295 | **0.191** |
| `c09_spgist_highwater` | spgist | 3.971 | 3.971 | 3.971 | 3.971 | 3.971 | 3.971 | 0.000 |
| `c10_brin_minmax` | brin | 0.209 | 0.209 | 0.209 | 0.209 | 0.209 | 0.209 | 0.000 |
| `c11_brin_mmmulti` | brin | 0.690 | 0.690 | 0.690 | 0.690 | 0.690 | 0.690 | 0.000 |
| `c12_partial` | hash | 0.178 | 0.180 | 0.178 | 0.181 | 0.178 | 0.179 | 0.003 |

Worst relative drift is 2.6% (`c08_spgist_prefix`, 7.228 to 7.419) and no cell comes
within 2% of its own threshold, so no recommendation can turn on it. The four movers are
the two repeated-replacement cells and the two whose row count the sample must estimate;
the stable nine either have an exactly-sampled population or a BRIN page-count
population, which is not sampled at all.

**Every probe reproduced, one of them including its own bug.** The six original probes
were re-run through their original script:

| Probe | Filed | Re-run |
|---|---|---|
| P1 comment size | 309 bytes / 212 payload | **340 / 243**, the `dbr` form; the difference is exactly 31 bytes both times, which is the length of the `"dbr": "...", ` key, so the filed figure is the same capture without it |
| P2 filenode and OID | 16649/16649 -> 16649/16650 -> 16651/16651 | same shape at 17275/17275 -> 17275/17276 -> 17277/17277; eval `valid` / 2.179 / `strong REINDEX candidate`, then `rebuilt since baseline` / 0.991 / `capture new baseline` twice — identical |
| P3 statistics reset | 3.167, churn `NULL`, `weak: inflated, churn unknown` | **identical**, with `d_ins` at `-200000` |
| P4 GIN pending list | 491 pages / 16,654,336 -> 0 / 21,905,408 | **byte-identical** |
| P5 `reltuples` progression | 43 / 200000 / 22 for BRIN | **reproduced the defect**: `ERROR: VACUUM cannot run inside a transaction block`, third column equal to the second |
| P6 parallel BRIN build | 348 against a true 70 | **346** against a true 70; serial 70 both times |

P5 is the useful one: running the original probe unchanged reproduces the aborted
transaction, which is direct evidence that the filed third column cannot have come from
it. The corrected probe re-run beside it returns **43 / 200000 / 22 for BRIN and 180000
for the other four**, matching the filed table exactly, 43 included.

**Three review findings were re-confirmed and one was corrected.** The BRIN
desummarize/summarize sequence reproduced byte for byte (114688 -> 114688 -> 196608,
`REINDEX` back to 114688). Two captures in the same second were identical and one two
seconds later differed only in `ts`. The bloated B-tree that scored `none` before the
guard now scores `unsupported access method`. The correction is the invalid-index arm:
the first review demonstrated it on a *B-tree* fixture, and with the guard in place that
same fixture now reads `unsupported access method`, because the guard is the first arm.
Re-tested on a target AM — an invalid **hash** index left behind by a
`CREATE INDEX CONCURRENTLY` whose expression divided by zero — the arm fires as designed
and reports `skip: index not valid`.

### Why REINDEX is the only thing that shrinks these five access methods

The heuristic is worth building because for all five target access methods, `VACUUM`
never returns a block to the operating system. The only truncation code in any index
AM is disabled: `spgvacuumscan` carries a `RelationTruncate` call inside
`#ifdef NOT_USED`, with a comment saying it is unsafe due to concurrent inserts and
that "btree doesn't do this either"
([spgvacuum.c#spgvacuumscan](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L877-L900)).
What `VACUUM` does instead is record freed blocks in the index's own free space map,
which is only an allocator hint
([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)).
`REINDEX` is different because it allocates a new relfilenumber and builds into it
([index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3782-L3789)).

Per access method, the reason a rebuild is smaller:

| AM | Why the file grows | Why VACUUM cannot give it back |
|---|---|---|
| hash | Overflow pages per bucket, and whole splitpoint batches of bucket pages allocated at once ([hashpage.c#_hash_alloc_buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L967-L1037)) | Freed overflow pages only clear a bit in the index's own bitmap ([hashovfl.c#_hash_freeovflpage](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L632-L642)); the README states there is no provision to shrink other than REINDEX ([hash/README](../../../../raw/postgres-17/src/backend/access/hash/README#L31-L34)) |
| gin | Posting lists outgrow the leaf and become posting trees; pending pages | VACUUM never deletes tuples or pages from the entry tree ([gin/README](../../../../raw/postgres-17/src/backend/access/gin/README#L390-L396)), and leaf vacuum deliberately does not re-encode segments — "You'll have to REINDEX anyway" ([gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L797-L813)) |
| gist | Page splits | Only *completely empty* leaf pages are ever deleted ([gistvacuum.c#gistvacuumpage](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L388-L403)), and the last downlink is never removed ([gistvacuum.c#gistvacuum_delete_empty_pages](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L541-L548)) |
| spgist | Deleted leaf tuples become placeholders, not free space | Placeholders are only physically removed when they form a contiguous run at the *end* of the page ([spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L569-L590)) |
| brin | A summary that no longer fits in place is written elsewhere and the old line pointer is orphaned ([brin_pageops.c#brin_doupdate](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L246-L262)) | `PageIndexTupleDeleteNoCompact` sets the line pointer unused but never sets `PD_HAS_FREE_LINES` ([bufpage.c#PageIndexTupleDeleteNoCompact](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L1333-L1347)), which is the only thing that makes `PageAddItem` recycle a slot ([bufpage.c#PageAddItemExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L249-L258)); and `brinbulkdelete` is a no-op ([brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301)) |

A fresh build is also denser than incremental growth for independent reasons: a hash
rebuild sizes its bucket count from the current `reltuples`
([hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L133-L137)),
and a sorted GiST build packs pages completely, explicitly ignoring fillfactor
([gistbuild.c#gist_indexsortbuild_levelstate_add](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L464-L470)).

### Why a current-over-baseline size ratio is the wrong question

Because it cannot separate a table that grew from an index that rotted. The measured
matrix contains a cell that makes the point without ambiguity: `c12_partial` grew its
index from 40,976 kB to 66,688 kB, a raw ratio of 1.63, and a `REINDEX` reclaimed
**2.5%**. The predicate-selected population had grown 9.00x — 200,000 rows of 2,000,000
to 1,800,000 — so nearly all of the growth was legitimate. The population-normalized
reading for the same cell is 0.178 to 0.181 across six runs, correctly below 1 in every
one of them.

### Three catalog facts the design depends on

#### COMMENT ON INDEX survives both REINDEX forms

This is what makes a comment-resident baseline viable at all. Plain `REINDEX` keeps
the index's pg_class row and only swaps the relfilenumber
([index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3782-L3789)),
so the `pg_description` row, which is keyed on the index OID, is untouched.
`REINDEX ... CONCURRENTLY` builds a *new* index with a *new OID* and explicitly moves
the comment across in a block commented "Move comment if any"
([index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784)).
The in-tree regression test asserts both
([create_index.out#testcomment](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2300-L2324)).

Measured on the pinned build, showing that the OID moves under CONCURRENTLY while the
comment follows it:

```text
step                                     idx_oid   filenode
oid/filenode before                        16649      16649
oid/filenode after plain REINDEX           16649      16650
oid/filenode after REINDEX CONCURRENTLY    16651      16651
```

The consequence for the design is a rule: **never cache the index OID; re-resolve it
by name.** The comment itself is `text` in a TOAST-enabled catalog
([pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L57))
with no length check in `CreateComments`
([comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L142-L226)),
and the measured payload is 212 bytes in a 309-byte comment, or 243 in 340 once `dbr`
is present, so the row stays inline either way.
`COMMENT ON` takes `ShareUpdateExclusiveLock` and requires ownership
([comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L77)).
`obj_description(oid, 'pg_class')` is the correct reader; it filters `objsubid = 0`
([system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301)).

#### An index reltuples means three different things

This is the finding that most shaped the design. Three different code paths write an
index's `pg_class.reltuples`, and they do not agree:

- `CREATE INDEX` / `REINDEX` write the AM's own `stats->index_tuples`
  ([index.c#index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135)).
- `ANALYZE` overwrites it with `ceil(tupleFract * totalrows)`, and `tupleFract` is
  initialized to `1.0`
  ([analyze.c:449](../../../../raw/postgres-17/src/backend/commands/analyze.c#L449))
  and only revised for expression or partial indexes, because `compute_index_stats`
  skips everything else
  ([analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L861-L863)).
  So for a plain index it becomes a copy of the *table's* row estimate
  ([analyze.c#do_analyze_rel](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663)).
- `VACUUM` writes the AM's `num_index_tuples`, but only when the count is exact
  ([vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3086-L3097)).
  For GIN that is deliberately the *heap* tuple count
  ([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L735-L739));
  for BRIN it is the number of summarized ranges
  ([brin.c#brinvacuumcleanup](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1324-L1327)).

Measured on the pin, one 200,000-row table (2,857 heap pages, so 23 BRIN ranges at the
default `pages_per_range`), then a 10% delete and `VACUUM`:

| AM | after CREATE INDEX | after ANALYZE | after DELETE 10% + VACUUM |
|---|---|---|---|
| hash | 200000 | 200000 | 180000 |
| gin | **300000** (entries) | **200000** (rows) | 180000 (rows) |
| gist | 200000 | 200000 | 180000 |
| spgist | 200000 | 200000 | 180000 |
| brin | **43** (parallel-build artifact; see below) | **200000** (rows) | **22** (ranges) |

A baseline captured before `ANALYZE` and evaluated after one would therefore show a
GIN index as 1.5x inflated and a BRIN index as 4,651x deflated on no physical change
at all. Two rules follow, and both are enforced: capture the baseline **after
`ANALYZE`**, and refuse to score when no `ANALYZE` has happened since the baseline.

BRIN's build-time number is worse than AM-specific: it is not even a property of the
finished index, because a parallel build counts *partial* summaries. Each participant
scans its own chunk of the heap and spills one summary per range it touched, bumping
its private `bs_numtuples`
([brin.c#form_and_spill_tuple](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1996-L2015)),
then adds that private count into the shared total
([brin.c#_brin_parallel_scan_and_build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2837-L2841)).
The leader copies the shared total into its own `bs_numtuples`
([brin.c#_brin_parallel_heapscan](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2568-L2596))
and then unions the duplicate per-range summaries into one index tuple each with
`brin_doinsert`, which does not recount
([brin.c#_brin_parallel_merge](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2688-L2720)).
So the number reported is the count of (participant, range) pairs. Overlap is the norm
rather than the exception, because the parallel scan sizes its chunk from the *table*
(`nblocks / 2048`, next power of two, capped at 8192 blocks), which for any table under
about 262,000 pages is smaller than a 128-page BRIN range
([tableam.c#table_block_parallelscan_startblock_init](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L434-L451)).

Measured on the 2,857-page table above, whose index holds 23 summary tuples by
`brin_page_items`, building the same index three ways:

| Build | `pg_class.reltuples`, first review | re-run next day |
|---|---|---|
| `max_parallel_maintenance_workers = 0` | **23** (correct) | **23** |
| default (one worker plus the leader) | **44**, then 45 / 45 / 45 on repeats | **44**, then 46 / 45 / 44 |
| `max_parallel_maintenance_workers = 4`, `min_parallel_table_scan_size = 0` | **102** | **90** |

Only the serial number is deterministic; the parallel ones depend on how the scan
happens to hand pages out, and eight observations of the default build span 43 to 46.
The filed 43 above is one of them, reproduced exactly on the re-run, which is why the
table flags it as an artifact rather than presenting it as the AM's own entry count. On
a 2,000,000-row table (8,850 pages, 70 true ranges) a serial build wrote
`reltuples = 70` both times, while a 4-worker build wrote **348** and then **346** —
about five participants times seventy ranges.

#### There is no per-table statistics reset timestamp

`PgStat_StatTabEntry` has no reset field
([pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-17/src/include/pgstat.h#L399-L429)),
and the relation stats kind registers no `reset_timestamp_cb`
([pgstat.c#pgstat_kind_builtin_infos](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L278-L290)).
`pg_stat_reset_single_table_counters()` does bump the *database* timestamp
([pgstat.c#pgstat_reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L734-L757)),
but that cannot tell you *which* table was reset. Measured: before the call
`pg_stat_database.stats_reset` was NULL, after it was
`2026-08-24 16:19:17.78836-04`, and the table's `n_tup_ins`, `n_tup_upd`,
`n_tup_del` and `vacuum_count` were all 0.

The only positive proof is therefore that a stored counter went backwards, which is
why the baseline stores the raw counters.

### The baseline field set

19 fields. Against the prompt's suggested list, `n_live_tup` and `n_dead_tup` were
**removed** and four fields were **added**.

| Key | Meaning | Why it is needed |
|---|---|---|
| `v` | payload version | lets the reader reject a format it does not understand |
| `ts` | baseline timestamp, UTC | reporting and baseline age |
| `am` | access method name | selects the normalization model; also detects an AM change |
| `fn` | `pg_relation_filenode()` | baseline validity; detects any rebuild |
| `isz` | `pg_relation_size()` | the B term of the model |
| `ipg` | index `relpages` | diagnostic cross-check on `isz` |
| `itup` | index `reltuples` | logical population for hash/GiST/SP-GiST; partial-fraction numerator |
| `tpg` | table `relpages` | BRIN logical ranges |
| `ttup` | table `reltuples` | partial-fraction denominator |
| `ppr` | `pages_per_range` (BRIN only) | BRIN range arithmetic; detects a reloption change |
| `iw` | summed `pg_stats.avg_width` of the key columns (GIN only) | GIN input-mass normalization |
| `ins`, `upd`, `hot`, `del` | raw cumulative tuple counters | churn deltas, **and** the monotonicity reset check |
| `vac`, `avac` | raw `vacuum_count`, `autovacuum_count` | "did VACUUM happen", and the reset check |
| `anl` | `analyze_count + autoanalyze_count` | gates the whole reading, per the `reltuples` finding above |
| `dbr` | `pg_stat_database.stats_reset` | secondary reset warning |

Removed, with reasons: **`live`** is redundant against `ttup` for every use the model
has, and **`dead`** has no decision power at baseline — the index was just built, so
its physical state is fresh regardless of how many dead heap tuples existed. The dead
tuple ratio that the heuristic actually gates on is a *current* reading. Added
against the prompt's list: `ppr`, `iw`, `anl`, and `dbr`.

`ppr` and `iw` are emitted only for the access method that uses them;
`jsonb_strip_nulls` drops them elsewhere.

### The comment format

```text
Search index used by the application.
Second human line with an @ sign and a } brace.

@idxmaint:{"v": 1, "am": "hash", "fn": "16643", "ts": "2026-08-24T20:19:15Z", "anl": 1, "del": 0, "hot": 0, "ins": 50000, "ipg": 258, "isz": 2113536, "tpg": 271, "upd": 0, "vac": 0, "avac": 0, "itup": 50000, "ttup": 50000}
```

That is a literal capture from the pinned build: 309 bytes total, 212 of them payload.
Everything before `@idxmaint:` is human text and is preserved verbatim, including an
`@` and a `}`. The payload is a single line so the marker regex is unambiguous.

Two qualifications on that sample, both measured:

- **`dbr` is absent only because that database had never had its statistics reset.**
  `pg_stat_database.stats_reset` was still NULL, so `jsonb_strip_nulls` dropped the key.
  Re-capturing the identical human comment on the same cluster after a reset produced
  **340 bytes total, 243 of them payload**, twice on two different days. Both deltas are
  exactly 31 bytes, which is the length of `"dbr": "2026-08-25T15:46:03Z", `, so the two
  captures are the same payload with and without that one key. Budget for the larger
  form; a 19-field payload is still far below any TOAST threshold.
- **Two captures are byte-identical only within the same clock second.** `jsonb` key
  ordering is canonical, so nothing moves *except* `ts` — but `ts` has one-second
  resolution and does move. Measured: two captures in the same second gave one md5,
  and a third two seconds later differed, in `ts` and in nothing else.

### SQL: capture a fresh baseline

Run this after `CREATE INDEX`, `REINDEX INDEX`, or `REINDEX INDEX CONCURRENTLY`, and
**after an `ANALYZE` of the table**. It creates no persistent object other than the
comment. `COMMENT ON` takes `ShareUpdateExclusiveLock`
([comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L77)),
so bound the wait:

```sql
SET /* wiki_idxmaint_guard */ statement_timeout = '30s';
SET /* wiki_idxmaint_guard */ lock_timeout = '5s';

DO /* wiki_idxmaint_capture_baseline */ $idxmaint$
DECLARE
    target    regclass := 'public.my_index'::regclass;
    old_text  text;
    human     text;
    payload   jsonb;
    new_text  text;
BEGIN
    SELECT obj_description(target, 'pg_class') INTO old_text;

    -- Preserve everything before the marker; drop only the old payload.
    human := rtrim(regexp_replace(coalesce(old_text, ''), '@idxmaint:.*$', ''), E' \t\r\n');

    SELECT jsonb_strip_nulls(jsonb_build_object(
             'v',    1,
             'ts',   to_char(now() AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"'),
             'am',   am.amname,
             'fn',   pg_relation_filenode(i.indexrelid),
             'isz',  pg_relation_size(i.indexrelid),
             'ipg',  ic.relpages,
             'itup', ic.reltuples::bigint,
             'tpg',  tc.relpages,
             'ttup', tc.reltuples::bigint,
             'ppr',  CASE WHEN am.amname = 'brin' THEN coalesce(
                            (SELECT o.option_value::int
                               FROM pg_options_to_table(ic.reloptions) o
                              WHERE o.option_name = 'pages_per_range'), 128)
                     END,
             'iw',   CASE WHEN am.amname = 'gin' THEN (
                            SELECT sum(s.avg_width)::int
                              FROM unnest(i.indkey::int2[]) WITH ORDINALITY k(attnum, ord)
                              JOIN pg_attribute a
                                ON a.attrelid = i.indrelid AND a.attnum = k.attnum
                              JOIN pg_stats s
                                ON s.schemaname = tn.nspname
                               AND s.tablename  = tc.relname
                               AND s.attname    = a.attname
                             WHERE k.ord <= i.indnkeyatts AND k.attnum <> 0)
                     END,
             'ins',  st.n_tup_ins,
             'upd',  st.n_tup_upd,
             'hot',  st.n_tup_hot_upd,
             'del',  st.n_tup_del,
             'vac',  st.vacuum_count,
             'avac', st.autovacuum_count,
             'anl',  st.analyze_count + st.autoanalyze_count,
             'dbr',  to_char(d.stats_reset AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"')
           ))
      INTO payload
      FROM pg_index i
      JOIN pg_class ic ON ic.oid = i.indexrelid
      JOIN pg_class tc ON tc.oid = i.indrelid
      JOIN pg_namespace tn ON tn.oid = tc.relnamespace
      JOIN pg_am am ON am.oid = ic.relam
      JOIN pg_stat_all_tables st ON st.relid = i.indrelid
      CROSS JOIN (SELECT stats_reset FROM pg_stat_database
                   WHERE datname = current_database()) d
     WHERE i.indexrelid = target;

    IF payload IS NULL THEN
        RAISE EXCEPTION 'no catalog row for index %', target;
    END IF;

    new_text := CASE WHEN human = '' THEN '' ELSE human || E'\n\n' END
                || '@idxmaint:' || payload::text;

    EXECUTE format('COMMENT ON INDEX %s IS %L', target::text, new_text);
END
$idxmaint$;
```

`128` is the correct fallback for a BRIN index with no explicit reloption
([brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-17/src/include/access/brin.h#L39-L45)).
`pg_relation_size(regclass)` is the main fork only
([system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)),
which is the same set of blocks `relpages` counts.

### SQL: retrieve and parse the baseline

```sql
SELECT /* wiki_idxmaint_read_baseline */
       c.oid::regclass                                                     AS index_name,
       rtrim(regexp_replace(coalesce(d.description, ''), '@idxmaint:.*$', ''),
             E' \t\r\n')                                                   AS human_comment,
       substring(d.description from '@idxmaint:(.*)$')                     AS payload_text,
       substring(d.description from '@idxmaint:(.*)$')::jsonb              AS payload,
       (substring(d.description from '@idxmaint:(.*)$')::jsonb ->> 'ts')   AS base_ts,
       (substring(d.description from '@idxmaint:(.*)$')::jsonb ->> 'am')   AS base_am,
       (substring(d.description from '@idxmaint:(.*)$')::jsonb ->> 'fn')::oid    AS base_filenode,
       (substring(d.description from '@idxmaint:(.*)$')::jsonb ->> 'isz')::bigint AS base_index_bytes
  FROM pg_class c
  LEFT JOIN pg_description d
         ON d.objoid = c.oid
        AND d.classoid = 'pg_class'::regclass
        AND d.objsubid = 0
 WHERE c.oid = 'public.my_index'::regclass;
```

To list every non-B-tree index that carries a baseline, swap the `WHERE` clause for
`c.relkind = 'i' AND d.description LIKE '%@idxmaint:%'` joined to
`pg_am am ON am.oid = c.relam AND am.amname <> 'btree'`. The six built-in index AM
names are fixed in
[pg_am.dat](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L18-L35).

### SQL: evaluate

One statement. It computes current metadata, the normalized inflation, all three
churn deltas, the churn and dead-tuple ratios, whether `VACUUM` occurred, and the
recommendation. It creates nothing.

```sql
SET /* wiki_idxmaint_guard */ statement_timeout = '30s';
SET /* wiki_idxmaint_guard */ lock_timeout = '5s';

WITH /* wiki_idxmaint_evaluate */ params AS (
    SELECT 'public.my_index'::regclass AS idx
),
raw AS (
    SELECT p.idx,
           substring(obj_description(p.idx, 'pg_class') from '@idxmaint:(.*)$') AS payload
      FROM params p
),
b AS (
    SELECT idx,
           CASE WHEN payload IS NULL THEN NULL
                WHEN payload ~ '^\s*\{' THEN payload::jsonb
           END AS j
      FROM raw
),
cur AS (
    SELECT i.indexrelid,
           i.indrelid,
           i.indpred IS NOT NULL                       AS is_partial,
           i.indisvalid,
           i.indislive,
           am.amname,
           ic.relpages::bigint                         AS ipg,
           ic.reltuples::numeric                       AS itup,
           tc.relpages::bigint                         AS tpg,
           tc.reltuples::numeric                       AS ttup,
           pg_relation_size(i.indexrelid)              AS isz,
           pg_relation_filenode(i.indexrelid)          AS fn,
           coalesce((SELECT o.option_value::int
                       FROM pg_options_to_table(ic.reloptions) o
                      WHERE o.option_name = 'pages_per_range'), 128) AS ppr,
           (SELECT sum(s.avg_width)::numeric
              FROM unnest(i.indkey::int2[]) WITH ORDINALITY k(attnum, ord)
              JOIN pg_attribute a ON a.attrelid = i.indrelid AND a.attnum = k.attnum
              JOIN pg_stats s ON s.schemaname = tn.nspname
                             AND s.tablename  = tc.relname
                             AND s.attname    = a.attname
             WHERE k.ord <= i.indnkeyatts AND k.attnum <> 0)          AS iw
      FROM pg_index i
      JOIN pg_class ic ON ic.oid = i.indexrelid
      JOIN pg_class tc ON tc.oid = i.indrelid
      JOIN pg_namespace tn ON tn.oid = tc.relnamespace
      JOIN pg_am am ON am.oid = ic.relam
     WHERE i.indexrelid = (SELECT idx FROM params)
),
st AS (
    SELECT * FROM pg_stat_all_tables WHERE relid = (SELECT indrelid FROM cur)
),
db AS (
    SELECT stats_reset FROM pg_stat_database WHERE datname = current_database()
),
m AS (
    SELECT
      cur.*,
      b.j,
      st.n_tup_ins, st.n_tup_upd, st.n_tup_hot_upd, st.n_tup_del,
      st.n_live_tup, st.n_dead_tup,
      st.vacuum_count, st.autovacuum_count,
      st.analyze_count + st.autoanalyze_count AS anl,
      db.stats_reset,
      -- baseline validity
      CASE
        WHEN b.j IS NULL                                   THEN 'no baseline'
        WHEN (b.j->>'v') IS DISTINCT FROM '1'              THEN 'unsupported payload version'
        WHEN NOT cur.indislive                             THEN 'index not live'
        WHEN cur.fn IS NULL                                THEN 'no storage'
        WHEN (b.j->>'am') IS DISTINCT FROM cur.amname      THEN 'access method changed'
        WHEN (b.j->>'fn')::oid IS DISTINCT FROM cur.fn     THEN 'rebuilt since baseline'
        ELSE 'valid'
      END AS baseline_state,
      -- churn deltas
      st.n_tup_ins     - (b.j->>'ins')::bigint  AS d_ins,
      st.n_tup_upd     - (b.j->>'upd')::bigint  AS d_upd,
      st.n_tup_hot_upd - (b.j->>'hot')::bigint  AS d_hot,
      st.n_tup_del     - (b.j->>'del')::bigint  AS d_del,
      st.vacuum_count     - (b.j->>'vac')::bigint  AS d_vac,
      st.autovacuum_count - (b.j->>'avac')::bigint AS d_avac,
      st.analyze_count + st.autoanalyze_count - (b.j->>'anl')::bigint AS d_anl,
      -- logical population, per access method
      CASE cur.amname
        WHEN 'brin' THEN ceil(GREATEST(cur.tpg, 0)::numeric / cur.ppr)
        WHEN 'gin'  THEN GREATEST(cur.itup, 0) * COALESCE(cur.iw, 1)
        ELSE             GREATEST(cur.itup, 0)
      END AS cur_pop,
      CASE (b.j->>'am')
        WHEN 'brin' THEN ceil(GREATEST((b.j->>'tpg')::numeric, 0) / (b.j->>'ppr')::numeric)
        WHEN 'gin'  THEN GREATEST((b.j->>'itup')::numeric, 0) * COALESCE((b.j->>'iw')::numeric, 1)
        ELSE             GREATEST((b.j->>'itup')::numeric, 0)
      END AS base_pop,
      (b.j->>'isz')::numeric AS base_isz,
      (b.j->>'ts')           AS base_ts
      FROM cur, b, st, db
),
d AS (
    SELECT m.*,
      -- statistics-reset detection: no per-table reset timestamp exists in v17,
      -- so a counter going backwards is the only positive proof.
      (d_ins < 0 OR d_upd < 0 OR d_hot < 0 OR d_del < 0 OR d_vac < 0 OR d_avac < 0)
        AS counters_went_backwards,
      (stats_reset IS DISTINCT FROM (j->>'dbr')::timestamptz) AS db_stats_reset_moved,
      CASE WHEN base_pop > 0 THEN base_isz * cur_pop / base_pop END AS expected_fresh_size,
      n_dead_tup::numeric / GREATEST(n_live_tup, 1) AS dead_ratio,
      CASE WHEN (j->>'ttup')::numeric > 0 AND ttup > 0
           THEN ((itup / ttup) / NULLIF((j->>'itup')::numeric / (j->>'ttup')::numeric, 0))
      END AS partial_fraction_shift
      FROM m
),
e AS (
    SELECT d.*,
      CASE WHEN expected_fresh_size > 0 THEN isz / expected_fresh_size END AS size_inflation,
      (d_upd - d_hot) AS d_nonhot_upd,
      (d_ins + (d_upd - d_hot) + d_del) AS churn,
      (d_vac + d_avac) > 0 AS vacuum_since_baseline
      FROM d
),
f AS (
    SELECT e.*,
      churn::numeric / GREATEST(n_live_tup, 1) AS churn_ratio,
      NOT counters_went_backwards AS churn_known
      FROM e
),
-- Experimental, per-access-method thresholds.  BRIN is deliberately the most
-- conservative: measured desummarize+summarize GREW a churned minmax_multi
-- index by 71%, and BRIN's size tracks table page count, not row churn.
-- This table is also the access-method guard: an AM with no row here is not
-- scored at all, because every threshold comparison against it would be NULL.
t AS (
    SELECT * FROM (VALUES
        ('hash',   1.30, 0.50, 1.50, 1.00),
        ('gist',   1.40, 0.75, 1.80, 1.50),
        ('spgist', 1.40, 0.75, 1.80, 1.50),
        ('gin',    1.50, 1.00, 2.00, 2.00),
        ('brin',   2.00, 1.00, 3.00, 2.00)
    ) AS v(amname, infl_cand, churn_cand, infl_strong, churn_strong)
)
SELECT
    f.indexrelid::regclass                       AS index_name,
    f.amname,
    f.is_partial,
    f.baseline_state,
    f.base_ts,
    pg_size_pretty(f.base_isz::bigint)           AS base_size,
    pg_size_pretty(f.isz)                        AS cur_size,
    pg_size_pretty(f.expected_fresh_size::bigint) AS expected_fresh,
    round(f.size_inflation, 3)                   AS size_inflation,
    CASE f.amname WHEN 'brin' THEN 'summarized ranges'
                  WHEN 'gin'  THEN 'tuples x avg_width'
                  ELSE 'indexed tuples' END      AS pop_unit,
    round(f.base_pop, 0)                         AS base_pop,
    round(f.cur_pop, 0)                          AS cur_pop,
    f.d_ins, f.d_nonhot_upd, f.d_del,
    -- never publish a churn number derived from counters that went backwards
    CASE WHEN f.churn_known THEN round(f.churn_ratio, 3) END AS churn_ratio,
    CASE WHEN f.churn_known THEN 'known'
         ELSE 'unknown: counters reset' END      AS churn_state,
    CASE WHEN f.churn_known THEN f.vacuum_since_baseline END AS vacuum_since_baseline,
    CASE WHEN f.churn_known THEN f.d_vac + f.d_avac END      AS vacuums_since_baseline,
    round(f.dead_ratio, 4)                       AS dead_ratio,
    round(f.partial_fraction_shift, 3)           AS pf_shift,
    CASE
      -- B-tree and any other AM the model was never calibrated for.  Without
      -- this arm the LEFT JOIN leaves every threshold NULL and the CASE falls
      -- through to 'none', which reads as a verdict rather than a refusal.
      WHEN t.amname IS NULL                                THEN 'unsupported access method'
      WHEN f.baseline_state <> 'valid'                     THEN 'capture new baseline'
      WHEN NOT f.indisvalid                                THEN 'skip: index not valid'
      WHEN f.d_anl = 0                                     THEN 'inconclusive: no ANALYZE since baseline'
      WHEN f.expected_fresh_size IS NULL                   THEN 'inconclusive: no baseline population'
      WHEN f.is_partial AND (f.partial_fraction_shift < 0.7
                          OR f.partial_fraction_shift > 1.43)
                                                           THEN 'suppressed: predicate population moved'
      -- A reset zeroes vacuum_count too, so the VACUUM and dead-tuple gates
      -- below cannot be evaluated.  Report the size reading at low confidence
      -- rather than inventing a churn number.
      WHEN NOT f.churn_known AND f.size_inflation >= t.infl_cand
                                                           THEN 'weak: inflated, churn unknown'
      WHEN NOT f.churn_known                               THEN 'none (churn unknown)'
      WHEN NOT f.vacuum_since_baseline                     THEN 'inconclusive: no VACUUM since baseline'
      WHEN f.dead_ratio > 0.20                             THEN 'inconclusive: dead tuples not yet reclaimed'
      -- Nothing worth rebuilding.  BRIN indexes are normally a handful of
      -- pages, and the multiplicative model has no intercept, so a fixed
      -- metapage + revmap prefix dominates the ratio at that size.
      WHEN f.isz < 1048576                                 THEN 'none: index below 1 MB'
      WHEN f.size_inflation >= t.infl_strong
       AND f.churn_ratio >= t.churn_strong                  THEN 'strong REINDEX candidate'
      WHEN f.size_inflation >= t.infl_cand
       AND f.churn_ratio >= t.churn_cand                    THEN 'REINDEX candidate'
      ELSE 'none'
    END AS recommendation
  FROM f
  LEFT JOIN t ON t.amname = f.amname;
```

### Access-method-specific normalization

| AM | Logical population | Rationale |
|---|---|---|
| hash | index `reltuples` | one entry per heap tuple; bucket count is sized from `reltuples` at build ([hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L133-L137)) |
| gist | index `reltuples` | one entry per heap tuple |
| spgist | index `reltuples` | one entry per heap tuple |
| gin | index `reltuples` x summed `pg_stats.avg_width` | one heap row yields N entries; `ginarrayextract` returns one key per element ([ginarrayproc.c#ginarrayextract](../../../../raw/postgres-17/src/backend/access/gin/ginarrayproc.c#L32-L59)), so row count alone under-describes the input mass |
| brin | `ceil(table relpages / pages_per_range)` | the revmap is indexed by heap *block* ([brin_revmap.c#HEAPBLK_TO_REVMAP_BLK](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L35-L43)) and `brinbuildCallback` closes a range on block boundaries ([brin.c#brinbuildCallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L998-L1008)) |

Two deliberate deviations from the prompt:

- **`churn_ratio` is normalized by `n_live_tup`, not by the logical population.** For
  hash, GiST and SP-GiST those are the same number. For BRIN they are not: dividing
  by summarized ranges would report a single full-table update on a 2,000,000-row
  table as a churn ratio in the thousands. `n_live_tup` keeps the ratio meaning "how
  many times over did the row population turn over" for every AM.
- **GIN's `avg_width` is a post-TOAST stored width**
  ([pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L40-L50)),
  so for a TOASTed column it measures an 18-byte pointer, not the datum. It is a
  usable proxy only for inline values; see [Known limitations](#known-limitations).

### Handling the four required edge cases

**Manual REINDEX via filenode.** `baseline_state` compares the stored `fn` against
`pg_relation_filenode()`. Measured on the pin: with an intact baseline the index read
`valid` / inflation 2.179 / `strong REINDEX candidate`; after a manual
`REINDEX INDEX` it read `rebuilt since baseline` / `capture new baseline`; after a
further `REINDEX INDEX CONCURRENTLY`, which also changes the index OID, it again read
`rebuilt since baseline`. `RelationSetNewRelfilenumber` is what makes this reliable —
it writes the new relfilenumber and resets `relpages`/`reltuples` on the same pg_class
row
([relcache.c#RelationSetNewRelfilenumber](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3943-L3955)).

**Statistics resets.** Detected by monotonicity. On a 200,000-row hash fixture churned
by two full-table updates, the statement read `strong REINDEX candidate` at
`churn_ratio` 2.000; after `pg_stat_reset_single_table_counters()` on the same index,
with no physical change whatever, it read:

```text
size_inflation         | 3.167
churn_ratio            |
churn_state            | unknown: counters reset
vacuum_since_baseline  |
recommendation         | weak: inflated, churn unknown
```

The size reading survives, the churn number is `NULL` rather than the fabricated
`-200000.000` an earlier revision printed, and the recommendation drops to `weak`.
`d_ins` is what proves the reset: it reads `-200000`, and no other catalog fact
distinguishes the state, because there is no per-table reset timestamp.

**Partial indexes.** `pf_shift` compares the current index/table `reltuples` fraction
with the stored one, and a shift outside `[0.7, 1.43]` suppresses the recommendation.
This works because `ANALYZE` genuinely measures the predicate fraction for a partial
index — `compute_index_stats` sets `tupleFract` from the sampled rows that pass the
predicate
([analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)).
Measured in `c12_partial`: suppressed in all six runs, with `REINDEX` reclaiming 2.5%
every time. The true predicate population moves exactly 9.00x — 200,000 of 2,000,000
rows to 1,800,000 — and `pf_shift` read 9.152, 9.024, 9.124, 8.976, 9.148 and 9.074, so
the column carries the `ANALYZE` sample's error on top of the shift. Read it to two
significant figures; a suppression window as wide as `[0.7, 1.43]` is unaffected by a
2% error, which is the point of choosing a wide one.

**Human comments.** Preserved by replacing only the marker onwards. Measured across
capture, re-capture, `REINDEX`, and `REINDEX CONCURRENTLY`: a two-line human comment
containing both `@` and `}` survived all four unchanged. A second capture of an
unchanged index reproduces the payload key for key, and differs only in `ts` when the
two captures fall in different seconds.

### Test methodology

Every cell followed the prompt's protocol exactly: create a disposable table with
`autovacuum_enabled = off`, insert the baseline dataset, `CREATE INDEX`, `ANALYZE`,
record B, capture the baseline, run the AM-specific churn, evaluate **before** any
`VACUUM`, `VACUUM (ANALYZE)`, record C, evaluate, `REINDEX INDEX`, `ANALYZE`, record
R. `VACUUM FULL` was never used. Fixtures are ~1,000,000 rows (2,000,000 for the BRIN
and partial cells).

The pre-`VACUUM` evaluation is the false-positive gate in action. In all 13 cells it
returned `inconclusive: no ANALYZE since baseline`, including `c05_gin_pending`, whose
pre-`VACUUM` raw inflation was 1.372 purely from pending pages. Over the six runs that
is 78 pre-`VACUUM` evaluations and the gate held in every one, with `c05`'s 1.372
identical each time.

`c01_hash_dup` had to be bounded. A single hot key puts every duplicate in one bucket,
because the bucket is a pure function of the hash code
([hashutil.c#_hash_hashkey2bucket](../../../../raw/postgres-17/src/backend/access/hash/hashutil.c#L121-L135)),
and `_hash_doinsert` walks that bucket's overflow chain on every insert. Observed once
during the original design session, one hot key completed **1 row in 90 seconds** and
was abandoned; the filed cell uses 100 hot keys. That timing is the one measured aside
on this page that the re-runs did not reproduce, because reproducing it means waiting on
a deliberately quadratic workload.

### The 13 test cells

Each cell is a setup script and a churn script; the driver supplies the rest of the
protocol. Every table is created `WITH (autovacuum_enabled = off)`.

```sql
-- c00_control  (hash, negative control: no churn at all)
CREATE TABLE c00 (id bigint, k bigint) WITH (autovacuum_enabled = off);
INSERT INTO c00 SELECT g, g FROM generate_series(1,1000000) g;
CREATE INDEX c00_i ON c00 USING hash (k);
ANALYZE c00;
-- churn: none

-- c01_hash_dup  (hash, highly duplicated hot key)
CREATE TABLE c01 (id bigint, k bigint) WITH (autovacuum_enabled = off);
INSERT INTO c01 SELECT g, g FROM generate_series(1,1000000) g;
CREATE INDEX c01_i ON c01 USING hash (k);
ANALYZE c01;
-- churn (100 hot keys; see note above on why not 1):
UPDATE c01 SET k = 1000000 + (id % 100);

-- c02_hash_highwater  (hash, grow 5x then return to baseline population)
CREATE TABLE c02 (id bigint, k bigint) WITH (autovacuum_enabled = off);
INSERT INTO c02 SELECT g, g FROM generate_series(1,1000000) g;
CREATE INDEX c02_i ON c02 USING hash (k);
ANALYZE c02;
-- churn:
INSERT INTO c02 SELECT g, g FROM generate_series(1000001,5000000) g;
DELETE FROM c02 WHERE id > 1000000;

-- c03_gin_common  (gin, common-key posting growth)
CREATE TABLE c03 (id bigint, arr text[]) WITH (autovacuum_enabled = off);
INSERT INTO c03 SELECT g, ARRAY['a'||g, 'b'||(g%50000), 'c'||(g%1000)]
  FROM generate_series(1,1000000) g;
CREATE INDEX c03_i ON c03 USING gin (arr) WITH (fastupdate = off);
ANALYZE c03;
-- churn:
INSERT INTO c03 SELECT g, ARRAY['hot1','hot2','hot3']
  FROM generate_series(1000001,4000000) g;
DELETE FROM c03 WHERE id > 1000000;

-- c04_gin_keychurn  (gin, repeated complete key replacement, constant width)
CREATE TABLE c04 (id bigint, arr text[]) WITH (autovacuum_enabled = off);
INSERT INTO c04 SELECT g, ARRAY['r0k'||g, 'r0m'||(g%20000), 'r0n'||(g%500)]
  FROM generate_series(1,1000000) g;
CREATE INDEX c04_i ON c04 USING gin (arr) WITH (fastupdate = off);
ANALYZE c04;
-- churn:
DO $$ DECLARE r int; BEGIN
  FOR r IN 1..6 LOOP
    UPDATE c04 SET arr = ARRAY['r'||r||'k'||id, 'r'||r||'m'||(id%20000), 'r'||r||'n'||(id%500)];
  END LOOP;
END $$;

-- c05_gin_pending  (gin, pending-list-only growth: false-positive control)
CREATE TABLE c05 (id bigint, arr text[]) WITH (autovacuum_enabled = off);
INSERT INTO c05 SELECT g, ARRAY['a'||g, 'b'||(g%50000), 'c'||(g%1000)]
  FROM generate_series(1,1000000) g;
CREATE INDEX c05_i ON c05 USING gin (arr) WITH (fastupdate = on);
ANALYZE c05;
-- churn (no deletes, so every inserted row is legitimate work for a rebuild):
SET gin_pending_list_limit = '1GB';
INSERT INTO c05 SELECT g, ARRAY['a'||g, 'b'||(g%50000), 'c'||(g%1000)]
  FROM generate_series(1000001,1400000) g;

-- c06_gist_range  (gist, repeated range relocation at constant row count)
CREATE TABLE c06 (id bigint, r int8range) WITH (autovacuum_enabled = off);
INSERT INTO c06 SELECT g, int8range(g, g+10) FROM generate_series(1,1000000) g;
CREATE INDEX c06_i ON c06 USING gist (r);
ANALYZE c06;
-- churn (the loop variable must not be named r; the column is r):
DO $$ DECLARE rnd int; BEGIN
  FOR rnd IN 1..6 LOOP
    UPDATE c06 SET r = int8range((id * 7 + rnd * 1000003) % 100000000,
                                 (id * 7 + rnd * 1000003) % 100000000 + 10);
  END LOOP;
END $$;

-- c07_gist_highwater
CREATE TABLE c07 (id bigint, r int8range) WITH (autovacuum_enabled = off);
INSERT INTO c07 SELECT g, int8range(g, g+10) FROM generate_series(1,1000000) g;
CREATE INDEX c07_i ON c07 USING gist (r);
ANALYZE c07;
-- churn:
INSERT INTO c07 SELECT g, int8range(g, g+10) FROM generate_series(1000001,4000000) g;
DELETE FROM c07 WHERE id > 1000000;

-- c08_spgist_prefix  (spgist, repeated prefix replacement, constant width)
CREATE TABLE c08 (id bigint, t text) WITH (autovacuum_enabled = off);
INSERT INTO c08 SELECT g, 'aaa' || lpad(g::text, 12, '0') FROM generate_series(1,1000000) g;
CREATE INDEX c08_i ON c08 USING spgist (t);
ANALYZE c08;
-- churn:
DO $$ DECLARE r int; p text; BEGIN
  FOR r IN 1..6 LOOP
    p := chr(98 + r) || chr(112 + r) || chr(103 + r);
    UPDATE c08 SET t = p || lpad(((id * 7919 + r * 104729) % 1000000)::text, 12, '0');
  END LOOP;
END $$;

-- c09_spgist_highwater
CREATE TABLE c09 (id bigint, t text) WITH (autovacuum_enabled = off);
INSERT INTO c09 SELECT g, 'aaa' || lpad(g::text, 12, '0') FROM generate_series(1,1000000) g;
CREATE INDEX c09_i ON c09 USING spgist (t);
ANALYZE c09;
-- churn:
INSERT INTO c09 SELECT g, 'zzz' || lpad(g::text, 12, '0') FROM generate_series(1000001,4000000) g;
DELETE FROM c09 WHERE id > 1000000;

-- c10_brin_minmax  (brin, negative control: huge value churn, same row count)
CREATE TABLE c10 (id bigint, v bigint) WITH (autovacuum_enabled = off, fillfactor = 90);
INSERT INTO c10 SELECT g, g FROM generate_series(1,2000000) g;
CREATE INDEX c10_i ON c10 USING brin (v) WITH (pages_per_range = 128);
ANALYZE c10;
-- churn:
DO $$ DECLARE r int; BEGIN
  FOR r IN 1..6 LOOP
    UPDATE c10 SET v = (id * 7919 + r * 104729) % 2000000;
  END LOOP;
END $$;

-- c11_brin_mmmulti  (brin minmax_multi: correlated -> scattered -> correlated)
CREATE TABLE c11 (id bigint, v bigint) WITH (autovacuum_enabled = off, fillfactor = 90);
INSERT INTO c11 SELECT g, g FROM generate_series(1,2000000) g;
CREATE INDEX c11_i ON c11 USING brin (v int8_minmax_multi_ops(values_per_range = 64))
  WITH (pages_per_range = 128);
ANALYZE c11;
-- churn:
UPDATE c11 SET v = (id * 7919) % 2000000;
UPDATE c11 SET v = (id * 104729) % 2000000;
UPDATE c11 SET v = id;

-- c12_partial  (partial hash index: predicate population grows 9x, no real inflation)
CREATE TABLE c12 (id bigint, state text, k bigint) WITH (autovacuum_enabled = off);
INSERT INTO c12 SELECT g, CASE WHEN g % 10 = 0 THEN 'pending' ELSE 'done' END, g
  FROM generate_series(1,2000000) g;
CREATE INDEX c12_i ON c12 USING hash (k) WHERE state = 'pending';
ANALYZE c12;
-- churn:
UPDATE c12 SET state = 'pending' WHERE id % 10 BETWEEN 1 AND 8;
```

`gin_pending_list_limit` is `PGC_USERSET`
([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)),
so the `SET` in `c05` is session/transaction scope only and needs no reload or
restart.

### Measurement queries

The four ground-truth sizes, taken at the four protocol points:

```sql
-- B: after CREATE INDEX + ANALYZE.  C: after churn + VACUUM (ANALYZE).
-- R: immediately after REINDEX INDEX.  Also taken post-churn, pre-VACUUM.
SELECT /* wiki_idxmaint_measure */
       c.oid::regclass                      AS index_name,
       am.amname,
       pg_relation_size(c.oid)              AS bytes,
       c.relpages,
       c.reltuples::bigint                  AS reltuples,
       pg_size_pretty(pg_relation_size(c.oid)) AS pretty
  FROM pg_class c
  JOIN pg_am am ON am.oid = c.relam
 WHERE c.oid = 'public.my_index'::regclass;
```

Reclaimed space, from the recorded C and R:

```sql
SELECT /* wiki_idxmaint_reclaimed */
       :c_bytes - :r_bytes                                        AS actual_reclaimed_bytes,
       round(100.0 * (:c_bytes - :r_bytes) / GREATEST(:c_bytes,1), 1)
                                                                  AS actual_reclaimed_pct;
```

### Results

Server: isolated 17.11 built from the pin (`--without-readline --without-zlib
--with-icu --enable-debug`), `block_size` 8192, `autovacuum = off`, `fsync = off`,
`shared_buffers` 512MB, `maintenance_work_mem` 256MB. B, C and R are
`pg_relation_size()` in kB. Ground truth per cell is a measured `REINDEX INDEX`. Every
run below and every re-run used this one cluster; a full pass takes about ten minutes.

| Cell | AM | B (kB) | C (kB) | R (kB) | Inflation | Reclaimed bytes | Reclaimed % | Recommendation | Correct? |
|---|---|---|---|---|---|---|---|---|---|
| `c00_control` | hash | 32784 | 32784 | 32784 | 1.000 | 0 | 0.0 | none | yes |
| `c01_hash_dup` | hash | 32784 | 76944 | 47904 | 2.347 | 29736960 | 37.7 | strong REINDEX candidate | yes |
| `c02_hash_highwater` | hash | 32784 | 165992 | 32784 | 5.063 | 136404992 | 80.2 | strong REINDEX candidate | yes |
| `c03_gin_common` | gin | 71664 | 83920 | 71664 | 1.171 | 12550144 | 14.6 | none | near-threshold miss |
| `c04_gin_keychurn` | gin | 81040 | 557496 | 81040 | 6.998 | 487890944 | 85.5 | strong REINDEX candidate | yes |
| `c05_gin_pending` | gin | 71664 | 125112 | 102168 | 1.247 | 23494656 | 18.3 | none | near-threshold miss |
| `c06_gist_range` | gist | 79200 | 384640 | 79200 | 4.907 | 312770560 | 79.4 | strong REINDEX candidate | yes |
| `c07_gist_highwater` | gist | 79200 | 293480 | 79200 | 3.706 | 219422720 | 73.0 | strong REINDEX candidate | yes |
| `c08_spgist_prefix` | spgist | 33168 | 241224 | 34432 | 7.405 | 211755008 | 85.7 | strong REINDEX candidate | yes |
| `c09_spgist_highwater` | spgist | 33168 | 131712 | 33168 | 3.971 | 100909056 | 74.8 | strong REINDEX candidate | yes |
| `c10_brin_minmax` | brin | 24 | 32 | 32 | 0.209 | 0 | 0.0 | none: index below 1 MB | yes |
| `c11_brin_mmmulti` | brin | 48 | 112 | 112 | 0.690 | 0 | 0.0 | none: index below 1 MB | yes |
| `c12_partial` | hash | 40976 | 66688 | 65008 | 0.178 | 1720320 | 2.5 | suppressed: predicate population moved | yes |

Eleven of thirteen recommendations are correct as filed. The two exceptions are both
GIN, both below the GIN candidate threshold of 1.50, and both leave 14.6% and 18.3%
on the table; they are threshold choices rather than model errors, and the inflation
figure predicted both amounts accurately (see below).

The whole matrix has now been executed **six times end to end** — runs 2, 3 and 4 on the
day it was filed, runs 5, 6 and 7 on review the next day — with run 1 discarded because
two cells were defective. **All six runs produced identical `(B, post-churn, C, R)`
quadruples on all 13 cells**, and identical `reclaimed_bytes`, `reclaimed_pct` and
recommendations: the physical measurements are exactly reproducible on this server,
across a restart. The table above is run 4, the run made with the statement as first
filed; runs 5-7 used the statement as it now stands, with the access-method guard.

The *inflation* figures are the only thing that moves, because the population term comes
from `ANALYZE`'s sample. Nine of the 13 cells are bit-identical in all six runs and four
drift, by at most 2.6% of the reading — see
[The three-run re-execution](#the-three-run-re-execution) for the full six-run table.
The consequence for a reader is unchanged: treat the inflation number as a
two-significant-figure estimate and do not compare it against a threshold it sits within
2% of.

### Inflation predicts reclaimable space, not just rank

If the model is right, a rebuild should land at `C / size_inflation`, so the
reclaimable fraction should be `1 - 1/size_inflation`. Measured against ground truth:

| Cell | Inflation | Predicted reclaim % | Actual reclaim % | Error (points) |
|---|---|---|---|---|
| `c00_control` | 1.000 | 0.0 | 0.0 | +0.0 |
| `c01_hash_dup` | 2.347 | 57.4 | 37.7 | **+19.7** |
| `c02_hash_highwater` | 5.063 | 80.2 | 80.2 | +0.0 |
| `c03_gin_common` | 1.171 | 14.6 | 14.6 | +0.0 |
| `c04_gin_keychurn` | 6.998 | 85.7 | 85.5 | +0.2 |
| `c05_gin_pending` | 1.247 | 19.8 | 18.3 | +1.5 |
| `c06_gist_range` | 4.907 | 79.6 | 79.4 | +0.2 |
| `c07_gist_highwater` | 3.706 | 73.0 | 73.0 | +0.0 |
| `c08_spgist_prefix` | 7.405 | 86.5 | 85.7 | +0.8 |
| `c09_spgist_highwater` | 3.971 | 74.8 | 74.8 | +0.0 |
| `c10_brin_minmax` | 0.209 | 0.0 | 0.0 | +0.0 |
| `c11_brin_mmmulti` | 0.690 | 0.0 | 0.0 | +0.0 |
| `c12_partial` | 0.178 | 0.0 | 2.5 | -2.5 |

The single large error is `c01_hash_dup`, and its cause is instructive. That cell
moves 1,000,000 distinct keys onto 100 duplicated keys. The logical population is
unchanged, so the model predicts a rebuild back to the baseline size — but a fresh
build of the *new* data is legitimately larger than a fresh build of the old data,
because a hash index of 100 keys is 100 long overflow chains rather than a wide
spread of bucket pages. The model assumes bytes-per-logical-unit is invariant across
the churn, and a change in key distribution violates that. This is the general shape
of the heuristic's over-estimates.

### BRIN: desummarize plus summarize made the index bigger

The prompt asks whether `brin_desummarize_range()` + `brin_summarize_range()` is a
better recommendation than `REINDEX` for some BRIN cases. Measured on a
2,000,000-row table with an `int8_minmax_multi_ops(values_per_range = 64)` index,
scattered then re-correlated:

| Step | `minmax_multi` bytes | `minmax` bytes | Table pages |
|---|---|---|---|
| fresh | 49152 | 24576 | 11977 |
| after churn + VACUUM | 114688 | 32768 | 40701 |
| after `brin_desummarize_range()` on all 318 ranges | **114688** | — | — |
| after `brin_summarize_range()` on all 318 ranges | **196608** | — | — |
| after `brin_summarize_new_values()` | 196608 (0 ranges left to add) | — | — |
| after `REINDEX` | 114688 | 32768 | — |

Desummarizing 318 ranges freed **nothing**, and resummarizing made the index **71%
larger** than it started. Both halves follow from source: `brinRevmapDesummarizeRange`
removes the tuple with `PageIndexTupleDeleteNoCompact` and carries the comment
`/* XXX record free space in FSM? */`
([brin_revmap.c#brinRevmapDesummarizeRange](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L395-L410)),
and because that routine never sets `PD_HAS_FREE_LINES`
([bufpage.c#PageIndexTupleDeleteNoCompact](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L1333-L1347)),
the orphaned slots cannot be reused by `PageAddItem`
([bufpage.c#PageAddItemExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L249-L258)).
`summarize_range` then inserts a placeholder and updates it into the real summary,
which usually takes the relocate branch again
([brin.c#summarize_range](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1740-L1864)).

So the design does allow a different BRIN recommendation, but the evidence points the
other way: **`brin_desummarize_range()` + `brin_summarize_range()` is a summary-quality
operation, not a space-reclamation one, and it costs space.** The BRIN arm therefore
uses the highest thresholds of any AM and never emits a resummarize recommendation.

The BRIN negative control behaved exactly as the prompt predicted: `c10_brin_minmax`
absorbed six full-table update rounds — a churn ratio of 5.31 to 5.48 over six runs,
against 10.8 million non-HOT updates on 2,000,000 rows — and grew from 3 pages to 4,
with 0 bytes reclaimable in every run, because a fixed-width minmax summary almost
always takes the in-place branch
([brin_pageops.c#brin_can_do_samepage_update](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L319-L328)).

### Experimental thresholds

**These are starting points from 13 cells on one server, not production values.**

| AM | Candidate: inflation / churn | Strong: inflation / churn | Confidence |
|---|---|---|---|
| hash | 1.30 / 0.50 | 1.50 / 1.00 | highest — 3 cells, all correct |
| gist | 1.40 / 0.75 | 1.80 / 1.50 | good — 2 positives, both correct |
| spgist | 1.40 / 0.75 | 1.80 / 1.50 | good — 2 positives, both correct |
| gin | 1.50 / 1.00 | 2.00 / 2.00 | lower — 1 clear positive, 2 near-threshold misses |
| brin | 2.00 / 1.00 | 3.00 / 2.00 | untested against any true positive |

Because `1 - 1/inflation` tracks the reclaimable fraction, a threshold is most
usefully chosen in terms of the space you want back:

| Inflation threshold | Implied reclaimable fraction |
|---|---|
| 1.15 | 13% |
| 1.30 | 23% |
| 1.50 | 33% |
| 2.00 | 50% |
| 3.00 | 67% |

An absolute floor is also applied: no index below 1 MB is ever recommended for
rebuild, regardless of ratio.

### Known limitations

1. **A change in key distribution breaks the model.** `c01_hash_dup` over-estimated
   by 19.7 points. Catalog metadata records how many rows are indexed, not how they
   are shaped, so a fresh build of new data can legitimately differ in
   bytes-per-tuple from a fresh build of the old data.
2. **The model has no intercept.** `expected = base_size * pop_ratio` assumes size is
   proportional to population through the origin. Every index has fixed overhead — a
   metapage, a revmap prefix, root pages — so the ratio is wrong for small indexes.
   For BRIN, where a whole index is often 3-6 pages, the fixed part dominates:
   `c10_brin_minmax` reported 0.209 where the truth is ~1.0. The 1 MB floor contains
   the damage but does not fix the arithmetic.
3. **The BRIN arm is unvalidated against a true positive.** Neither BRIN cell produced
   a single reclaimable byte.
4. **GIN's `avg_width` is post-TOAST**
   ([pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L40-L50)).
   For an indexed column whose values are TOASTed out of line, `iw` measures the
   pointer and the GIN input-mass normalization degrades toward a plain row count.
5. **Expression indexes are not normalized.** `iw` is summed from `pg_stats` rows for
   the *table's* columns, and an expression key has `indkey = 0`, so it contributes
   nothing. `ANALYZE` does store expression statistics against the index relation
   ([analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L828-L863)),
   which the statement does not read.
6. **GIN pending-list growth is not what intuition says.** Flushing the pending list
   makes the index *larger*, not smaller. Measured with `pageinspect` as ground truth,
   and reproduced byte for byte on the re-run: 491 pending pages / 16,654,336 bytes
   before `gin_clean_pending_list()`, and 0 pending pages / **21,905,408 bytes** after.
   Any GIN reading taken before the pending list is processed understates the eventual
   size, which is the real reason post-`VACUUM` evaluation matters for GIN — not the
   reverse.
7. **`VACUUM (INDEX_CLEANUP OFF)` leaves index relstats stale**
   ([vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L506-L513)),
   so `vacuum_count` can advance while the population reading does not refresh.
8. **`pg_relation_size()` is VOLATILE and stats the filesystem**
   ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7495),
   [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371)).
   The heuristic is catalog-plus-size, not purely catalog.
9. **A partial index whose predicate fraction is stable but whose rows churn heavily
   is scored normally**, and the `[0.7, 1.43]` suppression window is an unvalidated
   guess — only one partial cell was tested.
10. **Nothing here detects a `DROP INDEX`.** The comment is destroyed with the index
    ([dependency.c#deleteOneObject](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1326-L1336)),
    so a drop-and-recreate silently starts from no baseline. That is safe, but it
    means an index can lose its history without any signal.
11. **A build-time `itup` for BRIN is not a range count.** A parallel build stores the
    number of (participant, range) pairs, measured at 23 / 44 / 102 for the same
    23-range index. The `anl` gate makes this harmless for scoring, since a baseline
    is only valid after an `ANALYZE` has overwritten it, but the stored `ipg`/`itup`
    diagnostics carry the artifact.
12. **`churn_state` mislabels a missing baseline.** With no payload the churn deltas
    are NULL, so `churn_known` is NULL and the column prints
    `unknown: counters reset` even though no counter was ever reset. Measured; the
    `recommendation` for that row is still the correct `capture new baseline`.

## Context Reviewed

- Pinned checkout `raw/postgres-17/` at `786db8dcf168bd9df8f55047337525ac19118b1c`
  (17.11), the only evidence base used.
- Comment storage and lifecycle: `comment.c`, `pg_description.h`, `objectaddress.c`,
  `dependency.c`, `system_functions.sql`, `gram.y`, and the `create_index`
  regression test.
- Index rebuild paths: `index.c` (`reindex_index`, `index_concurrently_swap`,
  `index_build`, `index_update_stats`), `indexcmds.c`, `relcache.c`.
- Statistics: `system_views.sql`, `pgstatfuncs.c`, `pgstat.h`, `pgstat.c`,
  `pgstat_database.c`, `pgstat_relation.c`, `pgstat_shmem.c`.
- Relation statistics writers: `analyze.c`, `vacuum.c`, `vacuumlazy.c`.
- Access methods: `hash.c`, `hashovfl.c`, `hashpage.c`, `hashinsert.c`,
  `hashutil.c`, `hash/README`; `gininsert.c`, `ginvacuum.c`, `ginfast.c`,
  `gindatapage.c`, `ginutil.c`, `ginarrayproc.c`, `gin/README`; `gist.c`,
  `gistvacuum.c`, `gistbuild.c`, `gistutil.c`, `gist/README`; `spgdoinsert.c`,
  `spgvacuum.c`, `spgutils.c`, `spgist/README`; `brin.c`, `brin_pageops.c`,
  `brin_revmap.c`, `brin_minmax_multi.c`, `brin.h`, `brin_page.h`.
- Shared page code: `bufpage.c`, `indexfsm.c`.
- Sizing and options: `dbsize.c`, `reloptions.c`, `guc_tables.c`, `pg_am.dat`,
  `pg_class.h`, `pg_index.h`, `pg_statistic.h`.
- Exact-pin execution: an isolated 17.11 cluster built from this pin, 13-cell matrix
  run three times plus six edge-case probes, with `pageinspect` used only as ground
  truth. Sandbox was under `.wiki-runtime/tmp/idxm/`, deleted on 2026-08-25 at the
  user's instruction to make room for other work; the fixtures are rebuildable from
  the harness text quoted on this page.
- 2026-08-25 review, same pin and same cluster restarted with all 13 fixtures and the
  four recorded runs intact: the 57 distinct source ranges cited before the review all
  re-read; the filed sizes, inflation figures and cell scripts diffed against the
  harness's `results/metrics.csv`, `results/eval.csv`, `results/eval_prevacuum.csv`,
  `results_run{1,2,3}/` and `cells/`; and 13 fresh probe sections under `review/`
  covering the `reltuples` progression, serial versus parallel BRIN builds, the
  statistics reset, the BRIN desummarize/summarize pair, capture/parse/evaluate as
  filed, the re-capture `ts` question, fork scope, the `ppr` fallback, an invalid index
  and a missing baseline — plus a two-text regression diff over every index carrying a
  baseline and a check that the page's published SQL blocks are the texts that ran.
- Parallel-build accounting and parallel-scan chunk sizing: `brin.c`
  (`form_and_spill_tuple`, `_brin_parallel_scan_and_build`, `_brin_parallel_heapscan`,
  `_brin_parallel_merge`, `brin_fill_empty_ranges`, `brinsummarize`), `tableam.c`.
- 2026-08-25 re-execution, same pin and same cluster: the 13-cell matrix three more
  times (runs 5-7) through a copy of the filed harness whose only changes are the
  output paths and a switch to the statement extracted from this page; the six original
  probes through their own script, unmodified apart from those same paths; the 13 review
  probes; and one new probe for the invalid-index arm on a hash index. Artifacts under
  `.wiki-runtime/tmp/idxm/review2/`, with the filed `results*/`, `sql/`, `cells/` and
  `logs/` left untouched for comparison. That whole tree, cluster included, was
  deleted on 2026-08-25 at the user's instruction, so the recorded runs are no longer
  on disk and the numbers on this page cannot be re-diffed without rebuilding.

## Evidence Map

| Claim | Evidence |
|---|---|
| No index AM truncates during VACUUM | [spgvacuum.c#spgvacuumscan](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L877-L900) (`#ifdef NOT_USED`), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55) |
| Hash cannot shrink without REINDEX | [hash/README](../../../../raw/postgres-17/src/backend/access/hash/README#L31-L34), [hashovfl.c#_hash_freeovflpage](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L632-L642) |
| GIN entry tree is never pruned; leaves are not re-encoded | [gin/README](../../../../raw/postgres-17/src/backend/access/gin/README#L390-L396), [gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L797-L813) |
| GiST deletes only fully empty leaves | [gistvacuum.c#gistvacuumpage](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L388-L403), [gistvacuum.c#gistvacuum_delete_empty_pages](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L541-L548) |
| SP-GiST keeps interior placeholders | [spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L569-L590) |
| BRIN orphans line pointers on relocate; bulkdelete is a no-op | [brin_pageops.c#brin_doupdate](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c#L246-L262), [bufpage.c#PageIndexTupleDeleteNoCompact](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L1333-L1347), [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301) |
| Comment survives plain REINDEX | [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3782-L3789), [create_index.out#testcomment](../../../../raw/postgres-17/src/test/regress/expected/create_index.out#L2300-L2324) |
| Comment is moved by REINDEX CONCURRENTLY | [index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1740-L1784) |
| ANALYZE overwrites index reltuples with the table row estimate | [analyze.c:449](../../../../raw/postgres-17/src/backend/commands/analyze.c#L449), [analyze.c#compute_index_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L861-L863), [analyze.c#do_analyze_rel](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663) |
| CREATE INDEX/REINDEX write the AM's own index_tuples | [index.c#index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3135) |
| VACUUM writes the AM's num_index_tuples when exact | [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3086-L3097) |
| GIN reports heap tuples at vacuum cleanup | [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L735-L739) |
| BRIN reports summarized ranges at vacuum cleanup | [brin.c#brinvacuumcleanup](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1324-L1327) |
| BRIN parallel build over-counts index_tuples: each participant counts its own partial summaries, the leader adopts the sum, and the union that follows never recounts | [brin.c#form_and_spill_tuple](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1996-L2015), [brin.c#_brin_parallel_scan_and_build](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2837-L2841), [brin.c#_brin_parallel_heapscan](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2568-L2596), [brin.c#_brin_parallel_merge](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2688-L2720) |
| Empty-range backfill inserts without counting, so it cannot explain the over-count | [brin.c#brin_fill_empty_ranges](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2979-L3004) |
| A parallel scan's chunk is sized from the table and is smaller than a 128-page BRIN range below ~262,000 pages | [tableam.c#table_block_parallelscan_startblock_init](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L434-L451), [tableam.c#PARALLEL_SEQSCAN_NCHUNKS](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L41-L46) |
| A serial BRIN build counts one summary per range | [brin.c#brinbuild](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1215-L1248), [brin.c#form_and_insert_tuple](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1975-L1989) |
| BRIN vacuum cleanup counts new plus existing ranges, because both out-params are the same pointer | [brin.c#brinvacuumcleanup](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1324-L1327), [brin.c#brinsummarize](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1869-L1879), [brin.c:1948](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1948) |
| `pg_relation_size` is VOLATILE | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7495) |
| `PageIndexTupleDeleteNoCompact` is the only page routine that leaves LP_UNUSED slots without setting the hint | [bufpage.c#PageIndexTupleDeleteNoCompact](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L1284-L1295), [bufpage.c:810](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L810), [bufpage.c:893](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L893) |
| No per-table statistics reset timestamp | [pgstat.h#PgStat_StatTabEntry](../../../../raw/postgres-17/src/include/pgstat.h#L399-L429), [pgstat.c#pgstat_kind_builtin_infos](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L278-L290) |
| Single-table reset bumps the database timestamp | [pgstat.c#pgstat_reset](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L734-L757) |
| BRIN size follows heap block count | [brin_revmap.c#HEAPBLK_TO_REVMAP_BLK](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L35-L43), [brin.c#brinbuildCallback](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L998-L1008) |
| Desummarize does not publish freed space | [brin_revmap.c#brinRevmapDesummarizeRange](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c#L395-L410) |
| `pages_per_range` default is 128 | [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-17/src/include/access/brin.h#L39-L45) |
| `values_per_range` default 32, range 8-256 | [brin_minmax_multi.c#MINMAX_MULTI_DEFAULT_VALUES_PER_PAGE](../../../../raw/postgres-17/src/backend/access/brin/brin_minmax_multi.c#L127-L132), [brin_minmax_multi.c#brin_minmax_multi_options](../../../../raw/postgres-17/src/backend/access/brin/brin_minmax_multi.c#L2953-L2965) |
| GIN and BRIN have no fillfactor | [ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614), [brin.c#brinoptions](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1337-L1349) |
| `gin_pending_list_limit` is `PGC_USERSET` | [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585) |
| `pg_relation_size(regclass)` is the main fork only | [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289) |
| `obj_description` filters classoid and objsubid = 0 | [system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301) |
| pg_description is TOASTable text | [pg_description.h#pg_description](../../../../raw/postgres-17/src/include/catalog/pg_description.h#L48-L57) |
| COMMENT ON takes ShareUpdateExclusiveLock and needs ownership | [comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L77) |
| DROP INDEX destroys the comment | [dependency.c#deleteOneObject](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1326-L1336) |
| Hash bucket is a pure function of the hash code | [hashutil.c#_hash_hashkey2bucket](../../../../raw/postgres-17/src/backend/access/hash/hashutil.c#L121-L135) |
| Sorted GiST build ignores fillfactor | [gistbuild.c#gist_indexsortbuild_levelstate_add](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L464-L470) |

## Open Questions

1. **The BRIN arm has never been validated against a true positive.** Both BRIN cells
   reclaimed 0 bytes. Until a workload is found that leaves a BRIN index materially
   larger than a fresh build, the BRIN thresholds are guesses and the `minmax_multi`
   "maintenance case" in the required matrix is unreproduced.
2. **The no-intercept problem is unquantified.** An affine model
   (`expected = fixed + slope * population`) would need two baselines at different
   populations. Whether that is worth the extra comment bytes is untested.
3. **`c01_hash_dup`'s +19.7 point error was not decomposed.** How much is the
   legitimate cost of 100 long overflow chains versus reclaimable churn was not
   separated; that would need a fresh build of an identically-distributed table as a
   third reference point.
4. **The `[0.7, 1.43]` partial-index suppression window is arbitrary.** Only one
   partial cell was run, and it shifted 9.12x — far outside the window. The behaviour
   near the boundary is unmeasured.
5. **Only one fixture scale was tested.** Everything is ~1M rows (2M for BRIN and
   partial). Whether the thresholds hold at 100M rows, where the prompt's examples
   sit, is unknown.
6. **Autovacuum was off for every cell.** The interaction between the
   `vacuum_count`/`autovacuum_count` gate and a live autovacuum that fires mid-churn
   is untested, as is the autoanalyze path that flushes a GIN pending list
   ([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717)).
7. **Expression and multi-column indexes were not tested at all.** The `iw` term
   silently contributes nothing for an expression key, and no cell exercised a
   multi-column GIN index.
8. **Concurrency was not tested.** Every measurement is single-session. Whether a
   baseline captured while other sessions write produces a usable reading, and how
   the `ShareUpdateExclusiveLock` from `COMMENT ON` interacts with concurrent
   maintenance, is unmeasured.
9. **`REINDEX INDEX CONCURRENTLY` was never used as the rebuild in a scored cell.**
   Ground truth is always plain `REINDEX INDEX`. The two should produce the same
   size, but that was not verified per cell.
10. **The two GIN near-threshold misses may indicate the GIN threshold is too high**,
    or that `reltuples x avg_width` is the wrong mass proxy. A GIN-specific sweep
    across several array widths would settle it; it was not run.
11. **Statistics are cluster-local.** After a physical failover or a `pg_upgrade`, the
    cumulative counters do not carry over, and under `IsBinaryUpgrade`
    `index_update_stats` skips the `relpages`/`reltuples` write itself, leaving the
    values the dump carried
    ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2838-L2842)).
    The monotonicity check should catch this, but it was not tested.
12. **Everything is measured on one cluster.** Six matrix runs and every probe have now
    been re-executed, but always on the same 17.11 build, the same `block_size` 8192,
    the same `shared_buffers` and `maintenance_work_mem`, and the same filesystem. The
    byte-exact reproducibility is therefore a statement about repeatability on one
    machine, not about portability to another.
13. **The access-method guard is proven inert, not proven correct.** The added
    `unsupported access method` arm was regression-tested on 19 baselined indexes and
    moves only the two B-tree rows, and the three re-runs then reproduced all 13 matrix
    recommendations with it in place. Whether refusing is the right answer for a
    hypothetical seventh index AM, or for a `btree` index whose owner deliberately
    captured a baseline, is a design choice this page does not test. The guard being the
    first arm also means it pre-empts `skip: index not valid` for a non-target AM, which
    is deliberate but untested against any operator expectation.
14. **A parallel BRIN build's `reltuples` is not reproducible.** Eight observations of
    the default build on the same 23-range index span 43 to 46, and the four-worker
    build gave 102 and then 90. Only the serial number (23) is deterministic, and only
    that one equals the true range count.
15. **One measured aside was not re-run: the single-hot-key timing.** The note that one
    hot key completed "1 row in 90 seconds" comes from the original design session and
    was not reproduced, because reproducing it means waiting on a deliberately
    quadratic workload. The mechanism behind it is source-cited, and the filed cell's
    100-key form is what every run measures.

## Source References

- [index.c](../../../../raw/postgres-17/src/backend/catalog/index.c) - `reindex_index`, `index_concurrently_swap`, `index_build`, `index_update_stats`.
- [comment.c](../../../../raw/postgres-17/src/backend/commands/comment.c) - `CommentObject`, `CreateComments`, `DeleteComments`.
- [pg_description.h](../../../../raw/postgres-17/src/include/catalog/pg_description.h) - comment catalog.
- [system_functions.sql](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql) - `obj_description`, `pg_relation_size`.
- [system_views.sql](../../../../raw/postgres-17/src/backend/catalog/system_views.sql) - `pg_stat_all_tables`, `pg_stats`, `pg_stat_database`.
- [analyze.c](../../../../raw/postgres-17/src/backend/commands/analyze.c) - `do_analyze_rel`, `compute_index_stats`.
- [vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c) - `update_relstats_all_indexes`.
- [pgstat.h](../../../../raw/postgres-17/src/include/pgstat.h) - `PgStat_StatTabEntry`.
- [pgstat.c](../../../../raw/postgres-17/src/backend/utils/activity/pgstat.c) - `pgstat_reset`, kind table.
- [relcache.c](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c) - `RelationSetNewRelfilenumber`.
- [hash/README](../../../../raw/postgres-17/src/backend/access/hash/README), [hash.c](../../../../raw/postgres-17/src/backend/access/hash/hash.c), [hashovfl.c](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c), [hashpage.c](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c), [hashutil.c](../../../../raw/postgres-17/src/backend/access/hash/hashutil.c).
- [gin/README](../../../../raw/postgres-17/src/backend/access/gin/README), [ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c), [gindatapage.c](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c), [ginfast.c](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c), [ginutil.c](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c), [ginarrayproc.c](../../../../raw/postgres-17/src/backend/access/gin/ginarrayproc.c).
- [gistvacuum.c](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c), [gistbuild.c](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c).
- [spgvacuum.c](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c).
- [brin.c](../../../../raw/postgres-17/src/backend/access/brin/brin.c) - also `brinbuild`, `form_and_insert_tuple`, `form_and_spill_tuple`, `brinsummarize`, `brin_fill_empty_ranges`, and the four parallel-build routines.
- [brin_pageops.c](../../../../raw/postgres-17/src/backend/access/brin/brin_pageops.c), [brin_revmap.c](../../../../raw/postgres-17/src/backend/access/brin/brin_revmap.c), [brin_minmax_multi.c](../../../../raw/postgres-17/src/backend/access/brin/brin_minmax_multi.c), [brin.h](../../../../raw/postgres-17/src/include/access/brin.h).
- [tableam.c](../../../../raw/postgres-17/src/backend/access/table/tableam.c) - `table_block_parallelscan_startblock_init`, parallel-scan chunk sizing.
- [bufpage.c](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c), [indexfsm.c](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c).
- [dbsize.c](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c), [reloptions.c](../../../../raw/postgres-17/src/backend/access/common/reloptions.c), [guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c).
- [pg_am.dat](../../../../raw/postgres-17/src/include/catalog/pg_am.dat), [pg_proc.dat](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat), [pg_statistic.h](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h), [dependency.c](../../../../raw/postgres-17/src/backend/catalog/dependency.c).
- [create_index.out](../../../../raw/postgres-17/src/test/regress/expected/create_index.out) - comment-preservation regression test.

## Navigation

- [v17/index](../../index.md) - PostgreSQL 17 landing page.
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)](btree-bloat-with-pgstatindex.md) - the B-tree counterpart, which uses `pgstatindex` rather than catalog-only inputs.
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md) - the rebuild path this heuristic recommends.
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md) - what the planner does and does not see about a bloated index.
- [versions](../../../versions.md) - source pin manifest.
- [index](../../../index.md) - global wiki catalog.
