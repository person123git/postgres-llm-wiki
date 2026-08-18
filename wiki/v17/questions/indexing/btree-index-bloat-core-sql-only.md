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
  - [Verdict](#verdict)
  - [The current recommended statement](#the-current-recommended-statement)
  - [How the test was run](#how-the-test-was-run)
  - [The fifteen fixtures on 17.10](#the-fifteen-fixtures-on-1710)
  - [Method A against a rebuild](#method-a-against-a-rebuild)
  - [Deduplication is what breaks the model](#deduplication-is-what-breaks-the-model)
  - [The duplication-ratio sweep](#the-duplication-ratio-sweep)
  - [NULL runs are deduplicated too](#null-runs-are-deduplicated-too)
  - [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17)
  - [Method A-prime still fixes variable key width](#method-a-prime-still-fixes-variable-key-width)
  - [Method B: leaf counts still exact, density formula not](#method-b-leaf-counts-still-exact-density-formula-not)
  - [Method C: unchanged answer, different write path](#method-c-unchanged-answer-different-write-path)
  - [Method D: new message, no row count, new blind spot](#method-d-new-message-no-row-count-new-blind-spot)
  - [The v14 unknown reltuples sentinel](#the-v14-unknown-reltuples-sentinel)
  - [Partial indexes and the statistics state](#partial-indexes-and-the-statistics-state)
  - [The 54-cell matrix](#the-54-cell-matrix)
  - [Accuracy on 17 against the v12 page's reported accuracy](#accuracy-on-17-against-the-v12-pages-reported-accuracy)
  - [What to change before running the v12 SQL on 17](#what-to-change-before-running-the-v12-sql-on-17)
  - [Cost of each method on this server](#cost-of-each-method-on-this-server)
  - [Settings and apply scopes](#settings-and-apply-scopes)
  - [What no core-SQL method can measure on v17](#what-no-core-sql-method-can-measure-on-v17)
  - [Follow-up: the same sweep on a 12.2 server and a 17.10 server](#follow-up-the-same-sweep-on-a-122-server-and-a-1710-server)
  - [Follow-up: the INCLUDE-column false positive on v17](#follow-up-the-include-column-false-positive-on-v17)
  - [Follow-up: the v12 hazard the reltuples guard does not cover](#follow-up-the-v12-hazard-the-reltuples-guard-does-not-cover)
  - [Follow-up: one statement for PostgreSQL 12 through 17](#follow-up-one-statement-for-postgresql-12-through-17)
  - [What the proposed statement changes](#what-the-proposed-statement-changes)
  - [The posting-list arithmetic, derived from source](#the-posting-list-arithmetic-derived-from-source)
  - [The gate, and what each conjunct rejects](#the-gate-and-what-each-conjunct-rejects)
  - [How the same statement behaves on 12.2, 14.23 and 17.10](#how-the-same-statement-behaves-on-122-1423-and-1710)
  - [Measured accuracy, per fixture](#measured-accuracy-per-fixture)
  - [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate)
  - [Where the proposal is still wrong](#where-the-proposal-is-still-wrong)
  - [Settings the proposal touches](#settings-the-proposal-touches)
  - [Follow-up: two reporting defects, not arithmetic defects](#follow-up-two-reporting-defects-not-arithmetic-defects)
  - [Defect 1: the FSM column is a history bit](#defect-1-the-fsm-column-is-a-history-bit)
  - [Where a current count of empty, deleted and reusable pages comes from](#where-a-current-count-of-empty-deleted-and-reusable-pages-comes-from)
  - [Defect 2: the byte column is clamped and the percentage is not](#defect-2-the-byte-column-is-clamped-and-the-percentage-is-not)
  - [The corrected columns](#the-corrected-columns)
  - [What the corrections change, and whether the v17 sweep needs them too](#what-the-corrections-change-and-whether-the-v17-sweep-needs-them-too)
  - [Follow-up: the output columns say wasted_space, not bloat](#follow-up-the-output-columns-say-wasted_space-not-bloat)
  - [What the rename cannot change](#what-the-rename-cannot-change)
  - [What a consumer must change](#what-a-consumer-must-change)
  - [Follow-up: five changes from a twelve-issue review](#follow-up-five-changes-from-a-twelve-issue-review)
  - [Four issues the statement already fixed](#four-issues-the-statement-already-fixed)
  - [The mischaracterized issue: page packing](#the-mischaracterized-issue-page-packing)
  - [Change 1: round each key group up to whole posting tuples](#change-1-round-each-key-group-up-to-whole-posting-tuples)
  - [Change 2: extended statistics for a multicolumn key](#change-2-extended-statistics-for-a-multicolumn-key)
  - [Change 3: statistics that exist but are invisible](#change-3-statistics-that-exist-but-are-invisible)
  - [Change 4: a randomly inserted, never-deleted index](#change-4-a-randomly-inserted-never-deleted-index)
  - [Change 5: what the baseline is, and what a reading means](#change-5-what-the-baseline-is-and-what-a-reading-means)
  - [The corrected statement, with all five changes](#the-corrected-statement-with-all-five-changes)
  - [Measured on 17.11, per fixture](#measured-on-1711-per-fixture)
  - [Direction, magnitude, and whether the floor was immune](#direction-magnitude-and-whether-the-floor-was-immune)
  - [Two rejected fixes](#two-rejected-fixes)
  - [The corrected statement on a 12.2 server](#the-corrected-statement-on-a-122-server)
  - [Follow-up: seventeen mandatory tests for the deduplication gate](#follow-up-seventeen-mandatory-tests-for-the-deduplication-gate)
  - [The seventeen mandatory tests, and the verdict on each](#the-seventeen-mandatory-tests-and-the-verdict-on-each)
  - [How the mandatory tests were run](#how-the-mandatory-tests-were-run)
  - [What the engine decides, and when](#what-the-engine-decides-and-when)
  - [Change 6: name the support function, do not just count it](#change-6-name-the-support-function-do-not-just-count-it)
  - [Why prosrc and not proname](#why-prosrc-and-not-proname)
  - [What change 6 costs](#what-change-6-costs)
  - [What the mixed-key failure costs](#what-the-mixed-key-failure-costs)
  - [The earlier v17 sweep needs three conjuncts](#the-earlier-v17-sweep-needs-three-conjuncts)
  - [Post-build mutation, and why the metapage is still not the answer](#post-build-mutation-and-why-the-metapage-is-still-not-the-answer)
  - [The mandatory tests on a 12.2 server](#the-mandatory-tests-on-a-122-server)
  - [The harness, runnable](#the-harness-runnable)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17: test the SQL from the PostgreSQL 12 question "Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)" on PostgreSQL 17 and compare whether it measures bloat with the same accuracy as in version 12.

Follow-up: does the deduplication-aware sweep for v17 work for indexes from v12 and v17?

> Prompt note: the follow-up is filed as an approved grammar-corrected restatement
> of the original wording ("does A deduplication-aware sweep for v17 works for
> indexes from v12 and v7?"), per the repository's prompt-hygiene rule. The asker
> confirmed that "v7" meant v17, and that "indexes from v12" means indexes on a
> live PostgreSQL 12 server: does this statement parse, run, and stay correct when
> it is pointed at a 12 server. Indexes carried onto a 17 server by `pg_upgrade`
> are out of scope; see [Open Questions](#open-questions).

Follow-up: Propose a SQL statement to measure bloat on PostgreSQL v12 and later versions with support of deduplication.

> Prompt note: filed as an approved grammar-corrected restatement of "propose a
> sql to measure bloat on postgresql v12 and later versions with support of
> deduplication", per the repository's prompt-hygiene rule. The asker scoped
> "v12 and later versions" to servers 12 through 17 — one statement that runs
> unchanged on any of them and credits deduplication only where the catalog
> proves the engine would deduplicate — and asked for the proposal to be built
> and measured on servers, not only derived from source. Majors 13, 15 and 16
> have no checkout in this repo and were not run; see
> [Open Questions](#open-questions).

Follow-up: two defects in the PostgreSQL 12-through-17 bloat statement on this
page are reporting defects rather than arithmetic defects, because neither
changes expected_blocks. Confirm both against the pinned PostgreSQL 17 source
and correct the statement.

First, the has_freed_pages column is fsm_bytes > 0, which reports only that a
free space map fork exists with nonzero length. Establish what that expression
can and cannot prove about the index's current state, in both directions: an
index whose recorded free pages have all been handed out again, and an index
whose pages are deleted but not yet recyclable. State what a reader may
legitimately conclude from the column, rename it accordingly, and say which
core-SQL or contrib source could supply a current count of empty, deleted and
reusable pages instead.

Second, wasted is clamped at zero while bloat_pct is not, so one row can report
zero wasted bytes beside a large negative percentage. Establish what a negative
bloat_pct means about the model, name which consumers of the output the mixed
convention breaks, and choose one convention for both columns that keeps the
over-prediction visible rather than hiding it at zero.

For both defects, give the corrected column expressions, state whether the
correction changes any reported bloat percentage on the fixtures already measured
on this page, and say whether the earlier deduplication-aware sweep for v17 needs
the same two corrections.

Follow-up: add a correction. In the SQL, do not use bloat as the output; use
wasted_space.

> Prompt note: filed as an approved grammar-corrected restatement of "add
> correction: on the sql don't use bloat as the output but use wasted_space", per
> the repository's prompt-hygiene rule. The asker scoped "the output" to the
> statements' output column names and confirmed that all three are renamed
> (`wasted` -> `wasted_space`, `bloat_pct` -> `wasted_space_pct`,
> `bloat_pct_floor` -> `wasted_space_pct_floor`), that this page's prose follows
> the new names, and that the two `wiki_btree_bloat_sweep_*` statement tags are
> renamed too.

Follow-up on the PostgreSQL 17 page
wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md, section
"Follow-up: one statement for PostgreSQL 12 through 17".

An external review of that statement listed twelve issues. Establish first, against
the pinned PostgreSQL 17 checkout, that four of them are already fixed in the
statement as filed (the negative n_distinct branch, the nondeterministic-collation
conjunct in the deduplication gate, the INCLUDE-column conjunct
indnatts = indnkeyatts, and the two reporting defects that became
fsm_written_since_build and wasted_space_pct), and that one is mischaracterized.
Then apply exactly the five changes below and no others.

1. Correct the posting-tuple count so that each key group rounds up. In the
   classpages CTE, replace the tuple count class_rows / greatest(tids, 1) with
   least(class_groups * ceil(class_rows / class_groups / greatest(tids, 1)),
   class_rows), carrying class_groups through classfit as needed. Show the
   arithmetic on the page's own i_q1000 fixture (1000 keys, 1000 rows per key,
   nmax 132: 7576 tuples modelled against 8000 required) and state whether this
   accounts for the 847-modelled-against-896-rebuilt gap the page currently
   attributes to page packing, or only part of it. Re-derive i_cd (424 against
   462) under the corrected form.

2. Use extended statistics for multicolumn keys. When a CREATE STATISTICS object
   with the ndistinct kind covers the index's key columns, prefer
   pg_stats_ext.n_distinct over the per-column product now used in key_groups,
   and fall back to the product when no such object exists or its rows are not
   visible. State the privilege condition pg_stats_ext imposes, and how the
   statement behaves when the object exists but its entry does not cover the
   exact key-column set.

3. Add a caveat for invisible statistics. pg_stats drops rows the caller cannot
   see, and the current coalesce(..., 0) turns that into an all-distinct
   assumption, so the sweep silently degrades to the floor model and
   under-reports bloat on duplicate-heavy indexes. Distinguish "no statistics
   row exists" from "statistics exist but are not visible to this role" as far
   as core SQL allows, emit a caveat for it, and say whether that caveat belongs
   in the alert-suppression set alongside "never analyzed" and "row-count
   sources disagree".

4. Add a fixture for a randomly inserted, never-deleted index and measure it.
   Every existing reclaimable-space fixture is delete-driven or a statistics
   trap, so the page's "0 false positives at a 30% threshold on all three
   servers" result was never tested against an index whose leaves are at
   split-point density rather than fillfactor density. Report the live density,
   the rebuilt size, the point estimate, and the floor, and state plainly that
   this is the one valid issue the floor-plus-status-plus-caveats rule cannot
   defend against, because with distinct keys the floor equals the point
   estimate.

5. Sharpen the framing of the baseline. The model's reference is a sorted build
   at the index's fillfactor, which is what a rebuild produces; a live index's
   non-rightmost leaf splits target a different occupancy. Say what the output
   therefore means for a healthy index under steady random insertion, and how a
   reader should decide between "a rebuild would reclaim this" and "a rebuild is
   worth doing".

Record two rejected fixes, with reasons from source:

A. Do not replace least(reltuples, n_live_tup). n_live_tup is a live-row count
   maintained by DML deltas without ANALYZE, live rows are the correct rebuild
   baseline, and that expression is what makes the i_stale fixture read
   correctly. Name what the sweep would lose if it used reltuples alone.

B. Do not source the deduplication gate from bt_metap().allequalimage. The model
   predicts a fresh build, the build path recomputes the flag, an index carried
   over by pg_upgrade from PostgreSQL 12 reports false while a rebuild would set
   it true, and the function is superuser-only, which breaks the page's
   core-SQL-only constraint. State which question the metapage flag does answer.

Deliverable and constraints:

- Extend the existing question page as a new follow-up section. Do not create a
  new document, and do not renumber or rewrite the earlier follow-ups.
- Keep the two-column output contract (wasted_space_pct and
  wasted_space_pct_floor) and the wasted_space naming.
- For every change, state the direction of the error it fixes (does it produce
  false positives or false negatives), its measured or derived magnitude, and
  whether the floor-based alert rule was already immune to it.
- Verify that the statement still runs unchanged on PostgreSQL 12 and that every
  new construct exists in 12. If any does not, say so and give the fallback.
- Cite the pinned PostgreSQL 17 checkout for every behavioral claim. If a change
  is not measured on a server, put the unmeasured part under Open Questions
  rather than presenting derived numbers as measurements.

> Prompt note: filed verbatim, at the asker's direction, with two counts recorded
> here rather than silently corrected. First, the parenthesis lists five items for
> "four of them are already fixed"; the asker confirmed the two reporting defects
> count as one review issue, which makes the ledger 4 already fixed + 1
> mischaracterized + 5 applied + 2 rejected = 12. Second, the second reporting
> defect became the **signed `wasted_space` byte column**, not `wasted_space_pct`:
> `wasted_space_pct` is the percentage column, which was never defective and was
> renamed from `bloat_pct` by the fifth follow-up. The asker also named the
> mischaracterized issue: the attribution of the `i_q1000` gap to page packing.

Follow-up: in PostgreSQL 17, add a mandatory-tests section to this question page,
and make sure that all the proposed SQL passes all the mandatory tests. Add these
test requirements for deduplication:

- With a normal integer B-tree index: deduplication should be enabled.
- With a bigint B-tree index: deduplication should be enabled.
- With a text index using a deterministic collation: deduplication should be
  enabled.
- With a text index using a nondeterministic collation: deduplication must be
  disabled.
- With a numeric index: deduplication must be disabled.
- With a float4 or float8 index: deduplication must be disabled.
- With a multi-column index where all columns support equalimage, for example
  (integer, bigint): deduplication should be enabled.
- With a multi-column index where one column does not support equalimage, for
  example (integer, numeric): deduplication must be disabled.
- With an expression index, for example lower(text_column): verify that
  equalimage is evaluated using the index expression's opclass and collation.
- With an INCLUDE index: deduplication must be disabled.
- With deduplicate_items = off: deduplication must be disabled even if
  all_equalimage = true.
- With a custom opclass that has no equalimage support function: deduplication
  must be disabled.
- With a custom opclass where support function #4 exists but returns FALSE:
  all_equalimage must be FALSE and deduplication must be disabled. This is the
  main regression test for the original bug.
- With a custom opclass where support function #4 exists and returns TRUE:
  verify that all_equalimage is TRUE if the implementation can safely determine
  the result.
- Test a multi-column index where one key returns TRUE and another returns
  FALSE: the final all_equalimage must be FALSE.
- Test unknown or unsupported equalimage cases: they must resolve to FALSE,
  never TRUE.
- Run the same basic positive and negative tests to verify version
  compatibility.

> Prompt note: filed as an approved grammar-corrected restatement of "follow
> agents.md, in postgresql 17 , for question: Testing the PostgreSQL 12 Core-SQL
> B-Tree Bloat Method on PostgreSQL 17 (unverified) , create a section with
> mandatory tests to question, make sure that all proposed sql executes all
> mandatory tests and add these tests requirements for deduplication: ...", per
> the repository's prompt-hygiene rule. The seventeen requirements were reflowed
> to one bullet each; none was added, dropped, merged or reordered. The asker
> scoped "all the proposed SQL" to the two deduplication-aware statements on this
> page — [the corrected statement with all five
> changes](#the-corrected-statement-with-all-five-changes) and [the earlier
> deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17) —
> directed that a statement failing a mandatory test be corrected rather than
> only reported, and confirmed that "the original bug" is the existence-only
> equal-image gate: `EXISTS (... amprocnum = 4)` cannot see the support
> function's return value.

Follow-up: in PostgreSQL 17, add to this question the requirement to keep a
section that points to the current recommended SQL, selected as the most
accurate one, with the most fixes and the widest compatibility.

> Prompt note: filed as an approved grammar-corrected restatement of "follow
> agents.md, in postgresql 17 , for question: Testing the PostgreSQL 12 Core-SQL
> B-Tree Bloat Method on PostgreSQL 17 (unverified) , add to question the
> requirement to keep a section that point to the current recommended sql, select
> it based on the most acurate and with more fixes and more compatible", per the
> repository's prompt-hygiene rule. The asker confirmed three scoping decisions.
> The section points at a statement already on this page instead of restating its
> SQL, so no second copy can drift out of step; it names the exact substitution
> needed to assemble the recommendation from the two sections that hold it; and it
> sits directly after [Verdict](#verdict) rather than at the end of the page, so a
> reader meets it before the filing-order history. "Keep" is read as a standing
> requirement: that section is this page's single pointer to the recommended SQL,
> and it must be updated whenever a later follow-up changes the answer.

## Answer

### Verdict

Every statement in the v12 page runs unchanged on 17.10, and on indexes with distinct keys it is **exactly as accurate**: Method A hit the rebuilt block count to the block on 9 of 14 fixtures and on all 24 non-partial matrix cells outside the duplicate-key type, and Method B reproduced `pgstatindex.leaf_pages` exactly on all 12 eligible fixtures and all 36 eligible matrix cells — the same outcome the v12 page reports for v12.

One v13 feature breaks it. B-tree **deduplication** merges equal keys into posting-list tuples, both when a page is about to split and when a fresh index is built ([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1160)). The v12 page-fill model charges one full index tuple per row, so wherever duplicates exist it overestimates the rebuilt size, and the error scales with duplication:

| Symptom | v17 measurement |
|---|---|
| Method A on an all-duplicate 1,000,000-row index | model 2745 blocks, rebuild 849 blocks — **+223.3%** |
| Method A on 5 rows per key | model 2745, rebuild 1426 — **+92.5%**, on an index with 0% reclaimable |
| Method A on 25% NULL keys | model 2745, rebuild 2271 — **+20.9%** |
| Method B density formula on the same cells | 313.58% against `pgstatindex`'s 96.15% — **+217 points** |

Three smaller v17 differences also matter: `VACUUM VERBOSE` no longer prints an index row count and can skip the index line entirely (Method D), `pg_class.reltuples` now uses `-1` for "unknown" and turns a healthy 21 MB index into a 100.0% bloat reading, and a plain `ANALYZE` after the build costs partial-index cells their exactness. The [deduplication-aware sweep below](#a-deduplication-aware-sweep-for-v17) restores the worst case from +1896 blocks to −24 blocks (2.9%) without changing a single already-exact cell.

That corrected sweep is also safe to run against a PostgreSQL 12 server, where it silently reduces to the v12 page's own Method A; the follow-up sections measure it on both a 12.2 and a 17.10 server and name the one case where it is wrong on 17.10 — see [Follow-up: the same sweep on a 12.2 server and a 17.10 server](#follow-up-the-same-sweep-on-a-122-server-and-a-1710-server).

A later follow-up replaces it with a single statement intended for any server from 12 through 17, measured on 12.2, 14.23 and 17.10: exact posting-tuple arithmetic instead of a flat 6 bytes per row, a NULL-and-most-common-value key-group mixture, a nondeterministic-collation conjunct in the gate, both `reltuples` eras, and a second `wasted_space_pct_floor` column to alert on — see [Follow-up: one statement for PostgreSQL 12 through 17](#follow-up-one-statement-for-postgresql-12-through-17).

A fourth follow-up corrects two reporting defects that both statements shipped with — a free-space-map boolean that reports history rather than current state, and a byte column clamped at zero beside an unclamped percentage. Neither touches `expected_blocks`, so no percentage on this page moves; see [Follow-up: two reporting defects, not arithmetic defects](#follow-up-two-reporting-defects-not-arithmetic-defects).

A fifth follow-up renames the three reporting columns and both statement tags off the word "bloat": the statements now emit `wasted_space`, `wasted_space_pct` and `wasted_space_pct_floor`. That is an `AS` label change and nothing else, and it is worth making because the PostgreSQL glossary's "bloat" is per-page state that no core-SQL method here can see; see [Follow-up: the output columns say wasted_space, not bloat](#follow-up-the-output-columns-say-wasted_space-not-bloat).

A sixth follow-up works a twelve-issue external review of the portable statement, on a server built from the **current** 17.11 pin rather than the 17.10 one every earlier number came from. Four issues were already fixed, one — the page's own "page packing" attribution — is mischaracterized, five changes are applied, and two proposed fixes are rejected with reasons from source. The load-bearing results: extended statistics turn a genuinely 49.8%-reclaimable correlated two-column index from a missed `−38.3%` into `+49.9%`; a randomly inserted, never-deleted index reports **27.1% on both columns while the model is exactly right**, which is the one failure the floor-based alert rule cannot defend against; and the prescribed posting-tuple round-up is not a strict improvement — it removes every phantom-bloat reading above 264 rows per key and introduces over-prediction up to `−99.4%` just above a multiple of the 132-TID cap. See [Follow-up: five changes from a twelve-issue review](#follow-up-five-changes-from-a-twelve-issue-review).

### The current recommended statement

**Use [The corrected statement, with all five changes](#the-corrected-statement-with-all-five-changes) with its `all_equalimage` subquery replaced by the one in [Change 6](#change-6-name-the-support-function-do-not-just-count-it).** That pairing is the newest, most-fixed and most-portable variant on this page: it carries every correction filed here, it is the only variant whose deduplication gate agrees with the engine on all seventeen mandatory tests, and it was executed on both a 12.2 and a 17.11 server. It keeps the tag `wiki_btree_wasted_space_sweep_12_17` and the output contract `wasted_space_pct`, `wasted_space_pct_floor` and a signed `wasted_space`.

This section is the page's single pointer to that answer, and it is meant to be kept current. The rest of the page is in filing order, so the statement a reader meets first is not the one to run; whichever statement currently wins on accuracy, fixes and version coverage is named here.

**Assembling it, in two steps.**

1. Take the SQL block under [The corrected statement, with all five changes](#the-corrected-statement-with-all-five-changes).
2. In its `idx` CTE, replace the ten lines from `-- deduplication gate, in catalog terms: every key opclass must` through `WHERE k < x.indnkeyatts)                   AS all_equalimage,` with the block under [Change 6](#change-6-name-the-support-function-do-not-just-count-it), which ends in the same `AS all_equalimage,` label.

Nothing else changes: not the `all_deterministic` conjunct directly below it, not `dedup_applies`, not either model, not a column, not the two leading `SET`s. The 17.11 and 12.2 runs behind the mandatory-test verdicts were generated exactly this way, from this page's own SQL text, so the assembled statement is a text this page has run rather than a text it only describes.

**Why this variant, on the three criteria.** Each row below adds to the row above it, so "more fixes" is a superset relation rather than a judgement:

| Variant on this page | What it adds | Gate errors over the 28 mandatory-test fixtures | Servers it ran on |
|---|---|---|---|
| the v12 page's Method A, run verbatim | nothing; one index tuple charged per row | no gate at all, and not run on those fixtures; its worst phantom reading elsewhere on this page is `+223.3%` on an all-duplicate 1,000,000-row index with nothing to reclaim | 12.2, 14.23, 17.10 |
| [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17) | the posting-list term, `n_distinct > 0`, `deduplicate_items`, non-unique, and the `reltuples = -1` guard | 8 over-credits, 0 under-credits; 4 fixtures above 30% | 12.2, 17.10, 17.11 |
| [One statement for PostgreSQL 12 through 17](#follow-up-one-statement-for-postgresql-12-through-17) | exact posting-tuple arithmetic, MCV key classes, the `INCLUDE` and nondeterministic-collation conjuncts, the pre-14 stale zero, and the `wasted_space_pct_floor` column | gate expression identical to the two rows below; not scored separately on those fixtures | 12.2, 14.23, 17.10 |
| the same, [with both reporting corrections](#follow-up-two-reporting-defects-not-arithmetic-defects) and [the rename](#follow-up-the-output-columns-say-wasted_space-not-bloat) | `fsm_written_since_build`, a signed `wasted_space`, and the `wasted_space*` labels | reporting only; no percentage moves | source-only follow-ups |
| [The corrected statement, with all five changes](#the-corrected-statement-with-all-five-changes) | key-group round-up, extended statistics, the invisible-statistics caveat, and the baseline framing | 5 over-credits, 0 under-credits; 3 fixtures above 30% on `wasted_space_pct`, 0 on the floor | 12.2, 17.11 |
| **the same plus [Change 6](#change-6-name-the-support-function-do-not-just-count-it) — recommended** | the `prosrc` whitelist in place of the existence test | **0 over-credits, 1 under-credit; 0 fixtures above 30% on either column** | 12.2, 17.11 |

- **Most accurate.** Over-crediting is the dangerous direction, because it invents reclaimable space on a healthy index, and the recommended text is the only variant with none: the worst filed over-credit reports 78.1% waste on a 1931-block index a rebuild would reproduce block for block, and change 6 removes all five ([The seventeen mandatory tests, and the verdict on each](#the-seventeen-mandatory-tests-and-the-verdict-on-each), [What the mixed-key failure costs](#what-the-mixed-key-failure-costs)). The five changes supply the rest: `i_q1000` moves from `+5.5%` to `−0.6%`, `i_ext` and `i_sup` from `−206.4%` to `+0.1%`, and the genuinely 49.8%-reclaimable `i_ext50` from an unalertable `−38.3%` to `+49.9%` ([Measured on 17.11, per fixture](#measured-on-1711-per-fixture)).
- **Most fixes.** Six numbered changes on top of the portable statement's own gate conjuncts and both reporting corrections, and the gate it ends with is the catalog form of what the engine actually does: `_bt_allequalimage` looks the support function up **and calls it**, so a registered function that returns false is the same outcome as no function at all ([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183), [nbtutils.c:5156-5169](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5156-L5169)); a fresh build recomputes that answer from the current opclasses ([nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563)) and `_bt_load` deduplicates only when it, non-uniqueness and the reloption all agree ([nbtsort.c:1151-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)); the documentation states the same rule from the operator-class side ([btree.sgml#equalimage](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L499-L509)); and `prosrc`, not `proname`, is the identity the engine resolves for a `LANGUAGE internal` function ([fmgr.c:216-240](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240), [fmgr.c:166-178](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L166-L178)).
- **Most compatible.** It runs unchanged on 12 through 17, and change 6 adds no construct that 12 lacks: `pg_language.lanname` and `pg_proc.prosrc` both exist on 12.2, where the gate cannot open anyway because no B-tree opfamily has an `amprocnum = 4` row ([The mandatory tests on a 12.2 server](#the-mandatory-tests-on-a-122-server), [The corrected statement on a 12.2 server](#the-corrected-statement-on-a-122-server)). Measured coverage is 12.2 and 17.11 for this exact text, plus 14.23 and 17.10 for the pre-change-6 text ([How the same statement behaves on 12.2, 14.23 and 17.10](#how-the-same-statement-behaves-on-122-1423-and-1710)); majors 13, 15 and 16 have no checkout in this repo and were never run.

**What it costs, and in which direction.** Four residual errors survive. Three over-predict the rebuilt size, which surfaces as a negative reading — useless for sizing, harmless for a floor-based alert — and the fourth is the one honest false positive on this page, an index whose model is exactly right but whose reclaimable space comes back under continued random insertion:

| Residual error | Measured | Direction |
|---|---|---|
| change 1 prices each group's last, partial posting tuple as a full one | `−99.4%` at 133 rows per key, `−88.7%` at 143, `−33.5%` at 265 ([Change 1](#change-1-round-each-key-group-up-to-whole-posting-tuples)) | over-prediction, and not a strict improvement over the filed form |
| change 2 on independent columns | `i_ind2` moves from `−2.0%` to `−88.6%` ([Measured on 17.11, per fixture](#measured-on-1711-per-fixture)) | over-prediction |
| change 6 cannot call a custom support function, so it answers FALSE | `i_ei_true` reads `−226.4%`; adding a working `FUNCTION 4` to an opclass that had none makes a rebuild reclaim a true 69.4% that this gate reports as 0.1% ([What change 6 costs](#what-change-6-costs)) | one under-credit, i.e. a missed rebuild win, on custom operator classes only |
| a randomly inserted, never-deleted index | `27.1%` on both columns with the model exact to the block ([Change 4](#change-4-a-randomly-inserted-never-deleted-index)) | false positive that no gate or caveat catches |

**How to read the output.** Alert on `wasted_space_pct_floor`, not on the point estimate, and only when `status` is `ok` and `caveats` holds none of `never analyzed`, `row-count sources disagree: analyze first` or `statistics not visible to this role`; read `wasted_space_pct` and `wasted_space` for sizing only, and treat a wide gap between the two percentages as "this answer rests on a duplication estimate" ([Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate), [Change 5](#change-5-what-the-baseline-is-and-what-a-reading-means)). The statement sets `statement_timeout = '30s'` and `lock_timeout = '2s'` itself; both are `PGC_USERSET` ([guc_tables.c:2611-2631](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631)), so they apply at session/transaction scope and need no reload or restart. The gate swap costs no measurable time: 12.2 / 12.5 / 16.6 ms over 28 indexes and 34,164 blocks against 12.8 / 12.1 / 14.0 ms for the filed text, overlapping ranges, because the two added joins are on syscache-backed `pg_proc` and `pg_language` ([What change 6 costs](#what-change-6-costs)).

**Before adopting the gate change on an existing database**, run the zero-fixture audit query in [The harness, runnable](#the-harness-runnable). It lists exactly the indexes whose reading depends on the swap, and it returned 6 rows in the custom-opclass fixture database against 0 on a stock 17.11 database and 0 on 12.2, because every stock B-tree equal-image row names `btequalimage` or `btvarstrequalimage` and both are `LANGUAGE internal` in `pg_catalog` ([pg_amproc.dat:143](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L143), [pg_amproc.dat:206](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L206), [pg_amproc.dat:241](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L241)).

**What the recommendation does not replace.** The recommended statement is a catalog-only estimate of a rebuilt size, so three other methods on this page keep their own jobs: [Method C](#method-c-unchanged-answer-different-write-path), a `CREATE INDEX CONCURRENTLY` copy, is the only exact answer and is the arbiter every accuracy number here is scored against; [Method B](#method-b-leaf-counts-still-exact-density-formula-not) is the way to count live leaf pages without contrib, exact on 48 of 48 eligible cells; and [Method A-prime](#method-a-prime-still-fixes-variable-key-width) is the sampled-width repair for variable-width keys, which no variant of the sweep incorporates. [The earlier v17 sweep](#a-deduplication-aware-sweep-for-v17) is superseded for every purpose except reading this page in order, and if it is kept in place it needs the three conjuncts in [The earlier v17 sweep needs three conjuncts](#the-earlier-v17-sweep-needs-three-conjuncts).

**What would displace this recommendation.** Update this section when a later follow-up files a statement that is more accurate on the fixtures already measured here, carries a fix this one lacks, or widens the version coverage that was actually run. When a candidate wins on one criterion and loses on another — as change 1 does, and as change 6's single under-credit does — record the trade here instead of switching silently, and keep the losing direction named, since an over-prediction and an over-credit are not interchangeable.

### How the test was run

One isolated PostgreSQL 17.10 server, built out of tree from the pinned checkout under `.wiki-runtime/`, `block_size` 8192, `autovacuum = off`, `fsync = off`. `pgstattuple` was installed **only as ground truth**; no method below uses it.

Measurement provenance: every number on this page was produced on that 17.10 server, built from the **previous** pin `54eeefaedbee0385529f3edf321bb99e49232aaa`. Nothing was re-measured for the 2026-08-17 repin to 17.11 (`786db8dcf168bd9df8f55047337525ac19118b1c`). Two commits in that range touch code these methods read, and neither moves a number here:

- `355faed5a24` rewrote `IndexOnlyNext()` and `StoreIndexTuple()` so each tuple is deformed with the descriptor the index AM formed it with ([nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L62-L240), [nodeIndexonlyscan.c#StoreIndexTuple](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L255-L281)). The fix is for the GiST/SP-GiST `xs_hitup` path; the B-tree path still deforms `xs_itup` with `xs_itupdesc`, so Method B's index-only-scan census keeps the same semantics.
- `8434c938598` added an empty-index recheck to `_bt_endpoint`, but only inside `if (IsolationIsSerializable())` ([nbtsearch.c#_bt_endpoint-empty-index](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2590-L2606)). No fixture or method on this page runs in a `SERIALIZABLE` transaction.

Two populations, both taken from the v12 page's own descriptions:

- the **15 named fixtures** (`idx_seq` … `idx_empty`), each rebuilt from the recipe in the v12 page's fixture table;
- the **54-cell matrix**: 9 bloat types x 3 scales (200,000 / 500,000 / 1,000,000 rows) x {non-partial, partial}, two indexes per table over the same key, `flag = (id % 5 = 0)` so the partial index holds 20% of the rows, delete patterns on modulus 7 and 11.

Ground truth per index is a `CREATE INDEX CONCURRENTLY` rebuild (Method C, exact reclaimable size) plus `pgstatindex` page classes. The v12 page's SQL was executed verbatim; only the `actual_bytes > 1024 * 1024` triage filter was removed so that sub-megabyte partial indexes are scored, and `expected_blocks` was exposed so the model can be diffed against the rebuild.

The recipes are reconstructions. Where a v17 number differs from the v12 page's, the mechanism is named below and checked against v17 source; where the difference is a fixture artifact rather than a version change, it is called out as one.

### The fifteen fixtures on 17.10

`pgstatindex` ground truth, with the v12 page's reported figures beside it:

| index | v17 blocks | v17 leaf | v17 internal | v17 deleted | v17 density | v12 page blocks / density | same? |
|---|---|---|---|---|---|---|---|
| `idx_seq` | 2745 | 2733 | 11 | 0 | 90.06 | 2745 / 90.06 | identical |
| `idx_uuid` | 1543 | 1533 | 9 | 0 | 90.01 | 1543 / 90.01 | identical |
| `idx_text` | 3607 | 3572 | 34 | 0 | 89.98 | 3607 / 89.98 | identical |
| `idx_ff50` | 4971 | 4951 | 19 | 0 | 49.85 | 4971 / 49.85 | identical |
| `idx_part` | 139 | 137 | 1 | 0 | 89.83 | 139 / 89.83 | identical |
| `idx_del` | 2745 | 2733 | 11 | 0 | 9.27 | 2745 / 9.27 | identical |
| `idx_range` | 2745 | 411 | 3 | 2330 | 89.83 | 2745 / 89.83 | identical |
| `idx_stale` | 2745 | 2733 | 11 | 0 | 90.06 | 2745 / 90.06 | identical |
| `idx_empty` | 1 | 0 | 0 | 0 | `NaN` | 1 / `NaN` | identical |
| `idx_multi` | 3587 | 3572 | 14 | 0 | 89.58 | 3606 / 89.96 | fixture artifact |
| `idx_var` | 3211 | 3171 | 39 | 0 | 90.32 | 3169 / 90.43 | fixture artifact |
| `idx_rand` | 3788 | 3773 | 14 | 0 | 65.32 | 3758 / 65.81 | different seed |
| `idx_null` | 2271 | 2260 | 10 | 0 | 90.07 | 2746 / 90.09 | **v17 deduplication** |
| `idx_dup` | 396 | 392 | 3 | 0 | 96.12 | 1291 / 96.00 | **v17 deduplication** |
| `idx_churn` | 2471 | 2460 | 10 | 0 | 74.80 | 3293 / 67.63 | different update pattern |

Nine of the fifteen reproduce the v12 page's block count to the block, which is the control this comparison needs: the page-fill arithmetic itself did not move. `_bt_pagestate` still starts a leaf "full" threshold at `BLCKSZ * (100 - fillfactor) / 100` and internal levels at `BTREE_NONLEAF_FILLFACTOR` 70 ([nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)), `_bt_blnewpage` still pre-reserves the high-key line pointer ([nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629)), and a page is still finished when `PageGetFreeSpace()` drops below that threshold ([nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L809-L855)). The usable-space constant is unchanged as well ([nbtsplitloc.c:157-160](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L157-L160)).

`idx_multi`'s 14 internal pages against the v12 page's 33 is a fixture difference, not a version difference: this run's leading `int` column is unique, so `_bt_truncate` strips the trailing `text` attribute from every pivot tuple and the internal fanout roughly triples. The leaf count, 3572, is identical.

### Method A against a rebuild

Method A run verbatim, scored against the Method C rebuild:

| index | live | rebuilt | Method A | model − rebuilt | model bloat % | true bloat % |
|---|---|---|---|---|---|---|
| `idx_seq` | 2745 | 2745 | 2745 | 0 | 0.0 | 0.0 |
| `idx_uuid` | 1543 | 1543 | 1543 | 0 | 0.0 | 0.0 |
| `idx_text` | 3607 | 3607 | 3607 | 0 | 0.0 | 0.0 |
| `idx_ff50` | 4971 | 4971 | 4971 | 0 | 0.0 | 0.0 |
| `idx_empty` | 1 | 1 | 1 | 0 | 0.0 | 0.0 |
| `idx_rand` | 3788 | 2745 | 2745 | 0 | 27.5 | 27.5 |
| `idx_churn` | 2471 | 825 | 825 | 0 | 66.6 | 66.6 |
| `idx_range` | 2745 | 414 | 414 | 0 | 84.9 | 84.9 |
| `idx_del` | 2745 | 276 | 276 | 0 | 89.9 | 89.9 |
| `idx_stale` | 2745 | 276 | 276 | 0 | 89.9 | 89.9 |
| `idx_part` | 139 | 139 | 135 | −4 | 2.9 | 0.0 |
| `idx_multi` | 3587 | 3587 | 3607 | +20 | −0.6 | 0.0 |
| `idx_var` | 3211 | 3211 | 3316 | +105 | −3.3 | 0.0 |
| `idx_null` | 2271 | 2271 | 2745 | **+474** | −20.9 | 0.0 |
| `idx_dup` | 396 | 426 | 1374 | **+948** | −247.0 | −7.6 |

Nine cells exact, three inside the same error classes the v12 page documents (partial-index sampling, internal-page modelling, variable key width), and two that are new on 17. Both new failures are deduplication.

`idx_dup` also reproduces the v12 page's "a rebuild can make an index bigger" result, with different numbers: the live index sits at 96.12% density because an all-duplicate split uses `BTREE_SINGLEVAL_FILLFACTOR` ([nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416)), while the sorted rebuild caps each posting list at a tenth of a page, so rebuilding grew it from 396 to 426 blocks (−7.6%).

### Deduplication is what breaks the model

Two independent code paths merge equal keys into a single posting-list tuple that stores one copy of the key plus a 6-byte `ItemPointerData` per row.

**Build path.** `_bt_load` turns deduplication on for any non-unique index whose opclass is all-equal-image and whose `deduplicate_items` reloption is on, which is the default ([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1160), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1150)). Posting lists built this way are capped at `MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)`, i.e. 812 bytes at the default block size ([nbtsort.c:1292-1308](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1292-L1308)), and a TID is appended only while `MAXALIGN(basetupsize + nhtids * sizeof(ItemPointerData))` stays under that cap ([nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L484-L531)). For an 8-byte key that is 132 TIDs in an 808-byte tuple.

**Insert path.** A page about to split is deduplicated first, with a larger cap of `BTMaxItemSize(page) / 2` ([nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L75-L93)).

Because Method A charges `MAXALIGN(hoff + data) + 4` per **row**, it cannot see either. Measured on the 54-cell matrix, the `dup` rows are the only ones that move:

| cell | live | rebuilt | Method A | error |
|---|---|---|---|---|
| `m_dup_200_full` | 159 | 171 | 551 | +380 (+222.2%) |
| `m_dup_500_full` | 396 | 426 | 1374 | +948 (+222.5%) |
| `m_dup_1000_full` | 789 | 849 | 2745 | +1896 (+223.3%) |
| `m_dup_200_part` | 34 | 36 | 111 | +75 (+208.3%) |
| `m_dup_500_part` | 81 | 87 | 279 | +192 (+220.7%) |
| `m_dup_1000_part` | 159 | 171 | 557 | +386 (+225.7%) |

The v12 page reports these same six cells at 4 blocks / 0.2% and 2 blocks / 0.8%.

### The duplication-ratio sweep

Duplication is not binary, so a second exact-pin fixture set fixed the row count at 1,000,000 and varied only the rows per distinct key (`id = g / q`), each with `VACUUM (ANALYZE)` and a `CREATE INDEX CONCURRENTLY` rebuild:

| rows per key | `pg_stats.n_distinct` | live | rebuilt | true bloat | Method A | Method A error | dedup-aware model | its error |
|---|---|---|---|---|---|---|---|---|
| 2 | −0.509185 | 2475 | 2475 | 0.0% | 2745 | +270 (+10.9%) | 2745 | +270 |
| 5 | −0.208039 | 1426 | 1426 | 0.0% | 2745 | +1319 (+92.5%) | 2745 | +1319 |
| 10 | 97311 | 1157 | 1157 | 0.0% | 2745 | +1588 (+137.3%) | 1012 | −145 (−12.5%) |
| 20 | 50492 | 950 | 950 | 0.0% | 2745 | +1795 (+188.9%) | 922 | −28 (−2.9%) |
| 100 | 9991 | 839 | 839 | 0.0% | 2745 | +1906 (+227.2%) | 844 | +5 (+0.6%) |
| 1000 | 1000 | 896 | 896 | 0.0% | 2745 | +1849 (+206.4%) | 827 | −69 (−7.7%) |

Every one of these indexes is freshly built and has **zero** reclaimable space. The v12 sweep reports 10.9% to 227.2% phantom bloat on all six, and its answer never changes, because the model has no term that depends on duplication. At 5 rows per key it claims 92.5% of a 22 MB index is wasted when a rebuild reclaims nothing.

### NULL runs are deduplicated too

`idx_null` (1,000,000 `bigint`, 25% NULL) is 2271 blocks on 17.10 against 2745 predicted. NULLs are ordinary index entries and they are all equal to each other, so the 250,000 NULL rows collapse into posting lists exactly like any other duplicate run. This is worth separating from the `dup` case because `null_frac` is a directly measured statistic, so the correction below is reliable here even when `n_distinct` is not.

### A deduplication-aware sweep for v17

The fix is one extra term and one gate, both derived from source. Where the index can deduplicate, the leaf requirement becomes `key_groups * slot + (rows - key_groups) * 6` bytes instead of `rows * slot`, since each extra row after the first in a key group costs one `ItemPointerData` ([nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L484-L531)).

The gate matters as much as the term. `ANALYZE` stores `stadistinct` as a **positive absolute count** only when it estimated at most 10% of the table's rows to be distinct; above that it converts to a negative fraction ([analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612)), and the negative form is the unreliable one. This run measured `t_var` at `n_distinct = -0.87447` against a true 392,917 of 400,000 distinct values; crediting that 11% shortfall as deduplication moved the `idx_var` estimate from +105 to −271 blocks. So the sweep credits duplicate compression only when `n_distinct > 0`, and always accounts for NULLs from `null_frac`.

Three more source-level conditions gate the whole term: build-time deduplication is skipped for unique indexes (`!btspool->isunique`), requires `deduplicate_items` (default on), and requires an all-equal-image opclass, whose support function is visible in core SQL as `pg_amproc.amprocnum = 4`.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

WITH RECURSIVE
idx AS (
    SELECT /* wiki_btree_wasted_space_sweep_v17 */
           c.oid AS idxoid, n.nspname AS schemaname, t.relname AS tablename,
           c.relname AS indexname, t.oid AS tbloid, x.indkey, x.indisunique,
           (x.indpred IS NOT NULL)                      AS is_partial,
           coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'fillfactor'), 90)  AS fillfactor,
           coalesce((SELECT option_value::bool FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'deduplicate_items'), true) AS dedup_on,
           CASE WHEN c.reltuples = -1 THEN NULL
                WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
                ELSE least(c.reltuples::numeric,
                           coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
           END                                          AS live_rows,
           c.reltuples::numeric                         AS idx_reltuples,
           coalesce(s.n_dead_tup, 0)                    AS tbl_n_dead_tup,
           greatest(s.last_vacuum, s.last_autovacuum)   AS last_vacuum,
           greatest(s.last_analyze, s.last_autoanalyze) AS last_analyze,
           pg_relation_size(c.oid)                      AS actual_bytes,
           pg_relation_size(c.oid, 'fsm')               AS fsm_bytes,
           current_setting('block_size')::int           AS bs,
           (SELECT bool_and(EXISTS (SELECT 1 FROM pg_amproc ap
                                     WHERE ap.amprocfamily = op.opcfamily
                                       AND ap.amproclefttype = op.opcintype
                                       AND ap.amprocrighttype = op.opcintype
                                       AND ap.amprocnum = 4))
              FROM generate_subscripts(x.indclass, 1) k
              JOIN pg_opclass op ON op.oid = x.indclass[k]
             WHERE k < x.indnkeyatts)                   AS all_equalimage
      FROM pg_class c
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_am am       ON am.oid = c.relam
      LEFT JOIN pg_stat_all_tables s ON s.relid = t.oid
     WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
),
cols AS (
    SELECT i.idxoid, a.attlen, a.attalign,
           CASE WHEN a.attlen > 0 THEN a.attlen::numeric
                ELSE coalesce(se.avg_width, st.avg_width, 32)::numeric END AS width,
           coalesce(se.null_frac, st.null_frac, 0)::numeric                AS null_frac,
           coalesce(se.n_distinct, st.n_distinct, -1)::numeric             AS n_distinct
      FROM idx i
      JOIN pg_attribute a ON a.attrelid = i.idxoid AND a.attnum > 0 AND NOT a.attisdropped
      LEFT JOIN pg_stats se ON se.schemaname = i.schemaname
                           AND se.tablename = i.indexname AND se.attname = a.attname
      LEFT JOIN pg_attribute ta ON ta.attrelid = i.tbloid
                               AND ta.attnum = i.indkey[a.attnum - 1]
      LEFT JOIN pg_stats st ON st.schemaname = i.schemaname
                           AND st.tablename = i.tablename AND st.attname = ta.attname
),
tuple AS (
    SELECT i.*,
           (SELECT sum((1 - c.null_frac) *
                       CASE WHEN c.attlen < 0 AND c.width <= 127 THEN c.width
                            ELSE ceil(c.width / al.a) * al.a END)
              FROM cols c
              CROSS JOIN LATERAL (SELECT CASE c.attalign WHEN 'c' THEN 1 WHEN 's' THEN 2
                                              WHEN 'i' THEN 4 ELSE 8 END AS a) al
             WHERE c.idxoid = i.idxoid)                                    AS data_size,
           (SELECT 1 - coalesce(exp(sum(ln(greatest(1 - c.null_frac, 1e-9)))), 1)
              FROM cols c WHERE c.idxoid = i.idxoid)                       AS p_null,
           (SELECT least(round(exp(sum(ln(greatest(
                       CASE WHEN c.n_distinct > 0
                            THEN c.n_distinct
                                 + CASE WHEN c.null_frac > 0 THEN 1 ELSE 0 END
                            ELSE (1 - c.null_frac) * greatest(i.live_rows, 0)
                                 + CASE WHEN c.null_frac > 0 THEN 1 ELSE 0 END
                       END, 1))))),
                         greatest(i.live_rows, 0))
              FROM cols c WHERE c.idxoid = i.idxoid)                       AS key_groups
      FROM idx i
),
sized AS (
    SELECT t.*, ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8 + 4         AS slot
      FROM tuple t
),
fit AS (
    SELECT s.*,
           greatest(floor((s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100)) / s.slot), 1)
               AS leaf_cap,
           greatest(floor((s.bs - 48 - floor(s.bs * 30 / 100)) / s.slot), 2)
               AS int_cap,
           (s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100))          AS leaf_bytes,
           (NOT s.indisunique AND s.dedup_on AND coalesce(s.all_equalimage, false))
               AS dedup_applies,
           least(greatest(s.live_rows, 0), s.key_groups)                   AS groups_est
      FROM sized s
),
leaves AS (
    SELECT f.*,
           CASE WHEN f.dedup_applies AND f.groups_est < greatest(f.live_rows, 0)
                THEN ceil((f.groups_est * f.slot
                           + (greatest(f.live_rows, 0) - f.groups_est) * 6) / f.leaf_bytes)
                ELSE ceil(greatest(f.live_rows, 0) / f.leaf_cap)
           END AS leaf_pages
      FROM fit f
),
levels AS (
    SELECT idxoid, leaf_pages AS pages, int_cap FROM leaves
    UNION ALL
    SELECT l.idxoid, ceil(l.pages / l.int_cap), l.int_cap FROM levels l WHERE l.pages > 1
),
modelled AS (
    SELECT l.*, (SELECT sum(pages) FROM levels v WHERE v.idxoid = l.idxoid) + 1
                    AS expected_blocks
      FROM leaves l
)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(actual_bytes) AS index_size,
       CASE
         WHEN live_rows IS NULL THEN 'unmeasured: reltuples unknown'
         WHEN is_partial AND (tbl_n_dead_tup > 0 OR last_analyze IS NULL)
              THEN 'unmeasured: analyze the table first'
         ELSE 'ok'
       END                                            AS status,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         pg_size_pretty((actual_bytes - expected_blocks * bs)::bigint) END AS wasted_space,
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         round((100 * (1 - (expected_blocks * bs) / greatest(actual_bytes, 1)))::numeric, 1)
       END                                            AS wasted_space_pct,
       (dedup_applies AND groups_est < greatest(live_rows, 0)) AS dedup_credited,
       fsm_bytes > 0                                  AS fsm_written_since_build,
       tbl_n_dead_tup                                 AS dead_tuples,
       (last_vacuum IS NULL AND last_analyze IS NULL) AS never_analyzed,
       idx_reltuples::bigint                          AS idx_reltuples,
       groups_est::bigint                             AS key_groups,
       live_rows::bigint                              AS modelled_rows
  FROM modelled
 WHERE actual_bytes > 1024 * 1024
 ORDER BY (actual_bytes - expected_blocks * bs) DESC NULLS FIRST
 LIMIT 20;
```

Three labels in that final `SELECT`, plus the statement tag, were corrected after the fact, and none changes `expected_blocks` or any percentage measured below: `wasted_space` is signed rather than clamped at zero, the free-space-map boolean is renamed, and the two reporting columns and the tag no longer say "bloat". See [Follow-up: two reporting defects, not arithmetic defects](#follow-up-two-reporting-defects-not-arithmetic-defects) and [Follow-up: the output columns say wasted_space, not bloat](#follow-up-the-output-columns-say-wasted_space-not-bloat).

Scored on the same 54 cells, against the same rebuilds:

| | Method A (v12 SQL) | dedup-aware sweep |
|---|---|---|
| exact | 30 / 54 | 30 / 54 |
| within 5 blocks | 42 | 46 |
| within 16 blocks | 42 | 47 |
| worst absolute error | 1896 blocks | 499 blocks |
| worst `dup` error | +1896 (+223.3%) | −24 (−2.9%) |

The 499-block worst case is not deduplication; it is the partial-index staleness the v12 page already documents, and it clears the same way — see [Partial indexes and the statistics state](#partial-indexes-and-the-statistics-state). On the fixture set the same two changes move `idx_dup` from +948 to −12 blocks and `idx_null` from +474 to −5, and leave all nine already-exact fixtures untouched.

The correction is deliberately conservative and it under-corrects in two named places: it stops crediting deduplication once `n_distinct` flips to the negative form (rows 1 and 2 of the ratio sweep), and it ignores the posting-list size cap, which costs about 3% on all-duplicate indexes.

### Method A-prime still fixes variable key width

`pg_stats.avg_width` is a sample mean of the stored width, so a MAXALIGN of that single average mis-prices keys whose per-row width straddles an alignment boundary. Measuring the slot per row instead:

```sql
SET statement_timeout = '60s';

SELECT /* wiki_btree_measure_slot */
       count(*)                                                 AS rows_measured,
       avg(ceil((8 + pg_column_size(k)) / 8.0) * 8 + 4)         AS avg_slot_bytes
  FROM t_var TABLESAMPLE BERNOULLI (1) REPEATABLE (42);
```

returned 57.165 bytes from a 1% sample (3,883 rows, 23.9 ms) and 58.000 from a full scan, against Method A's catalog-derived 60. Feeding 58 back into the closed form gives `leaf_cap` 126, 3175 leaves, `int_cap` 98, and 3210 total blocks against a true rebuild of 3211 — the error falls from +105 blocks (−3.3% phantom negative bloat) to −1 block (−0.03%). Same conclusion as the v12 page.

### Method B: leaf counts still exact, density formula not

The census is unchanged: `live_leaf_pages = full_scan_blocks - descent_blocks`, both probes twice in one session, second reading used. A forward scan still reads one buffer per right link and skips ignorable pages ([nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2181-L2240)), and an index-only scan still falls back to the heap whenever the visibility-map bit is unset ([nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L151-L171)), reported as `Heap Fetches` ([explain.c:1993](../../../../raw/postgres-17/src/backend/commands/explain.c#L1993)).

Results: **exact on every eligible cell** — 12 of 12 fixtures and 36 of 36 matrix cells, partial and non-partial alike. The 18 ineligible matrix cells are exactly the three never-vacuumed bloat types, and the `Heap Fetches` check caught all of them (`m_churn_unvac_1000_full` reported 1,559,855 leaf pages against a true 8,197).

The density reconstruction is where v17 diverges. The v12 formula assumes one slot per row:

| bloat type | cells | v12 density formula error, non-partial | v12 formula error, partial |
|---|---|---|---|
| `fresh`, `scatter`, `range`, `random`, `churn_vac` | 30 | −0.05 to −0.04 points | −1.09 to +0.54 points |
| `dup` | 6 | **+216.67 to +217.43 points** | **+209.85 to +220.16 points** |

`m_dup_1000_full` is the clean example: 313.58% "density" against `pgstatindex`'s 96.15%. Substituting the same posting-list term — `(key_groups * slot + (rows - key_groups) * 6) / (leaf_pages * (BLCKSZ - 40))` — brings the six `dup` cells to −4.38 to −1.30 points and costs the other 30 cells about a quarter of a point. The denominator `BLCKSZ - 40` is still what `pgstatindex` uses ([pgstatindex.c:310-316](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L310-L316), [pgstatindex.c:363-367](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367)).

### Method C: unchanged answer, different write path

Method C is exact by construction on 17.10 and stays the arbiter used above. Its restrictions are the same: `CREATE INDEX CONCURRENTLY` is rejected inside a transaction block ([utility.c:1456-1466](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1456-L1466)), takes `ShareUpdateExclusiveLock` on the table ([indexcmds.c:672-682](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L672-L682)), is refused on a partitioned table ([indexcmds.c:723-733](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L723-L733)), leaves an invalid index behind on failure because `indisvalid` is set as the last step ([index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3478-L3550)), and costs two table scans ([create_index.sgml:625-635](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L625-L635)).

One v17 internal difference is worth knowing before quoting its cost: the sorted build now writes through the bulk-write facility rather than issuing its own page writes and an unconditional `smgrimmedsync` ([nbtsort.c:1145-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1152), [nbtsort.c:1370-1377](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1370-L1377)). `smgr_bulk_finish` still WAL-logs the built pages when the relation needs WAL, but it registers the fsync with the checkpointer and only calls `smgrimmedsync` when a checkpoint started concurrently ([bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L220)).

The DDL generator still needs the same two fixes, because `pg_get_indexdef` emits reloptions but neither `CONCURRENTLY` nor the tablespace ([ruleutils.c#pg_get_indexdef](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L1160-L1177)).

### Method D: new message, no row count, new blind spot

`VACUUM VERBOSE` output was restructured. v17 prints one consolidated report per table, and its per-index line carries four page classes and **no index row count**:

```text
index scan needed: 4425 pages from table (100.00% of total) had 900000 dead item identifiers removed
index "d_work_idx": pages: 2745 in total, 0 newly deleted, 0 currently deleted, 0 reusable
```

That is the exact format string in [vacuumlazy.c:718-732](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732), fed by `IndexBulkDeleteResult`, which gained a `pages_newly_deleted` counter ([genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L75-L84), [nbtree.c#btvacuumpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1160-L1190)). Anything parsing the v12 wording ("now contains N row versions in M pages") gets nothing, and the tuple count is simply not available from Method D any more.

The v12 caveat that a no-op VACUUM prints no index line still holds — `btvacuumcleanup` returns NULL when `_bt_vacuum_needs_cleanup` says no scan is needed ([nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L924)) — and v17 adds a second, larger hole. When fewer than 2% of the table's pages hold dead items, VACUUM bypasses index vacuuming entirely ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1864-L1935)) and prints a distinct line with no per-index output at all. Measured on a 4,425-page table with 4,000 dead rows on 18 pages:

```text
index scan bypassed: 18 pages from table (0.41% of total) have 4000 dead item identifiers
```

Re-running the same VACUUM with `INDEX_CLEANUP ON` — which sets `consider_bypass_optimization` false ([vacuumlazy.c:388-407](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L388-L407)) — produced the index line and reported 10 newly deleted pages. On v17, Method D therefore reports only when VACUUM both had work to do **and** did not bypass it.

### The v14 unknown reltuples sentinel

`pg_class.reltuples` now defaults to `-1`, documented in the catalog header as "unknown" ([pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)), and `index_update_stats` deliberately preserves it when an index is created on an empty table ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)). `TRUNCATE` puts an index back into that state: measured on a 300,000-row fixture, the index's `reltuples` went 300000 → −1 across a truncate and stayed −1 after the reload.

The v12 SQL has no `-1` branch, so the sentinel flows straight into the arithmetic. On a 1,000,000-row table whose index was created before the rows and never analyzed:

| index | live blocks | rebuilt blocks | true bloat | `idx_reltuples` | v12 `modelled_rows` | v12 model | v12 `bloat_pct` |
|---|---|---|---|---|---|---|---|
| `m1_neg_idx` | 2745 | 2745 | 0.0% | −1 | −1 | 1 block | **100.0** |

A perfectly healthy 21 MB index is reported as 100% wasted. Falling back to the cumulative statistics counter is not a fix either: at the moment of the sweep `pg_stat_all_tables.n_live_tup` for that table read 2,000,000 against 1,000,000 real rows. The sweep above therefore treats `reltuples = -1` as *unmeasured* and emits NULL for `wasted_space` and `wasted_space_pct` rather than a number. This capture predates the column rename, so its header carries the old labels:

```text
 tablename |  indexname  | index_size |             status            | wasted | bloat_pct | idx_reltuples
 m1_neg    | m1_neg_idx  | 21 MB      | unmeasured: reltuples unknown |        |           |            -1
```

`ANALYZE` or `VACUUM` on the table clears it. This is one place where v17 is *better* instrumented than the v12 model assumes: `0` and "unknown" are now distinguishable, so an empty index and an unmeasured index no longer produce the same reading.

### Partial indexes and the statistics state

Partial indexes were exact for the v12 page and are exact here too, but only in the statistics state the v12 recipes produce. Which of the two writers touched the index's `reltuples` last decides it:

- a build writes the true count through `index_update_stats` ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842));
- `ANALYZE` writes `ceil(tupleFract * totalrows)` from the heap sample ([analyze.c:637-660](../../../../raw/postgres-17/src/backend/commands/analyze.c#L637-L660));
- VACUUM writes the true count, but only when the index scan actually ran and produced a non-estimated count ([vacuumlazy.c:3078-3098](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3078-L3098), [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L924)).

Measured directly on the three `fresh` partial cells, first with build-time statistics and then after one plain `ANALYZE`:

| cell | live blocks | model with build-time reltuples | `idx_reltuples` after ANALYZE | model after ANALYZE | error |
|---|---|---|---|---|---|
| `m_fresh_200_part` | 112 | 112 (exact) | 40267 | 113 | +1 |
| `m_fresh_500_part` | 276 | 276 (exact) | 99850 | 275 | −1 |
| `m_fresh_1000_part` | 551 | 551 (exact) | 199100 | 548 | −3 |

The catastrophic partial-index case is unchanged from v12: an index whose predicate subset shrank, with no VACUUM and no ANALYZE, keeps its pre-delete `reltuples` because `pg_stat_all_tables.n_live_tup` counts the whole table. One `ANALYZE` repairs it:

| cell | live | rebuilt | model before | model after | error after | `reltuples` after |
|---|---|---|---|---|---|---|
| `m_lpdead_200_part` | 112 | 12 | 112 | 12 | 0 | 3636 |
| `m_lpdead_500_part` | 276 | 27 | 276 | 27 | 0 | 9116 |
| `m_lpdead_1000_part` | 551 | 52 | 551 | 52 | 0 | 18070 |
| `m_stale_200_part` | 112 | 12 | 112 | 12 | 0 | 3636 |
| `m_stale_500_part` | 276 | 27 | 276 | 27 | 0 | 9050 |
| `m_stale_1000_part` | 551 | 52 | 551 | 51 | −1 | 17910 |

Worst error 499 blocks before, 1 block after — the v12 page reports 510 blocks before and 1 after. The non-partial siblings on the same six tables were exact both times, because the collector's `n_live_tup` tracked the delete.

### The 54-cell matrix

Worst error per bloat type, as collected, before any repair `ANALYZE`:

| bloat type | non-partial: worst Δblocks (exact cells) | partial: worst Δblocks (exact cells) | dedup-aware, non-partial | dedup-aware, partial |
|---|---|---|---|---|
| `fresh` | 0 (3/3) | 5 (0/3) | 0 (3/3) | 5 (0/3) |
| `scatter` | 0 (3/3) | 1 (2/3) | 0 (3/3) | 1 (2/3) |
| `range` | 0 (3/3) | 1 (1/3) | 0 (3/3) | 1 (1/3) |
| `random` | 0 (3/3) | 4 (0/3) | 0 (3/3) | 4 (0/3) |
| `dup` | **1896** (0/3) | **386** (0/3) | 24 (0/3) | 3 (0/3) |
| `churn_vac` | 0 (3/3) | 3 (0/3) | 0 (3/3) | 3 (0/3) |
| `churn_unvac` | 0 (3/3) | 0 (3/3) | 0 (3/3) | 0 (3/3) |
| `lpdead` | 0 (3/3) | **499** (0/3) | 0 (3/3) | 499 (0/3) |
| `stale` | 0 (3/3) | **499** (0/3) | 0 (3/3) | 499 (0/3) |

Every non-partial cell outside `dup` is exact, which is the strongest single statement about accuracy transfer: for indexes with distinct keys, the v12 arithmetic predicts a v17 rebuild to the block at three scales and nine bloat shapes.

`churn_unvac` is worth one note. Its non-partial cells are exact at every scale, but the live index is a different size than the v12 page reports: two full-table updates left 8228 blocks here against a rebuild of 2745, and the `idx_churn` fixture reads 2471 blocks at 74.80% density against the v12 page's 3293 at 67.63%. v17 does have a mechanism the v12 checkout lacks — bottom-up index deletion deletes dead entries when a page is about to split, instead of splitting ([nbtdedup.c#_bt_bottomupdel_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L306-L421)) — but this run did not separate it from the update pattern, so the cause stays open. The model is unaffected either way, because it predicts the *rebuilt* size and the rebuild is a fresh sorted build.

### Accuracy on 17 against the v12 page's reported accuracy

Direct comparison, v17 measured here against the figures the v12 page reports for v12:

| Claim | v12 page | this v17 run |
|---|---|---|
| Method A exact on the fixture set | 10 of 14, ±2 blocks on 3, −4.6% on 1 | 9 of 14 (+ `idx_empty`), ±4 to +105 on 3, +474 and +948 on 2 |
| Method A on the matrix | exact on 39 of 54, within 5 on 47, within 16 on 48, 6 catastrophic | exact on 30 of 54, within 5 on 42, within 16 on 42, 12 catastrophic (6 `dup`, 6 `lpdead`/`stale` partial) |
| Method B `leaf_pages` | exact on 12 of 12 fixtures, 36 of 36 eligible cells | exact on 12 of 12 fixtures, 36 of 36 eligible cells |
| Method B density | −0.03 to −0.15 points | −1.09 to +0.54 points, except `dup` at +209.85 to +220.16 points |
| Method A-prime | fixes the variable-width fixture to +0.32% | fixes it to −0.03% |
| Method C | exact by construction | exact by construction |
| Partial-index repair | worst error 510 blocks → 1 after ANALYZE | worst error 499 blocks → 1 after ANALYZE |
| Method D | exact page census, silent when VACUUM has no work | no row count at all, silent when VACUUM has no work **or** bypasses the index |

So: same accuracy on distinct-key indexes, same failure mode and same repair for stale partial indexes, and one new class of catastrophic error that did not exist in v12. The v12 page's own scoreboard falls from 39 exact cells to 30 for a single reason, and the [dedup-aware sweep](#a-deduplication-aware-sweep-for-v17) recovers all but 2.9% of it.

### What to change before running the v12 SQL on 17

1. Add the posting-list term and its `n_distinct > 0` gate, or the sweep will condemn healthy indexes with duplicate keys. This is the only change that matters for correctness.
2. Add the `reltuples = -1` branch and report those indexes as unmeasured.
3. Flag partial indexes whose table has dead tuples or has never been analyzed as unmeasured; run `ANALYZE` before believing them.
4. Keep `least(reltuples, n_live_tup)` for non-partial indexes: it is still what rescues an unvacuumed bulk delete (`idx_stale`, model 276 against a rebuild of 276).
5. If a monitoring job parses Method D, rewrite the parser for `index "name": pages: N in total, N newly deleted, N currently deleted, N reusable`, and treat `index scan bypassed` as "no data", not "no bloat".
6. Method B and Method C need no changes; only Method B's density formula does.

### Cost of each method on this server

| Method | Reads | Writes | Measured cost | Accuracy against a rebuild |
|---|---|---|---|---|
| A: catalog sweep | catalogs only | none | 27.8 ms for 79 indexes over 128,306 blocks | exact on 9/14 fixtures and 24/27 non-partial matrix cells; +223% on duplicates |
| A-prime: `pg_column_size` | one 1% sample | none | 23.9 ms over 400,000 rows | fixes the variable-width fixture to −1 block |
| B: index-only-scan census | the whole index | none | 93 ms on a 21 MB index (warm) | `leaf_pages` exact on 48 of 48 eligible cells |
| C: CIC rebuild | table, writes a new index | yes | 204 ms plus 21 MB on the same index | exact by definition |
| D: `VACUUM VERBOSE` | the whole index | yes | a VACUUM | exact page classes, when it prints them |

### Settings and apply scopes

| Setting | Context in v17 | Apply scope |
|---|---|---|
| `statement_timeout` | `PGC_USERSET` ([guc_tables.c:2611-2620](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)) | session/transaction |
| `lock_timeout` | `PGC_USERSET` ([guc_tables.c:2622-2631](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)) | session/transaction |
| `maintenance_work_mem` | `PGC_USERSET` ([guc_tables.c:2465-2474](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)) | session/transaction |
| `enable_seqscan`, `enable_bitmapscan`, `max_parallel_workers_per_gather` | `PGC_USERSET` | session/transaction |
| `block_size` | `PGC_INTERNAL` preset ([guc_tables.c:3268-3277](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3268-L3277)) | read-only; always read it with `current_setting('block_size')` |
| `wal_level` | `PGC_POSTMASTER` ([guc_tables.c:4986-4994](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4986-L4994)) | restart; only relevant because it decides whether Method C WAL-logs every built page |

No method here requires changing a setting that needs a reload or a restart. The per-index `deduplicate_items` reloption read by the sweep is set with `ALTER INDEX ... SET (deduplicate_items = off)`, which takes `AccessExclusiveLock` and is not required to run the sweep.

### What no core-SQL method can measure on v17

The v12 page's list is unchanged, and deduplication adds one entry:

- **`leaf_fragmentation`** — it needs page headers in physical order ([pgstatindex.c:318-325](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L325)); nothing in core exposes them.
- **`LP_DEAD` space inside live leaves** — Method B counts returned rows, so it reports live density.
- **Deleted versus half-dead pages** — Methods A and B lump both into "not in the live leaf chain".
- **How much deduplication actually happened** — the number of posting-list tuples and their TID counts is per-page state; core SQL can only estimate it from `n_distinct` and `null_frac`.
- **Per-page detail of any kind** — there is still no core page reader. `pg_proc.dat` contains no `pgstatindex`, `pgstattuple`, `get_raw_page`, `bt_page_stats` or `pg_freespace` entry; all of them are contrib, none of `pgstattuple`, `pageinspect`, `pg_freespacemap` or `amcheck` sets `trusted = true` in its control file, and `read_extension_control_file` defaults `superuser` to true and `trusted` to false ([extension.c:778-790](../../../../raw/postgres-17/src/backend/commands/extension.c#L778-L790)), so a non-superuser without the trust flag gets `permission denied to create extension` ([extension.c#execute_extension_script](../../../../raw/postgres-17/src/backend/commands/extension.c#L1019-L1035)). `pgstatindex` re-checks `superuser()` in C at entry ([pgstatindex.c:145-160](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L145-L160)), as does `pageinspect` ([rawpage.c:148-154](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L148-L154)). `TABLESAMPLE` still cannot be applied to an index ([parse_clause.c:1136-1146](../../../../raw/postgres-17/src/backend/parser/parse_clause.c#L1136-L1146)).

### Follow-up: the same sweep on a 12.2 server and a 17.10 server

Yes on both, and the deduplication term is what makes it portable rather than what breaks it. Every v17-specific term in the sweep is gated on a catalog fact that a PostgreSQL 12 server cannot produce, so the same statement pointed at a 12 server turns the correction off by itself and leaves exactly the v12 page's Method A running.

The gate is the SQL form of the engine's own test. `_bt_load` deduplicates only when the build's `allequalimage` flag is set, the index is not unique, and `deduplicate_items` is on ([nbtsort.c:1147-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)), and that flag comes from `_bt_allequalimage`, which looks up a `BTEQUALIMAGE_PROC` support function per key column and returns false when any opclass lacks one ([nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563), [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)). `BTEQUALIMAGE_PROC` is support number 4 ([nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L702-L712)), which is why the sweep asks `pg_amproc` for `amprocnum = 4`.

What each v17 term needs, and what a 12.2 server offers:

| Sweep term | What it reads | 17.10 | 12.2, measured |
|---|---|---|---|
| posting-list leaf formula | a `pg_amproc` row at `amprocnum = 4` for every key column's opclass | present; `all_equalimage` is true on all 21 fixture indexes | no such row: `max(amprocnum)` over btree opfamilies is 3 and the count at 4 is 0, so `all_equalimage` is false on all 20 fixture indexes |
| `deduplicate_items` reloption | `pg_options_to_table(reloptions)` ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtutils.c#btoptions](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4561-L4576)) | exists, defaults to on | the option does not exist: `ALTER INDEX idx_dup SET (deduplicate_items = off)` fails with `ERROR: unrecognized parameter "deduplicate_items"` |
| `reltuples = -1` guard | `pg_class.reltuples` ([pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66)) | reads `-1` after `TRUNCATE` and reports `unmeasured: reltuples unknown` | the sentinel never appears: an index on an empty table and an index after `TRUNCATE` both read `0` |

None of the three can be back-ported into a 12 catalog by hand either. A support-function number is bounded by the access method's `amsupport`, which for B-tree is `BTNProcs` ([nbtree.c#bthandler](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L100-L107)) — 5 in v17, and `ALTER OPERATOR FAMILY ... ADD FUNCTION n` rejects anything above it ([opclasscmds.c:840-845](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L840-L845), [opclasscmds.c#invalid-function-number](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L956-L962)); `btvalidate` accepts exactly support numbers 1 through 5 and reports every other number as invalid ([nbtvalidate.c:90-126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L90-L126)). On 12.2 the same DDL returns `ERROR: invalid function number 4, must be between 1 and 3`.

All three constructs postdate 12 in this checkout's own history: `612a1ab7672` (2020-02-26, "Add equalimage B-Tree support functions.") and `0d861bbb702` (2020-02-26, "Add deduplication to nbtree.") first ship in `REL_13_0`, and `3d351d916b2` (2020-08-30, "Redefine pg_class.reltuples to be -1 before the first VACUUM or ANALYZE.") first ships in `REL_14_0`. None is an ancestor of `REL_12_2`.

Measured with the statement unchanged on two isolated servers, each built from its own pin, carrying the same DDL and the same generated data:

| | 12.2 | 17.10 |
|---|---|---|
| statement runs | yes | yes |
| indexes swept / blocks covered, rebuild copies included | 32 / 68,108 | 34 / 48,753 |
| warm run time, same statement | 10.9 ms | 10.5 ms |
| indexes credited with deduplication | 0 | 15 |
| cells whose `expected_blocks` differs from the v12 page's Method A | 0 of 32 | 15 of 34 |
| indexes reported `unmeasured` | 0 | 1 |

Per fixture, 1,000,000 rows each, `bigint` key, `pg_relation_size` in blocks and a `CREATE INDEX CONCURRENTLY` rebuild as ground truth:

| fixture | 12.2 live | 12.2 rebuilt | 12.2 model | 12.2 reported | 17.10 live | 17.10 rebuilt | 17.10 model | 17.10 reported |
|---|---|---|---|---|---|---|---|---|
| `idx_seq`, distinct keys | 2745 | 2745 | 2745 | 0.0% | 2745 | 2745 | 2745 | 0.0% |
| `idx_dup`, one key | 2749 | 2749 | 2745 | 0.1% | 849 | 849 | 825 | 2.8% |
| `idx_null`, 25% NULL | 2746 | 2746 | 2745 | 0.0% | 2271 | 2271 | 2265 | 0.3% |
| `idx_q2`, 2 rows/key | 2749 | 2749 | 2745 | 0.1% | 2475 | 2475 | 2745 | −10.9% |
| `idx_q5`, 5 rows/key | 2748 | 2748 | 2745 | 0.1% | 1426 | 1426 | 2745 | −92.5% |
| `idx_q10`, 10 rows/key | 2749 | 2749 | 2745 | 0.1% | 1157 | 1157 | 2745 | −137.3% |
| `idx_q100`, 100 rows/key | 2749 | 2749 | 2745 | 0.1% | 839 | 839 | 844 | −0.6% |
| `idx_seq_part`, 20% partial | 551 | 551 | 551 | 0.0% | 551 | 551 | 550 | 0.2% |
| `idx_dupdel`, 10 keys, 90% deleted + VACUUM | 2749 | 278 | 276 | 90.0% | 850 | 87 | 84 | 90.1% |

Four readings matter:

- **Every fresh fixture is 0% reclaimable on both servers** — live equals rebuilt in all sixteen cells — so any non-zero reading in the "reported" columns is model error, not bloat.
- **On 12.2 the sweep misses by at most 4 blocks across these nine fixtures**, including the all-duplicate index. That is the case the deduplication term exists for, and crediting it there would have been wrong: 12.2 stores one index tuple per row, so `idx_dup` really is 2749 blocks. The gate is what keeps the answer right.
- **`idx_dupdel` is the load-bearing case**: a duplicate-key index with real, VACUUM-confirmed bloat. Both servers report about 90% and both are right — 276 modelled blocks against a 12.2 rebuild of 278 (true bloat 89.9%), 84 against a 17.10 rebuild of 87 (true bloat 89.8%) — reached by different arithmetic on each server. The uncorrected v12 model would have reported 67.5% on the 17.10 index.
- **The negative percentages on 17.10 at 2, 5 and 10 rows per key** are the `n_distinct > 0` gate declining to credit deduplication, the blind spot already documented in [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17). The same cells are accurate on 12.2 because there is nothing to credit. At ten rows per key this run landed on the negative form (`key_groups` 1,000,000, no credit) where the earlier ratio sweep on this page recorded a positive `n_distinct` of 97311; 100,001 distinct values in 1,000,000 rows sits exactly on the 10%-of-rows boundary that decides the sign ([analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612)), and the estimate is sampled.

Six further index shapes over a 200,000-row table were swept on both servers without error — multi-column, `INCLUDE`, expression, unique, `text`, and a low-cardinality `int`. Scored against the freshly built live size rather than a rebuild, the model lands within one block on every one of the twelve cells, on both servers. The `indclass` probe survives the version change because `oidvector` subscripts start at 0, so `k < x.indnkeyatts` covers every key column on both servers (measured `array_lower(indclass, 1) = 0` on 12.2 and 17.10).

### Follow-up: the INCLUDE-column false positive on v17

This comparison exposed one case where the dedup-aware sweep is wrong on 17.10, and it is not a version-portability problem: `_bt_allequalimage` returns false immediately for any index whose total attribute count differs from its key attribute count, because "INCLUDE indexes can never support deduplication" ([nbtutils.c:5144-5148](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5148)). The sweep's catalog probe only inspects key columns and never checks for included ones, so it credits deduplication the engine will refuse.

It only bites when the *included* column is also low-cardinality, because `key_groups` multiplies per-column `n_distinct` across every index attribute. Measured on 17.10 over 1,000,000 rows with 1000 distinct `a` and 5 distinct `d`:

| index | key groups | model | live | rebuilt | reported | true |
|---|---|---|---|---|---|---|
| `idx_inc_lowcard` — `(a) INCLUDE (d)` | 5000 | 842 | 3853 | 3853 | **78.1%** | 0.0% |
| `idx_inc_bothkeys` — `(a, d)` | 5000 | 842 | 897 | 897 | 6.1% | 0.0% |
| `idx_inc_keyonly` — `(a)` | 1000 | 827 | 896 | 896 | 7.7% | 0.0% |
| `idx_inc_include` — `(a) INCLUDE (b)`, `b` distinct | 1,000,000 | 3853 | 3853 | 3853 | 0.0% | 0.0% |

The fix is one conjunct. Add `(x.indnatts = x.indnkeyatts) AS no_included_cols` to the `idx` CTE and require it in the gate:

```sql
(NOT s.indisunique AND s.dedup_on AND s.no_included_cols
     AND coalesce(s.all_equalimage, false))                    AS dedup_applies
```

Re-scored on the same server, `idx_inc_lowcard` moves from 78.1% to 0.0% with a model of 3853 blocks against a rebuild of 3853 — exact — and all 33 other indexes in that database are unchanged. On 12.2 the same three indexes read 0.1% to 0.2% with errors of −4 to −6 blocks, because the gate is closed there for every index anyway.

The 6.1% and 7.7% on the two indexes that *can* deduplicate are the posting-list-cap under-correction this page already records at 2.9%. It is not a constant: it was 24 blocks at 1,000,000 rows per key and 69 blocks at 1000 rows per key.

### Follow-up: the v12 hazard the reltuples guard does not cover

The `reltuples = -1` branch is dead code on 12.2, and the v12-era failure it was added for is still live there as a *zero*. Same fixture on both servers — 300,000 rows, `TRUNCATE`, reload, no `ANALYZE`:

| server | table / index `reltuples` | 825-block healthy index reported as |
|---|---|---|
| 12.2 | `0` / `0` | **99.9% bloat** |
| 17.10 | `-1` / `-1` | `unmeasured: reltuples unknown`, no number |

`least(reltuples, n_live_tup)` cannot rescue it: the collector had the right 300,000 while `pg_class` had 0, and `least()` takes the zero. So a v12 server needs its own unmeasured rule, and the size check is what separates a stale zero from a genuinely empty index:

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

SELECT /* wiki_btree_zero_reltuples_v12 */
       n.nspname                                              AS schemaname,
       c.relname                                              AS indexname,
       c.reltuples::bigint                                    AS reltuples,
       pg_relation_size(c.oid)                                AS bytes
  FROM pg_class c
  JOIN pg_index x     ON x.indexrelid = c.oid
  JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE c.relkind = 'i' AND x.indisvalid
   AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
   AND c.reltuples = 0
   AND pg_relation_size(c.oid) > current_setting('block_size')::int
 ORDER BY pg_relation_size(c.oid) DESC;
```

Run as written on the 12.2 test database, that returns the truncated index and nothing else — one row, `idx_trunc`, `reltuples` 0, 6,758,400 bytes — because the genuinely empty index is exactly one metapage (8192 bytes) and every populated index carries a non-zero `reltuples`.

So, pointing this sweep at a 12 server needs no change for deduplication and one change for statistics:

1. Leave the posting-list term and its `pg_amproc` gate alone. They cost nothing and cannot fire.
2. Replace the `reltuples = -1` branch with the zero rule above, or keep both — `-1` on 14 and later, `0`-with-bytes on 12 and 13.
3. Keep the partial-index `unmeasured` status. It is statistics-driven and version-independent.
4. Expect the model to sit 1 to 4 blocks *under* a 12.2 rebuild on duplicate-key and NULL-heavy indexes, against 0 blocks on distinct keys.

### Follow-up: one statement for PostgreSQL 12 through 17

Here it is. Every construct in it exists in 12 and still exists in 17, and it
credits deduplication only where the catalog proves the engine would deduplicate.
It was run on 12.2, 14.23 and 17.10; 13, 15 and 16 have no checkout in this repo
and were not run. On the 12.2 server every deduplication term switched itself off
and the statement reduced to the v12 page's Method A arithmetic — identical
`expected_blocks` in all 34 scored cells — while on 14.23 and 17.10 it credited
posting lists and landed within 5% of a `CREATE INDEX CONCURRENTLY` rebuild on 25
of 33 and 25 of 35 cells against Method A's 14 and 15.

It reports two numbers per index, not one:

- `wasted_space_pct` — the point estimate, with posting lists credited;
- `wasted_space_pct_floor` — the same model with one index tuple per row, which is
  what a pre-13 server would produce.

Alert on the floor and read the point estimate for size. On the 34-to-36 index
fixture set below, that rule finds 4 of the 5 genuinely bloated indexes with **0
false positives** on every server, and the one it misses is flagged
`never analyzed`; the point estimate alone produces up to 3 false positives.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

WITH RECURSIVE
idx AS (
    SELECT /* wiki_btree_wasted_space_sweep_12_17 */
           c.oid AS idxoid, n.nspname AS schemaname, t.relname AS tablename,
           c.relname AS indexname, t.oid AS tbloid, x.indkey, x.indisunique,
           x.indnkeyatts,
           (x.indpred IS NOT NULL)                      AS is_partial,
           (x.indnatts = x.indnkeyatts)                 AS keys_only,
           z.actual_bytes, z.fsm_bytes, z.bs, z.server_version_num,
           coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'fillfactor'), 90)            AS fillfactor,
           coalesce((SELECT option_value::bool FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'deduplicate_items'), true)   AS dedup_on,
           c.reltuples::numeric                         AS idx_reltuples,
           coalesce(s.n_live_tup, 0)::numeric           AS tbl_live_tup,
           coalesce(s.n_dead_tup, 0)::numeric           AS tbl_dead_tup,
           greatest(s.last_analyze, s.last_autoanalyze) AS last_analyze,
           -- rows to model: -1 is v14+ "unknown"; a 0 on a non-empty index
           -- whose table reports live rows is a pre-14 stale zero
           CASE
             WHEN c.reltuples < 0 THEN NULL
             WHEN c.reltuples = 0 AND z.actual_bytes > z.bs
                  AND coalesce(s.n_live_tup, 0) > 0 THEN NULL
             WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
             ELSE least(c.reltuples::numeric,
                        coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
           END                                          AS live_rows,
           -- deduplication gate, in catalog terms: every key opclass must
           -- advertise an equal-image support function (amprocnum 4)
           (SELECT bool_and(EXISTS (SELECT 1 FROM pg_amproc ap
                                     WHERE ap.amprocfamily = op.opcfamily
                                       AND ap.amproclefttype = op.opcintype
                                       AND ap.amprocrighttype = op.opcintype
                                       AND ap.amprocnum = 4))
              FROM generate_subscripts(x.indclass, 1) k
              JOIN pg_opclass op ON op.oid = x.indclass[k]
             WHERE k < x.indnkeyatts)                   AS all_equalimage,
           -- ... and no key column may use a nondeterministic collation
           NOT EXISTS (SELECT 1 FROM generate_subscripts(x.indcollation, 1) k
                         JOIN pg_collation cl ON cl.oid = x.indcollation[k]
                        WHERE k < x.indnkeyatts
                          AND NOT cl.collisdeterministic) AS all_deterministic
      FROM pg_class c
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_am am       ON am.oid = c.relam
      LEFT JOIN pg_stat_all_tables s ON s.relid = t.oid
      CROSS JOIN LATERAL (
            SELECT pg_relation_size(c.oid)                  AS actual_bytes,
                   pg_relation_size(c.oid, 'fsm')           AS fsm_bytes,
                   current_setting('block_size')::int       AS bs,
                   current_setting('server_version_num')::int AS server_version_num) z
     WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
),
cols AS (
    SELECT i.idxoid, a.attnum, a.attlen, a.attalign,
           CASE WHEN a.attlen > 0 THEN a.attlen::numeric
                ELSE coalesce(se.avg_width, st.avg_width, 32)::numeric END AS width,
           coalesce(se.null_frac, st.null_frac, 0)::numeric      AS null_frac,
           coalesce(se.n_distinct, st.n_distinct, 0)::numeric    AS n_distinct,
           coalesce(se.most_common_freqs, st.most_common_freqs)  AS mcf
      FROM idx i
      JOIN pg_attribute a ON a.attrelid = i.idxoid AND a.attnum > 0 AND NOT a.attisdropped
      LEFT JOIN pg_stats se ON se.schemaname = i.schemaname
                           AND se.tablename = i.indexname AND se.attname = a.attname
      LEFT JOIN pg_attribute ta ON ta.attrelid = i.tbloid
                               AND ta.attnum = i.indkey[a.attnum - 1]
      LEFT JOIN pg_stats st ON st.schemaname = i.schemaname
                           AND st.tablename = i.tablename AND st.attname = ta.attname
),
tuple AS (
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
           -- distinct key groups: per-column estimates multiplied over the key
           -- columns. A negative n_distinct is a fraction of the *table's*
           -- rows, so a partial index only trusts an absolute count.
           (SELECT least(round(exp(sum(ln(greatest(
                       CASE WHEN c.n_distinct > 0 THEN c.n_distinct
                            WHEN c.n_distinct < 0 AND NOT i.is_partial
                                 THEN (- c.n_distinct) * greatest(i.live_rows, 0)
                            ELSE (1 - c.null_frac) * greatest(i.live_rows, 0)
                       END
                       + CASE WHEN c.null_frac > 0 THEN 1 ELSE 0 END, 1))))),
                         greatest(i.live_rows, 0))
              FROM cols c
             WHERE c.idxoid = i.idxoid AND c.attnum <= i.indnkeyatts) AS key_groups
      FROM idx i
),
sized AS (
    SELECT t.*, ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8 + 4 AS slot
      FROM tuple t
),
fit AS (
    SELECT s.*,
           greatest(floor((s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100)) / s.slot), 1)
               AS leaf_cap,
           greatest(floor((s.bs - 48 - floor(s.bs * 30 / 100)) / s.slot), 2)
               AS int_cap,
           (s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100))     AS leaf_bytes,
           (NOT s.indisunique AND s.dedup_on AND s.keys_only
                AND coalesce(s.all_equalimage, false)
                AND s.all_deterministic)                             AS dedup_applies,
           least(greatest(s.live_rows, 0), greatest(s.key_groups, 1)) AS groups_est,
           -- maxpostingsize = MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)
           floor(floor(s.bs * 10 / 100) / 8) * 8 - 4                  AS maxposting
      FROM sized s
),
posting AS (
    SELECT f.*,
           -- largest n with MAXALIGN(base + n * sizeof(ItemPointerData)) <= maxposting
           greatest(floor(4 * floor((f.maxposting - (f.slot - 4)) / 8) / 3), 1) AS nmax
      FROM fit f
),
kstat AS (
    SELECT p.idxoid,
           CASE WHEN p.is_partial THEN 0 ELSE c.null_frac END AS null_frac,
           CASE WHEN p.is_partial THEN '{}'::real[]
                ELSE coalesce(c.mcf, '{}'::real[]) END        AS mcf
      FROM posting p
      JOIN cols c ON c.idxoid = p.idxoid AND c.attnum = 1
     WHERE p.indnkeyatts = 1 AND p.dedup_applies AND p.live_rows > 0
),
gclass AS (
    -- the NULL run is one key group
    SELECT p.idxoid, greatest(p.live_rows, 0) * k.null_frac AS class_rows,
           1::numeric AS class_groups
      FROM posting p JOIN kstat k ON k.idxoid = p.idxoid
     WHERE k.null_frac > 0
    UNION ALL
    -- every most-common value is one key group
    SELECT p.idxoid, greatest(p.live_rows, 0) * f, 1::numeric
      FROM posting p JOIN kstat k ON k.idxoid = p.idxoid
      CROSS JOIN LATERAL unnest(k.mcf) f
    UNION ALL
    -- the rest of the rows spread over the remaining distinct values
    SELECT p.idxoid,
           greatest(greatest(p.live_rows, 0)
                    * (1 - k.null_frac
                         - coalesce((SELECT sum(f) FROM unnest(k.mcf) f), 0)), 0),
           greatest(p.groups_est
                    - CASE WHEN k.null_frac > 0 THEN 1 ELSE 0 END
                    - coalesce(array_length(k.mcf, 1), 0), 1)
      FROM posting p JOIN kstat k ON k.idxoid = p.idxoid
    UNION ALL
    -- multi-column keys: one uniform class over the product estimate
    SELECT p.idxoid, greatest(p.live_rows, 0), p.groups_est
      FROM posting p
     WHERE p.indnkeyatts > 1 AND p.dedup_applies AND p.live_rows > 0
),
classfit AS (
    SELECT g.idxoid, g.class_rows,
           least(g.class_rows / greatest(g.class_groups, 1), p.nmax) AS tids,
           p.slot, p.leaf_bytes
      FROM gclass g
      JOIN posting p ON p.idxoid = g.idxoid
     WHERE g.class_rows > 0
),
classpages AS (
    -- posting tuples are MAXALIGNed and each leaf page holds
    -- floor((leaf_bytes + truncated posting list) / tuple size) of them
    SELECT c.idxoid,
           sum((c.class_rows / greatest(c.tids, 1))
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
    SELECT p.*, coalesce(cp.max_tids, 1) AS tids,
           CASE WHEN p.dedup_applies AND cp.leaf_frac IS NOT NULL
                THEN greatest(ceil(cp.leaf_frac), 1)
                ELSE ceil(greatest(p.live_rows, 0) / p.leaf_cap)
           END                                                AS leaf_pages,
           ceil(greatest(p.live_rows, 0) / p.leaf_cap)         AS leaf_pages_floor
      FROM posting p
      LEFT JOIN classpages cp ON cp.idxoid = p.idxoid
),
levels AS (
    SELECT idxoid, 'dedup'::text AS variant, leaf_pages AS pages, int_cap FROM leaves
    UNION ALL
    SELECT idxoid, 'floor'::text, leaf_pages_floor, int_cap FROM leaves
    UNION ALL
    SELECT l.idxoid, l.variant, ceil(l.pages / l.int_cap), l.int_cap
      FROM levels l WHERE l.pages > 1
),
modelled AS (
    SELECT l.*,
           (SELECT sum(v.pages) FROM levels v
             WHERE v.idxoid = l.idxoid AND v.variant = 'dedup') + 1 AS expected_blocks,
           (SELECT sum(v.pages) FROM levels v
             WHERE v.idxoid = l.idxoid AND v.variant = 'floor') + 1 AS floor_blocks
      FROM leaves l
)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(actual_bytes)                     AS index_size,
       CASE
         WHEN idx_reltuples < 0 THEN 'unmeasured: reltuples unknown'
         WHEN live_rows IS NULL THEN 'unmeasured: reltuples 0, table has live rows'
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
         CASE WHEN NOT is_partial
                   AND greatest(tbl_live_tup, idx_reltuples)
                       > 1.1 * greatest(least(tbl_live_tup, idx_reltuples), 1)
              THEN 'row-count sources disagree: analyze first' END,
         CASE WHEN is_partial AND (tbl_dead_tup > 0 OR last_analyze IS NULL)
              THEN 'partial: predicate subset may be stale' END,
         CASE WHEN is_partial AND dedup_applies AND tids > 1
              THEN 'partial: duplicates from table statistics' END,
         CASE WHEN dedup_applies AND tids > 1 THEN 'deduplication credited' END
       ], NULL), '; ')                                  AS caveats,
       key_groups::bigint                               AS key_groups,
       round(tids::numeric, 1)                          AS tids_per_tuple,
       live_rows::bigint                                AS modelled_rows,
       idx_reltuples::bigint                            AS idx_reltuples,
       fsm_bytes > 0                                    AS fsm_written_since_build,
       server_version_num
  FROM modelled
 WHERE actual_bytes > 1024 * 1024
 ORDER BY (actual_bytes - floor_blocks * bs) DESC NULLS FIRST
 LIMIT 20;
```

Remove `WHERE actual_bytes > 1024 * 1024` and `LIMIT 20` to score every index;
that is how the numbers below were collected.

Three expressions in that final `SELECT` — `wasted_space`, the free-space-map
boolean, and the `ORDER BY` key — were corrected after the measurements below were
taken, and the three reporting columns plus the statement tag were then renamed off
the word "bloat". No correction touches `expected_blocks`, `floor_blocks` or any
reported percentage; see
[Follow-up: two reporting defects, not arithmetic defects](#follow-up-two-reporting-defects-not-arithmetic-defects)
and
[Follow-up: the output columns say wasted_space, not bloat](#follow-up-the-output-columns-say-wasted_space-not-bloat).

### What the proposed statement changes

Relative to [the deduplication-aware sweep earlier on this page](#a-deduplication-aware-sweep-for-v17), five things changed. Each one is a measured fix, not a preference:

| Change | Why | Measured on |
|---|---|---|
| Posting tuples are sized as `MAXALIGN(base + tids * 6) + 4` and capped at the 1/10-page posting-list limit, and a leaf's capacity gets the high-key truncation credit | the flat "6 bytes per extra row" term ignores alignment padding and the cap | on 17.10, `i_q5` reads −92.5% under the flat term and −2.5% under this one; `i_qall` moves from +2.8% to +0.2% |
| Key groups come from a mixture — the NULL run, each most-common value, then the remaining distinct values — instead of one uniform group size | one hot value plus mostly-distinct keys is not a uniform distribution | `i_null` (25% NULL, rest distinct) models 2909 blocks under the uniform form (−28.1% on a 2271-block index) and 2281 under the mixture (−0.4%) |
| A negative `n_distinct` is credited, but only for a whole-table index | the negative form is a fraction of the *table's* rows, so a partial index's subset can have a completely different duplication ratio | on 17.10 `i_q5_part` reads 45.2% when the fraction is applied to the subset and −2.4% when it is not; `i_q10_part` moves from 53.7% to −8.9% and `i_q2_part` from 10.9% to 0.9%, all on 0%-reclaimable indexes |
| The gate adds "no key column uses a nondeterministic collation" | `btvarstrequalimage` returns false for one, while the `pg_amproc` row still exists | dropping that one conjunct makes a healthy 28 MB ICU index read 88.2% instead of 0.1% |
| Both `reltuples` eras are handled, plus a `caveats` column and the floor | `-1` only exists from 14, a stale `0` only bites 12 and 13, and two row-count sources can disagree | `i_trunc` reports `unmeasured: reltuples 0, table has live rows` on 12.2 and `unmeasured: reltuples unknown` on 14.23/17.10; `i_grow` reports `row-count sources disagree: analyze first` on all three |

The INCLUDE-column conjunct from [the previous follow-up](#follow-up-the-include-column-false-positive-on-v17) is folded in as `x.indnatts = x.indnkeyatts`.

### The posting-list arithmetic, derived from source

Everything in the deduplication branch comes from three places in the build path.

**The cap.** A sorted build limits a posting list to `MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)` ([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1292-L1308)) — 812 bytes at `block_size` 8192, which the statement computes as `floor(floor(bs * 10 / 100) / 8) * 8 - 4`.

**The tuple size.** A TID is accepted while `MAXALIGN(basetupsize + (nhtids + 1) * sizeof(ItemPointerData))` stays inside that cap ([nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L503-L531)), and the finished tuple is exactly `MAXALIGN(keysize + nhtids * sizeof(ItemPointerData))` ([nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L863-L911)). `keysize` is already MAXALIGNed because `index_form_tuple` rounds up ([indextuple.c:154-163](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L154-L163)). Since the base is a multiple of 8, `MAXALIGN(base + 6n) = base + 8 * ceil(3n/4)`, so the largest usable TID count is `floor(4 * floor((maxposting - base) / 8) / 3)` — 132 for an 8-byte key, in an 808-byte tuple.

**The page capacity.** `_bt_buildadd` finishes a leaf when `pgspc + last_truncextra < btps_full`, where `last_truncextra` is the size of the previous tuple's posting list, because that list is truncated away when the tuple becomes the page's high key ([nbtsort.c:769-781](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L769-L781), [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L855)). With `btps_full` at `BLCKSZ * (100 - fillfactor) / 100` ([nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L666), [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1145)) that makes the data-tuple capacity `floor((leaf_bytes + tids * 6) / tuple_size)`, which is why the statement adds `tids * 6` to the numerator. For an 808-byte posting tuple, 812 with its line pointer, that is 9 per leaf, and 1,000,000 rows under one key need 7576 tuples, so 843 leaves — the 17.10 rebuild is 849 blocks including internal pages and the metapage.

NULL runs deduplicate because `_bt_keep_natts_fast` treats two NULLs as equal ([nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4890-L4902)), which is why the NULL run enters the mixture as one key group sized from `pg_stats.null_frac`. The most-common-value classes use `pg_stats.most_common_freqs`, which `ANALYZE` stores as sample frequencies of the total row count ([analyze.c:2664-2684](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2664-L2684), [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L218)), and the remainder class uses `n_distinct`, whose sign is decided by the 10%-of-rows rule ([analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612)).

### The gate, and what each conjunct rejects

Build-time deduplication needs `allequalimage`, a non-unique index and the `deduplicate_items` reloption ([nbtsort.c:1147-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)), and `_bt_allequalimage` refuses INCLUDE indexes outright before it looks up `BTEQUALIMAGE_PROC` — support function 4 — for every key opclass ([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5170), [nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L707-L712)). Each of those conditions has a catalog form, and on the 17.10 fixture set each one rejects a different index:

| Conjunct | Catalog source | What it rejected here |
|---|---|---|
| `NOT indisunique` | `pg_index.indisunique` | `i_uniq` |
| `deduplicate_items` | `pg_options_to_table(reloptions)` ([reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)) | `i_dupoff`, 2749 blocks with 100 rows per key, read 0.1% |
| `indnatts = indnkeyatts` | `pg_index` | `i_inc_lowcard` and `i_inc_hicard` |
| every key opclass has `pg_amproc.amprocnum = 4` | `pg_amproc` | all 68 indexes on 12.2, none on 14.23/17.10 |
| no key collation with `collisdeterministic = false` | `pg_index.indcollation` ([pg_index.h:53-54](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L53-L54)) joined to `pg_collation` ([pg_collation.h:40](../../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40)) | `i_ci` on the ICU build |

That last row is the new one, and it is load-bearing. The `pg_amproc` probe accepts all 72 indexes on the 17.10 server, `i_ci` included, because `text_ops` really does have an equal-image support function — `btvarstrequalimage`, which returns false at run time for a nondeterministic collation ([varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2599-L2613)). Measured on a 17.10 build configured `--with-icu`, over 500,000 rows with 100 rows per key:

| index | collation | live | rebuilt | statement as filed | same statement, collation conjunct removed |
|---|---|---|---|---|---|
| `i_ci` | `provider = icu, deterministic = false` | 3611 | 3611 | **0.1%** | **88.2%** |
| `i_cd` | default | 462 | 462 | 8.2% | 8.2% |

Same data, same 100-rows-per-key shape: the engine deduplicated the deterministic index down to 462 blocks and refused to deduplicate the nondeterministic one, leaving it 7.8x larger. Without the conjunct the sweep would have called a healthy 28 MB index 88.2% waste.

### How the same statement behaves on 12.2, 14.23 and 17.10

Three isolated servers, each built from its own pin, carrying the same DDL and the same generated data; `autovacuum = off`, `fsync = off`, `block_size` 8192. Ground truth per index is a `CREATE INDEX CONCURRENTLY` copy.

| | 12.2 | 14.23 | 17.10 |
|---|---|---|---|
| statement runs unchanged | yes | yes | yes |
| B-tree indexes swept, rebuild copies included | 68 | 68 | 72 |
| blocks covered | 133,677 | 94,910 | 103,056 |
| three consecutive runs | 26.1 / 19.9 / 17.7 ms | 32.9 / 20.6 / 17.6 ms | 28.7 / 20.4 / 17.2 ms |
| indexes the `pg_amproc` probe accepts | 0 of 68 | 68 of 68 | 72 of 72 |
| indexes credited with deduplication | 0 | 34 | 36 |
| `expected_blocks` identical to v12 Method A | 34 of 34 scored | 17 of 34 | 18 of 36 |
| rows reported `unmeasured` | 1 | 1 | 1 |
| that row's status | `reltuples 0, table has live rows` | `reltuples unknown` | `reltuples unknown` |

The 12.2 column is the portability claim in numbers: same statement, same answers as the v12 arithmetic, in every cell, because the gate cannot open there. The 14.23 column matters because 14 is the first major with both deduplication and the `-1` sentinel, so it exercises the two version branches at once; 13 has the first and not the second, while 15 and 16 have both; none of the three was run.

### Measured accuracy, per fixture

Every fixture in this first table is freshly built with **0% reclaimable space** — live equals rebuilt in all 54 cells — so any non-zero reading is model error. 1,000,000 rows, `bigint` key, unless noted.

| fixture | shape | 12.2 live / reported | 14.23 live / reported | 17.10 live / reported |
|---|---|---|---|---|
| `i_q1` | distinct keys | 2745 / 0.0% | 2745 / 0.0% | 2745 / 0.0% |
| `i_q2` | 2 rows per key | 2749 / 0.1% | 2475 / 2.4% | 2475 / 0.4% |
| `i_q5` | 5 rows per key | 2748 / 0.1% | 1426 / −0.3% | 1426 / −2.5% |
| `i_q10` | 10 rows per key | 2749 / 0.1% | 1157 / −2.3% | 1157 / −0.6% |
| `i_q100` | 100 rows per key | 2749 / 0.1% | 839 / −0.2% | 839 / −0.1% |
| `i_q1000` | 1000 rows per key | 2749 / 0.1% | 896 / 5.5% | 896 / 5.5% |
| `i_qall` | one key | 2749 / 0.1% | 849 / 0.2% | 849 / 0.2% |
| `i_null` | 25% NULL, rest distinct | 2746 / 0.0% | 2271 / −0.1% | 2271 / −0.4% |
| `i_skew` | one value 25%, rest distinct | 2746 / 0.0% | 2271 / **53.1%** | 2271 / **53.4%** |
| `i_ff50` | 50 rows per key, `fillfactor = 50` | 4978 / 0.1% | 1547 / 0.2% | 1547 / 0.4% |
| `i_var` | variable-width text, 400,000 rows | 3222 / −2.9% | 3211 / −2.3% | 3211 / −4.4% |
| `i_uniq` | unique, distinct keys | 2745 / 0.0% | 2745 / 0.0% | 2745 / 0.0% |
| `i_dupoff` | 100 rows per key, `deduplicate_items = off` | 2749 / 0.1% | 2749 / 0.1% | 2749 / 0.1% |
| `i_inc_bothkeys` | `(a, d)`, 1000 x 5 distinct | 2749 / 0.1% | 896 / 5.5% | 896 / 5.5% |
| `i_inc_lowcard` | `(a) INCLUDE (d)`, same data | 2749 / 0.1% | 2749 / 0.1% | 2749 / 0.1% |
| `i_q5_part` | 20% partial sibling of `i_q5` | 551 / 0.0% | 551 / −0.7% | 551 / −2.4% |
| `i_q100_part` | 20% partial sibling of `i_q100` | 552 / 0.2% | 191 / −5.8% | 191 / −5.8% |
| `i_qall_part` | 20% partial sibling of `i_qall` | 552 / 0.2% | 171 / −0.6% | 171 / 1.2% |

The second table is the one that matters operationally: indexes with real reclaimable space, plus the three statistics traps. `reported / floor` are the statement's two columns.

| fixture | what it is | 12.2 live -> rebuilt (true) reported / floor | 14.23 | 17.10 |
|---|---|---|---|---|
| `i_seqdel` | distinct keys, 90% deleted + VACUUM | 2745 -> 276 (89.9%) **89.9 / 89.9** | same | same |
| `i_dupdel` | 10 keys, 90% deleted + VACUUM | 2749 -> 278 (89.9%) **90.0 / 90.0** | 850 -> 87 (89.8%) **89.8 / 67.5** | 850 -> 87 (89.8%) **89.8 / 67.5** |
| `i_dupdelp` | partial over 7 keys, 90% of the subset deleted + VACUUM | 552 -> 57 (89.7%) **89.7 / 89.7** | 171 -> 19 (88.9%) **88.9 / 67.3** | 171 -> 19 (88.9%) **88.3 / 64.9** |
| `i_alldead` | every row deleted + VACUUM + ANALYZE | 551 -> 1 (99.8%) **99.8 / 99.8** | same | same |
| `i_stale` | 19% deleted, never vacuumed or analyzed | 2745 -> 2224 (19.0%) **19.0 / 19.0** | same | same |
| `i_stale_part` | partial subset drained, never analyzed | 551 -> 30 (94.6%) **0.0 / 0.0**, caveat `never analyzed` | same | same |
| `i_trunc` | `TRUNCATE` and reload, no ANALYZE | 825 -> 825 (0.0%) **unmeasured** | unmeasured | unmeasured |
| `i_grow` | doubled since the last ANALYZE | 2745 -> 2745 (0.0%) **49.9 / 49.9**, caveat `row-count sources disagree` | same | same |

`i_dupdel` is the load-bearing cell: a duplicate-key index with 89.8% real bloat reads 89.8% on 14.23 and 17.10 through posting-list arithmetic and 90.0% on 12.2 through one-tuple-per-row arithmetic, from the same statement text. The v12 model on the 17.10 index reads 67.5%, which is the floor column and still actionable.

Scored against the rebuilds, excluding the single `unmeasured` row:

| | 12.2 | 14.23 | 17.10 |
|---|---|---|---|
| cells scored | 33 | 33 | 35 |
| exact — Method A / v17 sweep / proposal | 11 / 11 / 11 | 7 / 7 / 9 | 7 / 7 / 8 |
| within 1% — Method A / v17 sweep / proposal | 30 / 30 / 30 | 12 / 16 / 20 | 12 / 14 / 19 |
| within 5% — Method A / v17 sweep / proposal | 31 / 31 / 31 | 14 / 22 / 25 | 15 / 23 / 25 |
| largest model-vs-rebuild gap in blocks, statistics traps excluded | 94 (`i_var`, 3316 against 3222) | 73 (`i_var`, 3284 against 3211) | 142 (`i_var`, 3353 against 3211) |
| largest relative model-vs-rebuild gap, traps excluded | 2.9% (`i_var`) | 9.5% (`i_q10_part`) | 8.9% (`i_q10_part`) |

The traps excluded from those two rows are `i_grow`, `i_skew`, `i_stale` and `i_stale_part`, all four of which are statistics states rather than arithmetic; each is covered in [Where the proposal is still wrong](#where-the-proposal-is-still-wrong). The remaining gaps run both ways: a model *below* the rebuild reports phantom bloat (`i_q1000`, 847 against 896, 5.5%), and a model *above* it reports a small negative bloat (`i_var`, `i_q10_part`).

### Read the floor, not the point estimate

Alert when `wasted_space_pct_floor` crosses the threshold, the `status` is `ok`, and `caveats` contains neither `never analyzed` nor `row-count sources disagree`. At a 30% threshold, on every one of the three servers:

| rule | true positives | false positives | false negatives |
|---|---|---|---|
| floor + status + caveats (proposed) | 4 of 5 | **0 on all three servers** | 1 (`i_stale_part`) |
| `wasted_space_pct` point estimate alone | 4 of 5 | 1 on 12.2 (`i_grow`), 2 on 14.23 and 17.10 (`i_grow`, `i_skew`) | 1 |
| the deduplication-aware sweep already on this page | 4 of 5 | 1 on 12.2, 2 on 14.23, 3 on 17.10 (`i_grow`, `i_skew`, `i_ci`) | 1 |

The rows the status and caveats take out of alerting are exactly `i_grow` (0% true), `i_stale` (19.0% true, below the threshold anyway), `i_stale_part` (94.6% true — the one false negative) and `i_trunc` (0% true). The false negative is the partial-index staleness this page already documents: one `ANALYZE` on the table repairs it.

The floor is what rescues `i_skew`: its point estimate reads 53.4% on a healthy index while its floor reads −20.9%, so the rule never fires. That is the general shape of the guard — a wide gap between the two columns means the answer rests entirely on a duplication estimate, and only the floor is safe to act on.

### Where the proposal is still wrong

**One hot value plus mostly-distinct keys, at the default statistics target.** `i_skew` is 25% one value and 75% distinct. `ANALYZE` estimated 81,281 distinct values against a true 750,001 and stored it as a positive absolute count, so the statement credited compression that does not exist. This is a statistics failure, not an arithmetic one, and it is fixable from outside the statement — measured on 17.10:

| statistics state | `n_distinct` | reported | floor |
|---|---|---|---|
| default target (`default_statistics_target` 100) | 81281 | 53.4% | −20.9% |
| `ALTER TABLE t_skew ALTER COLUMN k SET STATISTICS 1000; ANALYZE` | −0.473248 | −12.4% | −20.9% |
| `ALTER TABLE t_skew ALTER COLUMN k SET (n_distinct = -0.75); ANALYZE` | −0.75 | **0.1%** | −20.9% |
| reset back to the default | 81402 | 53.3% | −20.9% |

**Long posting lists lose 5-8% to page packing.** Where a key group needs more than one posting tuple, the last one is partial and pages no longer pack evenly. `i_q1000`, `i_inc_keyonly` and `i_inc_bothkeys` all model 847 blocks against a 896-block rebuild on both 14.23 and 17.10, 5.5% short, so each reports 5.5% bloat on an index with nothing to reclaim. The 33-byte-key `i_cd` is the same effect at 424 modelled against 462 built: 8.2%.

**A partial index's duplication ratio is a guess.** The statement only lets a partial index use an absolute `n_distinct` count, and labels those rows `partial: duplicates from table statistics`. When the predicate correlates with the key it under-credits instead of over-crediting: `i_q10_part` models 541 blocks against a 497-block index on 17.10 (544 on 14.23), 8.9% high, because it declined to credit the subset's two-rows-per-key at all. An over-modelled index reports negative bloat — −8.9% here — which is harmless for alerting and useless for sizing.

**`i_var` drifts with `avg_width`.** A variable-width text index models 3316 blocks against a 3222-block rebuild on 12.2 and 3353 against 3211 on 17.10 — 2.9% and 4.4% high — for the reason [Method A-prime](#method-a-prime-still-fixes-variable-key-width) documents: one MAXALIGN of a sampled mean width is not the mean of the MAXALIGNs.

### Settings the proposal touches

| Setting | Context in v17 | Apply scope |
|---|---|---|
| `statement_timeout`, `lock_timeout` | `PGC_USERSET` ([guc_tables.c:2611-2631](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631)) | session/transaction; both are set by the statement itself |
| `default_statistics_target` | `PGC_USERSET` ([guc_tables.c:2071-2078](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2078)) | session/transaction; raising it only changes estimates after the next `ANALYZE` |
| `block_size` | `PGC_INTERNAL` preset ([guc_tables.c:3268-3277](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3268-L3277)) | read-only; the statement reads it with `current_setting('block_size')` |

Neither the statement nor the two statistics repairs above need a setting that requires a reload or a restart. `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS` and `SET (n_distinct = ...)` are DDL on the table, not GUC changes, and take effect at the next `ANALYZE`; the per-index `deduplicate_items` reloption is read, never written, by the sweep.

### Follow-up: two reporting defects, not arithmetic defects

Both defects are confirmed against the pinned checkout, and the framing holds: neither column feeds `expected_blocks`, `floor_blocks`, `live_rows`, `key_groups`, `tids`, the deduplication gate, `status` or `caveats`. **No bloat percentage anywhere on this page changes.** What changes is what a reader may conclude from a row.

| Column, as filed | What the expression actually reports | Correction |
|---|---|---|
| `fsm_bytes > 0 AS has_freed_pages` | that this index's free space map has been written at least once since its current relfilenode was created — a fact about history, not about now | rename to `fsm_written_since_build`; get page classes from `VACUUM VERBOSE` or contrib |
| `pg_size_pretty(greatest(actual_bytes - expected_blocks * bs, 0)::bigint) AS wasted` | reclaimable bytes where the model under-predicts, and `0 bytes` where it over-predicts by *any* amount | drop the `greatest(..., 0)` clamp, in the column and in the `ORDER BY` key, so both columns carry the same sign convention |

Both as-filed labels are quoted above as they shipped. The columns are `fsm_written_since_build` and `wasted_space` in the statements now, the second having also lost the word "bloat" from its percentage siblings in [Follow-up: the output columns say wasted_space, not bloat](#follow-up-the-output-columns-say-wasted_space-not-bloat).

### Defect 1: the FSM column is a history bit

`pg_relation_size(c.oid, 'fsm')` is a live `stat()` over one fork's segment files ([dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343)), so `fsm_bytes > 0` asks one question: does this index have a free space map file of nonzero length. For a B-tree index the map is a bitmap in all but name — it "only track[s] whether pages are completely free or in-use", storing 0 for a used page and `BLCKSZ - 1` for a free one ([indexfsm.c#NOTES](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)) — and its file length is fixed, not proportional to the number of free pages. `RecordPageWithFreeSpace` reaches `fsm_readbuf(rel, addr, true)`, whose `extend` flag creates the fork ([freespace.c#RecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L186-L204), [freespace.c#fsm_set_and_search](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L648-L682), [freespace.c#fsm_readbuf](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L557-L631), [freespace.c#fsm_extend](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L633-L646)). At `block_size` 8192, `SlotsPerFSMPage` is 4069 ([fsm_internals.h#SlotsPerFSMPage](../../../../raw/postgres-17/src/include/storage/fsm_internals.h#L47-L61)), above the 1626 a three-level tree needs ([freespace.c#FSM_TREE_DEPTH](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L78)), so the slot for any of an index's first 4069 blocks sits on physical FSM block 2 ([freespace.c#fsm_get_location](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L497-L510), [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L461-L495)): the first recorded page extends the fork to three blocks, 24576 bytes, and the four-thousandth adds nothing.

**Direction one: every recorded page handed back out, and the column still reads true.** `_bt_allocbuf` takes pages from the map through `GetFreeIndexPage`, which marks the page used as a documented side effect — `RecordUsedIndexPage` writes category 0 into the same slot ([nbtpage.c#_bt_allocbuf](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L868-L905), [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L32-L46), [indexfsm.c#RecordUsedIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L57-L65)). The slot's value changes; the file's length does not. Nothing in nbtree ever shortens it: the only in-place shortening path is `FreeSpaceMapPrepareTruncateRel` ([freespace.c#FreeSpaceMapPrepareTruncateRel](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L273-L359)), reached only from `RelationTruncate` ([storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L288-L326)), whose live callers are heap truncation and the index truncation inside `heap_truncate` ([vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2638-L2650), [heapam_handler.c:629](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L629), [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3055-L3092)); the one call in an index AM is inside `#ifdef NOT_USED` ([spgvacuum.c#NOT_USED-truncate](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L889-L901)). The column therefore goes back to false only when the index's storage is replaced — `REINDEX` and transactional `TRUNCATE` both call `RelationSetNewRelfilenumber`, which drops the old storage and creates the main fork alone ([index.c#reindex_index-relfilenumber](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789), [tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2160-L2189), [relcache.c#RelationSetNewRelfilenumber](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3842-L3899)). Even the truncation path leaves it true: `FreeSpaceMapPrepareTruncateRel(rel, 0)` maps block 0 to leaf slot 0 and so returns the two upper-level FSM blocks rather than zero, leaving a 16384-byte fork of zeroed slots ([freespace.c#FreeSpaceMapPrepareTruncateRel](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L299-L359)). The column reads false precisely when the index is fresh and true from its first recyclable page onward, which is the opposite of a health signal.

**Direction two: thousands of pages deleted, and the column can still read false.** `btvacuumpage` records a deleted page in the map only if it is recyclable *now*, and otherwise just counts it — "Already deleted page ... Can't recycle yet" ([nbtree.c#btvacuumpage-page-classes](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1189)). Recyclable means deleted *and* the page's stored `safexid` is old enough to be invisible to every snapshot ([nbtree.h#BTPageIsRecyclable](../../../../raw/postgres-17/src/include/access/nbtree.h#L279-L318), [nbtree.h#BTDeletedPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L230-L236)). Pages that this VACUUM deleted are queued in `BTVacState.pendingpages` and revisited once at the end, where `_bt_pendingfsm_finalize` stops at the first page whose `safexid` is still visible and leaves the rest to a future VACUUM ([nbtpage.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2984-L3055)); `btvacuumscan` then vacuums the FSM only if something was recorded ([nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1043-L1059)). Half-dead pages are never recorded free at all ([nbtree.c#btvacuumpage-halfdead](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1180-L1189)). And the bloat class this whole page is about never reaches the map in the first place: nbtree deletes a page only once it is "completely empty of items" ([README#deleting-entire-pages](../../../../raw/postgres-17/src/backend/access/nbtree/README#L232-L246)), so partly-emptied leaves — the documented mechanism ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)) — stay in the live chain and out of the FSM.

What a reader may legitimately conclude is one sentence: **at least one page of this index was recorded free in its free space map at some point since the index's current relfilenode was created.** Equivalently, the index's storage has not been replaced since it last had a recyclable page. It says nothing about how many pages are free, deleted or half-dead now, and nothing about how many blocks a rebuild would return — `expected_blocks` is the only estimate of that on the row. Two more reasons not to over-read it: the map is a hint, written with `MarkBufferDirtyHint`, not WAL-logged, and zeroed rather than reported when corrupt, because "The FSM information is not accurate anyway" ([freespace.c#fsm_readbuf-zero-on-error](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L586-L607)); and `_bt_allocbuf` marks a page used before validating it, so a page it then rejects with `elog(DEBUG2, "FSM returned nonrecyclable page")` "will be lost to use till the next VACUUM" ([nbtpage.c#_bt_allocbuf-reject](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L894-L969)). The map can under-report as well as over-report.

Renamed accordingly:

```sql
       fsm_bytes > 0                                    AS fsm_written_since_build,
```

### Where a current count of empty, deleted and reusable pages comes from

Not from core SQL. No catalog column and no core function reports a B-tree index's page classes; the page-reader inventory in [What no core-SQL method can measure on v17](#what-no-core-sql-method-can-measure-on-v17) is unchanged. There is exactly one core source, and it is not a query:

| Source | Kind | What it reports | Cost and access |
|---|---|---|---|
| `VACUUM VERBOSE` per-index line | core, text output | `pages: N in total, N newly deleted, N currently deleted, N reusable`, printed straight from `IndexBulkDeleteResult`, where `pages_deleted` and `pages_free` "refer to free space within the index file" and nbtree treats `pages_free` as whole-index state rather than per-VACUUM state ([vacuumlazy.c#verbose-index-line](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732), [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L66-L84), [nbtree.c#pages_free-is-index-state](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L949-L967)) | a VACUUM, and it prints nothing when no index scan ran or the 2% bypass fired — see [Method D](#method-d-new-message-no-row-count-new-blind-spot) |
| `pgstatindex` (`pgstattuple`) | contrib | `deleted_pages` counts every `P_ISDELETED` page, recyclable or not, and `empty_pages` is in fact the half-dead count — the code comments it as `/* this is the "half dead" state */` ([pgstatindex.c#page-classes](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#result-tuple](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L372), [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31)) | one buffered read of every index page under `AccessShareLock`; superuser |
| `pg_freespace` (`pg_freespacemap`) | contrib | the current reusable count, as `count(*) FILTER (WHERE avail > 0)` over `pg_freespace(idx)`; for an index `avail` is 0 or `MaxFSMRequestSize`, because `RecordFreeIndexPage` stores `BLCKSZ - 1` and that maps to category 255 ([pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L18-L50), [freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L249-L271), [freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L423-L435)) | one FSM lookup per main-fork block; `REVOKE`d from `PUBLIC`, granted to `pg_stat_scan_tables` at version 1.2 ([pg_freespacemap--1.1.sql#L6-L25](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L6-L25), [pg_freespacemap--1.1--1.2.sql#GRANT](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L1-L7)) |
| `bt_page_stats` / `bt_multi_page_stats` (`pageinspect`) | contrib | per-page `type`: `d` deleted leaf, `D` deleted internal, `e` half-dead, `l` leaf, `i` internal, `r` root, plus raw `btpo_flags` ([btreefuncs.c#page-type](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L125-L163), [pageinspect--1.11--1.12.sql#bt_multi_page_stats](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.11--1.12.sql#L6-L23)) | one buffer per page; superuser |

The pair to reach for is `pgstatindex` plus `pg_freespace`: `deleted_pages` is a superset of the reusable pages, so `deleted_pages` minus the `avail > 0` count is the deleted-but-not-yet-recyclable population on a quiet index, and `empty_pages` gives the half-dead class separately. The `avail > 0` idiom is the one `pg_freespacemap`'s own regression test uses, on a B-tree index among others ([sql/pg_freespacemap.sql:8-12](../../../../raw/postgres-17/contrib/pg_freespacemap/sql/pg_freespacemap.sql#L8-L12), [expected/pg_freespacemap.out#btree-rows](../../../../raw/postgres-17/contrib/pg_freespacemap/expected/pg_freespacemap.out#L29-L53)). Both are still contrib, still superuser to install, and still outside this page's core-SQL constraint — which is the honest answer: the statement cannot report page classes, so it should not imply that it does.

One core-SQL route exists in principle and is not taken here. `pg_read_binary_file` will return the `_fsm` fork's bytes to a member of `pg_read_server_files` ([genfile.c#convert_and_check_filename](../../../../raw/postgres-17/src/backend/utils/adt/genfile.c#L41-L91), [genfile.c#pg_read_binary_file_common](../../../../raw/postgres-17/src/backend/utils/adt/genfile.c#L254-L273)), the same trick [the deduplication-after-upgrade page](btree-deduplication-after-pg-upgrade.md) uses on a metapage byte, and an FSM page is a flat `uint8` array whose leaf nodes follow `fp_next_slot` and `NonLeafNodesPerPage` upper nodes ([fsm_internals.h#FSMPageData](../../../../raw/postgres-17/src/include/storage/fsm_internals.h#L20-L61)). That yields the *reusable* count only — the map has no notion of deleted versus half-dead — and the offsets depend on `BLCKSZ` and `MAXALIGN`, so it is a decoding exercise, not a column. Nothing on this page implemented or measured it.

### Defect 2: the byte column is clamped and the percentage is not

`wasted_space_pct` is `100 * (1 - expected_blocks * bs / actual_bytes)`, so it goes negative exactly when the model predicts a fresh sorted rebuild *larger* than the index on disk. That has two causes, and they are not the same:

1. **The model over-predicts.** Every negative *reported* cell in the fixture tables above is this case: `i_var` (one MAXALIGN of a sampled mean width), `i_q10_part` (a partial index's duplication declined), `i_q5`/`i_q10`/`i_q100`/`i_null` (posting-list packing loss), and — by construction — `wasted_space_pct_floor` on any index the engine really did deduplicate, which is why `i_skew`'s floor reads −20.9% on a healthy 2271-block index.
2. **A rebuild really would grow the index.** This page measured one, in the *true* bloat column rather than the reported one: `idx_dup` sits at 396 blocks live and 426 after a `CREATE INDEX CONCURRENTLY` rebuild, true bloat −7.6%, because an all-duplicate leaf split packs to `BTREE_SINGLEVAL_FILLFACTOR` 96 while the sorted build caps each posting list at a tenth of a page ([nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416), [nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)).

So a negative `wasted_space_pct` does not prove model error; it proves that *a rebuild is not predicted to shrink this index*, and only a rebuild separates the two causes. Either way the magnitude is information: it is the distance between the model and the file, which is the one signal on the row that says how much weight the point estimate can carry.

The clamp deletes that magnitude. Clamped, `wasted_space` collapses "the model is exactly right" and "the model over-predicts by 10 MB" into the same `0 bytes`. Three examples, derived arithmetically at `block_size` 8192 from block counts already reported above, not re-measured:

| Row | actual | model | signed difference | reported as filed |
|---|---|---|---|---|
| `i_var`, 12-through-17 statement on 17.10 | 3211 blocks, 26,304,512 B | 3353 blocks, 27,467,776 B | −1,163,264 B (−142 blocks) | `0 bytes` beside −4.4% |
| 5 rows per key, [v17 dedup-aware sweep](#a-deduplication-aware-sweep-for-v17) on 17.10 | 1426 blocks, 11,681,792 B | 2745 blocks, 22,487,040 B | −10,805,248 B (−1319 blocks) | `0 bytes` beside −92.5% |
| `idx_dup`, v12 Method A on 17.10 | 396 blocks, 3,244,032 B | 1374 blocks, 11,255,808 B | −8,011,776 B (−978 blocks) | `0 bytes` beside −247.0% |

Consumers the mixed convention breaks:

- **Any byte threshold or ranking.** "Alert when `wasted_space` exceeds 100 MB" and "show me the largest `wasted_space`" cannot distinguish a well-modelled index from one the model over-shot by 10 MB, because both print `0 bytes`. The failure is in the diagnostic, not in the bloat detection.
- **The statement's own top-N triage.** The key as filed, `ORDER BY greatest(actual_bytes - floor_blocks * bs, 0) DESC NULLS FIRST` with `LIMIT 20`, ties every over-predicted row at zero, and PostgreSQL documents that a `LIMIT` without a unique ordering returns "an unpredictable subset of the query's rows" ([queries.sgml#limit-ordering](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1940-L1947)). Unclamped, those rows get distinct decreasing keys and a deterministic tail.
- **Any aggregation.** Summing the byte difference behind `wasted_space` over a database charges each over-prediction as zero instead of a negative, so a cluster-wide "reclaimable" total is biased upward and cannot be reconciled with the per-row percentages.
- **A human reading one row.** `0 bytes` next to −92.5% reads as a tool bug and invites the reader to pick whichever column suits. Worse, it silently disables this page's own guard: [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate) says a wide gap between the two columns means the answer rests entirely on a duplication estimate, and that rule is a sign-reading rule.

One convention, applied to both columns: **signed, positive means the index is larger than the model (reclaimable if the model is right), negative means the model is larger than the index (over-prediction, or a rebuild that would grow it).** Clamping both instead would be the wrong trade — it would hide the over-prediction in the percentage too, and `wasted_space_pct_floor` exists precisely so that the two models can be seen disagreeing. `pg_size_pretty(bigint)` renders negatives symmetrically: the unit is chosen from the absolute value, `half_rounded` rounds away from zero, and the divisor comment says division is used "to ensure positive and negative values are rounded in the same way" ([dbsize.c#half_rounded](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L34-L57), [dbsize.c#pg_size_pretty](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L565-L604)). The in-tree regression test asserts that shape for both the `bigint` and `numeric` variants — `-10 bytes`, `-977 kB`, `-954 MB`, `-931 GB`, `-909 TB` ([sql/dbsize.sql:1-4](../../../../raw/postgres-17/src/test/regress/sql/dbsize.sql#L1-L4), [expected/dbsize.out#negatives](../../../../raw/postgres-17/src/test/regress/expected/dbsize.out#L1-L13)). Applying that algorithm by hand, the three rows above print `-1136 kB`, `-10 MB` and `-7824 kB`.

### The corrected columns

Three expressions in the [12-through-17 statement](#follow-up-one-statement-for-postgresql-12-through-17), all in its final `SELECT`; these two defects change nothing above that `SELECT`, though the later rename does move the statement tag. Both statements on this page now carry these corrections, so the blocks above are the corrected text and the diff is here:

```sql
       -- signed, like wasted_space_pct: negative means the model predicts a
       -- rebuild larger than the index on disk, so this is not a size
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         pg_size_pretty((actual_bytes - expected_blocks * bs)::bigint) END  AS wasted_space,
       -- the FSM fork has been written at least once since this index's
       -- relfilenode was created; not a count of free pages, and not current
       fsm_bytes > 0                                    AS fsm_written_since_build,
...
 ORDER BY (actual_bytes - floor_blocks * bs) DESC NULLS FIRST
```

Notes on the shape of the fix:

- `wasted_space_pct` and `wasted_space_pct_floor` are already signed, so the clamp correction leaves their expressions untouched; only their labels moved, and that came later.
- The `ORDER BY` keeps ranking on `floor_blocks` rather than `expected_blocks`, deliberately: the alerting rule on this page reads the floor. Only the clamp goes.
- The `live_rows IS NULL` guard stays, so an `unmeasured` row still emits NULL rather than a signed number.
- Clamp late, not early. A consumer that genuinely wants a non-negative size can wrap the column in `greatest(..., 0)` itself; the statement should not make that choice on its behalf.
- If a consumer parses the output, emit `(actual_bytes - expected_blocks * bs)::bigint` raw — as `wasted_space_bytes` — and let the consumer format it; `pg_size_pretty` is for human reading.

### What the corrections change, and whether the v17 sweep needs them too

**No reported bloat percentage changes.** `wasted_space_pct` and `wasted_space_pct_floor` are the same expressions before and after, over the same `expected_blocks`, `floor_blocks` and `live_rows`, so every cell in [Measured accuracy, per fixture](#measured-accuracy-per-fixture), every error figure in [How the same statement behaves on 12.2, 14.23 and 17.10](#how-the-same-statement-behaves-on-122-1423-and-1710), and the 30%-threshold true/false-positive counts in [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate) stand unchanged — that alerting rule reads `wasted_space_pct_floor`, `status` and `caveats` only, none of which is touched.

What does change on the already-measured fixtures:

- The `wasted_space` cell on every row where the model over-predicts. Across the 54 cells of the fresh-fixture table (18 fixtures on 3 servers), at least 16 of them: `i_var` on 12.2; `i_q5`, `i_q10`, `i_q100`, `i_null`, `i_var`, `i_q5_part`, `i_q100_part` and `i_qall_part` on 14.23; and that same list minus `i_qall_part` on 17.10. It is "at least" because a cell printed `0.0%` may be a small negative rounded to one decimal, and the rounded tables cannot be un-rounded.
- Nothing in the real-bloat table: every percentage there is positive, and `i_trunc` stays NULL in both columns.
- Nothing that was ever printed for the renamed column. No table on this page reports `has_freed_pages`, and its per-fixture values were not recorded, so the rename cannot be scored against the fixtures without re-running them.
- Nothing that depends on the sort key: the measurements were collected with `WHERE actual_bytes > 1024 * 1024` and `LIMIT 20` removed, as [the statement's own note](#follow-up-one-statement-for-postgresql-12-through-17) says. The un-clamped key only matters to a production run that keeps the `LIMIT`.

**The earlier deduplication-aware sweep needs both corrections, in the same words, and now carries them.** As filed, [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17) had `fsm_bytes > 0 AS has_freed_pages`, the same clamp on its byte column (then labelled `AS wasted`), an unclamped percentage (then `AS bloat_pct`), and the same clamped `ORDER BY` key. Two differences in how much they matter there:

- The clamp hides more, because that sweep has no floor column: the sign of `wasted_space_pct` is the only trustworthiness signal it emits, and its documented blind spot is an order of magnitude larger than the proposal's — 10,805,248 bytes of over-prediction printed as `0 bytes` at 5 rows per key, against 1,163,264 bytes in the proposal's worst fresh cell.
- Its `ORDER BY` key uses `expected_blocks`, the same quantity as its `wasted_space`, so removing the clamp there also makes the ranking and the size column agree, which the proposal reaches differently by ranking on the floor.

The corrected columns for that sweep are the same two expressions, with its own `status` semantics untouched:

```sql
       CASE WHEN live_rows IS NULL THEN NULL ELSE
         pg_size_pretty((actual_bytes - expected_blocks * bs)::bigint) END  AS wasted_space,
       fsm_bytes > 0                                  AS fsm_written_since_build,
...
 ORDER BY (actual_bytes - expected_blocks * bs) DESC NULLS FIRST
```

### Follow-up: the output columns say wasted_space, not bloat

Done, and it costs nothing: both statements now label their reporting columns `wasted_space` and `wasted_space_pct`, the 12-through-17 statement adds `wasted_space_pct_floor`, and both statement tags say `wasted_space` too. The change is confined to `AS` labels and two comments. Every value expression, both models, the deduplication gate, `status` and `caveats` are unchanged, so **no number in any table on this page moves**.

| As filed | Now | The value behind it |
|---|---|---|
| `wasted` | `wasted_space` | `pg_size_pretty(actual_bytes - expected_blocks * bs)`, signed since [Defect 2](#defect-2-the-byte-column-is-clamped-and-the-percentage-is-not) |
| `bloat_pct` | `wasted_space_pct` | `round(100 * (1 - expected_blocks * bs / actual_bytes), 1)` |
| `bloat_pct_floor` | `wasted_space_pct_floor` | the same percentage over `floor_blocks`, the one-tuple-per-row model |
| `/* wiki_btree_bloat_sweep_v17 */` | `/* wiki_btree_wasted_space_sweep_v17 */` | the v17 sweep's statement tag |
| `/* wiki_btree_bloat_sweep_12_17 */` | `/* wiki_btree_wasted_space_sweep_12_17 */` | the portable statement's tag |
| — | `wasted_space_bytes` | the name to give the raw `::bigint` if a consumer parses the output instead of reading it |

The rename is not only cosmetic, because "bloat" overclaims what the column holds. The PostgreSQL glossary defines bloat as "Space in data pages which does not contain current row versions, such as unused (free) space or outdated row versions" ([glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)) — per-page state, free space inside still-used pages included. Neither statement can see per-page state. Both subtract a modelled fresh-rebuild size from the file's size, which is why [What no core-SQL method can measure on v17](#what-no-core-sql-method-can-measure-on-v17) lists `LP_DEAD` space inside live leaves, the deleted-versus-half-dead split and `leaf_fragmentation` as invisible, why the number can be negative at all, and why a positive reading is a prediction about a rebuild rather than a measurement of dead space. "Wasted space" is the docs' own phrase for space a maintenance command is expected to recover ([copy.sgml#recover-the-wasted-space](../../../../raw/postgres-17/doc/src/sgml/ref/copy.sgml#L613-L621)), which is what these columns estimate.

The vocabulary also matches what ships. No SQL-visible interface in the pinned tree spells the word: `system_views.sql` contains no occurrence of the string `bloat`, no contrib SQL script and no `pg_proc.dat` entry does either, and `pgstattuple` names this class of quantity `free_space` and `free_percent` ([pgstattuple--1.4.sql#free_space](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L14-L15)). Where "bloat" does appear it is documentation prose or a C comment — such as the non-B-tree paragraph of the reindexing advice this page rests on ([maintenance.sgml#non-btree-bloat](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)) — with one unrelated exception, a static variable in the imported IANA timezone compiler ([zic.c#bloat](../../../../raw/postgres-17/src/timezone/zic.c#L641-L646)).

Two artifacts keep the old spelling on purpose: the psql capture in [The v14 unknown reltuples sentinel](#the-v14-unknown-reltuples-sentinel), because that is what the server printed, and the as-filed expression quotes in [Follow-up: two reporting defects, not arithmetic defects](#follow-up-two-reporting-defects-not-arithmetic-defects). Every other mention on this page names the columns as they stand now.

### What the rename cannot change

- **Any value.** `AS` assigns an output column label, used for display and for reference elsewhere in the same query; the select-list expression is untouched ([queries.sgml#Column-Labels](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1556-L1602)).
- **Row order.** Both statements sort on an expression — `ORDER BY (actual_bytes - floor_blocks * bs) DESC NULLS FIRST` in the portable statement, the `expected_blocks` form in the v17 sweep — never on a label. `ORDER BY` may name an output column, but only when the sort key is that bare name ([queries.sgml#sort-by-output-column](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1867-L1886)), and neither statement does that.
- **The query ID.** Two independent reasons. A comment never reaches the parse tree, and `pg_stat_statements` hashes the tree, not the text: `JumbleQuery` walks the finished `Query` node ([queryjumblefuncs.c#JumbleQuery](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L104-L127)), and the docs say the value "is computed on the post-parse-analysis representation of the queries" ([pgstatstatements.sgml#queryid](../../../../raw/postgres-17/doc/src/sgml/pgstatstatements.sgml#L613-L639)). The label does reach the tree, as `TargetEntry.resname`, but it is declared `pg_node_attr(query_jumble_ignore)` ([primnodes.h#TargetEntry](../../../../raw/postgres-17/src/include/nodes/primnodes.h#L2186-L2203)) and `gen_node_support.pl` emits no jumble code for such a field ([gen_node_support.pl#query_jumble_ignore](../../../../raw/postgres-17/src/backend/nodes/gen_node_support.pl#L1283-L1324)) into the generated `queryjumblefuncs.funcs.c` that `_jumbleNode` dispatches through ([queryjumblefuncs.c#generated-includes](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L227-L255)). Renaming a column therefore cannot split one `pg_stat_statements` entry into two.
- **Identifier limits.** The longest new label is 22 bytes. Truncation only starts at `NAMEDATALEN`, 64, so 63 usable bytes ([pg_config_manual.h#NAMEDATALEN](../../../../raw/postgres-17/src/include/pg_config_manual.h#L22-L29)), where the lexer's `downcase_truncate_identifier` ([scan.l:1096-1103](../../../../raw/postgres-17/src/backend/parser/scan.l#L1096-L1103)) reaches `truncate_identifier` and raises `NOTICE: identifier "..." will be truncated` ([scansup.c#truncate_identifier](../../../../raw/postgres-17/src/backend/parser/scansup.c#L84-L105)).

### What a consumer must change

1. **Anything that selects the old names.** Wrapping either statement in a view or a subquery and reading `bloat_pct` now fails with `ERRCODE_UNDEFINED_COLUMN` and `column "bloat_pct" does not exist`, possibly with a "Perhaps you meant to reference the column" hint pointing at the neighbour it fuzzy-matched ([parse_relation.c#errorMissingColumn](../../../../raw/postgres-17/src/backend/parser/parse_relation.c#L3712-L3733)). Output labels have no alias-compatibility mechanism; there is nothing to deprecate gradually.
2. **Anything that sorts on the size column.** `pg_size_pretty` returns `text` for both its `int8` and `numeric` forms ([pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507)), so `ORDER BY wasted_space DESC` sorts collated strings through `bttextcmp` ([varlena.c#bttextcmp](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L1875-L1888), [varlena.c#varstr_cmp](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L1583-L1590)) and puts `9 bytes` above `10 MB`. This was already true of `wasted`; it is why the ranking belongs on the byte expression, and why a parsing consumer should take `wasted_space_bytes` instead.
3. **Log and `pg_stat_statements` text matching.** The tag survives into both, so a grep for `wiki_btree_bloat_sweep_12_17` stops matching. `log_statement` and `log_min_duration_statement` print the source string as received ([postgres.c#log_statement](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L1071-L1081), [postgres.c#duration-statement](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L1369-L1380)); `pg_stat_statements` trims only leading and trailing whitespace from the statement's slice of the source text ([queryjumblefuncs.c#CleanQuerytext](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L61-L102)) and its normalization replaces constants, so an inline comment is preserved. One wrinkle follows from the unchanged query ID: the text is written only when the hash entry is created ([pg_stat_statements.c#pgss_store-entry](../../../../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1312-L1369)), so an entry that already exists keeps showing the old tag until it is evicted or `pg_stat_statements_reset()` runs.

### Follow-up: five changes from a twelve-issue review

The twelve issues resolve into four already fixed, one mischaracterized, five applied and two rejected. Every number in this follow-up was measured on a **17.11** server built out of tree from the current pin `786db8dcf168bd9df8f55047337525ac19118b1c` — the first measurements on this page taken on the pinned commit rather than the previous 17.10 one — plus a 12.2 server for portability. `block_size` 8192, `autovacuum = off`, `fsync = off`; `pgstattuple` and `pageinspect` were installed as ground truth only and neither statement reads them. Ground truth per index is a `CREATE INDEX CONCURRENTLY` copy.

| # | Issue | Disposition |
|---|---|---|
| 1 | Negative `n_distinct` is not credited | already fixed: credited for a whole-table index since the portable statement was filed |
| 2 | The gate ignores nondeterministic collations | already fixed: `all_deterministic` conjunct |
| 3 | The gate credits INCLUDE indexes | already fixed: `indnatts = indnkeyatts` |
| 4 | Two reporting defects | already fixed: `fsm_written_since_build`, signed `wasted_space` |
| 5 | The `i_q1000` gap is page packing | **mischaracterized**: 48 of its 49 blocks are a posting-tuple count error |
| 6 | Posting tuples are not counted per key group | applied, change 1 |
| 7 | Multicolumn keys use a per-column product | applied, change 2 |
| 8 | Invisible statistics look like all-distinct keys | applied, change 3 |
| 9 | No randomly-inserted fixture | applied, change 4 |
| 10 | The baseline is not stated | applied, change 5 |
| 11 | Replace `least(reltuples, n_live_tup)` | rejected, A |
| 12 | Source the gate from `bt_metap().allequalimage` | rejected, B |

The output contract does not move. The statement still emits `wasted_space_pct`, `wasted_space_pct_floor` and a signed `wasted_space`, still alerts on the floor, and still carries the tag `wiki_btree_wasted_space_sweep_12_17`. One caveat string is added and one is refined; nothing is removed.

### Four issues the statement already fixed

Each of the four is in the [statement as filed](#follow-up-one-statement-for-postgresql-12-through-17), and three of them are measurable on 17.11 with the filed text run verbatim:

| Already fixed | Where it lives in the filed statement | Source rule it encodes | Measured on 17.11 |
|---|---|---|---|
| Negative `n_distinct` | the `key_groups` `CASE`: `WHEN c.n_distinct < 0 AND NOT i.is_partial THEN (- c.n_distinct) * greatest(i.live_rows, 0)` | `ANALYZE` stores `stadistinct` as a negative fraction of the table's rows once it estimates more than 10% distinct ([analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612), [pg_statistic.h#stadistinct](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L52-L69)) | `t_rand`, `t_seq`, `t_stale` and `t_wide` all read `n_distinct = -1`, and their key groups come out at the full row count |
| Nondeterministic collation | `all_deterministic` in `idx`, required by `dedup_applies` | `btvarstrequalimage` returns false for a nondeterministic collation even though the `pg_amproc` row exists ([varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613), [pg_collation.h:40](../../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40)) | not re-measured: this build is `--without-icu`, so no nondeterministic collation exists on it |
| INCLUDE columns | `keys_only` = `(x.indnatts = x.indnkeyatts)`, required by `dedup_applies` | `_bt_allequalimage` returns false for any index whose attribute count exceeds its key-attribute count ([nbtutils.c:5144-5148](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5148)) | `i_inc_lowcard` — `(a) INCLUDE (d)`, 1000 x 5 distinct over 1,000,000 rows — reads `dedup_applies = false`, models 2745 against a 2749-block index and a 2749-block rebuild, and reports 0.1%. The 78.1% false positive [the third follow-up](#follow-up-the-include-column-false-positive-on-v17) measured is gone |
| The two reporting defects | `fsm_written_since_build` and the unclamped `pg_size_pretty((actual_bytes - expected_blocks * bs)::bigint) AS wasted_space` | the FSM fork's length is history, not state ([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [freespace.c#fsm_extend](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L633-L646)); `pg_size_pretty` renders negatives symmetrically ([dbsize.c#half_rounded](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L34-L57)) | every row of the 17.11 run prints a signed `wasted_space` and `fsm_written_since_build = f` on the freshly built fixtures |

One more conjunct in the same gate is confirmed by measurement rather than by text: `i_dupoff` — 1000 keys x 1000 rows with `deduplicate_items = off` — reads `dedup_applies = false` and 0.1% on a 2749-block index, because `dedup_on` comes from `pg_options_to_table(reloptions)` and the build path requires it ([nbtsort.c:1151-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152), [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)).

### The mischaracterized issue: page packing

[Where the proposal is still wrong](#where-the-proposal-is-still-wrong) says of `i_q1000`, `i_inc_keyonly` and `i_inc_bothkeys`: "Where a key group needs more than one posting tuple, the last one is partial and pages no longer pack evenly." That is the wrong mechanism, and it is why the page also concluded that fixing it "would need the same per-page simulation `_bt_buildadd` performs".

Reproduced on 17.11, `i_q1000` is 896 blocks live, 896 rebuilt, 889 leaf pages and 6 internal pages, and the filed statement models 847. The 49-block gap decomposes into two errors, neither of which is uneven packing:

| Gap component | Blocks | Cause |
|---|---|---|
| Posting-tuple count | 48 | the model divides a class's rows by its TID count, which lets one posting tuple span two key groups. Groups are disjoint: `_bt_load` flushes the pending posting list whenever the next tuple's key differs ([nbtsort.c:1323-1335](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1323-L1335), [nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1029-L1057)) |
| Internal-level fanout | 1 | `int_cap` prices a pivot tuple at the leaf slot size, 20 bytes. Above a deduplicated leaf level a pivot generally carries a heap TID, so `pageinspect` measures level-1 block 3 of `i_q1000` at 212 items of 8 to 24 bytes, 185 of them with an `htid`, against the modelled 284 — five level-1 pages where the model computes four, under a 5-item root at level 2 |

Leaf **packing** is modelled correctly here: 8000 posting tuples over 889 leaves is exactly 9 data items per leaf, which is what the statement's capacity expression already returns, and `pageinspect` confirms the page layout the model assumes — leaf block 890 holds 10 items, nine 808-byte posting tuples plus a 24-byte high key whose posting list was truncated away. So the count error is expressible in SQL — that is change 1 — and no per-page simulation is needed for it.

Where the page's claim does hold is `i_cd`, and change 1 leaves it untouched; see the end of the next section.

### Change 1: round each key group up to whole posting tuples

Applied exactly as prescribed. `classfit` now carries `class_groups`, and `classpages` counts tuples per class instead of per row:

```sql
classfit AS (
    SELECT g.idxoid, g.class_rows, g.class_groups,
           least(g.class_rows / greatest(g.class_groups, 1), p.nmax) AS tids,
           p.slot, p.leaf_bytes
      FROM gclass g
      JOIN posting p ON p.idxoid = g.idxoid
     WHERE g.class_rows > 0
),
classpages AS (
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
)
```

**The arithmetic on `i_q1000`.** An 8-byte key gives `slot` 20, so the base tuple is 16 bytes and `nmax` is `floor(4 * floor((812 - 16) / 8) / 3)` = 132 TIDs in an 808-byte tuple, 812 with its line pointer. One key group of 1000 rows therefore needs `ceil(1000 / 132)` = 8 posting tuples, seven full and one holding 76 TIDs, and the build has no choice about it: `_bt_dedup_save_htid` refuses the 133rd TID because `MAXALIGN(16 + 133 * 6)` = 816 exceeds `maxpostingsize` 812 ([nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L504-L531), [nbtsort.c:1292-1308](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1292-L1308)), and the run then ends at a key change, flushing whatever is pending ([nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1029-L1057)).

| | Tuples | Leaves | Blocks | Against a 896-block rebuild |
|---|---|---|---|---|
| As filed, `class_rows / tids` | 1,000,000 / 132 = 7575.8 | `ceil(7575.8 / 9)` = 842 | 842 + 3 + 1 + 1 = 847 | +5.5%, phantom |
| Change 1, one class of 1000 groups | 1000 x `ceil(1000/132)` = 8000 | `ceil(8000 / 9)` = 889 | 889 + 4 + 1 + 1 = 895 | +0.1% |
| The build | 8000 | 889 | 889 + 6 + 1 = 896 | — |

That closes 48 of the 49 blocks. The last block is the internal-fanout error above, not packing.

**The measured statement lands at 901, not 895**, because `ANALYZE` stored 11 most-common values for `t_q1000` with frequencies near 0.0018 even though the distribution is uniform, and the [key-group mixture](#the-posting-list-arithmetic-derived-from-source) then splits the index into 11 single-group classes plus a 989-group remainder. Each class rounds up separately and each rounded-up tuple is priced at the full 812 bytes, so the model over-predicts by 6 leaves: 901 blocks against 896, `−0.6%`. The sign of the error flips and its magnitude drops from 49 blocks to 5.

**`i_cd` does not move.** Reproduced on 17.11 at 462 blocks live, 462 rebuilt, 455 leaf pages: `avg_width` 33 gives `slot` 52, `nmax` 126, and `n_distinct` 4995 over 500,000 rows is 100.1 rows per group. `ceil(100.1 / 100.1)` = 1, so the tuple count is unchanged at 4995 and the model stays at 424 blocks and +8.2%. Its 38-block gap is a **capacity** error, and it is exact rather than statistical: a 100-TID posting tuple is `MAXALIGN(48 + 600) + 4` = 652 bytes, `_bt_buildadd` keeps adding while `PageGetFreeSpace()` covers the next tuple plus a heap TID ([nbtsort.c:853-854](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854)), so 12 tuples go on the page and the twelfth is then moved to the next page and left behind as the truncated high key ([nbtsort.c#_bt_buildadd-highkey](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L873-L940)). Eleven data items per leaf, `ceil(4995 / 11)` = 455 leaves, which is what `pgstatindex` reports; the statement's `floor((leaf_bytes + tids * 6) / tuple_size)` says 12. The exact rule, `floor((bs - 48 - greatest(tuple_size + 4, floor(bs * (100 - fillfactor) / 100) - tids * 6)) / tuple_size)`, reproduces 11 for `i_cd`, 9 for `i_q1000` and 366 for a non-deduplicated `bigint` leaf, but it is a sixth change and was not applied; see [Open Questions](#open-questions).

**Direction, magnitude, immunity.** The count error produced **false positives** — phantom reclaimable space on indexes with nothing to reclaim — at 5.5% on `i_q1000`, `i_r500` and `i_r1000` and 10.8% on `i_r265`. Change 1 removes them and replaces them with over-prediction, which is a **false negative** risk, at up to `−99.4%`; both directions are confined to `wasted_space_pct`, because `wasted_space_pct_floor` never credits deduplication at all, so the floor-based alert rule was already immune to the defect *and* is unaffected by the fix. The full band is in [Direction, magnitude, and whether the floor was immune](#direction-magnitude-and-whether-the-floor-was-immune).

### Change 2: extended statistics for a multicolumn key

The per-column product is wrong in one direction only: it multiplies away every functional dependency, so a correlated multicolumn key looks far more distinct than it is, the statement declines to credit deduplication, and the modelled rebuild comes out too large. Too large means a **false negative**: a bloated index reports negative waste and never alerts.

The fix reads an `ndistinct` extended-statistics object when one covers the whole key. `pg_ndistinct`'s output function spells its keys as ascending, comma-space-separated table attnums, so the entry can be looked up as JSON:

```sql
keyatts AS (
    SELECT i.idxoid,
           string_agg(i.indkey[k]::text, ', ' ORDER BY i.indkey[k]) AS ext_key,
           count(*)         AS nkeys,
           min(i.indkey[k]) AS min_attnum
      FROM idx i, generate_subscripts(i.indkey, 1) k
     WHERE k < i.indnkeyatts
     GROUP BY i.idxoid
),
extstat AS (
    SELECT k.idxoid, max(e.nd) AS ext_ndistinct
      FROM keyatts k
      JOIN idx i           ON i.idxoid = k.idxoid
      JOIN pg_stats_ext se ON se.schemaname = i.schemaname
                          AND se.tablename = i.tablename
      CROSS JOIN LATERAL (
            SELECT ((se.n_distinct::text)::json ->> k.ext_key)::numeric AS nd) e
     WHERE k.nkeys > 1 AND k.min_attnum > 0 AND e.nd > 0
     GROUP BY k.idxoid
)
```

and `key_groups` prefers it: `CASE WHEN e.ext_ndistinct IS NOT NULL THEN least(e.ext_ndistinct, greatest(i.live_rows, 0)) ELSE <the product> END`.

Four source facts make this safe:

- **The value is an absolute count, never a fraction.** `statext_ndistinct_build` runs the same Duj1 estimator `analyze.c` uses and clamps the result to `[d, totalrows]` ([mvdistinct.c#estimate_ndistinct](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L519-L542)), so the negative-fraction convention that forces the per-column branch to be careful does not exist here.
- **The key spelling is fixed by the output function.** `pg_ndistinct_out` writes `{"1, 2": 1000}` — quoted attnum lists, `", "` separated, in the stored ascending order ([mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L354-L385)). Measured: an index declared `(b, a)` finds the same `"1, 2"` entry as one declared `(a, b)`, because `ORDER BY i.indkey[k]` sorts numerically.
- **A superset object still covers the key.** `statext_ndistinct_build` emits an item for every combination of 2 to N of the object's columns ([mvdistinct.c#statext_ndistinct_build](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L87-L141)). Measured: `i_sup` on `(a, b)` with `CREATE STATISTICS ... ON a, b, c` picks up the `"1, 2"` entry out of the four entries stored.
- **Expression keys are excluded.** Expressions are stored as negative attnums offset by the expression count ([mvdistinct.c:83-86](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L83-L86)), which no `indkey` attnum can match, so `min_attnum > 0` keeps the lookup off them. Measured: `i_expr` on `((a + b), a)` gets no extended estimate and falls back to the product.

**The privilege condition.** `pg_stats_ext` filters on `pg_has_role(c.relowner, 'USAGE')` and on RLS ([system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L309)); the documentation states it "only exposes information about tables the current user owns" ([catalogs.sgml#pg_statistic_ext_data](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L7764-L7775)). That is stricter than `pg_stats`, which needs only column-level `SELECT` ([system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L271-L273)). So a monitoring role that is not a member of the table owner's role gets no extended estimate and silently falls back to the product. Measured: the 17.11 database exposes 8 `pg_stats_ext` rows to the table owner and 0 to an unprivileged role.

**When the object exists but its entry does not cover the exact key set**, `->>` returns NULL for the missing key, the `WHERE e.nd > 0` predicate drops the row, `extstat` produces nothing for that index, and `key_groups` uses the product. Measured: `i_gap` on `(a, b, c)` with `CREATE STATISTICS ... ON a, b` keeps the product and stays at `−329.5%`, unchanged from the filed statement. Two further limits come from source: an object needs at least 2 columns ([statscmds.c:415-419](../../../../raw/postgres-17/src/backend/commands/statscmds.c#L415-L419)) and at most `STATS_MAX_DIMENSIONS` = 8 ([statistics.h#STATS_MAX_DIMENSIONS](../../../../raw/postgres-17/src/include/statistics/statistics.h#L19), [statscmds.c:216-224](../../../../raw/postgres-17/src/backend/commands/statscmds.c#L216-L224)), so a 9-or-more-column key can never be covered, and the data only exists after an `ANALYZE` builds it ([extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L112-L133), [extended_stats.c:200-210](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L200-L210)).

**Direction, magnitude, immunity.** It fixes **false negatives**, and the floor was *not* immune. `i_ext50` is a correlated `(a, b)` index over 1,000,000 rows with half the rows deleted and vacuumed: 896 blocks live, 450 rebuilt, so 49.8% is genuinely reclaimable. The filed statement models 1239 blocks and reports `−38.3%`; its floor models 1374 and reports `−53.3%`; both are silent. With the extended estimate the model is 449 blocks and the report is `+49.9%`, one block from the rebuild. On healthy fixtures the same change moves `i_ext` and `i_sup` from `−206.4%` to `+0.1%`, tightening the size estimate 3.06x. A new caveat, `key groups from extended statistics`, marks every row that used it.

### Change 3: statistics that exist but are invisible

`pg_stats` is publicly readable and silently row-filtered: it returns nothing for a column the caller cannot `SELECT`, and nothing when RLS is active on the table ([system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L273)), because `pg_statistic` itself is revoked from `public` ([system_views.sql:275](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L275), [catalogs.sgml#pg_statistic](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L7431-L7440)). The filed statement's `coalesce(..., 0)` then reads a hidden row as "no duplicates", and `coalesce(se.avg_width, st.avg_width, 32)` reads a hidden width as 32 bytes.

Core SQL can separate the two cases, because the statement can evaluate the view's own filter itself:

```sql
           -- no pg_stats row was returned for this column ...
           (se.attname IS NULL AND st.attname IS NULL)           AS no_stats_row,
           -- ... and pg_stats would have withheld one had it existed
           (NOT coalesce(has_column_privilege(i.tbloid, ta.attnum, 'SELECT'),
                         has_table_privilege(i.tbloid, 'SELECT'))
            OR (i.tbl_rls AND row_security_active(i.tbloid)))    AS stats_hidden
```

aggregated per index in a new `statvis` CTE and reported as two caveats: `statistics not visible to this role` when a row is missing *and* the filter would have removed it, `no statistics row for an index column` when a row is missing while the caller could have seen it and the table has been analyzed. `pg_stat_all_tables.last_analyze` is the corroborating signal and is not privilege-filtered ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L687-L693)). What core SQL cannot do is prove a hidden row exists: `pg_statistic` is unreadable, so "not visible" is a statement about the filter, not about the catalog.

**Yes, the caveat belongs in the alert-suppression set**, and for a reason the review did not give. Losing `n_distinct` only collapses the point estimate onto the floor, which is a false negative the floor rule already tolerates. Losing `avg_width` moves **the floor itself**, because `slot` feeds `leaf_cap`. Measured on 17.11, same server, same indexes, as the table owner and then as a role with no `SELECT` privilege:

| Index | Key | As owner | As an unprivileged role | Why |
|---|---|---|---|---|
| `i_wide` | 300,000 distinct 100-byte `text` keys | 0.0 / 0.0, model 4864 = live = rebuilt | **62.5 / 62.5**, model 1825 | `width` defaults to 32, so `slot` is 44 instead of 116 and `leaf_cap` 166 instead of 63 |
| `i_seq` | 1,000,000 distinct `bigint` | 0.0 / 0.0 | 0.0 / 0.0 | `attlen > 0`, so the width never comes from statistics |
| `i_q1000` | 1000 x 1000 `bigint` | −0.6 / −206.4 | −206.4 / −206.4 | `n_distinct` defaults to 0, the point estimate collapses onto the floor |
| `i_cd` | 5000 x 100, 33-byte `text` | 8.2 / −680.7 | −557.8 / −557.8 | both effects at once |

A 62.5-point **false positive on both columns** of a healthy index is exactly what `never analyzed` and `row-count sources disagree` are suppressed for, so the third caveat joins them: alert when `wasted_space_pct_floor` crosses the threshold, `status` is `ok`, and `caveats` contains none of `never analyzed`, `row-count sources disagree` and `statistics not visible to this role`. The narrower `no statistics row for an index column` carries the same width risk and is worth suppressing on the same grounds; it is kept separate so that a genuinely un-analyzed column can be told apart from a permissions problem.

### Change 4: a randomly inserted, never-deleted index

`i_rand` is 1,000,000 distinct `bigint` keys inserted in random order into an index that already existed, with no update and no delete, then analyzed. Nothing about it is bloated by the documented mechanism ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040)); it is simply at the density random insertion produces.

| Measurement | Value |
|---|---|
| Live size | 3765 blocks, 29 MB |
| `pgstatindex` leaf pages / `avg_leaf_density` / `leaf_fragmentation` | 3752 / **65.68%** / 49.87% |
| Deleted, half-dead, empty pages | 0 / 0 / 0 |
| `CREATE INDEX CONCURRENTLY` rebuild | 2745 blocks, 21 MB, 90.06% density |
| `wasted_space_pct` | **27.1** |
| `wasted_space_pct_floor` | **27.1** |
| `wasted_space` | 8160 kB |
| `status` / `caveats` | `ok` / empty |
| Modelled blocks | 2745, which is the rebuilt size **exactly** |

This is the one valid review issue the floor-plus-status-plus-caveats rule cannot defend against. The floor is the one-tuple-per-row model; with distinct keys the point estimate *is* the one-tuple-per-row model, so both columns carry the same number, `status` is `ok` because both row counts agree, and there is no caveat to emit. On the 17.11 database this row sorts first under the statement's own `ORDER BY` — an 8,355,840-byte key in the state above, against 4,702,208 for the three 89.8%-reclaimable duplicate-key indexes behind it — so it is also the row a `LIMIT 20` triage shows first. At a 30% threshold it does not fire, at 27.1%, but the margin is 2.9 points on one fixture, and a `fillfactor` above 90 would raise the modelled density and push the same shape over the line; that was not measured.

What the rule can still say is narrower and true: the model was right. A rebuild really would return 1020 blocks. The question change 5 answers is whether that makes a rebuild worth doing.

### Change 5: what the baseline is, and what a reading means

The model's reference is a sorted build at the index's `fillfactor`: `_bt_pagestate` sets a leaf's target free space from `BTGetTargetPageFreeSpace` ([nbtsort.c:663-665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L663-L665)), which is what `REINDEX` and `CREATE INDEX` produce. A live index only reaches that density where inserts arrive in key order: `_bt_findsplitloc` applies `fillfactor` to a **rightmost** leaf split and splits every other leaf 50:50 ([nbtsplitloc.c:286-291](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L286-L291), [nbtsplitloc.c:329-335](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L329-L335)), with a middle case for a new item at the right edge of a localized grouping ([nbtsplitloc.c:292-303](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L292-L303)). The header comment states the consequence outright: ordered insertion "will end up with a tree whose pages are about fillfactor% full, instead of the 50% full result that we'd get without this special case. This is the same as nbtsort.c produces for a newly-created tree" ([nbtsplitloc.c:88-102](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L88-L102)).

So for a healthy index under steady random insertion the output means: **a rebuild would shrink this index to the density a sorted build produces, and ordinary inserts would then take it back.** It is a prediction about a rebuild, not a measurement of dead space — the same limit the [column rename](#follow-up-the-output-columns-say-wasted_space-not-bloat) records.

Measured, in one sequence on the same table: 500,000 more random keys were inserted into `t_rand`, then a second rebuild taken.

| Index | State | Blocks | `avg_leaf_density` | Reported |
|---|---|---|---|---|
| `i_rand` | never rebuilt, 1.5M rows | 5590 | 66.36% | 26.4 / 26.4 |
| the first rebuild | built sorted at 1.0M rows, then took 500,000 random inserts | 4622 | 80.21% | 10.9 / 10.9 |
| a fresh rebuild | built sorted at 1.5M rows | 4116 | 90.07% | 0.0 / 0.0, model 4116 exactly |

The 27.1% the rebuild reclaimed at 1.0M rows was 10.9% gone again after 50% more rows arrived, while the index that was never rebuilt held steady near 66% density. That is the decision rule:

1. **"A rebuild would reclaim this"** is what a positive `wasted_space_pct_floor` says, once `status` is `ok` and no suppressing caveat is set. It is a size prediction and, on these fixtures, an accurate one.
2. **"A rebuild is worth doing"** needs one more fact the statement does not have: whether the index is *converging* on its current density or *retaining* dead space. Sources for that fact, in order of cost: a non-zero deleted-page or reusable-page count from `VACUUM VERBOSE` or contrib ([Where a current count of empty, deleted and reusable pages comes from](#where-a-current-count-of-empty-deleted-and-reusable-pages-comes-from)); a workload fact — bulk delete, `TRUNCATE`-and-reload, retention window, or a monotonic key that has stopped being appended to; and the shape of the row itself, since a wide gap between the two percentage columns means the answer rests on a duplication estimate.
3. If the index takes random inserts, has no deleted pages, and reads near 25-35%, the honest reading is "this index is at split-point density", and a rebuild buys temporary space at the cost of a full index build. `leaf_fragmentation` 49.87% on `i_rand` against 0.00% on every sorted build is the contrib-only signal that separates the two, and no core-SQL method reaches it ([pgstatindex.c:318-325](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L325)).

### The corrected statement, with all five changes

Two CTEs are new (`keyatts`, `extstat` for change 2, and `statvis` for change 3), `classfit`/`classpages` carry change 1, `idx` gains `t.relrowsecurity`, `cols` gains two booleans, `key_groups` gains its extended-statistics branch, and the final `SELECT` gains two caveat strings. Nothing else moved: the gate, both models, `status`, the two percentage columns, the signed `wasted_space`, the `ORDER BY` and the tag are as filed.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

WITH RECURSIVE
idx AS (
    SELECT /* wiki_btree_wasted_space_sweep_12_17 */
           c.oid AS idxoid, n.nspname AS schemaname, t.relname AS tablename,
           c.relname AS indexname, t.oid AS tbloid, x.indkey, x.indisunique,
           x.indnkeyatts, t.relrowsecurity                AS tbl_rls,
           (x.indpred IS NOT NULL)                      AS is_partial,
           (x.indnatts = x.indnkeyatts)                 AS keys_only,
           z.actual_bytes, z.fsm_bytes, z.bs, z.server_version_num,
           coalesce((SELECT option_value::int FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'fillfactor'), 90)            AS fillfactor,
           coalesce((SELECT option_value::bool FROM pg_options_to_table(c.reloptions)
                      WHERE option_name = 'deduplicate_items'), true)   AS dedup_on,
           c.reltuples::numeric                         AS idx_reltuples,
           coalesce(s.n_live_tup, 0)::numeric           AS tbl_live_tup,
           coalesce(s.n_dead_tup, 0)::numeric           AS tbl_dead_tup,
           greatest(s.last_analyze, s.last_autoanalyze) AS last_analyze,
           -- rows to model: -1 is v14+ "unknown"; a 0 on a non-empty index
           -- whose table reports live rows is a pre-14 stale zero
           CASE
             WHEN c.reltuples < 0 THEN NULL
             WHEN c.reltuples = 0 AND z.actual_bytes > z.bs
                  AND coalesce(s.n_live_tup, 0) > 0 THEN NULL
             WHEN x.indpred IS NOT NULL THEN c.reltuples::numeric
             ELSE least(c.reltuples::numeric,
                        coalesce(nullif(s.n_live_tup, 0), c.reltuples)::numeric)
           END                                          AS live_rows,
           -- deduplication gate, in catalog terms: every key opclass must
           -- advertise an equal-image support function (amprocnum 4)
           (SELECT bool_and(EXISTS (SELECT 1 FROM pg_amproc ap
                                     WHERE ap.amprocfamily = op.opcfamily
                                       AND ap.amproclefttype = op.opcintype
                                       AND ap.amprocrighttype = op.opcintype
                                       AND ap.amprocnum = 4))
              FROM generate_subscripts(x.indclass, 1) k
              JOIN pg_opclass op ON op.oid = x.indclass[k]
             WHERE k < x.indnkeyatts)                   AS all_equalimage,
           -- ... and no key column may use a nondeterministic collation
           NOT EXISTS (SELECT 1 FROM generate_subscripts(x.indcollation, 1) k
                         JOIN pg_collation cl ON cl.oid = x.indcollation[k]
                        WHERE k < x.indnkeyatts
                          AND NOT cl.collisdeterministic) AS all_deterministic
      FROM pg_class c
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_am am       ON am.oid = c.relam
      LEFT JOIN pg_stat_all_tables s ON s.relid = t.oid
      CROSS JOIN LATERAL (
            SELECT pg_relation_size(c.oid)                  AS actual_bytes,
                   pg_relation_size(c.oid, 'fsm')           AS fsm_bytes,
                   current_setting('block_size')::int       AS bs,
                   current_setting('server_version_num')::int AS server_version_num) z
     WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
       AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
),
keyatts AS (
    -- the index's key columns as table attnums, spelled the way
    -- pg_ndistinct spells its keys: ascending, comma-space separated
    SELECT i.idxoid,
           string_agg(i.indkey[k]::text, ', ' ORDER BY i.indkey[k]) AS ext_key,
           count(*)         AS nkeys,
           min(i.indkey[k]) AS min_attnum
      FROM idx i, generate_subscripts(i.indkey, 1) k
     WHERE k < i.indnkeyatts
     GROUP BY i.idxoid
),
extstat AS (
    -- a CREATE STATISTICS ... (ndistinct) object whose column list covers the
    -- whole key: its estimate replaces the per-column product. pg_stats_ext
    -- shows a row only to a member of the table owner's role, and the entry
    -- is present only for the exact column set asked for below.
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
    SELECT i.idxoid, a.attnum, a.attlen, a.attalign,
           CASE WHEN a.attlen > 0 THEN a.attlen::numeric
                ELSE coalesce(se.avg_width, st.avg_width, 32)::numeric END AS width,
           coalesce(se.null_frac, st.null_frac, 0)::numeric      AS null_frac,
           coalesce(se.n_distinct, st.n_distinct, 0)::numeric    AS n_distinct,
           coalesce(se.most_common_freqs, st.most_common_freqs)  AS mcf,
           -- no pg_stats row was returned for this column ...
           (se.attname IS NULL AND st.attname IS NULL)           AS no_stats_row,
           -- ... and pg_stats would have withheld one had it existed, because
           -- the view filters on has_column_privilege() and row_security_active()
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
    SELECT c.idxoid,
           bool_or(c.no_stats_row)                    AS any_no_stats,
           bool_or(c.no_stats_row AND c.stats_hidden) AS any_stats_hidden
      FROM cols c
     GROUP BY c.idxoid
),
tuple AS (
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
           -- distinct key groups: an ndistinct extended-statistics estimate for
           -- the whole key when one is visible, else the per-column product.
           -- A negative n_distinct is a fraction of the *table's* rows, so a
           -- partial index only trusts an absolute count.
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
           v.any_no_stats, v.any_stats_hidden
      FROM idx i
      LEFT JOIN extstat e ON e.idxoid = i.idxoid
      LEFT JOIN statvis v ON v.idxoid = i.idxoid
),
sized AS (
    SELECT t.*, ceil((8 + 8 * t.p_null + t.data_size) / 8) * 8 + 4 AS slot
      FROM tuple t
),
fit AS (
    SELECT s.*,
           greatest(floor((s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100)) / s.slot), 1)
               AS leaf_cap,
           greatest(floor((s.bs - 48 - floor(s.bs * 30 / 100)) / s.slot), 2)
               AS int_cap,
           (s.bs - 48 - floor(s.bs * (100 - s.fillfactor) / 100))     AS leaf_bytes,
           (NOT s.indisunique AND s.dedup_on AND s.keys_only
                AND coalesce(s.all_equalimage, false)
                AND s.all_deterministic)                             AS dedup_applies,
           least(greatest(s.live_rows, 0), greatest(s.key_groups, 1)) AS groups_est,
           -- maxpostingsize = MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)
           floor(floor(s.bs * 10 / 100) / 8) * 8 - 4                  AS maxposting
      FROM sized s
),
posting AS (
    SELECT f.*,
           -- largest n with MAXALIGN(base + n * sizeof(ItemPointerData)) <= maxposting
           greatest(floor(4 * floor((f.maxposting - (f.slot - 4)) / 8) / 3), 1) AS nmax
      FROM fit f
),
kstat AS (
    SELECT p.idxoid,
           CASE WHEN p.is_partial THEN 0 ELSE c.null_frac END AS null_frac,
           CASE WHEN p.is_partial THEN '{}'::real[]
                ELSE coalesce(c.mcf, '{}'::real[]) END        AS mcf
      FROM posting p
      JOIN cols c ON c.idxoid = p.idxoid AND c.attnum = 1
     WHERE p.indnkeyatts = 1 AND p.dedup_applies AND p.live_rows > 0
),
gclass AS (
    -- the NULL run is one key group
    SELECT p.idxoid, greatest(p.live_rows, 0) * k.null_frac AS class_rows,
           1::numeric AS class_groups
      FROM posting p JOIN kstat k ON k.idxoid = p.idxoid
     WHERE k.null_frac > 0
    UNION ALL
    -- every most-common value is one key group
    SELECT p.idxoid, greatest(p.live_rows, 0) * f, 1::numeric
      FROM posting p JOIN kstat k ON k.idxoid = p.idxoid
      CROSS JOIN LATERAL unnest(k.mcf) f
    UNION ALL
    -- the rest of the rows spread over the remaining distinct values
    SELECT p.idxoid,
           greatest(greatest(p.live_rows, 0)
                    * (1 - k.null_frac
                         - coalesce((SELECT sum(f) FROM unnest(k.mcf) f), 0)), 0),
           greatest(p.groups_est
                    - CASE WHEN k.null_frac > 0 THEN 1 ELSE 0 END
                    - coalesce(array_length(k.mcf, 1), 0), 1)
      FROM posting p JOIN kstat k ON k.idxoid = p.idxoid
    UNION ALL
    -- multi-column keys: one uniform class over the whole-key estimate
    SELECT p.idxoid, greatest(p.live_rows, 0), p.groups_est
      FROM posting p
     WHERE p.indnkeyatts > 1 AND p.dedup_applies AND p.live_rows > 0
),
classfit AS (
    SELECT g.idxoid, g.class_rows, g.class_groups,
           least(g.class_rows / greatest(g.class_groups, 1), p.nmax) AS tids,
           p.slot, p.leaf_bytes
      FROM gclass g
      JOIN posting p ON p.idxoid = g.idxoid
     WHERE g.class_rows > 0
),
classpages AS (
    -- posting tuples are MAXALIGNed and each leaf page holds
    -- floor((leaf_bytes + truncated posting list) / tuple size) of them.
    -- Each key group rounds up to a whole number of posting tuples.
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
    SELECT p.*, coalesce(cp.max_tids, 1) AS tids,
           CASE WHEN p.dedup_applies AND cp.leaf_frac IS NOT NULL
                THEN greatest(ceil(cp.leaf_frac), 1)
                ELSE ceil(greatest(p.live_rows, 0) / p.leaf_cap)
           END                                                AS leaf_pages,
           ceil(greatest(p.live_rows, 0) / p.leaf_cap)         AS leaf_pages_floor
      FROM posting p
      LEFT JOIN classpages cp ON cp.idxoid = p.idxoid
),
levels AS (
    SELECT idxoid, 'dedup'::text AS variant, leaf_pages AS pages, int_cap FROM leaves
    UNION ALL
    SELECT idxoid, 'floor'::text, leaf_pages_floor, int_cap FROM leaves
    UNION ALL
    SELECT l.idxoid, l.variant, ceil(l.pages / l.int_cap), l.int_cap
      FROM levels l WHERE l.pages > 1
),
modelled AS (
    SELECT l.*,
           (SELECT sum(v.pages) FROM levels v
             WHERE v.idxoid = l.idxoid AND v.variant = 'dedup') + 1 AS expected_blocks,
           (SELECT sum(v.pages) FROM levels v
             WHERE v.idxoid = l.idxoid AND v.variant = 'floor') + 1 AS floor_blocks
      FROM leaves l
)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(actual_bytes)                     AS index_size,
       CASE
         WHEN idx_reltuples < 0 THEN 'unmeasured: reltuples unknown'
         WHEN live_rows IS NULL THEN 'unmeasured: reltuples 0, table has live rows'
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
         CASE WHEN any_no_stats AND NOT any_stats_hidden AND last_analyze IS NOT NULL
              THEN 'no statistics row for an index column' END,
         CASE WHEN NOT is_partial
                   AND greatest(tbl_live_tup, idx_reltuples)
                       > 1.1 * greatest(least(tbl_live_tup, idx_reltuples), 1)
              THEN 'row-count sources disagree: analyze first' END,
         CASE WHEN is_partial AND (tbl_dead_tup > 0 OR last_analyze IS NULL)
              THEN 'partial: predicate subset may be stale' END,
         CASE WHEN is_partial AND dedup_applies AND tids > 1
              THEN 'partial: duplicates from table statistics' END,
         CASE WHEN dedup_applies AND tids > 1 THEN 'deduplication credited' END,
         CASE WHEN ext_used THEN 'key groups from extended statistics' END
       ], NULL), '; ')                                  AS caveats,
       key_groups::bigint                               AS key_groups,
       round(tids::numeric, 1)                          AS tids_per_tuple,
       live_rows::bigint                                AS modelled_rows,
       idx_reltuples::bigint                            AS idx_reltuples,
       fsm_bytes > 0                                    AS fsm_written_since_build,
       server_version_num
  FROM modelled
 WHERE actual_bytes > 1024 * 1024
 ORDER BY (actual_bytes - floor_blocks * bs) DESC NULLS FIRST
 LIMIT 20;
```

Cost, over the 54 B-tree indexes and 75,847 blocks the 17.11 database ended with, three consecutive runs with the triage filter and `LIMIT` removed: 41.7 / 34.4 / 37.6 ms for this statement against 30.2 / 31.9 / 36.6 ms for the filed one. The three new CTEs cost single-digit milliseconds at that scale, and the ranges overlap.

### Measured on 17.11, per fixture

Every fixture in the first group is freshly built with **nothing to reclaim** — live equals rebuilt — so any non-zero reading is model error. 1,000,000 rows and a `bigint` key unless noted.

| Fixture | Shape | Live = rebuilt | As filed | Corrected | Floor |
|---|---|---|---|---|---|
| `i_seq` | distinct, inserted in key order | 2745 | 0.0 | 0.0 | 0.0 |
| `i_q1000` | 1000 keys x 1000 rows | 896 | +5.5 | **−0.6** | −206.4 |
| `i_cd` | 5000 keys x 100 rows, 33-byte `text`, 500,000 rows | 462 | +8.2 | +8.2 | −680.7 |
| `i_ext` | `(a, b)` with `b = a`, 1000 distinct | 896 | −206.4 | **+0.1** | −206.4 |
| `i_sup` | `(a, b)`, statistics object on `(a, b, c)` | 896 | −206.4 | **+0.1** | −206.4 |
| `i_gap` | `(a, b, c)`, statistics object on `(a, b)` only | 897 | −329.5 | −329.5 | −329.5 |
| `i_ind2` | `(a, d)`, 1000 x 7 independent | 830 | −2.0 | **−88.6** | −230.7 |
| `i_inc_lowcard` | `(a) INCLUDE (d)`, 1000 x 5 | 2749 | +0.1 | +0.1 | +0.1 |
| `i_dupoff` | 1000 x 1000, `deduplicate_items = off` | 2749 | +0.1 | +0.1 | +0.1 |
| `i_wide` | 300,000 distinct 100-byte `text` keys | 4864 | 0.0 | 0.0 | 0.0 |
| `i_rand` | 1,000,000 distinct, random insertion | 3765 live, **2745 rebuilt** | +27.1 | +27.1 | +27.1 |

The duplication band isolates change 1. Each row is 1,000,000 rows at `r` rows per key, freshly built, live equal to rebuilt, and `nmax` is 132:

| `r` | Live = rebuilt | As filed | Corrected | Why |
|---|---|---|---|---|
| 100 | 839 | −0.6 | −0.7 | one tuple per group; the round-up is a no-op |
| 132 | 847 | −0.8 | −0.8 | exactly one full tuple per group |
| 133 | 843 | −0.5 | **−99.4** | two tuples per group, the second holding one TID, priced as a full one |
| 143 | 829 | −2.2 | −88.7 | same shape |
| 200 | 841 | −0.7 | −33.2 | second tuple holds 68 of 132 TIDs |
| 264 | 854 | +0.8 | **+0.4** | exactly two full tuples per group |
| 265 | 950 | **+10.8** | −33.5 | worst as-filed false positive; the round-up then charges three full tuples for 2 x 132 + 1 |
| 500 | 896 | +5.5 | **−0.2** | |
| 1000 | 896 | +5.5 | **−0.3** | |

Then the fixtures with real reclaimable space, and the statistics states:

| Fixture | What it is | Live -> rebuilt (true) | As filed | Corrected | Floor |
|---|---|---|---|---|---|
| `i_dupdel` | 10 keys, 90% of rows deleted + VACUUM | 850 -> 87 (89.8%) | 89.8 | 89.8 | 67.5 |
| `i_ext50` | `(a, b)` with `b = a`, 50% deleted + VACUUM | 896 -> 450 (49.8%) | **−38.3** | **+49.9** | −53.3 |
| `i_stale` | 19% deleted, never vacuumed or analyzed | 2745 -> 2224 (19.0%) | 19.0 | 19.0 | 19.0 |
| `i_rand` | random insertion, nothing deleted | 3765 -> 2745 (27.1%) | 27.1 | 27.1 | 27.1 |

`i_ext50` is the row that changes an operational outcome: 446 reclaimable blocks that the filed statement reports as `−38.3%` and does not surface at any threshold, on either column.

### Direction, magnitude, and whether the floor was immune

| Change | Error it fixes | Direction | Measured magnitude | Floor already immune? |
|---|---|---|---|---|
| 1, key-group round-up | posting tuples counted across group boundaries | fixes **false positives** | +5.5% on `i_q1000`/`i_r500`/`i_r1000` and +10.8% on `i_r265` removed; introduces over-prediction to `−99.4%` at `r` = 133 | **Yes**, in both directions: the floor never credits deduplication, so neither the defect nor the fix can move `wasted_space_pct_floor` |
| 2, extended statistics | the per-column product multiplies away functional dependencies | fixes **false negatives** | `i_ext50`: a true 49.8% reported as `−38.3%` becomes `+49.9%`; `i_ext`/`i_sup` over-prediction 3.06x -> 1.001x | **No.** The floor read `−53.3%` on the same index and was equally silent |
| 3, invisible statistics | `coalesce(..., 0)` and the 32-byte default width | fixes both, and the width half is a **false positive** | `i_wide` reads 62.5 / 62.5 on a healthy index as an unprivileged role, against 0.0 / 0.0 as owner | **No.** `avg_width` feeds `slot`, so the floor moves too. This is why the caveat joins the suppression set |
| 4, random-insert fixture | an untested class of index | exposes a **false positive** | `i_rand`: 27.1 / 27.1 on an index with no dead space, model exact to the block | **No**, and it cannot be: with distinct keys the floor is the point estimate |
| 5, baseline framing | reading a size prediction as a bloat measurement | neither; it changes interpretation | a rebuild reclaimed 1020 blocks, and 500,000 further random inserts gave 10.9% of it back | n/a |

Two things follow for the alerting rule in [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate). The suppression set grows by one string, `statistics not visible to this role`. And the rule's "0 false positives" claim needs the qualifier the review asked for: 0 false positives *among delete-driven and statistics-trap fixtures*, and one at 27.1% among randomly-inserted ones, which is below a 30% threshold by 2.9 points rather than by construction.

### Two rejected fixes

**A. `least(reltuples, n_live_tup)` stays.** The two sources are not interchangeable. `pg_class.reltuples` is written only by an index build, by `ANALYZE`'s sample, or by a `VACUUM` that counted exactly ([index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842), [analyze.c:637-660](../../../../raw/postgres-17/src/backend/commands/analyze.c#L637-L660), [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1470)), while `pg_stat_all_tables.n_live_tup` is maintained by DML: each statement's inserts and deletes accumulate per transaction ([pgstat_relation.c#pgstat_count_heap_insert](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L360-L369)), commit turns them into a signed delta — "insert adds a live tuple, delete removes one" ([pgstat_relation.c#AtEOXact_PgStat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L553-L575)) — and the flush adds that delta to the shared counter, clamped at zero ([pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L838-L866)). `ANALYZE` then re-bases it, discounting rows in still-open transactions ([pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L277-L311)).

Live rows are the right baseline because a rebuild indexes live rows. Measured on 17.11, with the same statement modified only to drop the `least()`:

| `i_stale`, 19% deleted, never vacuumed or analyzed | Modelled rows | Modelled blocks | Reported |
|---|---|---|---|
| As filed, `least(reltuples, n_live_tup)` | 810,000 | 2224 | **19.0%** — the rebuild is 2224 blocks |
| `reltuples` alone | 1,000,000 | 2745 | **0.0%** — the live size |

What the sweep would lose is every index whose table shrank since the last `ANALYZE`: 521 blocks and 4168 kB on this one fixture, reported as nothing at all. That is a **false negative**, and the floor does not rescue it, because `live_rows` feeds both models — the floor reads 0.0% too. Three other fixtures (`i_dupdel`, `i_ext50`, `i_seq`) are bit-identical under both forms, so the guard costs nothing where the two sources agree. The reverse hazard is already handled elsewhere: when the collector is the stale one, the `row-count sources disagree: analyze first` caveat fires, as it does on `i_stale` itself.

**B. `bt_metap().allequalimage` stays out.** Four reasons, three from source and one measured:

1. **The model predicts a fresh build, and the build recomputes the flag.** `_bt_leafbuild` sets `wstate.inskey->allequalimage = _bt_allequalimage(wstate.index, true)` from the current opclasses, never from the metapage ([nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563)), uses that value to decide whether to deduplicate ([nbtsort.c:1151-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152)), and only then writes it into the new metapage ([nbtsort.c:1124-1127](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1124-L1127), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L60-L90)). The sweep's `pg_amproc` probe plus the collation conjunct is the catalog form of that same function.
2. **A pg_upgrade'd v12 index reports false while a rebuild would set it true.** The header says so: "Even version 4 indexes created on PostgreSQL v12 will need a REINDEX to make use of deduplication, though, since there is no other way to set `btm_allequalimage` to true (pg_upgrade hasn't been taught to set the metapage field)" ([nbtree.h:135-142](../../../../raw/postgres-17/src/include/access/nbtree.h#L135-L142)), and `bt_metap` relies on the same assumption when it reports the column ([btreefuncs.c:905-921](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L905-L921)). A gate built on it would model those indexes as un-deduplicatable and hide exactly the win a `REINDEX` would deliver — measured separately, on real 12.2 -> 17.10 upgrades, in [Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade From PostgreSQL 12 to 17 (unverified)](btree-deduplication-after-pg-upgrade.md).
3. **It is superuser-only.** `bt_metap` raises `must be superuser to use pageinspect functions` before it opens the relation ([btreefuncs.c:840-857](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L840-L857)); measured on 17.11, an unprivileged role gets exactly that error. `pageinspect` is also contrib and not `trusted`, which is the same wall [What no core-SQL method can measure on v17](#what-no-core-sql-method-can-measure-on-v17) already documents.
4. **The flag is true for indexes that hold no posting lists.** Measured on 17.11: `i_dupoff` (1000 keys x 1000 rows, `deduplicate_items = off`) reports `allequalimage = t` and `version = 4` on a 2749-block index with no deduplication anywhere in it, because the reloption, not the flag, is what turned it off. The metapage would have credited it.

**The question the metapage flag does answer** is the complement: *may this index, as it stands on disk, deduplicate?* `_bt_metaversion` reads `btm_allequalimage` into the insertion scankey ([nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L718-L795)), and the insert path runs a deduplication pass only if that flag and the reloption both allow it ([nbtinsert.c:2772-2782](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2772-L2782)). Operationally that is "does this index need a `REINDEX` before deduplication can ever happen in it", which is a rebuild-decision question, not a size-model question — and it is why the upgrade page reads the metapage byte while this page reads `pg_amproc`.

### The corrected statement on a 12.2 server

The statement runs unchanged on 12.2, and every construct the five changes add exists there. Verified by running the same file on an isolated 12.2 server, `server_version_num` 120002:

| New construct | Exists in 12? | Evidence from the 12.2 run |
|---|---|---|
| `pg_stats_ext.n_distinct`, `.kinds`, `.schemaname`, `.tablename` | yes | the view's column list on 12.2 is `schemaname, tablename, statistics_schemaname, statistics_name, statistics_owner, attnames, kinds, n_distinct, dependencies, most_common_vals, most_common_val_nulls, most_common_freqs, most_common_base_freqs` |
| `pg_ndistinct` rendered as JSON | yes | `n_distinct::text` returns `{"1, 2": 1000}`, and `((n_distinct::text)::json ->> '1, 2')::numeric` yields 1000 |
| `string_agg(... ORDER BY ...)`, `generate_subscripts(indkey, 1)`, `min()` over `int2vector` subscripts | yes | `i_ext` on `(a, b)` resolves `ext_key` and is credited |
| `has_column_privilege`, `has_table_privilege`, `row_security_active`, `pg_class.relrowsecurity` | yes | as an unprivileged role, `i_wide` reports 62.5 / 62.5 with `statistics not visible to this role`, the same as 17.11 |

Two columns the v17 view has are **absent** in 12 and are deliberately not referenced: `inherited` (from `pg_statistic_ext_data.stxdinherit`, which the v17 view exposes at [system_views.sql:290-291](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L290-L291)) and `exprs`. Referencing either would break the statement on 12. The cost of not filtering on `inherited` is that an inheritance parent can expose two rows for the same object on 15 and later; `max(e.nd)` takes the larger estimate, which credits less deduplication and therefore under-reports rather than over-reports. That is the fallback, and it is the conservative direction.

What 12.2 does with the rest is unchanged from [How the same statement behaves on 12.2, 14.23 and 17.10](#how-the-same-statement-behaves-on-122-1423-and-1710): `pg_amproc` has no `amprocnum = 4` row for any B-tree opfamily — measured count 0 — so the gate never opens, `tids_per_tuple` is 1.0 in all 12 rows, and every point estimate equals its floor. Change 1 is therefore dead code on 12. Change 2 fills in `key_groups` and its caveat on two of the twelve indexes but cannot move a percentage while the gate is closed. Change 3 does move percentages on 12: the unprivileged-role run there reports the same 62.5% on `i_wide`, which is the strongest reason to treat its caveat as version-independent.

| | 12.2 | 17.11 |
|---|---|---|
| Statement runs unchanged | yes | yes |
| B-tree indexes swept, rebuild copies included | 12 | 54 |
| Blocks covered | 37,668 | 75,847 |
| Three consecutive runs | 30.5 / 19.4 / 21.7 ms | 41.7 / 34.4 / 37.6 ms |
| Indexes credited with deduplication | 0 | 39 |
| Indexes using an extended-statistics estimate | 2 | 15 |
| Rows where `wasted_space_pct` differs from `wasted_space_pct_floor` | 0 of 12 | 39 of 54 |
| `i_rand`, random insertion, nothing deleted | 26.6 / 26.6 | 27.1 / 27.1 |
| `i_stale`, 19% deleted, unvacuumed | 19.0 / 19.0 | 19.0 / 19.0 |

The `i_rand` row is the portability point that matters for change 5: split-point density is not a v13-and-later behavior, so a 12 server reports the same healthy index as 26.6% waste.

### Follow-up: seventeen mandatory tests for the deduplication gate

**The statement as filed fails 3 of the 17 mandatory tests and the earlier deduplication-aware sweep fails 6. One replaced conjunct — change 6 — passes all 17 in both statements, at no measurable cost.** Every failure is the same defect wearing a different disguise: the gate asks whether an equal-image support function *exists* in `pg_amproc`, while the engine asks what that function *returns*.

Measured on 28 freshly built fixtures, each with nothing to reclaim, on an isolated 17.11 server built from this page's pin:

| Statement | Gate over-credits | Gate under-credits | Fixtures over 30% on `wasted_space_pct` | ... on `wasted_space_pct_floor` |
|---|---|---|---|---|
| [The corrected statement, with all five changes](#the-corrected-statement-with-all-five-changes) | **5 of 28** | 0 | **3** | 0 |
| The same statement with change 6 | **0** | 1 | **0** | 0 |
| [A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17) | **8 of 28** | 0 | **4** | n/a, it has no floor |
| The same sweep with three conjuncts added | **0** | 1 | **0** | n/a |

Over-crediting is the dangerous direction, because it invents reclaimable space on a healthy index: the worst cell reports **78.1%** waste on a 1931-block index that a rebuild would reproduce block for block. Under-crediting is the safe direction, and change 6 trades exactly one of those for the five over-credits.

The one under-credit is unavoidable in core SQL, and the mandatory list anticipates it. Test 14 asks for `all_equalimage = TRUE` on a custom opclass whose support function returns true *"if the implementation can safely determine the result"*; a SQL statement cannot call a function it can only see by OID, so it cannot safely determine it, and test 16 requires that such cases resolve to FALSE rather than TRUE. Change 6 answers FALSE, and pays for it on one fixture.

### The seventeen mandatory tests, and the verdict on each

The `Engine` column is the engine's own answer, taken from the `DEBUG1` line `_bt_allequalimage` emits during the build and cross-checked against the metapage flag and against whether the first leaf really holds posting tuples. Every percentage is `wasted_space_pct` on an index with 0% genuinely reclaimable space, so any non-zero number is model error.

| # | Requirement | Fixture | Engine | Filed | Change 6 | v17 sweep |
|---|---|---|---|---|---|---|
| 1 | integer key deduplicates | `i_int4` | yes, 421 blocks | pass, −0.5% | pass, −0.5% | pass |
| 2 | `bigint` key deduplicates | `i_int8` | yes, 421 | pass, −0.5% | pass, −0.5% | pass |
| 3 | `text` with a deterministic collation deduplicates | `i_text_det` (default), `i_text_icu_det` (ICU `und`) | yes, 460 each | pass, 7.8% | pass, 7.8% | pass |
| 4 | `text` with a nondeterministic collation must not | `i_text_nondet` (ICU `und-u-ks-level2`) | no, 1931 | pass, 0.2% | pass, 0.2% | **FAIL, 77.7%** |
| 5 | `numeric` must not | `i_numeric` | no, 1376 | pass, 0.1% | pass, 0.1% | pass |
| 6 | `float4`/`float8` must not | `i_float4`, `i_float8` | no, 1376 each | pass, 0.1% | pass, 0.1% | pass |
| 7 | all-equal-image multicolumn key deduplicates | `i_multi_ok`, `i2_ok` `(int4, int8)` | yes, 459 each | pass, −320.0% / 8.1% | pass, same | pass |
| 8 | multicolumn key with one non-equal-image column must not | `i_multi_bad` `(int4, numeric)` | no, 1931 | pass, 28.8% | pass, 28.8% | pass |
| 9 | an expression key is judged by the expression's opclass and collation | `i_expr_lower`, `i_expr_lower_ci`, `i_expr_num`, `i_expr_int8` | yes / no / no / yes | pass, 4 of 4 | pass, 4 of 4 | **FAIL on `i_expr_lower_ci`, 77.7%** |
| 10 | an `INCLUDE` index must not | `i_inc` | no, 1376 | pass, 0.1% | pass, 0.1% | **FAIL, 65.0%** |
| 11 | `deduplicate_items = off` must not, even at `all_equalimage = true` | `i_dupoff`, `i_text_off`, `i2_off` | flag true, no posting lists | pass, 0.1-0.2% | pass | pass |
| 12 | custom opclass with no support function must not | `i_ei_none` | no, 1376 | pass, 0.1% | pass, 0.1% | pass |
| 13 | support function exists and returns FALSE must not | `i_ei_false` | no, 1376 | **FAIL, 69.3%** | pass, 0.1% | **FAIL, 69.2%** |
| 14 | support function exists and returns TRUE, if safely determinable | `i_ei_true`, `i_ei_alias` | yes, 421 each | pass, −0.5% | `i_ei_alias` pass; `i_ei_true` **FALSE by design**, −226.4% | pass |
| 15 | one key TRUE, another FALSE, must resolve FALSE | `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft` | no, 1931 each | **FAIL, 4 of 4** (0.2% latent, **78.1%** visible) | pass, 0.2% | **FAIL, 4 of 4** |
| 16 | unknown or unsupported cases resolve FALSE, never TRUE | `i_ei_null`, `i_ei_boom`, `i_squat`, a replaced built-in | 2 builds fail outright; `i_squat` refuses; the replaced built-in still deduplicates | **FAIL on `i_squat`** | pass | **FAIL on `i_squat`** |
| 17 | the same positive and negative tests across versions | 10 fixtures on 12.2 | no deduplication exists there | pass, 10 of 10 | pass, 10 of 10 | pass |

Tests 1, 2, 3, 7 and 14 are the positives; the rest are negatives. Two readings in that table are arithmetic error rather than gate error, and both are already filed above: the 7.8% on the text fixtures is [the posting-tuple packing loss](#the-mischaracterized-issue-page-packing) that `i_cd` reports as 8.2%, and `i_multi_ok`'s −320.0% is the per-column product rule that [change 2](#change-2-extended-statistics-for-a-multicolumn-key) fixes — the same index with a `CREATE STATISTICS ... (ndistinct)` object is `i2_ok`, at 8.1%. The gate verdict is identical either way.

### How the mandatory tests were run

One isolated **17.11** server, built out of tree from this page's pin `786db8dcf168bd9df8f55047337525ac19118b1c` and configured `--without-readline --without-zlib --with-icu --enable-debug`, `block_size` 8192, `autovacuum = off`, `fsync = off`, in a scratch database created for this run. `pageinspect` and `amcheck` were installed as ground truth only. One isolated **12.2** server from its own pin, without ICU, carried test 17.

Two fixture tables, both 500,000 rows with 5,000 distinct keys, so every candidate key repeats 100 times:

- `t (u int4 unique, a int4, b int8, s text, n numeric, f4 float4, f8 float8, d int4)`, carrying 24 B-tree indexes.
- `t2 (a int4, b int8)` with `CREATE STATISTICS s2_ab (ndistinct) ON a, b`, carrying 4. It exists so that test 15's failure is visible as a percentage and not only as a flag; see [What the mixed-key failure costs](#what-the-mixed-key-failure-costs).

Eight custom operator classes supply the equal-image cases that no built-in type can. The support functions are plain SQL and PL/pgSQL, which the access method accepts: `amvalidate` returned true for all eight custom opclasses ([amapi.c#amvalidate](../../../../raw/postgres-17/src/backend/access/index/amapi.c#L110-L143)), because `btvalidate` only checks that support function 4 takes one `oid` and returns `bool` ([nbtvalidate.c:110-113](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L110-L113)) and `CREATE OPERATOR CLASS` only checks the same two properties plus non-cross-typedness ([opclasscmds.c:1310-1333](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L1310-L1333), reached through `amadjustmembers` at [opclasscmds.c:696-704](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L696-L704)). Neither checks the language, the volatility or the return value:

| Function | Language | Registered in | What the engine does with it |
|---|---|---|---|
| `ei_true(oid) → true` | `sql` | `int4_ei_true`, `int8_ei_true` | deduplicates; `i_ei_true` is 421 blocks |
| `ei_false(oid) → false` | `sql` | `int4_ei_false`, `int8_ei_false` | refuses; `i_ei_false` is 1376 blocks |
| `ei_none` — none registered | — | `int4_ei_none` | refuses; 1376 blocks |
| `ei_null(oid) → NULL` | `sql` | `int4_ei_null` | `CREATE INDEX` fails: `ERROR: function 16575 returned NULL` |
| `ei_boom(oid)` raises | `plpgsql` | `int4_ei_boom` | `CREATE INDEX` fails: `ERROR: equalimage probe exploded` |
| `ei_alias(oid)` | `internal`, `AS 'btequalimage'`, in `public` | `int4_ei_alias` | deduplicates; 421 blocks |

The two build failures are not a quirk of the harness. `_bt_allequalimage` calls the support function through `OidFunctionCall1Coll` ([fmgr.c#OidFunctionCall1Coll](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L1411-L1418)), whose `FunctionCall1Coll` rejects a NULL result with `elog(ERROR, "function %u returned NULL", ...)` ([fmgr.c:1141-1143](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L1141-L1143)), and an exception inside the function propagates. So tests 12 through 16 are satisfiable only by opclasses whose function returns a real boolean — which is why `i_ei_null` and `i_ei_boom` contribute "no such index can exist" rather than a row.

### What the engine decides, and when

`_bt_allequalimage` is the whole of it ([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)), and four of its properties are exactly what the mandatory tests probe:

1. **`INCLUDE` short-circuits before any lookup.** If the index has more attributes than key attributes it returns false immediately ([nbtutils.c:5144-5147](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147)) — before the `debugmessage` block, which is why `i_inc` is the one fixture that logs *no* deduplication verdict at all while every other index logs one.
2. **The loop is over index key attributes, not table columns.** It reads `rd_opfamily[i]`, `rd_opcintype[i]` and `rd_indcollation[i]` ([nbtutils.c:5149-5157](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5149-L5157)), so an expression key is judged by the expression's own opclass and collation. Measured: `i_expr_int8` on `(n::int8)` over a **numeric** column deduplicates to 421 blocks, while `i_expr_num` on `(a::numeric)` over an **int4** column refuses at 1376, and `i_expr_lower` on `lower(s)` deduplicates at 460 while `i_expr_lower_ci` on `(lower(s)) COLLATE ci` refuses at 1931. That is test 9, in both directions, twice.
3. **The catalog lookup is only half the test.** `get_opfamily_proc` finds the OID and then the function is *called*, with the index collation passed as the function's collation and `opcintype` as its argument; a missing OID and a false return are the same outcome ([nbtutils.c:5156-5169](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5156-L5169)). The loop `break`s on the first false, so `i_mixed_tf` and `i_mixed_ft` both answer false regardless of which key is the false one.
4. **It runs at build time only.** `_bt_leafbuild` calls it with `debugmessage = true` and stores the answer in the insertion scankey ([nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563)); `_bt_load` then deduplicates only when that flag, non-uniqueness and the reloption all agree ([nbtsort.c:1151-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1151-L1152), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)). The documentation states the same rule from the operator-class side: deduplication is safe only when *every* indexed column registers an `equalimage` function *and each one actually returns true* ([btree.sgml#equalimage](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L499-L509)).

The two stock functions are what make tests 1 through 6 predictable. `btequalimage` returns true unconditionally ([datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L415-L438)); `btvarstrequalimage` returns true for the C collation, the default collation, or any collation marked deterministic, and false otherwise ([varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)). `numeric`, `float4`, `float8`, `jsonb` and the container types register nothing at all — measured on a stock database, `numeric_ops`, `float8_ops`, `jsonb_ops`, `array_ops`, `range_ops` and `record_ops` have no support function 4, while `int4_ops`, `text_ops` and `text_pattern_ops` do — which is the catalog form of the documentation's list of unsafe cases ([btree.sgml:834-909](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L909)).

Two catalog facts bound how much the gate can even see. Support-function numbers are per access method: the same stock database has 113 `pg_amproc` rows at `amprocnum = 4`, of which only **29** belong to a B-tree family, the rest being `brin_minmax_union`, GIN extract routines and their kin. And all 29 name one of the two stock functions — 26 `btequalimage`, 3 `btvarstrequalimage`, every one of them `LANGUAGE internal` in `pg_catalog` ([pg_amproc.dat:143](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L143), [pg_amproc.dat:206](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L206), [pg_amproc.dat:241](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L241)). On a database with no custom opclass, therefore, the filed gate and change 6 return the same answer for every index.

Four unsupported shapes never reach the gate at all, because the DDL refuses them. Measured on 17.11:

| Attempted | Error |
|---|---|
| `ALTER OPERATOR FAMILY int4_ei_true USING btree ADD FUNCTION 4 (int4, int2) ei_true(oid)` | `btree equal image functions must not be cross-type` ([opclasscmds.c:1321-1332](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L1321-L1332)) |
| the same with a two-argument function | `btree equal image functions must have one argument` ([opclasscmds.c:1312-1315](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L1312-L1315)) |
| the same with a function returning `int` | `btree equal image functions must return boolean` ([opclasscmds.c:1316-1319](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L1316-L1319)) |
| `CREATE INDEX ON t (s COLLATE ci text_pattern_ops)` | `nondeterministic collations are not supported for operator class "text_pattern_ops"` ([index.c:807-850](../../../../raw/postgres-17/src/backend/catalog/index.c#L807-L850)) |

The last one matters for the collation conjunct. `text_pattern_ops` registers `btequalimage`, not `btvarstrequalimage` ([pg_amproc.dat:241](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L241)), so an index on that opclass with a nondeterministic collation would be the one case where the statement's whole-index collation test is stricter than the engine's per-column one — and no such index can be created. The cross-type rejection is covered upstream by a regression test ([alter_generic.sql:444-446](../../../../raw/postgres-17/src/test/regress/sql/alter_generic.sql#L444-L446), [alter_generic.out:504-507](../../../../raw/postgres-17/src/test/regress/expected/alter_generic.out#L504-L507)), and `opr_sanity` separately asserts which core opclasses may omit `btequalimage` ([opr_sanity.sql:1336-1353](../../../../raw/postgres-17/src/test/regress/sql/opr_sanity.sql#L1336-L1353), [opr_sanity.out:2204-2222](../../../../raw/postgres-17/src/test/regress/expected/opr_sanity.out#L2204-L2222)).

### Change 6: name the support function, do not just count it

Replace the `all_equalimage` subquery in the `idx` CTE. Nothing else in [the corrected statement](#the-corrected-statement-with-all-five-changes) changes — not the collation conjunct next to it, not `dedup_applies`, not either model, not a column.

```sql
           -- deduplication gate, in catalog terms: every key opclass must
           -- register a *known* equal-image support function (amprocnum 4).
           -- A row's mere existence proves nothing: the engine calls the
           -- function, and a custom one may return false.
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
             WHERE k < x.indnkeyatts)                   AS all_equalimage,
```

The two whitelisted names are safe to hard-code because their behavior is fixed in C: one always returns true, and the other returns true exactly when the collation conjunct beside it already requires. Everything else — a SQL function, a PL/pgSQL function, a third-party C function, or no row at all — resolves to false, which is test 16.

### Why `prosrc` and not `proname`

The obvious spelling is `pr.proname IN ('btequalimage', 'btvarstrequalimage')` with `pronamespace = pg_catalog`. Two measured fixtures reject it, and both follow from how `fmgr` resolves a function:

- For a built-in OID, `fmgr_info` never consults `pg_proc` at all ([fmgr.c:166-178](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L166-L178)).
- For a non-built-in OID declared `LANGUAGE internal`, the C entry point is looked up **by `prosrc`**, and the comment says why: a user may create an alias whose name differs from the internal function's ([fmgr.c:216-240](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)).

So `prosrc` is the identity the engine uses, and the name is decoration. Measured, on the same server:

| Probe | Engine | `EXISTS` (as filed) | `proname` whitelist | `prosrc` whitelist (change 6) |
|---|---|---|---|---|
| `i_ei_alias`: `LANGUAGE internal AS 'btequalimage'`, in schema `public` | deduplicates, 421 blocks | true | **false** — wrong name, wrong schema | true |
| `i_squat`: the real `btvarstrequalimage` renamed away and a SQL impostor created under its name in `pg_catalog`, registered in a new `text_squat` opclass | refuses, 1931 blocks, metapage false | true | **true** — over-credits | false |
| `i_text_det`, measured during that same rename | deduplicates, 460 blocks | true | **false** — the name it wants no longer exists | true |
| `pg_catalog.btequalimage` replaced by `CREATE OR REPLACE ... LANGUAGE sql AS 'SELECT false'` | still deduplicates, 421 blocks, because of the built-in fast path | true | true | **false** — under-credits |

The impostor got a fresh OID (16738) and therefore no fast path, so the engine really did call it and really did refuse. Scoring those four rows against the engine: the existence test is wrong once (`i_squat`, an over-credit), the `proname` whitelist is wrong **three** times and in both directions, and the `prosrc` whitelist is wrong once — on the replaced built-in, where it under-credits, and where being wrong needs a superuser to rewrite a catalog row that the running engine then ignores.

### What change 6 costs

One fixture, and one shape of database. `i_ei_true` is a genuinely deduplicated 421-block index that change 6 models as if it held one tuple per row, so it reports **−226.4%** where the filed statement reports −0.5%. A negative reading is the harmless direction for alerting and useless for sizing, which is the same trade this page records for [`i_q10_part`](#where-the-proposal-is-still-wrong).

The real cost is a missed positive, and it is worth stating plainly. Starting from `i_ei_none` — 1376 blocks, built with no support function — a single `ALTER OPERATOR FAMILY int4_ei_none USING btree ADD FUNCTION 4 (int4, int4) ei_alias(oid)` makes a rebuild deduplicate: `REINDEX` took it to **421 blocks**, a true 69.4% reclaimable. The filed gate reported 69.3% and was right; change 6 reported 0.1% and was wrong. That is the price, and it is bounded by three facts:

- It needs a custom opclass. On a stock 17.11 database all 29 B-tree support-function-4 rows pass change 6, so no reading moves.
- The documentation calls a custom function the recommended practice for third-party extensions ([btree.sgml:538-550](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L538-L550)), so the affected population is exactly "indexes on extension-supplied operator classes", not "indexes on built-in types".
- The error is a false negative on a rebuild win, not an invented one. Nothing in the output claims space that is not there.

Cost in time is nil: three consecutive runs over the 28 indexes and 34,164 blocks of this database took 12.8 / 12.1 / 14.0 ms as filed against 12.2 / 12.5 / 16.6 ms with change 6, and on the 12.2 server 14.0 ms against 10.3 ms. The two extra joins are on `pg_proc` and `pg_language`, both syscache-backed.

### What the mixed-key failure costs

Test 15 is where the filed statement's over-credit is easiest to miss, because on `t` it is *latent*. `i_mixed_tf` and `i_mixed_ft` open the gate but still report 0.2%: with 5,000 distinct values in each key column the per-column product is 25,000,000, which `least(..., live_rows)` clamps to the row count, so `tids_per_tuple` comes back 1.0 and the posting-list arithmetic has nothing to compress.

Give the same two-column key an `ndistinct` object and the flag turns into a number. On `t2`, where `CREATE STATISTICS s2_ab (ndistinct) ON a, b` makes [change 2](#change-2-extended-statistics-for-a-multicolumn-key) fire, `key_groups` reads 5004 and `tids_per_tuple` 99.9:

| Fixture | Live = rebuilt | Engine | Filed | Change 6 |
|---|---|---|---|---|
| `i2_ok` `(a, b)`, both keys equal-image | 459 | deduplicates | 8.1% | 8.1% |
| `i2_tf` `(a int4_ei_true, b int8_ei_false)` | 1931 | refuses | **78.1%** | 0.2% |
| `i2_ft` `(a int4_ei_false, b int8_ei_true)` | 1931 | refuses | **78.1%** | 0.2% |
| `i2_off` `(a, b)`, `deduplicate_items = off` | 1931 | refuses | 0.2% | 0.2% |

So the two changes interact: the better the duplication estimate gets, the more a wrong gate costs. The floor column is immune — 0.2% in both failing rows — which is why the alerting rule in [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate) survives all five over-credits, and why a sizing decision taken from `wasted_space_pct` or `wasted_space` does not.

### The earlier v17 sweep needs three conjuncts

[A deduplication-aware sweep for v17](#a-deduplication-aware-sweep-for-v17) predates the INCLUDE and collation work, so its gate is `NOT indisunique AND dedup_on AND all_equalimage` and it fails six tests. Adding the two conjuncts the portable statement already carries, plus change 6, brings it to 0 over-credits on the same 28 fixtures:

| Fixture | Test | Engine | Sweep as filed | Sweep + 3 conjuncts |
|---|---|---|---|---|
| `i_inc` | 10 | refuses | **65.0%** | 0.1% |
| `i_text_nondet` | 4 | refuses | **77.7%** | 0.2% |
| `i_expr_lower_ci` | 9 | refuses | **77.7%** | 0.2% |
| `i_ei_false` | 13 | refuses | **69.2%** | 0.1% |
| `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft` | 15 | refuses | gate opens; 0.2% reported | 0.2% |

```sql
-- in the sweep's idx CTE, beside all_equalimage
           (x.indnatts = x.indnkeyatts)                  AS keys_only,
           NOT EXISTS (SELECT 1 FROM generate_subscripts(x.indcollation, 1) k
                         JOIN pg_collation cl ON cl.oid = x.indcollation[k]
                        WHERE k < x.indnkeyatts
                          AND NOT cl.collisdeterministic) AS all_deterministic,
-- ... and in its fit CTE
           (NOT s.indisunique AND s.dedup_on AND coalesce(s.all_equalimage, false)
                AND s.keys_only AND s.all_deterministic)  AS dedup_applies,
```

The sweep still reports −320.0% on `i2_ok`, because it has no extended-statistics branch and no floor column. It is superseded by the portable statement for every purpose except reading this page in order.

### Post-build mutation, and why the metapage is still not the answer

The mandatory tests also settle a question [rejected fix B](#two-rejected-fixes) answered from source only: could the gate just read `bt_metap().allequalimage`? A support function is ordinary catalog state, and replacing it invalidates the metapage's stored verdict without touching the index. Measured, on `i_ei_true`, in one session:

| Step | Blocks | Metapage flag | `bt_index_check` | Filed | Change 6 |
|---|---|---|---|---|---|
| As built | 421, posting tuples present | true | passes | −0.5% | −226.4% |
| `CREATE OR REPLACE FUNCTION ei_true(oid) → false` | 421, unchanged | still true | **`ERROR: index "i_ei_true" metapage incorrectly indicates that deduplication is safe`** | −0.5% | −226.4% |
| `REINDEX INDEX i_ei_true` | **1376**, no posting tuples | false | passes | 0.1% | 0.1% |

Two things follow. First, core tooling treats the divergence as corruption, not as a stale cache: `bt_index_check` re-runs `_bt_allequalimage` against the current catalog and raises `ERRCODE_INDEX_CORRUPTED` when the metapage disagrees ([verify_nbtree.c:379-400](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L379-L400)). The catalog, which is what change 6 reads, is the authority. Second, change 6's "over-prediction" was the correct prediction: −226.4% against an actual rebuild of 1376 blocks from 421, which is −226.8%. The metapage would have said "true" and the filed gate "−0.5%", both of which described an index that no longer existed in that form.

### The mandatory tests on a 12.2 server

Test 17 on an isolated 12.2 server, `server_version_num` 120002, built without ICU. The engine has no equal-image concept at all, so the correct verdict for every fixture — positive and negative alike — is "deduplication disabled", and all four statement variants delivered it:

| Probe | Result on 12.2 |
|---|---|
| B-tree `pg_amproc` rows at `amprocnum = 4` | **0**, while 55 rows exist for other access methods |
| `CREATE OPERATOR CLASS ... FUNCTION 4 ei_true(oid)` | `ERROR: invalid function number 4, must be between 1 and 3` |
| `WITH (deduplicate_items = off)` | `ERROR: unrecognized parameter "deduplicate_items"` |
| `CREATE COLLATION ... provider = icu, deterministic = false` | `ERROR: ICU is not supported in this build` |
| `DEBUG1` deduplication verdict during `CREATE INDEX` | never logged |
| 10 fixtures scored | `all_equalimage` false and `dedup_applies` false in all 10, for all four variants: 0 over-credits, 0 under-credits |
| Reported percentages | 0.0-0.2% on nine fixtures; 28.8% on `i_multi_bad`, identical to 17.11 |
| Change 6 runs unchanged | yes; `pg_language.lanname` and `pg_proc.prosrc` both exist |

Tests 4, 11, 12, 13, 14, 15 and 16 are therefore **unconstructible** on 12: the DDL that would create the fixture is rejected by that major. The positives (1, 2, 3, 7) and the type negatives (5, 6, 8, 10) all ran, and every one of them read as "not deduplicated" against a physical index that really is not deduplicated — `i_int4` at 1376 blocks there against 421 on 17.11, `i_text_det` at 1931 against 460. `i_multi_bad`'s 28.8% appearing identically on both majors is the useful accident: it proves that cell is the `avg_width` arithmetic rather than anything to do with the gate, because on 12.2 the gate cannot open.

That 28.8% is worth one sentence of explanation, since it is the largest non-gate reading in the table. `pg_stats.avg_width` for the `numeric` column is **4**, because `stawidth` is an `int32` assigned `total_width / nonnull_cnt` ([analyze.c:2536-2540](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540)) and the true mean is 4.9996 — `pg_column_size` is 5 for 499,900 rows and 3 for the 100 zero rows. The model therefore prices the `(int4, numeric)` tuple in a 20-byte slot at 366 per leaf, while the built index holds 24-byte items at 262 per leaf.

### The harness, runnable

Two artifacts. The first is an audit query that needs no fixtures: run it against any 12-or-later database to list the indexes whose reading depends on the gate change, before adopting change 6.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

SELECT /* wiki_btree_dedup_gate_audit */
       n.nspname, t.relname AS tablename, c.relname AS indexname,
       pg_size_pretty(pg_relation_size(c.oid))                     AS index_size,
       g.exists_aei                                                AS gate_as_filed,
       g.builtin_aei                                               AS gate_change6,
       (SELECT string_agg(DISTINCT coalesce(pr.proname::text, '(none)'), ', ')
          FROM generate_subscripts(x.indclass, 1) k
          JOIN pg_opclass op ON op.oid = x.indclass[k]
          LEFT JOIN pg_amproc ap ON ap.amprocfamily = op.opcfamily
                                AND ap.amproclefttype = op.opcintype
                                AND ap.amprocrighttype = op.opcintype
                                AND ap.amprocnum = 4
          LEFT JOIN pg_proc pr ON pr.oid = ap.amproc
         WHERE k < x.indnkeyatts)                                   AS equalimage_procs
  FROM pg_class c
  JOIN pg_index x     ON x.indexrelid = c.oid
  JOIN pg_class t     ON t.oid = x.indrelid
  JOIN pg_namespace n ON n.oid = c.relnamespace
  JOIN pg_am am       ON am.oid = c.relam
  CROSS JOIN LATERAL (
    SELECT
      (SELECT bool_and(EXISTS (SELECT 1 FROM pg_amproc ap
                                WHERE ap.amprocfamily = op.opcfamily
                                  AND ap.amproclefttype = op.opcintype
                                  AND ap.amprocrighttype = op.opcintype
                                  AND ap.amprocnum = 4))
         FROM generate_subscripts(x.indclass, 1) k
         JOIN pg_opclass op ON op.oid = x.indclass[k]
        WHERE k < x.indnkeyatts)                                    AS exists_aei,
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
        WHERE k < x.indnkeyatts)                                    AS builtin_aei
  ) g
 WHERE am.amname = 'btree' AND c.relkind = 'i' AND x.indisvalid
   AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
   AND coalesce(g.exists_aei, false) IS DISTINCT FROM coalesce(g.builtin_aei, false)
 ORDER BY pg_relation_size(c.oid) DESC;
```

Every row it returns is an index whose deduplication credit is a guess as filed. Measured: it returns 6 rows in the fixture database — `i_ei_false`, `i_ei_true`, `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft`, with `equalimage_procs` naming `ei_true` and `ei_false` — and **0 rows** both in a stock 17.11 database and on the 12.2 server. `i_ei_alias` and `i_ei_none` are correctly absent: the two gates agree on them, true and false respectively.

The second artifact is the fixture harness. It writes a 500,000-row table and 23 of the 24 indexes measured on `t` above — the `text` `deduplicate_items = off` twin is left out — so run it in a scratch database, never beside production data.

```sql
-- fixtures for the seventeen mandatory tests (scratch database only)
CREATE COLLATION ci   (provider = icu, locale = 'und-u-ks-level2', deterministic = false);
CREATE COLLATION cdet (provider = icu, locale = 'und');

CREATE FUNCTION ei_true(oid)  RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT true $$;
CREATE FUNCTION ei_false(oid) RETURNS bool LANGUAGE sql IMMUTABLE AS $$ SELECT false $$;
CREATE FUNCTION ei_alias(oid) RETURNS bool LANGUAGE internal IMMUTABLE AS 'btequalimage';

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

CREATE TABLE t AS
SELECT i::int4 AS u, (i % 5000)::int4 AS a, (i % 5000)::int8 AS b,
       'key' || lpad((i % 5000)::text, 8, '0') AS s, ((i % 5000)::numeric) AS n,
       (i % 5000)::float4 AS f4, (i % 5000)::float8 AS f8, (i % 7)::int4 AS d
  FROM generate_series(1, 500000) i;

SET client_min_messages = debug1;          -- logs the engine's own verdict
CREATE INDEX i_int4          ON t (a);                                   -- 1
CREATE INDEX i_int8          ON t (b);                                   -- 2
CREATE INDEX i_text_det      ON t (s);                                   -- 3
CREATE INDEX i_text_icu_det  ON t (s COLLATE cdet);                      -- 3
CREATE INDEX i_text_nondet   ON t (s COLLATE ci);                        -- 4
CREATE INDEX i_numeric       ON t (n);                                   -- 5
CREATE INDEX i_float4        ON t (f4);                                  -- 6
CREATE INDEX i_float8        ON t (f8);                                  -- 6
CREATE INDEX i_multi_ok      ON t (a, b);                                -- 7
CREATE INDEX i_multi_bad     ON t (a, n);                                -- 8
CREATE INDEX i_expr_lower    ON t (lower(s));                            -- 9
CREATE INDEX i_expr_lower_ci ON t ((lower(s)) COLLATE ci);               -- 9
CREATE INDEX i_expr_num      ON t ((a::numeric));                        -- 9
CREATE INDEX i_expr_int8     ON t ((n::int8));                           -- 9
CREATE INDEX i_inc           ON t (a) INCLUDE (d);                       -- 10
CREATE INDEX i_dupoff        ON t (a) WITH (deduplicate_items = off);     -- 11
CREATE INDEX i_ei_none       ON t (a int4_ei_none);                      -- 12
CREATE INDEX i_ei_false      ON t (a int4_ei_false);                     -- 13
CREATE INDEX i_ei_true       ON t (a int4_ei_true);                      -- 14
CREATE INDEX i_ei_alias      ON t (a int4_ei_alias);                     -- 14
CREATE INDEX i_mixed_tf      ON t (a int4_ei_true, b int8_ei_false);     -- 15
CREATE INDEX i_mixed_ft      ON t (a int4_ei_false, b int8_ei_true);     -- 15
CREATE UNIQUE INDEX i_uniq   ON t (u);
RESET client_min_messages;
ANALYZE t;
```

`client_min_messages` is `PGC_USERSET` ([guc_tables.c:4777-4785](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4777-L4785)), so `debug1` applies to the session or transaction that sets it and needs no reload or restart. The two build-failure cases are omitted because they abort `CREATE INDEX` by design; add them with `ei_null(oid) → NULL::bool` and a PL/pgSQL `ei_boom(oid)` that raises. Both `CREATE COLLATION` lines need a server built `--with-icu`: without it they fail with `ICU is not supported in this build`, which drops tests 4 and half of test 9.

The block above was run verbatim in a fresh database on the same 17.11 server: 23 indexes built, the two ICU collations created, and the engine logged `can safely use deduplication` for 11 of them, `cannot use deduplication` for 11, and nothing at all for `i_inc`.

Scoring needs the engine's answer per index. Three sources agree in all 28 fixtures, and the first needs no extension:

1. the `DEBUG1` line — `index "x" can safely use deduplication` or `cannot use deduplication` — except for `INCLUDE` indexes, which log nothing;
2. `bt_metap(indexname).allequalimage`, which is superuser-only and true even for `deduplicate_items = off` and unique indexes;
3. `count(tids) > 0` over `bt_page_items(indexname, 1)`, the only one of the three that proves posting lists were actually written.

## Context Reviewed

- Pinned checkout `raw/postgres-17/` at commit `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11, `REL_17_11-7-g786db8dcf16`); repinned from `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on 2026-08-17. Every measured number here is a 17.10 observation and was not re-measured; the two code changes in the range (`355faed5a24`, `8434c938598`) are recorded in [How the test was run](#how-the-test-was-run) and leave the B-tree read paths these methods use unchanged.
- nbtree build, split and deduplication: `nbtsort.c` (`_bt_blnewpage`, `_bt_pagestate`, `_bt_buildadd`, `_bt_load`, `_bt_sort_dedup_finish_pending`), `nbtdedup.c` (`_bt_dedup_pass`, `_bt_dedup_start_pending`, `_bt_dedup_save_htid`, `_bt_form_posting`, `_bt_bottomupdel_pass`), `nbtsplitloc.c` (`_bt_findsplitloc`, single-value strategy), `nbtree.h` (fillfactor constants, `BTGetDeduplicateItems`, `BTMaxItemSize`, `BTPageOpaqueData`, `P_HIKEY`), `README`.
- VACUUM and page recycling: `vacuumlazy.c` (VERBOSE report, `BYPASS_THRESHOLD_PAGES`, `lazy_vacuum`, `lazy_cleanup_all_indexes`, `lazy_cleanup_one_index`, index relstats update), `nbtree.c` (`btvacuumcleanup`, `btvacuumscan`, `btvacuumpage`), `nbtpage.c` (`_bt_pagedel`, `BTPageIsRecyclable`), `indexfsm.c`, `genam.h`.
- Catalog and statistics surfaces: `pg_class.h`, `index.c` (`index_update_stats`, `index_set_state_flags`), `vacuum.c` (`vac_update_relstats`), `analyze.c` (`do_analyze_rel`, `compute_scalar_stats`, `compute_distinct_stats`), `pg_statistic.h`, `system_views.sql` (`pg_stats`, `pg_stat_all_tables`, `pg_stat_all_indexes`), `pg_proc.dat`, `dbsize.c`, `relpath.c`, `varlena.c`, `guc_tables.c`.
- Executor and EXPLAIN: `nodeIndexonlyscan.c`, `nbtsearch.c` (`_bt_readnextpage`), `explain.c` (`show_buffer_usage`, `Heap Fetches`), `explain.sgml`.
- Rebuild path: `indexcmds.c` (`DefineIndex`), `utility.c`, `ruleutils.c` (`pg_get_indexdef`), `bulk_write.c`, `create_index.sgml`, `maintenance.sgml`.
- Contrib boundary: `pgstattuple.control`, `pgstatindex.c`, `pageinspect.control`, `rawpage.c`, `extension.c`, the 22 `trusted = true` contrib control files, `pgstattuple.sgml`.
- Exact-pin execution: one isolated 17.10 server built from the pinned checkout under `.wiki-runtime/`, carrying the 15 named fixtures, the 9 x 3 x {full, partial} matrix (54 indexes over 27 tables), a six-point duplication-ratio sweep, a `reltuples = -1` probe, and Method D bypass fixtures. Methods A, A-prime, B, C and D were executed against them, with `pgstattuple` installed solely as ground truth and Method C rebuilds as the reclaimable-size arbiter. Method B was driven through `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` so the plan's index name, `Heap Fetches` and buffer counts could be scored programmatically.
- Deduplication gate and its version dependence, for the v12/v17 follow-up: `nbtsort.c` (`_bt_load`'s `deduplicate` condition, `_bt_leafbuild` setting `inskey->allequalimage`), `nbtutils.c` (`_bt_allequalimage`, `btoptions`), `nbtree.h` (`BTORDER_PROC` through `BTNProcs`, `BTGetDeduplicateItems`), `nbtree.c` (`bthandler`'s `amsupport`), `nbtvalidate.c` (accepted support numbers), `reloptions.c` (`deduplicate_items` entry), `opclasscmds.c` (`maxProcNumber` and the `invalid function number` error), `varlena.c` (`btvarstrequalimage`), `pg_class.h` (`reltuples` `-1`), `analyze.c` (the negative-`stadistinct` rule). Commit history for the three constructs' first release tags was read from the same pinned checkout.
- Portable-statement follow-up, source coverage: `nbtsort.c` (`_bt_load`'s deduplicate condition and `maxpostingsize`, `_bt_buildadd`'s `truncextra` soft-limit rule and its header comment, `_bt_pagestate`, `_bt_blnewpage`), `nbtdedup.c` (`_bt_dedup_start_pending` `basetupsize`, `_bt_dedup_save_htid` `mergedtupsz`, `_bt_form_posting` `newsize`), `nbtutils.c` (`_bt_allequalimage`, `_bt_keep_natts_fast`'s NULL handling, `btoptions`), `nbtree.h` (`BTEQUALIMAGE_PROC`, `BTNProcs`, `BTGetDeduplicateItems`, `BTGetTargetPageFreeSpace`, `BTMaxItemSize`, fillfactor constants), `indextuple.c` (`index_form_tuple_context`'s MAXALIGN), `varlena.c` (`btvarstrequalimage`), `pg_collation.h` and `pg_index.h` (`collisdeterministic`, `indcollation`, `indclass`, `indnkeyatts`, `indnatts`), `reloptions.c`, `pg_class.h` (`reltuples`), `analyze.c` (`compute_scalar_stats`: the 10%-of-rows `stadistinct` sign rule and MCV frequency storage), `system_views.sql` (`pg_stats`, `pg_stat_all_tables`), `vacuumlazy.c` (index relstats update), `guc_tables.c` (`statement_timeout`, `lock_timeout`, `default_statistics_target`, `block_size`).
- Portable-statement follow-up, exact-pin execution on three servers: isolated 12.2, 14.23 and 17.10 clusters, each built out of tree from its own pinned checkout under `.wiki-runtime/`, all with `autovacuum = off`, `fsync = off`, `block_size` 8192; the 17.10 build was configured `--with-icu` so that a nondeterministic collation could be created. Identical fixture DDL on all three: a seven-point duplication band at 1,000,000 rows (1, 2, 5, 10, 100, 1000 and 1,000,000 rows per key) each with a 20% partial sibling, a 25%-NULL index, a one-hot-value skew index, variable-width and fixed-width text, a unique index, four multi-column/`INCLUDE` variants, a `fillfactor = 50` duplicate-key index, a `deduplicate_items = off` index, five real-bloat fixtures (scattered delete plus VACUUM on distinct and duplicate keys, a partial duplicate-key index, an all-rows-deleted index, an unvacuumed delete), a drained partial predicate, a `TRUNCATE`-and-reload index, a grown-since-ANALYZE index, an empty index, and two 100-rows-per-key text indexes differing only in collation determinism. Ground truth per index is a `CREATE INDEX CONCURRENTLY` copy; the v12 Method A arithmetic, the v17 sweep on this page, a uniform-group variant, and the proposed statement were installed as views and scored against those rebuilds in one query. Catalog probes covered `pg_amproc` support numbers, `ALTER OPERATOR FAMILY ... ADD FUNCTION 4`, `ALTER INDEX ... SET (deduplicate_items = off)`, `collisdeterministic`, `array_lower(indclass, 1)`, and `reltuples` after build, `TRUNCATE`, reload and full delete. The statistics repairs (`SET STATISTICS 1000`, `SET (n_distinct = ...)`) were exercised on 17.10 only. All three servers were stopped afterwards and their data directories removed.
- Reporting-defect follow-up, source coverage (no server run; this follow-up is source-only): free space map internals in `freespace.c` (`RecordPageWithFreeSpace`, `GetPageWithFreeSpace`, `GetRecordedFreeSpace`, `fsm_readbuf`'s `extend` flag and ZERO_ON_ERROR path, `fsm_extend`, `fsm_set_and_search`, `fsm_get_location`, `fsm_logical_to_physical`, `fsm_space_avail_to_cat`/`fsm_space_cat_to_avail`, `FreeSpaceMapPrepareTruncateRel`, `FreeSpaceMapVacuum`, the category table and `FSM_TREE_DEPTH`), `indexfsm.c` (all four exported routines and the header NOTES), `fsm_internals.h` (`NodesPerPage` through `SlotsPerFSMPage`); nbtree page recycling in `nbtree.c` (`btvacuumscan`'s `pages_free`-is-index-state comment and its FSM-vacuum condition, `btvacuumpage`'s recyclable/deleted/half-dead branches), `nbtpage.c` (`_bt_allocbuf`'s FSM loop and its two reject paths, `_bt_pendingfsm_init`, `_bt_pendingfsm_add`, `_bt_pendingfsm_finalize`), `nbtree.h` (`BTPageIsRecyclable`, `BTDeletedPageData`, `BTPageSetDeleted`, `P_ISDELETED`/`P_ISHALFDEAD`/`P_IGNORE`, `BTVacState`), `README` ("Deleting entire pages during VACUUM"); storage lifecycle in `storage.c` (`RelationTruncate`'s per-fork preparation), `heap.c` (`RelationTruncateIndexes`), `relcache.c` (`RelationSetNewRelfilenumber`), `index.c` (`reindex_index`), `tablecmds.c` (`ExecuteTruncateGuts`), plus the `RelationTruncate` caller set (`vacuumlazy.c`, `heapam_handler.c`, and the `#ifdef NOT_USED` call in `spgvacuum.c`); reporting surfaces in `vacuumlazy.c` (the VERBOSE per-index line), `genam.h` (`IndexBulkDeleteResult` field semantics), `dbsize.c` (`half_rounded`, `size_pretty_units`, `pg_size_pretty`, `pg_size_pretty_numeric`), `queries.sgml` (`LIMIT` without a unique ordering), `genfile.c` (`convert_and_check_filename`'s `pg_read_server_files` check and `pg_read_binary_file_common`); contrib page-class sources `pgstatindex.c` with `pgstattuple--1.4.sql`, `pg_freespacemap.c` with `pg_freespacemap--1.1.sql`/`--1.1--1.2.sql` and its control file, `pageinspect`'s `btreefuncs.c` with `pageinspect--1.11--1.12.sql`; and tests `contrib/pg_freespacemap/{sql,expected}/pg_freespacemap` (the `avail > 0` idiom over a B-tree index) and `src/test/regress/{sql,expected}/dbsize` (negative `pg_size_pretty` for both variants).
- Follow-up exact-pin execution, two servers: the same DDL and generated data on one isolated 12.2 server and one isolated 17.10 server, each built from its own pin under `.wiki-runtime/`, both with `autovacuum = off`, `fsync = off`, `block_size` 8192, and no contrib extension installed. Fixtures: nine 1,000,000-row indexes (distinct, all-duplicate, 25% NULL, four duplication ratios, a 20% partial sibling, and a 10-key index with 90% of rows deleted and vacuumed), six shape indexes over a 200,000-row table, three `INCLUDE`-versus-key-column indexes over a 1,000,000-row table, an empty-table index, and a `TRUNCATE`-and-reload index. The dedup-aware sweep and the v12 page's Method A were installed as views on both servers with only the 1 MB triage filter and `LIMIT` removed and `expected_blocks` exposed; `CREATE INDEX CONCURRENTLY` rebuilds were the ground truth. Catalog probes covered `pg_amproc` support numbers, `array_lower(indclass, 1)`, `ALTER INDEX ... SET (deduplicate_items = off)`, `ALTER OPERATOR FAMILY ... ADD FUNCTION 4`, and `pg_class.reltuples` after build, `TRUNCATE` and reload. Both servers were stopped afterwards, the test databases dropped, and the 17.10 data directory removed.
- Twelve-issue-review follow-up, source coverage: nbtree build and deduplication in `nbtsort.c` (`_bt_pagestate`'s `btps_full`, `_bt_blnewpage`'s reserved high-key line pointer, `_bt_buildadd`'s hard and soft page-full rule and its high-key move, `_bt_load`'s `maxpostingsize` and its dedup loop, `_bt_sort_dedup_finish_pending`, `_bt_leafbuild`'s recomputed `allequalimage` and metapage write), `nbtdedup.c` (`_bt_dedup_start_pending`'s `basetupsize`, `_bt_dedup_save_htid`'s `mergedtupsz` test, `_bt_form_posting`), `nbtsplitloc.c` (`_bt_findsplitloc`'s header comment on fillfactor versus 50:50, the rightmost-leaf branch, `_bt_afternewitemoff`, the other-leaf branch, single-value strategy), `nbtutils.c` (`_bt_allequalimage`'s INCLUDE early return), `nbtpage.c` (`_bt_initmetapage`, `_bt_metaversion`), `nbtinsert.c` (the insert-time deduplication gate), `nbtree.h` (the btree-version comment on `btm_allequalimage` and pg_upgrade); extended statistics in `mvdistinct.c` (`statext_ndistinct_build`'s combination generator, `ndistinct_for_combination`, `estimate_ndistinct`'s clamping, `pg_ndistinct_out`'s key spelling, the negative-attnum expression convention), `extended_stats.c` (`BuildRelationExtStatistics`, the ndistinct build call, `stxdinherit`), `statscmds.c` (the two-column minimum and the `STATS_MAX_DIMENSIONS` limit), `statistics.h`; statistics visibility in `system_views.sql` (`pg_stats` and its `has_column_privilege`/RLS filter, the `REVOKE` on `pg_statistic`, `pg_stats_ext` and its `pg_has_role` filter, `pg_stats_ext_exprs`, `pg_stat_all_tables`'s live/dead/last-analyze columns) and `catalogs.sgml` (the `pg_statistic` and `pg_statistic_ext_data` visibility paragraphs); row-count sources in `pgstat_relation.c` (`pgstat_count_heap_insert`, `AtEOXact_PgStat`'s live/dead deltas, `pgstat_relation_flush_cb`'s clamped accumulation, `pgstat_report_analyze`'s re-basing), `index.c`, `analyze.c`, `vacuum.c`; contrib boundary in `btreefuncs.c` (`bt_metap`'s superuser check and its `allequalimage` output) and `pgstatindex.c` (`leaf_fragmentation`).
- Twelve-issue-review follow-up, exact-pin execution: one isolated **17.11** server built out of tree from the current pin under `.wiki-runtime/`, configured `--without-readline --without-zlib --without-icu`, `block_size` 8192, `autovacuum = off`, `fsync = off`, plus one isolated 12.2 server from its own pin for the portability run. Fixtures on 17.11: a nine-point duplication band at 1,000,000 rows (100, 132, 133, 143, 200, 264, 265, 500 and 1000 rows per key) chosen around the 132-TID cap; `i_q1000` and `i_cd` rebuilt to the shapes the earlier follow-up measured; `i_seq` and `i_rand` as ordered-versus-random insertion twins over the same 1,000,000 distinct keys; `i_wide` at 300,000 distinct 100-byte `text` keys; correlated, independent, superset-covered, gap-covered, reversed-column and expression multicolumn indexes with `CREATE STATISTICS ... (ndistinct)` objects; `i_inc_lowcard` and `i_dupoff` for the two gate conjuncts; `i_stale`, `i_dupdel`, `i_extdel2` and `i_ext50` for real reclaimable space; and an unprivileged `probe` role with no table privileges. The statement as filed and the corrected statement were installed as views, a `security_invoker` copy was used to read internals as the unprivileged role, and a third view dropped only the `least()` guard for rejected fix A. `CREATE INDEX CONCURRENTLY` copies are the ground truth; `pgstattuple` supplied page classes, densities and `leaf_fragmentation`, and `pageinspect` supplied `bt_metap`, `bt_page_stats`, `bt_multi_page_stats` and `bt_page_items` — both as ground truth only. The 12.2 server carried the ordered/random twins, the correlated multicolumn index with its statistics object, `i_q1000`, `i_wide` and `i_stale`, plus its own `probe` role.
- Column-rename follow-up, source coverage (no server run; this follow-up is source-only): output column labelling and sorting in `queries.sgml` (the Column Labels section and the `ORDER BY`-by-output-column rule); query-jumble surfaces in `queryjumblefuncs.c` (`CleanQuerytext`, `JumbleQuery`, `_jumbleNode` and the two generated includes), `gen_node_support.pl` (per-field `query_jumble_ignore` emission), `primnodes.h` (`TargetEntry.resname`), and `pgstatstatements.sgml` (the `queryid` post-parse-analysis and stability paragraphs); query-text surfaces in `pg_stat_statements.c` (`pgss_store`'s entry lookup, `generate_normalized_query`, `qtext_store`) and `postgres.c` (`log_statement` and duration logging in `exec_simple_query`); identifier limits in `pg_config_manual.h` (`NAMEDATALEN`), `scan.l` (the `{identifier}` rule) and `scansup.c` (`downcase_truncate_identifier`, `truncate_identifier`); type and error surfaces in `pg_proc.dat` (`pg_size_pretty` return types), `varlena.c` (`bttextcmp`, `text_cmp`, `varstr_cmp`) and `parse_relation.c` (`errorMissingColumn`); vocabulary in `glossary.sgml` (the Bloat entry), `ref/copy.sgml` ("recover the wasted space"), `maintenance.sgml` (routine reindexing), `pgstattuple--1.4.sql` (`free_space`/`free_percent`), plus whole-tree string searches for `bloat` across `system_views.sql`, `pg_proc.dat` and every contrib SQL script.
- Mandatory-tests follow-up, source coverage: the equal-image decision in `nbtutils.c` (`_bt_allequalimage`'s INCLUDE early return, its per-key-attribute loop over `rd_opfamily`/`rd_opcintype`/`rd_indcollation`, the `get_opfamily_proc` lookup followed by the actual call and the first-false `break`, and the `debugmessage` block that the early return skips), `nbtsort.c` (`_bt_leafbuild`'s `debugmessage = true` call, `_bt_load`'s three-way `deduplicate` condition), `nbtree.h` (`BTEQUALIMAGE_PROC`, `BTNProcs`, `BTGetDeduplicateItems`); the two stock support functions in `datum.c` (`btequalimage`'s unconditional true and its header comment about `opcintype`) and `varlena.c` (`btvarstrequalimage`'s C-collation/default-collation/`get_collation_isdeterministic` branches); function resolution in `fmgr.c` (`fmgr_info_cxt_security`'s built-in fast path that never consults `pg_proc`, the `INTERNALlanguageId` branch that resolves an alias by `prosrc` through `fmgr_lookupByName`, `OidFunctionCall1Coll`, and `FunctionCall1Coll`'s NULL-result `elog`); support-function validation in `opclasscmds.c` (`assignProcTypes`'s one-argument, boolean-return and non-cross-type checks for `BTEQUALIMAGE_PROC`, and the two `amadjustmembers` call sites) and `nbtvalidate.c` (`btvalidate`'s `check_amproc_signature` for support number 4 and its `invalid support number` default), with the SQL entry point in `amapi.c` (`amvalidate`); the nondeterministic-collation DDL refusal for the three pattern opclasses in `index.c`; catalog shape in `pg_amproc.dat` (every B-tree `amprocnum => '4'` row), `pg_proc.dat` (`btequalimage`, `btvarstrequalimage`), `pg_index.h` (`indclass`, `indcollation`, `indnatts`, `indnkeyatts`), `pg_collation.h` (`collisdeterministic`); `analyze.c`'s `stawidth = total_width / nonnull_cnt` assignment; `guc_tables.c` (`client_min_messages`); the contrib cross-check in `amcheck/verify_nbtree.c` (`bt_index_check`'s metapage-versus-`_bt_allequalimage` comparison and its `ERRCODE_INDEX_CORRUPTED` message) and `pageinspect/btreefuncs.c` (`bt_metap`, `bt_page_items`); the documentation in `btree.sgml` (the `equalimage` support-function contract, the every-column-must-return-true rule, the stock-function convention and the third-party-extension recommendation, and the unsafe-case list covering nondeterministic collations, `numeric`, `jsonb`, `float4`/`float8`, container types and `INCLUDE`); and the upstream tests `alter_generic.sql`/`.out` (cross-type `ADD FUNCTION 4`) and `opr_sanity.sql`/`.out` (which core opclasses may omit `btequalimage`).
- Mandatory-tests follow-up, exact-pin execution: one isolated **17.11** server built out of tree from the current pin under `.wiki-runtime/`, configured `--without-readline --without-zlib --with-icu --enable-debug`, `block_size` 8192, `autovacuum = off`, `fsync = off`, in a scratch database created for this run, plus the isolated 12.2 server from its own pin for test 17. Fixtures: `t`, 500,000 rows with 5,000 distinct keys over `int4`, `int8`, `text`, `numeric`, `float4`, `float8`, a unique `int4` and a 7-value `int4`, carrying 24 B-tree indexes that cover the seventeen requirements — two deterministic-collation text indexes (default and ICU `und`), one nondeterministic ICU index, four expression indexes chosen so the expression's type and collation differ from the underlying column's, an `INCLUDE` index, a `deduplicate_items = off` twin per key type, a unique index, and six custom-opclass indexes; and `t2`, the same shape on `(int4, int8)` with a `CREATE STATISTICS ... (ndistinct)` object so the mixed-key over-credit becomes visible as a percentage. Eight custom operator classes registered SQL, PL/pgSQL and `LANGUAGE internal` support functions returning true, false and NULL, plus one that raises. Ground truth per index is the build's own `DEBUG1` verdict, `bt_metap().allequalimage`, and `count(tids) > 0` over `bt_page_items(index, 1)`, with a `deduplicate_items = off` twin as the physical size baseline; `pageinspect` and `amcheck` were installed as ground truth only. Both statements on this page were installed as views generated mechanically from this page's own SQL text — the filed text and a copy with only the `all_equalimage` subquery replaced — with the 1 MB triage filter, `ORDER BY` and `LIMIT` removed and `dedup_applies`, `all_equalimage`, `slot`, `leaf_cap`, `expected_blocks` and `floor_blocks` exposed. Additional probes: `amvalidate` on every custom opclass, four rejected `ADD FUNCTION 4`/`CREATE INDEX` DDL shapes, `CREATE OR REPLACE FUNCTION pg_catalog.btequalimage`, a rename-and-squat of `pg_catalog.btvarstrequalimage`, a post-build support-function mutation followed by `bt_index_check` and `REINDEX`, the reverse mutation via `ALTER OPERATOR FAMILY ... ADD FUNCTION 4` followed by `REINDEX`, and a stock-database census of B-tree `amprocnum = 4` rows. Both servers were stopped afterwards, with the scratch databases dropped and `pg_catalog.btequalimage` restored to `LANGUAGE internal`.
- Recommended-statement follow-up (no server run; it selects among statements already filed and measured above): re-read the equal-image decision in `nbtutils.c` (`_bt_allequalimage`'s lookup-then-call and its first-false `break`), `nbtsort.c` (`_bt_leafbuild`'s recomputed flag, `_bt_load`'s three-way `deduplicate` condition), function resolution in `fmgr.c` (`fmgr_info_cxt_security`'s built-in fast path and the `INTERNALlanguageId` branch that resolves by `prosrc`), the stock B-tree `amprocnum => '4'` rows in `pg_amproc.dat`, the operator-class rule in `btree.sgml`, and `guc_tables.c` for `statement_timeout` and `lock_timeout`; plus every statement variant, measurement table, caveat and open question already on this page, which is where the ranking's numbers come from.

## Evidence Map

| Claim | Evidence |
|---|---|
| Leaf fill rule, high-key line pointer, and non-leaf fillfactor 70 are unchanged | [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtsort.c#_bt_blnewpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L605-L629), [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L809-L855), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202) |
| Usable page space and the 8152-byte denominator | [nbtsplitloc.c:157-160](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L157-L160), [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L210-L216), [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L69) |
| Entry cost is `MAXALIGN(hoff + data) + 4`, NULLs widen `hoff` | [itup.h#IndexInfoFindDataOffset](../../../../raw/postgres-17/src/include/access/itup.h#L96-L110), [indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L65-L140), [heaptuple.c#heap_compute_data_size](../../../../raw/postgres-17/src/backend/access/common/heaptuple.c#L215-L263), [itemid.h#ItemIdData](../../../../raw/postgres-17/src/include/storage/itemid.h#L25-L31) |
| A sorted build deduplicates non-unique, all-equal-image indexes by default | [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1160), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1150) |
| Build-time posting lists are capped at 812 bytes; each extra row costs one 6-byte TID | [nbtsort.c:1292-1308](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1292-L1308), [nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L484-L531) |
| Insert-time deduplication uses a different, larger cap | [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L75-L93) |
| All-duplicate leaf splits use fillfactor 96 | [nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L406-L416), [nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202) |
| `ANALYZE` stores `stadistinct` as a negative fraction above 10% distinct | [analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612), [analyze.c:2544-2549](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2544-L2549), [pg_statistic.h#stadistinct](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L52-L69) |
| `reltuples = -1` means unknown, and index creation on an empty table preserves it | [pg_class.h:62-66](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842) |
| Index `reltuples` is written by the build, by ANALYZE's sample, or by a non-estimated VACUUM count | [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842), [analyze.c:637-660](../../../../raw/postgres-17/src/backend/commands/analyze.c#L637-L660), [vacuumlazy.c:3078-3098](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3078-L3098), [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1470) |
| `n_live_tup`/`n_dead_tup` come from the cumulative statistics system | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L700) |
| `pg_stats` exposes `avg_width`, `null_frac` and `n_distinct` | [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L215) |
| `pg_relation_size` is a live filesystem measurement of one fork | [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343), [relpath.c#forkNames](../../../../raw/postgres-17/src/common/relpath.c#L33-L40) |
| `pg_column_size` returns the stored datum size | [varlena.c#pg_column_size](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L5062-L5102) |
| A forward index scan reads one buffer per right link and ignores dead pages | [nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2181-L2240) |
| Index-only scans fall back to the heap when the VM bit is unset, reported as `Heap Fetches` | [nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L151-L171), [explain.c:1993](../../../../raw/postgres-17/src/backend/commands/explain.c#L1993) |
| `BUFFERS` is one per-node counter with no per-relation split | [explain.c#show_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3743-L3800), [explain.sgml#BUFFERS](../../../../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L182-L200) |
| v17 `VACUUM VERBOSE` prints four page classes per index and no row count | [vacuumlazy.c:718-732](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732), [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L75-L84) |
| Index vacuuming is bypassed below 2% of pages, and forced by `INDEX_CLEANUP ON` | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1864-L1935), [vacuumlazy.c:388-407](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L388-L407) |
| A no-op VACUUM prints no index line | [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L851-L924) |
| Deleted pages stay in the file and are only recorded in the FSM | [nbtree.c#btvacuumpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1160-L1190), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [README:236-246](../../../../raw/postgres-17/src/backend/access/nbtree/README#L236-L246) |
| Partly-emptied pages remain allocated, the documented bloat mechanism | [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1032-L1040) |
| CIC restrictions, lock level and invalid-index leftover | [utility.c:1456-1466](../../../../raw/postgres-17/src/backend/tcop/utility.c#L1456-L1466), [indexcmds.c:672-682](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L672-L682), [indexcmds.c:723-733](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L723-L733), [index.c#index_set_state_flags](../../../../raw/postgres-17/src/backend/catalog/index.c#L3478-L3550), [create_index.sgml:625-635](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L625-L635) |
| v17 builds write through the bulk-write facility, syncing via the checkpointer | [nbtsort.c:1145-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1145-L1152), [nbtsort.c:1370-1377](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1370-L1377), [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L220) |
| `pg_get_indexdef` emits reloptions but not `CONCURRENTLY` or the tablespace | [ruleutils.c#pg_get_indexdef](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L1160-L1177) |
| `pgstatindex`'s density denominator and fragmentation definition | [pgstatindex.c:310-316](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L310-L316), [pgstatindex.c:363-367](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367), [pgstatindex.c:318-325](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L325) |
| The bloat tooling is contrib and superuser-gated | [extension.c:778-790](../../../../raw/postgres-17/src/backend/commands/extension.c#L778-L790), [extension.c#execute_extension_script](../../../../raw/postgres-17/src/backend/commands/extension.c#L1019-L1035), [pgstatindex.c:145-160](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L145-L160), [rawpage.c:148-154](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L148-L154) |
| `TABLESAMPLE` cannot be applied to an index | [parse_clause.c:1136-1146](../../../../raw/postgres-17/src/backend/parser/parse_clause.c#L1136-L1146) |
| GUC contexts used by these methods | [guc_tables.c:2611-2631](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631), [guc_tables.c:2465-2474](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474), [guc_tables.c:3268-3277](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3268-L3277), [guc_tables.c:4986-4994](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4986-L4994) |
| The build-time deduplication decision is `allequalimage AND NOT isunique AND deduplicate_items` | [nbtsort.c:1147-1152](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152), [nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563) |
| `allequalimage` is false when any key column's opclass lacks a `BTEQUALIMAGE_PROC` entry, and unconditionally false for an INCLUDE index | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183), [nbtutils.c:5144-5148](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5148) |
| Equal image is B-tree support function number 4, which is why the sweep probes `pg_amproc.amprocnum = 4` | [nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L702-L712) |
| A support-function number cannot exceed the access method's `amsupport`, so a pre-13 catalog cannot advertise number 4 | [nbtree.c#bthandler](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L100-L107), [opclasscmds.c:840-845](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L840-L845), [opclasscmds.c#invalid-function-number](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L956-L962), [nbtvalidate.c:90-126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L90-L126) |
| `deduplicate_items` is a B-tree reloption defaulting to on | [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtutils.c#btoptions](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4561-L4576), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1150) |
| Equal-image support functions, nbtree deduplication and the `-1` `reltuples` sentinel all postdate 12 | this checkout's history: `612a1ab7672` and `0d861bbb702` (2020-02-26, first in `REL_13_0`), `3d351d916b2` (2020-08-30, first in `REL_14_0`); none is an ancestor of `REL_12_2` |
| The equal-image function can exist and still return false, so a presence-only catalog probe is not exact | [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613) |
| A posting tuple's size is `MAXALIGN(keysize + nhtids * sizeof(ItemPointerData))`, and `keysize` is already MAXALIGNed | [nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L863-L911), [nbtdedup.c#_bt_dedup_start_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L432-L475), [indextuple.c:154-163](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L154-L163) |
| A leaf page under construction accepts posting tuples until `pgspc + last_truncextra < btps_full`, so the posting list of the future high key raises the effective capacity | [nbtsort.c:769-781](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L769-L781), [nbtsort.c:845-855](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L855), [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L661-L666), [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1144-L1145) |
| Two NULL keys count as equal for deduplication, so a NULL run forms one key group | [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4890-L4902) |
| `most_common_freqs` are sample frequencies over the total row count, stored as `STATISTIC_KIND_MCV` | [analyze.c:2664-2684](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2664-L2684), [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L218) |
| A nondeterministic collation is visible in core SQL as `pg_collation.collisdeterministic = false` for one of `pg_index.indcollation`'s entries | [pg_collation.h:40](../../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40), [pg_index.h:53-54](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L53-L54), [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2599-L2613) |
| `n_mod_since_analyze`, `n_live_tup` and `n_dead_tup` all come from the cumulative statistics system, not from `pg_class` | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L686-L694) |
| `default_statistics_target` is session-scoped and only changes estimates at the next `ANALYZE` | [guc_tables.c:2071-2078](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2078), [analyze.c:2605-2612](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2605-L2612) |
| An index's FSM stores only "free" or "in-use", written as `BLCKSZ - 1` and 0 | [indexfsm.c#NOTES](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [indexfsm.c#RecordUsedIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L57-L65) |
| Recording free space creates and extends the FSM fork; at `block_size` 8192 the first recorded page makes it three blocks | [freespace.c#RecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L186-L204), [freespace.c#fsm_set_and_search](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L648-L682), [freespace.c#fsm_readbuf](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L557-L631), [freespace.c#fsm_extend](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L633-L646), [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L461-L495), [fsm_internals.h#SlotsPerFSMPage](../../../../raw/postgres-17/src/include/storage/fsm_internals.h#L47-L61) |
| Taking a page out of an index's FSM marks the slot used but never shortens the fork | [nbtpage.c#_bt_allocbuf](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L868-L905), [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L32-L46) |
| The only in-place FSM shortening path is heap/table truncation, not any B-tree VACUUM path | [freespace.c#FreeSpaceMapPrepareTruncateRel](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L273-L359), [storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L288-L326), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2638-L2650), [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3055-L3092), [spgvacuum.c#NOT_USED-truncate](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L889-L901) |
| A rebuild or transactional `TRUNCATE` replaces the relfilenode and creates the main fork alone, so the FSM fork returns to zero length | [index.c#reindex_index-relfilenumber](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789), [tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2160-L2189), [relcache.c#RelationSetNewRelfilenumber](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3842-L3899) |
| A deleted B-tree page enters the FSM only once its `safexid` is invisible to every snapshot; the rest wait for a later VACUUM | [nbtree.c#btvacuumpage-page-classes](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1189), [nbtree.h#BTPageIsRecyclable](../../../../raw/postgres-17/src/include/access/nbtree.h#L279-L318), [nbtpage.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2984-L3055), [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1043-L1059) |
| The FSM is a hint: not WAL-logged, zeroed when corrupt, and a page it hands out can be lost until the next VACUUM | [freespace.c#fsm_readbuf-zero-on-error](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L586-L607), [nbtpage.c#_bt_allocbuf-reject](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L894-L969) |
| `VACUUM VERBOSE` is the only core source of current index page classes, and `pages_free` is whole-index state | [vacuumlazy.c#verbose-index-line](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732), [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L66-L84), [nbtree.c#pages_free-is-index-state](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L949-L967) |
| `pgstatindex.deleted_pages` counts every deleted page and `empty_pages` is the half-dead class | [pgstatindex.c#page-classes](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L331), [pgstatindex.c#result-tuple](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L372), [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31) |
| `pg_freespace` reports the map's current per-block verdict, and its own test uses the `avail > 0` idiom on a B-tree index | [pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L18-L50), [freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L249-L271), [sql/pg_freespacemap.sql:8-12](../../../../raw/postgres-17/contrib/pg_freespacemap/sql/pg_freespacemap.sql#L8-L12), [expected/pg_freespacemap.out#btree-rows](../../../../raw/postgres-17/contrib/pg_freespacemap/expected/pg_freespacemap.out#L29-L53) |
| `pageinspect` classifies each B-tree page as deleted leaf/internal, half-dead, leaf, internal or root | [btreefuncs.c#page-type](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L125-L163), [pageinspect--1.11--1.12.sql#bt_multi_page_stats](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.11--1.12.sql#L6-L23) |
| `pg_size_pretty` renders negative sizes symmetrically, and the regression suite asserts it | [dbsize.c#half_rounded](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L34-L57), [dbsize.c#pg_size_pretty](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L565-L604), [sql/dbsize.sql:1-4](../../../../raw/postgres-17/src/test/regress/sql/dbsize.sql#L1-L4), [expected/dbsize.out#negatives](../../../../raw/postgres-17/src/test/regress/expected/dbsize.out#L1-L13) |
| A `LIMIT` without a unique ordering returns an unpredictable subset | [queries.sgml#limit-ordering](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1940-L1947) |
| `AS` names an output column and nothing else; `ORDER BY` may use that name only as a bare sort key | [queries.sgml#Column-Labels](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1556-L1602), [queries.sgml#sort-by-output-column](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1867-L1886) |
| An output column label cannot change `queryid`: the jumble walks the post-analysis tree and `TargetEntry.resname` is `query_jumble_ignore`, which `gen_node_support.pl` emits no code for | [queryjumblefuncs.c#JumbleQuery](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L104-L127), [primnodes.h#TargetEntry](../../../../raw/postgres-17/src/include/nodes/primnodes.h#L2186-L2203), [gen_node_support.pl#query_jumble_ignore](../../../../raw/postgres-17/src/backend/nodes/gen_node_support.pl#L1283-L1324), [queryjumblefuncs.c#generated-includes](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L227-L255), [pgstatstatements.sgml#queryid](../../../../raw/postgres-17/doc/src/sgml/pgstatstatements.sgml#L613-L639) |
| An inline comment survives into the logged and stored statement text, and the stored text is written only when the hash entry is created | [postgres.c#log_statement](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L1071-L1081), [postgres.c#duration-statement](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L1369-L1380), [queryjumblefuncs.c#CleanQuerytext](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L61-L102), [pg_stat_statements.c#pgss_store-entry](../../../../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1312-L1369) |
| Identifiers are truncated only at `NAMEDATALEN - 1` bytes, with a `NOTICE` | [pg_config_manual.h#NAMEDATALEN](../../../../raw/postgres-17/src/include/pg_config_manual.h#L22-L29), [scan.l:1096-1103](../../../../raw/postgres-17/src/backend/parser/scan.l#L1096-L1103), [scansup.c#truncate_identifier](../../../../raw/postgres-17/src/backend/parser/scansup.c#L84-L105) |
| Selecting a name the statement no longer emits raises `ERRCODE_UNDEFINED_COLUMN` | [parse_relation.c#errorMissingColumn](../../../../raw/postgres-17/src/backend/parser/parse_relation.c#L3712-L3733) |
| `pg_size_pretty` returns `text`, so ordering by its result is a collated string comparison | [pg_proc.dat#pg_size_pretty](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507), [varlena.c#bttextcmp](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L1875-L1888), [varlena.c#varstr_cmp](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L1583-L1590) |
| "Bloat" is defined in the docs as per-page space, while "wasted space" is the docs' phrase for space a maintenance command recovers; contrib names the quantity `free_space` | [glossary.sgml#Bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250), [copy.sgml#recover-the-wasted-space](../../../../raw/postgres-17/doc/src/sgml/ref/copy.sgml#L613-L621), [maintenance.sgml#non-btree-bloat](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046), [pgstattuple--1.4.sql#free_space](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L14-L15), [zic.c#bloat](../../../../raw/postgres-17/src/timezone/zic.c#L641-L646) |
| A run of equal keys is flushed as its own posting tuple or tuples, so key groups never share one, and the last tuple of a group is partial | [nbtsort.c:1323-1335](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1323-L1335), [nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1029-L1057), [nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L504-L531) |
| A leaf under construction is finished on a hard space test or a fillfactor test discounted by the previous tuple's posting list, and the last item added becomes the truncated high key rather than a data item | [nbtsort.c:853-854](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L853-L854), [nbtsort.c#_bt_buildadd-highkey](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L873-L940), [nbtsort.c:663-665](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L663-L665) |
| An `ndistinct` extended-statistics value is an absolute count clamped to the sampled distinct count and the row count, never a negative fraction | [mvdistinct.c#estimate_ndistinct](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L519-L542), [mvdistinct.c#ndistinct_for_combination](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L424-L517) |
| `pg_ndistinct` renders as a JSON object keyed by ascending comma-space-separated attnums, with an item for every 2-to-N column combination | [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L354-L385), [mvdistinct.c#statext_ndistinct_build](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L87-L141) |
| Expressions in an extended-statistics object are stored as negative attnums, so an `indkey` attnum cannot match one | [mvdistinct.c:83-86](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L83-L86) |
| A statistics object needs 2 to 8 columns, and its data exists only after `ANALYZE` builds it | [statscmds.c:415-419](../../../../raw/postgres-17/src/backend/commands/statscmds.c#L415-L419), [statscmds.c:216-224](../../../../raw/postgres-17/src/backend/commands/statscmds.c#L216-L224), [statistics.h#STATS_MAX_DIMENSIONS](../../../../raw/postgres-17/src/include/statistics/statistics.h#L19), [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L112-L133) |
| `pg_stats` is publicly readable but row-filtered by column privilege and RLS, and `pg_statistic` itself is revoked from `public` | [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L271-L273), [system_views.sql:275](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L275), [catalogs.sgml#pg_statistic](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L7431-L7440) |
| `pg_stats_ext` is stricter: it needs membership in the table owner's role | [system_views.sql#pg_stats_ext](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L308-L309), [catalogs.sgml#pg_statistic_ext_data](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L7764-L7775) |
| `pg_stat_all_tables` exposes live/dead tuples and the last analyze time with no privilege filter | [system_views.sql#pg_stat_all_tables-live](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L687-L693) |
| `n_live_tup` is maintained by DML deltas at commit and re-based by `ANALYZE`, without needing `ANALYZE` to move | [pgstat_relation.c#pgstat_count_heap_insert](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L360-L369), [pgstat_relation.c#AtEOXact_PgStat](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L553-L575), [pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L838-L866), [pgstat_relation.c#pgstat_report_analyze](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L277-L311) |
| A rightmost leaf split targets `fillfactor` while every other leaf split targets 50:50, so only ordered insertion reaches a sorted build's density | [nbtsplitloc.c:88-102](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L88-L102), [nbtsplitloc.c:286-291](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L286-L291), [nbtsplitloc.c:292-303](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L292-L303), [nbtsplitloc.c:329-335](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L329-L335) |
| A build recomputes `allequalimage` from the catalog and only then writes it to the new metapage, while the metapage copy is what the insert path reads | [nbtsort.c:561-563](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L561-L563), [nbtsort.c:1124-1127](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1124-L1127), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L60-L90), [nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L718-L795), [nbtinsert.c:2772-2782](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2772-L2782) |
| A v12-built index keeps `btm_allequalimage` false until a `REINDEX`, and `bt_metap` is superuser-only | [nbtree.h:135-142](../../../../raw/postgres-17/src/include/access/nbtree.h#L135-L142), [btreefuncs.c:905-921](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L905-L921), [btreefuncs.c:840-857](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L840-L857) |
| The equal-image decision looks the function up **and calls it**, so a registered support function that returns false is the same outcome as no support function at all | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183), [nbtutils.c:5156-5169](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5156-L5169), [btree.sgml#equalimage](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L499-L509) |
| An `INCLUDE` index returns false before any lookup, and before the debug message, so it logs no deduplication verdict | [nbtutils.c:5144-5147](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147), [nbtutils.c:5172-5180](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180), [btree.sgml:897-909](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L897-L909) |
| The loop is over index key attributes, using each key's own opclass, input type and collation, so an expression key is judged by the expression | [nbtutils.c:5149-5157](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5149-L5157), [pg_index.h:48-54](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L48-L54) |
| The two stock support functions are unconditional-true and deterministic-collations-only | [datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L415-L438), [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613), [btree.sgml:538-550](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L538-L550) |
| `numeric`, `jsonb`, `float4`/`float8`, container types and `INCLUDE` are the documented unsafe cases, and the catalog form is a missing support-function row | [btree.sgml:834-909](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L909), [pg_amproc.dat:143](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L143), [opr_sanity.sql:1336-1353](../../../../raw/postgres-17/src/test/regress/sql/opr_sanity.sql#L1336-L1353), [opr_sanity.out:2204-2222](../../../../raw/postgres-17/src/test/regress/expected/opr_sanity.out#L2204-L2222) |
| A support function is validated for arity, return type and non-cross-typedness only — never for language or return value | [opclasscmds.c:1310-1333](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L1310-L1333), [opclasscmds.c:696-704](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L696-L704), [nbtvalidate.c:110-126](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L110-L126), [amapi.c#amvalidate](../../../../raw/postgres-17/src/backend/access/index/amapi.c#L110-L143), [alter_generic.sql:444-446](../../../../raw/postgres-17/src/test/regress/sql/alter_generic.sql#L444-L446) |
| A NULL-returning support function aborts the build rather than disabling deduplication | [fmgr.c#OidFunctionCall1Coll](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L1411-L1418), [fmgr.c:1141-1143](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L1141-L1143) |
| `prosrc`, not `proname`, is the identity the engine resolves: built-in OIDs skip `pg_proc` entirely, and a `LANGUAGE internal` alias is looked up by `prosrc` | [fmgr.c:166-178](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L166-L178), [fmgr.c:216-240](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240) |
| A nondeterministic collation cannot be combined with the three pattern opclasses, which is why a whole-index collation test cannot be wrong in that direction | [index.c:807-850](../../../../raw/postgres-17/src/backend/catalog/index.c#L807-L850), [pg_amproc.dat:241](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L241) |
| A metapage flag that disagrees with the current opclasses is corruption to `amcheck`, not a stale cache | [verify_nbtree.c:379-400](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L379-L400) |
| `pg_stats.avg_width` is an integer truncation of the mean stored width | [analyze.c:2536-2540](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2536-L2540), [pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L40-L50) |
| `client_min_messages` is `PGC_USERSET`, so the harness's `debug1` needs no reload or restart | [guc_tables.c:4777-4785](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4777-L4785) |
| The recommended gate swap can only move an index whose key opclass is custom, because every stock B-tree equal-image row names one of the two whitelisted `LANGUAGE internal` functions and a built-in OID never reaches `pg_proc` at all | [pg_amproc.dat:143](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L143), [pg_amproc.dat:206](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L206), [pg_amproc.dat:241](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L241), [fmgr.c:166-178](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L166-L178) |

## Open Questions

- **The v12 fixtures were reconstructed, not recovered.** The v12 page records each fixture's shape and its resulting block counts but not its DDL, so `idx_multi`, `idx_var`, `idx_rand` and `idx_churn` differ from the v12 page's block counts for reasons that include fixture choice. Nine of fifteen fixtures reproduce the v12 block count exactly, which bounds but does not eliminate the risk that a difference attributed to v17 is really a difference in the recipe.
- **v12 numbers in the original comparison are quoted, not re-measured.** Every "v12 page" figure in the sections above comes from that page's own tables, so those figures are attributions rather than evidence from the v12 checkout. The 12.2 server used for the v12/v17 follow-up carried new fixtures and does not re-measure the v12 page's own numbers.
- **The 12.2 column of the follow-up is measurement plus history, not v12 source citation.** This page may cite only `raw/postgres-17/`, so every 12.2 statement above rests on exact-pin execution against a 12.2 server plus this checkout's own commit history. The v12-side source analysis — where v12's `BTNProcs` is 3, what its `btoptions` accepts, and what writes its `reltuples` — belongs on [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md) and is not filed there yet.
- **Only 12 and 17 were exercised.** Majors 13 through 16 were not run. 13 is the interesting gap: deduplication and the equal-image support function exist there, so the sweep would credit the posting-list term, while the `-1` `reltuples` sentinel does not exist until 14, so the stale-zero hazard described above applies at the same time.
- **The `pg_upgrade` reading was not measured here, but has since been measured separately.** A duplicate-key index built by 12 and carried onto a 17 server holds no posting lists, while the sweep predicts the size a 17 rebuild would produce, so the reported "bloat" should be the deduplication win a `REINDEX` would realize. That reading is now confirmed by real 12.2 → 17.10 `pg_upgrade` runs in [Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade From PostgreSQL 12 to 17 (unverified)](btree-deduplication-after-pg-upgrade.md), which measured a carried-over 10-distinct-value index at 22,519,808 bytes against 6,963,200 after a rebuild, exactly the size of a fresh `deduplicate_items = off` twin. That page also records the trap this one does not cover: `pg_upgrade` leaves `relpages` and `reltuples` at 0 for every carried-over index, so this sweep is blind until the first `ANALYZE`.
- **On 12.2 the model ran 1 to 6 blocks under a rebuild whenever keys repeated** — 2745 modelled against 2749 built on four single-column fixtures, 2748 on one, 2746 on the NULL fixture, and 3853 against 3859 on the two two-attribute `INCLUDE` cases — while distinct keys were exact. The cause was not isolated; no page-level tool was installed on that server, so leaf-versus-internal attribution was not possible.
- **The nondeterministic-collation false positive is source-derived here.** `btvarstrequalimage` returns false for a nondeterministic collation while the `pg_amproc` row still exists, so the sweep's presence-only probe would credit deduplication the engine refuses. This build has no ICU support (`CREATE COLLATION ... provider = icu` fails with `ICU is not supported in this build`), so the case was not measured on this page. It has since been measured on a separate ICU-enabled 17.10 build in [Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade From PostgreSQL 12 to 17 (unverified)](btree-deduplication-after-pg-upgrade.md): the nondeterministic-collation index stayed at 4,521,984 bytes where its deterministic twin deduplicated to 1,400,832, and the engine logged "cannot use deduplication" for it.
- **The INCLUDE fix was scored on one server.** Adding `x.indnatts = x.indnkeyatts` to the gate corrected the one affected index and left the other 33 in that 17.10 database unchanged, but it was not re-scored against the 54-cell matrix above.
- **The dedup term ignores the posting-list cap.** Charging a flat 6 bytes per extra row underestimates an all-duplicate rebuild by about 3% (825 modelled against 849 measured at 1,000,000 rows) because it ignores both the 812-byte cap and the base tuple that each capped posting list repeats. The exact arithmetic is derivable from `_bt_dedup_save_htid` plus the `_bt_buildadd` `truncextra` rule, but it was not implemented in SQL.
- **The gate is a heuristic with a measured blind spot.** Refusing to credit deduplication when `n_distinct` is negative leaves +10.9% error at 2 rows per key and +92.5% at 5. A `SET (n_distinct = ...)` column override would close it, but that was not tested.
- **Multi-column group estimation is a product of per-column `n_distinct`.** No cell in the 54-cell matrix exercised a multi-column index with duplicates. The follow-up added one: `(a, d)` with 1000 and 5 distinct values over 1,000,000 rows produced 5000 groups and modelled 842 blocks against a rebuild of 897, so the product rule was 6.1% optimistic on the one case measured. Whether that error is the product rule or the posting-list cap was not separated.
- **`n_live_tup` read 2,000,000 for a 1,000,000-row table** after a truncate-and-reload in one session, while a separate clean run of the same sequence showed the counter resetting correctly. The discrepancy was not traced to a specific flush path; the recommendation above avoids depending on it, but the cause is unresolved.
- **Half-dead pages were never produced.** No fixture reached a non-zero `empty_pages`, so the claim that Methods A and B lump half-dead pages in with deleted ones follows from the code path, not from measurement.
- **Bottom-up index deletion was not isolated.** `idx_churn` and the `churn_*` matrix cells differ from the v12 page's sizes, and v14's bottom-up deletion is the obvious mechanism, but no fixture here separates it from the update pattern; the model's accuracy does not depend on which it is.
- **Block sizes other than 8192 were not exercised**, and `MAXALIGN` was assumed to be 8.
- **No upstream test covers these estimates.** The pinned tree has no regression test comparing a modelled index size against a built one, so every accuracy number here rests on the exact-pin fixtures described above.
- **Majors 13, 15 and 16 were never run.** The portable statement was measured on 12.2, 14.23 and 17.10 only, because those are the checkouts this repo pins. 13 is the interesting hole: deduplication and the equal-image support function exist there, while the `-1` `reltuples` sentinel does not, so a 13 server should take the deduplication branch and the stale-zero branch at the same time. The statement covers that combination by construction — the two `reltuples` branches are independent — but no 13 server confirmed it. 15 and 16 should behave exactly like 14.23 for every construct the statement reads.
- **The skew repairs were measured on one server and one shape.** `SET STATISTICS 1000` and `SET (n_distinct = -0.75)` were applied to a single 25%-hot-value column on 17.10. Neither the cost of the larger statistics target nor the staleness risk of a hard-coded `n_distinct` override was measured, and no equivalent run was made on 12.2 or 14.23.
- **The posting-list packing loss is measured, not modelled.** Where a key group needs more than one posting tuple the last one is partial, and the statement's uniform tuple size understates the rebuild by 5.5% on three fixtures and 8.2% on a 33-byte-key text index. Deriving the exact mixed-size packing would need the same per-page simulation `_bt_buildadd` performs, which is not expressed in the SQL.
- **The multi-column mixture is still the product rule.** Only one two-column duplicate case (`(a, d)` with 1000 x 5 distinct values) was measured, and it lands at −5.5%, indistinguishable from the single-column packing loss on the same data. Whether the product rule or the packing loss dominates was not separated, and no NULL-plus-MCV mixture was applied across two key columns.
- **Choosing caveats over suppression trades one error for another.** The earlier sweep on this page reports a stale partial index as `unmeasured`; the proposal reports the number with a `partial: predicate subset may be stale` caveat instead. That keeps the true 88.9% detection on `i_dupdelp`, whose statistics really were current, and keeps the false negative on `i_stale_part`, which reads 0.0% against a true 94.6%. Which default is better was not tested against a real monitoring workload.
- **The stale-zero rule leans on the cumulative statistics counters.** On a 12 or 13 server the rule fires only when `pg_stat_all_tables.n_live_tup` is above zero. After `pg_stat_reset()`, or on a standby, a genuinely stale `reltuples = 0` would pass the rule and report ~100% bloat on a healthy index. That path was not measured.
- **Two counter artifacts were observed and not traced.** Immediately after `VACUUM (ANALYZE)`, `t_dupdelp` reported `n_dead_tup` and `n_mod_since_analyze` of 180,000 and `t_alldead` reported 200,000 dead, on all three servers; `t_trunc` reported `n_live_tup` 600,000 for 300,000 real rows on 17.10. Message-ordering between the analyze report and the DML report is the obvious suspect, but no source path was confirmed. This is why the `caveats` column tests `pg_class.reltuples` against `n_live_tup` rather than trusting `n_mod_since_analyze`.
- **The 17.10 server for this follow-up is a different build of the same pin.** It was configured `--with-icu` so that `CREATE COLLATION ... deterministic = false` would work, unlike the `--without-icu` build used earlier on this page. The ICU-specific rows are `i_ci` and `i_cd`; no other cell depends on the build option, but the two builds were not diffed cell by cell.
- **The 30% alert threshold is arbitrary.** True-positive and false-positive counts are reported at that one threshold on 36 fixtures. No sweep over thresholds, and no production index population, was measured.
- **The two reporting corrections were not run on a server.** Both are source-derived: the renamed boolean reads the same `fsm_bytes` expression, and the un-clamped `wasted_space` is the same subtraction without `greatest(..., 0)`. The claim that no percentage changes follows from the expressions, which share `expected_blocks`, `floor_blocks` and `live_rows` with the versions that produced the tables above, but no server re-ran the corrected statement to confirm it row for row.
- **The rendered negative sizes are derived, not observed.** `-1136 kB`, `-10 MB` and `-7824 kB` come from applying `pg_size_pretty`'s unit selection and `half_rounded` by hand to −1,163,264, −10,805,248 and −8,011,776 bytes. The regression suite proves the sign and unit behavior but not these three specific strings.
- **No fixture's FSM state was ever recorded.** The `has_freed_pages` column was in both statements but never printed in a table here, so neither direction of its failure was observed on the fixtures: no run confirmed a true reading on an index whose recorded pages had all been reused, and none confirmed a false reading on an index full of deleted-but-not-recyclable pages. `idx_range` (2330 deleted pages) and `idx_del` are the fixtures that would show it, and `pg_freespace` was never installed on those servers.
- **The FSM fork size arithmetic assumes `block_size` 8192 and `MAXALIGN` 8.** `SlotsPerFSMPage` 4069, `FSM_TREE_DEPTH` 3, the resulting 24576-byte first extension and the 16384 bytes a truncation leaves behind are all computed from the pinned constants at that block size; smaller block sizes take the four-level branch and were not worked through, and no server confirmed either byte count.
- **The replacement page-class sources were not exercised here.** `pg_freespace`, `pgstatindex.deleted_pages`/`empty_pages` and `bt_page_stats` are named from source and from `pg_freespacemap`'s own regression output; no run on this page compared their counts against each other on the same index, so the "`deleted_pages` minus reusable equals not-yet-recyclable" identity is a source-level deduction about a quiescent index, not a measurement.
- **Whether a signed `wasted_space` breaks a real consumer was not tested.** The consumer list is reasoned from the statement's own output shape and the documented `LIMIT` ordering rule. No monitoring pipeline was pointed at either version of the statement.
- **The renamed statements were never executed.** No server ran either statement under its new labels, so the claim that only the labels moved rests on reading the two `SELECT` lists: the same `expected_blocks`, `floor_blocks`, `live_rows`, `key_groups` and `tids` feed the same expressions, and the `ORDER BY` keys never named a label. Nothing was re-measured, and no psql capture with the new header exists on this page.
- **The query-ID and query-text consequences of the tag rename are source-derived.** `queryid` stability follows from `JumbleQuery` walking the parse tree and from `TargetEntry.resname` carrying `query_jumble_ignore`; the surviving comment and the write-once query text follow from `CleanQuerytext` and `pgss_store`. No 17 server was started with `pg_stat_statements` loaded to confirm that one entry keeps the old tag while the new tag is what a fresh entry shows, and the generated `queryjumblefuncs.funcs.c` was read only through `gen_node_support.pl`, since generated files are not in the checkout.
- **`wasted_space_bytes` is a recommendation, not part of either statement.** Neither statement emits the raw `bigint`; the name is proposed for a consumer that parses output, and no run compared a text `wasted_space` ordering against a byte ordering to demonstrate the `9 bytes` before `10 MB` inversion.
- **The negative string searches are searches, not proofs of intent.** "No SQL-visible interface says bloat" comes from string searches of `system_views.sql`, `pg_proc.dat` and every contrib SQL script in the pinned tree. A name assembled at run time, or one in an out-of-tree extension, would not be found that way.
- **The 17.11 follow-up did not re-measure the rest of the page.** Its server carries new fixtures, not the 15 named fixtures, the 54-cell matrix, or the 12-through-17 fixture family, and 14.23 was not rebuilt for it. So the two servers on this page now disagree in provenance: everything above [Follow-up: five changes from a twelve-issue review](#follow-up-five-changes-from-a-twelve-issue-review) is a 17.10 observation and everything inside it is a 17.11 one. No cell was compared across the two builds.
- **The corrected statement's tuple count is right and its tuple price is not.** Change 1 counts posting tuples per key group exactly — 14,008 modelled against 14,000 built on `i_ind2` — and then charges every one of them the full `nmax`-TID size, so a group whose last posting list is nearly empty is over-charged by up to `(nmax - remainder) * 6` bytes. That is the whole of the `−99.4%` at 133 rows per key and the `−88.7%` at 143. A two-size mixture (`floor(rpg / nmax)` full tuples plus one partial) is the obvious repair, was not implemented, and cannot be closed-form on its own: the packing of mixed-size tuples into leaves needs the per-page rule `_bt_buildadd` runs, and the measured per-page utilisation across three fixtures was 6929, 7165 and 7689 bytes, so no single constant reproduces it.
- **The exact leaf-capacity rule was derived and not applied.** `floor((bs - 48 - greatest(tuple_size + 4, floor(bs * (100 - fillfactor) / 100) - tids * 6)) / tuple_size)` reproduces the measured 11 data items per leaf on `i_cd`, 9 on `i_q1000` and 366 on a plain `bigint` leaf, which would close `i_cd`'s remaining 38 blocks. It was left out because the brief was exactly five changes, and it was checked on three shapes only, all at `block_size` 8192, `fillfactor` 90 and `MAXALIGN` 8.
- **The internal-level fanout for a deduplicated index is still modelled from the leaf slot.** `int_cap` uses `slot`, and the pivots above a posting-list leaf level measured 8 to 24 bytes with 185 of 212 carrying a heap TID, which is why `i_q1000` needs five level-1 pages against the modelled four. The one-block error this leaves was not corrected, and only one index's internal levels were inspected.
- **The nondeterministic-collation conjunct was not re-measured on 17.11.** That build is `--without-icu`, so the `all_deterministic` claim in this follow-up rests on the statement text plus `btvarstrequalimage`, not on a fresh run. The measured version of it is the 17.10 ICU build recorded earlier on this page.
- **`i_rand`'s numbers come from one seed and one shape.** `setseed(0.42)` and a `generate_series ... ORDER BY random()` load produced 65.68% density and 27.1% on both columns; no second seed, no second key type, no `fillfactor` other than 90, and no concurrent-insert workload were tried. Whether a production random-insert index sits above or below a 30% threshold is therefore unquantified, and the 2.9-point margin here should not be read as a safety margin.
- **The re-randomisation experiment mutated the fixture.** The 500,000 extra inserts that produced the 5590/4622/4116-block drift table were run after the `i_rand` row of the fixture tables was captured, on the same table and in the same database, so the two sets of `i_rand` numbers describe two different states rather than a repeatable pair. The rebuilt copy that drifted to 80.21% was also the ground-truth index for the earlier state.
- **Extended statistics were exercised on plain-column keys only.** The measured cases are two-column and three-column `int` keys on ordinary tables. Partitioned parents, inheritance children, expression statistics objects, `MCV`-only or `dependencies`-only objects, objects on more than 8 columns, and the `inherited = true` row that 15 and later can produce were not run; the `max(e.nd)` choice for duplicate rows is an argument about direction, not a measurement.
- **The invisible-statistics caveat cannot prove a hidden row exists.** `pg_statistic` is unreadable to the role in question, so the statement reports that `pg_stats`'s filter would have removed a row, not that one is there. A column that genuinely has no statistics and a column whose statistics are hidden are distinguished only by privilege state plus `last_analyze`, and no case was constructed where a table was analyzed while one column legitimately had no row.
- **RLS was not exercised.** The `stats_hidden` expression tests `relrowsecurity AND row_security_active(tbloid)` because that is `pg_stats`'s own condition, but no fixture enabled row-level security, so only the column-privilege half of the caveat is measured.
- **The alert-rule counts were not re-run.** [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate) reports true and false positives at a 30% threshold over the 12-through-17 fixture family on three servers. This follow-up adds one true false positive (`i_rand`) and one recovered true positive (`i_ext50`) and one new suppression string, but the whole table was not recomputed on 17.11, so the "4 of 5, 0 false positives" line still describes the older fixture set.
- **Neither statement was re-run on 14.23, 13, 15 or 16.** The corrected statement was measured on 17.11 and 12.2 only. 14.23 was measured for the earlier follow-up with the filed statement, and change 2 would behave there as on 17.11 because `CREATE STATISTICS` and `pg_stats_ext` predate 12; that is a source-and-catalog argument, not a run.
- **The `−0.6%` on `i_q1000` depends on the sampled most-common-value list.** `ANALYZE` stored 11 MCVs for a uniform 1000-value column, which is what splits the key groups into classes and adds 6 leaves to the model. A second `ANALYZE` with a different sample, or a different `default_statistics_target`, would move that cell; `i_r1000` — the same shape built independently — read `−0.3%` in the same run, which is the size of that effect on one server.
- **The two-server timing comparison is not a benchmark.** 41.7 / 34.4 / 37.6 ms against 30.2 / 31.9 / 36.6 ms was measured on one warm database with 54 indexes, with the triage filter removed, on a machine running other work. The ranges overlap and no attempt was made to attribute the difference to a specific CTE.
- **The mandatory tests ran on their own fixture set, not on the 54-cell matrix or the twelve-issue-review fixtures.** Change 6 touches only `all_equalimage`, and the 28 fixtures show it moving exactly the six custom-opclass rows, so on any index whose key opclasses are built-in the two statements are the same expression. That is an argument from the changed subquery plus 22 unchanged rows, not a re-run of the earlier tables. `i_q1000`, `i_cd`, `i_rand`, `i_stale`, `i_dupdel`, `i_ext50` and the duplication band were not re-measured with change 6 applied, so every number above this follow-up remains a reading of the statement without it.
- **The 30%-threshold alert table above was not recomputed.** [Read the floor, not the point estimate](#read-the-floor-not-the-point-estimate) counts true and false positives on the earlier fixture set. The new fixtures add three point-estimate false positives at that threshold and none on the floor, but they were counted on this run's 28 indexes, not merged into that table's population.
- **Test 14 is answered "FALSE by design", which is a policy choice, not a proof.** A statement that could call the support function — a `security_invoker` view over a PL/pgSQL wrapper, or an extension — would answer test 14 correctly and keep test 16 safe. That was not built, because this page's constraint is core SQL and read-only catalog access. Whether calling an arbitrary opclass support function from a monitoring query is acceptable was not analyzed; the function may be volatile, may raise, and `ei_boom` shows what raising costs.
- **The whitelist is a fixed two-name list, so a future core function would need editing.** If a later major adds a third stock equal-image function, change 6 silently declines to credit every opclass that registers it. Nothing in the statement detects that; the only symptom is under-reporting. No mechanism for discovering "known-safe" support functions from the catalog alone was found.
- **The replaced-built-in case was measured for the gate but not for the reported percentage.** With `pg_catalog.btequalimage` rewritten as a SQL function returning false, change 6 answered false for `i_int4` and `i_text_det` while the engine still deduplicated, so the statement would over-predict every built-in-keyed index in that database. The gate values were captured; the resulting percentages were not, because the built-in was restored first.
- **`i_mixed_tf` and `i_mixed_ft` were only made visible through an extended-statistics object.** On `t` their over-credit is latent, because the per-column product exceeds the row count. Whether a real two-column key with genuine correlation and no `CREATE STATISTICS` object can show the same failure was not tested; the `t2` pair proves it appears as soon as change 2 supplies a group estimate.
- **The `i_multi_bad` 28.8% reading is explained but not fixed.** It reproduces identically on 12.2, which places it in the `avg_width` family of errors [Method A-prime](#method-a-prime-still-fixes-variable-key-width) documents, and 28.8% sits just under the 30% threshold used throughout this page. No fixture was built to find the width at which a two-column key with a variable-width member crosses it.
- **The 12.2 run could not construct seven of the seventeen tests.** Tests 4, 11, 12, 13, 14, 15 and 16 need a nondeterministic collation, the `deduplicate_items` reloption or a custom support function 4, and that server rejects all three. Test 17 therefore proves that the statement stays correct where deduplication cannot exist, not that it stays correct on a 13-through-16 server where deduplication exists and custom opclasses are constructible. Majors 13 through 16 were again not run.
- **The 12.2 server has no ICU.** `CREATE COLLATION ... provider = icu, deterministic = false` fails there with `ICU is not supported in this build`, so the collation half of the gate was exercised on 17.11 only. That is a build-option gap in this repository's checkout, not a statement about what PostgreSQL 12 supports.
- **The harness scores the gate, not the whole statement.** Pass and fail above mean "`dedup_applies` agrees with the engine". A test that also asserted the reported percentage would fail on `i_text_det` (7.8%), `i_multi_ok` (−320.0%) and `i_multi_bad` (28.8%) for reasons this page already files as arithmetic limits. No threshold for "close enough" was defined, and none of the three has a fix in this follow-up.
- **One seed, one block size, one fillfactor.** All 28 fixtures use 500,000 rows, exactly 100 rows per key, `block_size` 8192 and the default `fillfactor`. The 100-rows-per-key choice puts every deduplicated fixture below the 132-TID cap for an 8-byte key, so no mandatory-test fixture exercises the multi-posting-tuple path that [change 1](#change-1-round-each-key-group-up-to-whole-posting-tuples) addresses.
- **The post-build mutation was measured on one fixture in one session.** `ei_true` was rewritten, `bt_index_check` raised, and `REINDEX` produced 1376 blocks. Whether an ordinary `VACUUM`, an insert-time deduplication pass or a standby replaying the same index would behave the same way after such a catalog change was not tested, and `amcheck`'s verdict was taken as the definition of "wrong", not as a guarantee that reads of that index misbehave.
- **The recommendation ranks this page's own statements and nothing else.** [The current recommended statement](#the-current-recommended-statement) compares the six variants filed here on the fixtures filed here. No third-party bloat query, no `pgstattuple`-based estimator and no database built for a purpose other than this page was scored against it, so "most accurate" is a statement about this population, not about the space of possible bloat queries.
- **The recommended pairing itself has two servers and one fixture set behind it.** Five changes plus change 6 ran on 17.11 against the 28 mandatory-test fixtures and on 12.2 against 10 of them. Every per-fixture accuracy number the recommendation inherits — `i_q1000`, `i_cd`, `i_ext50`, `i_rand`, `i_stale`, `i_dupdel` and the duplication band — was measured with the same statement *without* change 6, for the reason the mandatory-tests open question above records, and 14.23, 13, 15 and 16 never ran the assembled text at all. The "0 fixtures above 30%" and "0 over-credits" figures are therefore scoped to the 28-fixture run.
- **"0% reclaimable" is an argument from freshly built, not a measured rebuild.** Unlike the earlier fixture families on this page, no `CREATE INDEX CONCURRENTLY` copy was taken for the 28 mandatory-test fixtures; each one was built once from static data and never modified, so live equals rebuilt by construction. The only two rebuilds actually performed are the mutation pair (`REINDEX` produced 1376 blocks for `i_ei_true` and 421 for `i_ei_none`, both matching the corresponding fresh builds exactly), which supports the construction argument without replacing it.

## Source References

- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L641-L671)
- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L783-L940)
- [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1135-L1377)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L535-L571)
- [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)
- [nbtutils.c#btoptions](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4561-L4576)
- [nbtvalidate.c#btvalidate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L40-L287)
- [nbtree.c#bthandler](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L100-L153)
- [reloptions.c#deduplicate_items](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)
- [opclasscmds.c#AlterOpFamily](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L816-L878)
- [opclasscmds.c#AlterOpFamilyAdd](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L880-L1028)
- [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)
- [nbtdedup.c#_bt_dedup_pass](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L58-L210)
- [nbtdedup.c#_bt_dedup_start_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L432-L475)
- [nbtdedup.c#_bt_dedup_save_htid](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L477-L544)
- [nbtdedup.c#_bt_form_posting](../../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L856-L920)
- [nbtutils.c#_bt_keep_natts_fast](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L4852-L4905)
- [indextuple.c#index_form_tuple_context](../../../../raw/postgres-17/src/backend/access/common/indextuple.c#L64-L200)
- [pg_collation.h#collisdeterministic](../../../../raw/postgres-17/src/include/catalog/pg_collation.h#L28-L50)
- [pg_index.h#indcollation](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L28-L60)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L672-L700)
- [guc_tables.c#default_statistics_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2070-L2079)
- [nbtsplitloc.c#_bt_findsplitloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L129-L430)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)
- [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1131-L1150)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L845-L924)
- [nbtree.c#btvacuumpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1160-L1190)
- [vacuumlazy.c#heap_vacuum_rel](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L618-L735)
- [vacuumlazy.c#lazy_vacuum](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1864-L1935)
- [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L55-L70)
- [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2789-L2929)
- [analyze.c#compute_scalar_stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2440-L2620)
- [system_views.sql#pg_stats](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L215)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L308-L343)
- [varlena.c#pg_column_size](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L5062-L5102)
- [nbtsearch.c#_bt_readnextpage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsearch.c#L2181-L2338)
- [nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L62-L240)
- [explain.c#show_buffer_usage](../../../../raw/postgres-17/src/backend/commands/explain.c#L3743-L3906)
- [indexcmds.c#DefineIndex](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L740)
- [bulk_write.c#smgr_bulk_finish](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L220)
- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L380)
- [extension.c#execute_extension_script](../../../../raw/postgres-17/src/backend/commands/extension.c#L993-L1060)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1018-L1054)
- [create_index.sgml#CONCURRENTLY](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L610-L700)
- [indexfsm.c#exported-routines](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L1-L74)
- [freespace.c#fsm-categories-and-depth](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L78)
- [freespace.c#GetPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L120-L142)
- [freespace.c#FreeSpaceMapVacuum](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L361-L394)
- [freespace.c#fsm_space_avail_to_cat](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L396-L435)
- [fsm_internals.h#FSMPageData](../../../../raw/postgres-17/src/include/storage/fsm_internals.h#L20-L61)
- [nbtpage.c#_bt_allocbuf](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L859-L987)
- [nbtpage.c#_bt_pendingfsm_init](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2940-L2982)
- [nbtpage.c#_bt_pendingfsm_add](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3057-L3114)
- [nbtree.h#BTPageSetDeleted](../../../../raw/postgres-17/src/include/access/nbtree.h#L214-L318)
- [nbtree.h#BTVacState](../../../../raw/postgres-17/src/include/access/nbtree.h#L320-L346)
- [nbtree.c#btvacuumscan](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L926-L1060)
- [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L50-L88)
- [storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L280-L439)
- [relcache.c#RelationSetNewRelfilenumber](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3772-L3974)
- [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3048-L3092)
- [dbsize.c#pg_size_pretty](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L565-L608)
- [pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L1-L50)
- [btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L101-L200)
- [queries.sgml#LIMIT](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1900-L1960)
- [queries.sgml#queries-column-labels](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1556-L1620)
- [queries.sgml#sorting-rows](../../../../raw/postgres-17/doc/src/sgml/queries.sgml#L1830-L1893)
- [queryjumblefuncs.c#CleanQuerytext-and-JumbleQuery](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L61-L160)
- [queryjumblefuncs.c#_jumbleNode](../../../../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L212-L290)
- [gen_node_support.pl#jumble-emission](../../../../raw/postgres-17/src/backend/nodes/gen_node_support.pl#L1258-L1340)
- [primnodes.h#TargetEntry-fields](../../../../raw/postgres-17/src/include/nodes/primnodes.h#L2130-L2203)
- [pgstatstatements.sgml#queryid-stability](../../../../raw/postgres-17/doc/src/sgml/pgstatstatements.sgml#L604-L660)
- [pg_stat_statements.c#pgss_store](../../../../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1270-L1420)
- [pg_stat_statements.c#generate_normalized_query](../../../../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L2780-L2900)
- [postgres.c#exec_simple_query-logging](../../../../raw/postgres-17/src/backend/tcop/postgres.c#L1060-L1090)
- [pg_config_manual.h#NAMEDATALEN-comment](../../../../raw/postgres-17/src/include/pg_config_manual.h#L16-L30)
- [scansup.c#downcase_truncate_identifier](../../../../raw/postgres-17/src/backend/parser/scansup.c#L23-L105)
- [parse_relation.c#errorMissingColumn-full](../../../../raw/postgres-17/src/backend/parser/parse_relation.c#L3658-L3751)
- [nbtsort.c#_bt_sort_dedup_finish_pending](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1029-L1057)
- [nbtsort.c#_bt_load-dedup-loop](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1280-L1356)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1075-L1130)
- [nbtsplitloc.c#_bt_findsplitloc-strategy](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L270-L340)
- [nbtsplitloc.c#_bt_afternewitemoff](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L629-L746)
- [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L60-L100)
- [nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L718-L795)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2683-L2790)
- [nbtree.h#btree-versions](../../../../raw/postgres-17/src/include/access/nbtree.h#L112-L150)
- [mvdistinct.c#statext_ndistinct_build](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L76-L141)
- [mvdistinct.c#pg_ndistinct_out](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L346-L386)
- [mvdistinct.c#estimate_ndistinct](../../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L519-L542)
- [extended_stats.c#BuildRelationExtStatistics](../../../../raw/postgres-17/src/backend/statistics/extended_stats.c#L104-L230)
- [statscmds.c#CreateStatistics](../../../../raw/postgres-17/src/backend/commands/statscmds.c#L200-L230)
- [statistics.h#STATS_MAX_DIMENSIONS](../../../../raw/postgres-17/src/include/statistics/statistics.h#L14-L30)
- [system_views.sql#pg_stats-view](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L275)
- [system_views.sql#pg_stats_ext-view](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L277-L310)
- [catalogs.sgml#pg_statistic-visibility](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L7425-L7441)
- [catalogs.sgml#pg_statistic_ext_data-visibility](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml#L7760-L7776)
- [pgstat_relation.c#pgstat_count_heap_insert](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L355-L400)
- [pgstat_relation.c#AtEOXact_PgStat-relations](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L530-L590)
- [pgstat_relation.c#pgstat_relation_flush_cb](../../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L800-L880)
- [btreefuncs.c#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L828-L939)
- [pg_proc.dat#pg_size_pretty-entries](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7500-L7507)
- [varlena.c#text_cmp](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L1583-L1888)
- [glossary.sgml#glossary-bloat](../../../../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)
- [copy.sgml#wasted-space](../../../../raw/postgres-17/doc/src/sgml/ref/copy.sgml#L613-L621)
- [pgstattuple--1.4.sql#pgstattuple](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L6-L17)
- [datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L415-L438)
- [fmgr.c#fmgr_info_cxt_security](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L142-L260)
- [fmgr.c#FunctionCall1Coll](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L1128-L1146)
- [fmgr.c#OidFunctionCall1Coll](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L1393-L1418)
- [fmgr.c#fmgr_isbuiltin](../../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L70-L95)
- [opclasscmds.c#assignProcTypes-btree](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L1244-L1334)
- [opclasscmds.c#DefineOpClass-amadjustmembers](../../../../raw/postgres-17/src/backend/commands/opclasscmds.c#L690-L710)
- [nbtvalidate.c#btvalidate-procs](../../../../raw/postgres-17/src/backend/access/nbtree/nbtvalidate.c#L83-L138)
- [amapi.c#amvalidate](../../../../raw/postgres-17/src/backend/access/index/amapi.c#L110-L143)
- [index.c#index_create-nondeterministic](../../../../raw/postgres-17/src/backend/catalog/index.c#L807-L850)
- [pg_amproc.dat#btree-equalimage-rows](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L18-L295)
- [pg_proc.dat#btvarstrequalimage](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L1055-L1057)
- [pg_proc.dat#btequalimage](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L10380-L10382)
- [pg_statistic.h#stawidth](../../../../raw/postgres-17/src/include/catalog/pg_statistic.h#L40-L50)
- [analyze.c#compute_scalar_stats-width](../../../../raw/postgres-17/src/backend/commands/analyze.c#L2530-L2545)
- [guc_tables.c#client_min_messages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4776-L4786)
- [verify_nbtree.c#bt_index_check-metapage](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L365-L405)
- [btree.sgml#equalimage-support-function](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L459-L552)
- [btree.sgml#btree-deduplication-restrictions](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L909)
- [opr_sanity.sql#btree-equalimage](../../../../raw/postgres-17/src/test/regress/sql/opr_sanity.sql#L1336-L1353)
- [opr_sanity.out#btree-equalimage](../../../../raw/postgres-17/src/test/regress/expected/opr_sanity.out#L2204-L2222)
- [alter_generic.sql#cross-type-equalimage](../../../../raw/postgres-17/src/test/regress/sql/alter_generic.sql#L444-L446)
- [alter_generic.out#cross-type-equalimage](../../../../raw/postgres-17/src/test/regress/expected/alter_generic.out#L504-L507)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](../../../v12/questions/indexing/btree-index-bloat-core-sql-only.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](create-index-concurrently.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md)
- [versions](../../../versions.md)
