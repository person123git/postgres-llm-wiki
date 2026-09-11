---
type: common-concept
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Mandatory B-Tree Bloat Tests (unverified)

## Contents

- [Definition](#definition)
- [Why It Exists](#why-it-exists)
- [How It Works](#how-it-works)
  - [The five phases of a run](#the-five-phases-of-a-run)
  - [Rule 1 splitting every recipe at the index build](#rule-1-splitting-every-recipe-at-the-index-build)
  - [Rule 2 the uniform heap-block drain](#rule-2-the-uniform-heap-block-drain)
  - [Rule 3 the simulated auto-analyze](#rule-3-the-simulated-auto-analyze)
  - [The oracle and the verdict bands](#the-oracle-and-the-verdict-bands)
  - [Family 1 the deduplication gate](#family-1-the-deduplication-gate)
  - [Family 2 partial indexes](#family-2-partial-indexes)
  - [Family 3 false-positive constructions](#family-3-false-positive-constructions)
  - [Family 4 false-negative constructions](#family-4-false-negative-constructions)
  - [Family 5 the change A to D controls](#family-5-the-change-a-to-d-controls)
  - [Family 6 the drained queue and zero row counts](#family-6-the-drained-queue-and-zero-row-counts)
  - [Catalog forgeries run last](#catalog-forgeries-run-last)
  - [Feature gates that skip a fixture](#feature-gates-that-skip-a-fixture)
  - [What the suite does not cover](#what-the-suite-does-not-cover)
- [Where It Appears in Source](#where-it-appears-in-source)
- [Related Structures and Functions](#related-structures-and-functions)
- [Interactions with Other Concepts](#interactions-with-other-concepts)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Definition

The mandatory B-tree bloat tests are this wiki's shared acceptance suite for any claim about how much space a PostgreSQL B-tree index is wasting, or whether it should be rebuilt. The suite is 121 numbered tests in six families, run on disposable fixtures against one isolated server, where the only oracle is a measured `REINDEX INDEX`: build a fixture, record its as-built state, churn it, let the method under test decide, rebuild, and compare. Each family targets one engine behavior that makes a catalog-only or page-level reading of bloat go wrong - the deduplication equal-image gate, a partial index whose population is not its table's, statistics that mislead in one direction, statistics that mislead in the other, the controls that pin a threshold, and the row counts the engine leaves at zero or stale. A method is not "tested" against a claim in this wiki until these families have been run against the exact text that claim describes, and the suite's contract is that a failing test is corrected in the method, not merely reported.

## Why It Exists

Three properties of the v17 engine make an untested bloat method plausible and wrong at the same time, and each is the reason one family exists.

A B-tree's on-disk size does not follow from its row count. Deduplication merges duplicate keys into posting lists, so two indexes over identical data can hold their entries in very different numbers of pages depending on one metapage flag and one reloption. The flag is `btm_allequalimage`, recorded once when the index is built and read back by `_bt_metaversion` ([nbtree.h#btm_allequalimage](../../../raw/postgres-17/src/include/access/nbtree.h#L113-L119), [nbtpage.c#_bt_metaversion](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L739-L750)). What it records is the verdict of `_bt_allequalimage`, which refuses `INCLUDE` indexes outright and otherwise calls each key opclass's support function 4, returning false if any opclass lacks the procedure or answers false ([nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)). The build path applies a second, independent switch: it deduplicates only when the index is equal-image, is not unique, and has `deduplicate_items` on ([nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)).

A partial index's population is not its table's row count. The executor skips an index whose predicate the new row fails ([execIndexing.c#partial-predicate-skip](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L378-L387)), and `ANALYZE` estimates the index's own population by counting the sampled rows that pass the predicate and scaling: `tupleFract = numindexrows / numrows`, then `ceil(tupleFract * totalrows)` ([analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)). A method reading the table's `reltuples` for a partial index is reading a different population, and the size of that error is set by the fixture, not by the method.

Row counts in the catalog are written by whichever command ran last, and sometimes not at all. `reltuples` is documented as "not always up-to-date; -1 means unknown" ([pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L64-L66)); a build writes an exact count through `index_update_stats` ([index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2830)); `ANALYZE` writes the scaled estimate above through `vac_update_relstats` ([analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660), [vacuum.c#vac_update_relstats](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1425)); `VACUUM` writes an exact count only when the index AM did not mark it estimated ([vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096)), and `btvacuumcleanup` marks it estimated for a cleanup-only scan ([nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894)); and a new relfilenode resets it to `-1` ([relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3945-L3953)). Every one of those writers is a fixture in family 6.

The families of deliberately misleading fixtures exist for the fourth reason: a bloat method has two ways to be wrong, and only one of them is visible without an oracle. Family 3 builds indexes that are fresh and healthy but whose statistics invite a rebuild; family 4 builds indexes that are genuinely reclaimable but whose statistics invite doing nothing. Without both, a method can be tuned until it looks right on whichever error it happens to make.

## How It Works

### The five phases of a run

Every fixture goes through the same five phases in one isolated cluster with `autovacuum` off, so no background worker moves a fixture between the phases. `autovacuum` is `PGC_SIGHUP`, so turning it off is a reload, not a restart ([guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457)).

| Phase | What happens | Why it is a separate phase |
|---|---|---|
| build | create the table, load it, `ANALYZE`, create the index that is scored | the as-built state is the only state in which the index is known to be dense |
| baseline | record size, the table's `reltuples` and the index's own `reltuples` | a before-and-after method has nothing to compare against otherwise |
| churn | run the recipe's own writes, or the uniform drain | this is the state the method is asked about |
| decide | run the method under test, unmodified | the text that is scored must be the text that is published |
| oracle | `REINDEX INDEX`, then measure the file again | the only ground truth for "how much would a rebuild give back" |

The three rules below are what turns a one-shot estimator fixture into a five-phase fixture, and they are what a consumer page has to state it followed.

### Rule 1 splitting every recipe at the index build

Each numbered recipe is cut in two. Everything up to and including `CREATE INDEX` is the build phase; everything after it is the churn phase. A recipe written for a one-shot estimator ends in a single final state and asks one question about it, which leaves a before-and-after method with no baseline to compare against. Splitting at the build also fixes what the baseline means: a fresh serial build packs leaves to the index's fillfactor, `BTGetTargetPageFreeSpace` for a leaf level and `BTREE_NONLEAF_FILLFACTOR` above it ([nbtsort.c#_bt_pagestate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L660-L666), [nbtree.h#BTGetTargetPageFreeSpace](../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)), so the baseline is a known density rather than an arbitrary one.

### Rule 2 the uniform heap-block drain

Fixtures whose recipe has no churn of its own get one: delete every heap tuple outside one heap block in ten, then `VACUUM` and `ANALYZE`. The drain selects by block number out of the tuple's `ctid`, the self item pointer whose two fields are a block id and an offset ([itemptr.h#ItemPointerData](../../../raw/postgres-17/src/include/storage/itemptr.h#L36-L40), [sysattr.h#SelfItemPointerAttributeNumber](../../../raw/postgres-17/src/include/access/sysattr.h#L21)), so the survivors are spread across the heap rather than clustered at one end, and the index loses entries from every leaf page instead of one contiguous run. The `VACUUM` is what turns those dead entries into reclaimable index pages: `btbulkdelete` scans the index and removes the entries, and `_bt_pagedel` marks emptied pages deleted ([nbtree.c#btbulkdelete](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L821-L832), [nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802-L1815)), which is the state `pgstatindex` reports as `deleted_pages` and half-dead pages as `empty_pages` ([pgstatindex.c#page-classes](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L295-L320)).

The drain is not applied to the fixtures whose whole point is that a fresh index must not be touched: tests 70 and 71, the whole of family 3, and controls 96, 97, 101 to 105, 108 to 112, 116 and 120. Draining those would destroy the test.

### Rule 3 the simulated auto-analyze

Any fixture that changes more than a tenth of a table's heap tuples must `ANALYZE` that table before the decide phase, because a server with autovacuum on would have. The rule is not hand-annotated per fixture. It applies the engine's own test: the launcher analyzes a table when `mod_since_analyze` exceeds `autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * reltuples` ([autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3066-L3076), [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095)), which at the shipped defaults of 50 and 0.1 is 50 rows plus a tenth of the estimated row count ([guc_tables.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375), [guc_tables.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914)). Both settings are `PGC_SIGHUP`, so a cluster-wide change is a reload; the per-table `autovacuum_analyze_threshold` and `autovacuum_analyze_scale_factor` reloptions take `ShareUpdateExclusiveLock` instead ([reloptions.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L243-L251), [reloptions.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L416-L425)).

The counter the census reads is `pg_stat_all_tables.n_mod_since_analyze` ([system_views.sql#n_mod_since_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L689)), which `pgstat_report_analyze` resets when the `ANALYZE` completes ([pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L330-L338)) and which later flushes of a backend's pending counts increase again ([pgstat_relation.c#mod_since_analyze](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L855-L860)).

The rule costs coverage and the cost has to be named, not hidden: every fixture whose point was a stale row count has its table analyzed by this step, so those fixtures test the method against fresh statistics instead. The stale-count shapes are preserved by the catalog forgeries below, which run after the census.

### The oracle and the verdict bands

The oracle is a measured `REINDEX INDEX` on the churned fixture, and `actual_pct` is the fraction of the churned file the rebuild gave back. `reindex_index` builds a fresh index into a new relfilenode and refreshes the catalog counts as any build does ([index.c#reindex_index](../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3600), [indexcmds.c#ReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2815)), which is why the baseline written after a rebuild is an exact count and not an estimate ([index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2830)).

Four bands score each fixture, and the two false-positive bands are separate because their costs differ by an order of magnitude:

| Verdict | Rule |
|---|---|
| `CRITICAL FALSE POSITIVE` | rebuilt, and the rebuild gave back less than 10 % |
| `FALSE POSITIVE` | rebuilt, and the rebuild gave back less than 35 % |
| `FALSE NEGATIVE` | not rebuilt, and a rebuild would have given back 50 % or more |
| `PASS` | everything else |

Two further columns are mandatory for a run to be readable: `expected_stage`, the gate arithmetic recomputed independently from the recorded baseline, so a disagreement between the method and the arithmetic is visible; and `want_stage`, a per-fixture prediction filed before the run, which may not be rewritten afterwards.

### Family 1 the deduplication gate

Tests 1 to 17, built on two 500,000-row tables whose deduplication key columns each carry 5,000 distinct values, 100 rows per key, beside one strictly increasing column for the unique control. The family asks one question in seventeen ways: can this index deduplicate, and does the method under test know it. `_bt_allequalimage` logs its verdict at `DEBUG1`, which makes the engine its own oracle for this family ([nbtutils.c#_bt_allequalimage-debug](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180)).

| Test | Fixture index | What it builds | What it must prove |
|---|---|---|---|
| 1, 2 | `i_int4`, `i_int8` | integer keys, 100 rows per key | the integer opclasses register `btequalimage`, which returns true unconditionally ([datum.c#btequalimage](../../../raw/postgres-17/src/backend/utils/adt/datum.c#L432-L438)) |
| 3 | `i_text_det`, `i_text_icu_det` | `text` key under the default collation, and under a deterministic ICU collation | `btvarstrequalimage` returns true for C, the default collation, or any deterministic collation ([varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)) |
| 4 | `i_text_nondet` | `text` key under a nondeterministic ICU collation | the same function returns false, so the index is not equal-image; `deterministic = false` is stored in `pg_collation` ([pg_collation.h#collisdeterministic](../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40), [collationcmds.c#deterministic](../../../raw/postgres-17/src/backend/commands/collationcmds.c#L200-L207)) |
| 5, 6 | `i_numeric`, `i_float4`, `i_float8` | `numeric` and float keys | an opclass with no `BTEQUALIMAGE_PROC` is assumed unsafe ([nbtutils.c#allequalimage-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5149-L5170), [nbtree.h#BTEQUALIMAGE_PROC](../../../raw/postgres-17/src/include/access/nbtree.h#L706-L712)) |
| 7 | `i_multi_ok`, `i2_ok` | two equal-image key columns | every key column is tested, not only the first |
| 8 | `i_multi_bad` | one equal-image and one non-equal-image column | one failing column breaks the loop and condemns the index |
| 9 | `i_expr_num`, `i_expr_lower_ci` | expression keys, numeric and a `lower()` under a nondeterministic collation | the expression's result type and collation decide, not the base column's |
| 10 | `i_inc` | an `INCLUDE` column | `INCLUDE` indexes are refused before any opclass lookup ([nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5143-L5147), [pg_index.h#indnkeyatts](../../../raw/postgres-17/src/include/catalog/pg_index.h#L33-L36)) |
| 11 | `i_dupoff`, `i_text_off`, `i2_off` | equal-image keys with `deduplicate_items = off` | the reloption is a second switch independent of the metapage flag ([reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtree.h#BTGetDeduplicateItems](../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)) |
| 12 | `i_ei_none` | a custom operator class declaring no `FUNCTION 4` | a missing procedure is unsafe, distinct from one that answers false |
| 13 | `i_ei_false` | a custom `FUNCTION 4` returning false | the callback is really called, not merely looked up |
| 14 | `i_ei_true`, `i_ei_alias` | a custom `FUNCTION 4` returning true, and a `LANGUAGE internal` alias of `btequalimage` | an internal function is resolved by `prosrc`, so an alias under another name still resolves to the builtin ([fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)) |
| 15 | `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft` | true/false and false/true callbacks in both column orders | the verdict does not depend on which column fails |
| 16 | `i_squat`, `i_text_det2` | a SQL impostor named `btequalimage`, and a renamed internal alias of `btvarstrequalimage` | the name is not the identity; `prolang` and `prosrc` are |
| 17 | `i_uniq` | a unique index over equal-image keys | a serial build never deduplicates a unique index ([nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)) |

What the family does *not* pin is the page-level merge itself: `_bt_dedup_pass` runs on insert to avoid a page split, so a deduplicating index's density is a function of its write history as well as its keys ([nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)).

### Family 2 partial indexes

Tests 18 to 77, the largest family. Every fixture is a partial index whose subset differs from its table in one named way, which is exactly the axis on which a catalog-only method has to guess. The predicate lives in `pg_index.indpred` ([pg_index.h#indpred](../../../raw/postgres-17/src/include/catalog/pg_index.h#L58-L62)), and `ANALYZE` refines the index's own row estimate only for an index that has statistics columns or a predicate ([analyze.c#partial-index-skip](../../../raw/postgres-17/src/backend/commands/analyze.c#L859-L863), [analyze.c#tupleFract-default](../../../raw/postgres-17/src/backend/commands/analyze.c#L445-L450)).

| Tests | Fixtures | What they build | What they must prove |
|---|---|---|---|
| 18-21 | `p18`-`p21` | one 1,000,000-row table, four predicates selecting roughly 20 %, 1 %, 10 % and 80 % | selectivity alone moves the subset's population from about a hundredth of the table to four fifths of it while the table's `reltuples` never changes |
| 22-26 | `p22`-`p26` | duplication, uniqueness, `n_distinct` and most-common-value distributions that differ inside the subset | per-column statistics describe the table, not the subset, so key-modelling assumptions break here |
| 27-29 | `p27`-`p29` | NULL-heavy subset, NULL-free subset of a NULL-heavy table, and an all-NULL partial index under `WHERE s IS NULL` | NULLs are indexed entries in a B-tree, and the table's null fraction does not describe the subset's |
| 30-33 | `p30`-`p33` | subset values wider than the rest, narrower than the rest, extreme mismatch, and variable widths with the same range inside and out | the average width `ANALYZE` records is the table's, so entry size is a guess |
| 34-38 | `p34`-`p38` | a deduplication-heavy subset, a unique subset of a duplicate-heavy table, one key group of 100,000 TIDs, an all-NULL key subset, and `deduplicate_items = off` | posting lists change the file size for an unchanged entry count ([nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)) |
| 39 | `p39` | a partial `UNIQUE` index | uniqueness suppresses build-time deduplication even where the keys are equal-image |
| 40-45 | `p40`-`p45` | two-column keys correlated only inside the subset, independent only inside the subset, duplicated, unique, with and without a `CREATE STATISTICS (ndistinct)` object, and one whose extended statistics are wrong for the subset | extended statistics are collected for the table, and an `ndistinct` object does not know about the predicate ([mvdistinct.c#statext_ndistinct_build](../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L86-L100), [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../../../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L84)) |
| 46-47 | `p46`, `p47` | partial indexes with `INCLUDE` columns, one with wide payloads inside the subset | non-key columns occupy leaf space and are counted by no key statistic |
| 48-50 | `p48`-`p50b` | partial expression indexes, each in two orders: `ANALYZE` before the build, and `ANALYZE` after it | an expression index has no statistics row until an `ANALYZE` runs with the index in place |
| 51-52 | `p51`, `p52` | the same subset under a deterministic and a nondeterministic ICU collation | the collation decides deduplication safety for a partial index exactly as it does in family 1 |
| 53-55 | `p53`-`p55` | the same subset at fillfactor 90, 100 and 70 | target leaf free space is `BLCKSZ * (100 - fillfactor) / 100`, so the as-built density is a parameter ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-17/src/include/access/nbtree.h#L197-L202)) |
| 56-63 | `p56`-`p63` | boolean, equality, range, `IS NULL`, `IS NOT NULL`, multi-column, and correlated and anti-correlated predicates | predicate shape changes which rows enter the index, and correlation changes where they land in key order |
| 64-69 | `p64`-`p69` | inserts into the subset, deletes with no `VACUUM`, rows entering the predicate, rows leaving it, heavy churn with `VACUUM` and `ANALYZE`, and `VACUUM` with no `ANALYZE` | the four writers of a row count are separated here; a delete with no `VACUUM` leaves entries physically present |
| 70-71 | `p70`, `p71` | a freshly created partial index, and one freshly rebuilt behind the method's back | neither may be touched; 71 also checks that a rebuild nobody recorded does not become a false positive |
| 72-75 | `p72`-`p75` | a quarter, a half, three quarters and nine tenths of the subset deleted, each with `VACUUM` and `ANALYZE` | this is the calibration ladder: four known reclaimable fractions with a measured rebuild behind each |
| 76 | `p76` | rows updated so the indexed key changes while the predicate still holds | the file grows while the entry count barely moves, which no row-count test can see |
| 77 | `p77` | a contiguous 95 % of the subset deleted, then `VACUUM` | produces empty and deleted pages rather than sparse leaves ([nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802-L1815)) |

### Family 3 false-positive constructions

Tests 78 to 85: eight freshly built indexes whose statistics invite a rebuild and whose rebuild would return nothing. None of them is churned, and none of them may be rebuilt. Five differ from their table in a way conditioned on the predicate, one has no statistics row for its expressions, one forges a catalog count, and one carries table statistics that predate a widening `UPDATE`.

| Test | Fixture | The trap |
|---|---|---|
| 78 | `f78` | subset values far wider than the table average |
| 79 | `f79` | the subset is the only non-NULL part of the column |
| 80 | `f80` | the subset is unique while the table is duplicate-heavy |
| 81 | `f81` | the table's most common value does not occur in the subset |
| 82 | `f82` | two key columns correlated in the table and not in the subset, with an `ndistinct` object present |
| 83 | `f83` | a two-expression index with no statistics row for either expression |
| 84 | `f84` | the index's own `reltuples` is forged low, so a method reading it sees a shrunken population |
| 85 | `f85` | the table's statistics predate an `UPDATE` that widened every subset row |

### Family 4 false-negative constructions

Tests 86 to 91: six indexes that really are reclaimable, each `VACUUM`ed and `ANALYZE`d after the churn so that nothing about them is stale. All six must be rebuilt. They exist because the cheapest way to score zero false positives is to rebuild nothing, and this family makes that strategy fail.

| Test | Fixture | Why a model under-reads it |
|---|---|---|
| 86 | `f86` | duplicates concentrated inside the subset, so the real entry count per key is far above the table's |
| 87 | `f87` | NULLs concentrated inside the subset |
| 88 | `f88` | the subset is much narrower than the table's average width |
| 89 | `f89` | two key columns perfectly correlated only inside the subset |
| 90 | `f90` | real deduplication compresses more than a per-key model predicts |
| 91 | `f91` | many deleted pages *and* an over-predicting width model in the same fixture |

### Family 5 the change A to D controls

Tests 92 to 112, the controls that pin thresholds and isolate one input at a time.

| Tests | Fixtures | What they hold fixed |
|---|---|---|
| 92-95 | `b92`-`b95` | a reclaimable partial index plus a known number of row changes, twice at cluster defaults and twice with per-table `autovacuum_analyze_threshold` and `autovacuum_analyze_scale_factor` reloptions set to 100/0 and 200000/1 ([reloptions.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L243-L251)) |
| 96-99 | `np96`-`np99` | non-partial controls: a plain index with fresh statistics, an expression index with no statistics row, a plain index after 300,000 inserts and no `ANALYZE`, and a duplicate-heavy plain index that is genuinely reclaimable |
| 100-105 | `i100`-`i105` | width controls: partial and non-partial `INCLUDE` payloads, payloads wider inside the subset, narrower inside the subset, mixed non-key widths, and one partial index whose *key* column is wide and unique inside the subset |
| 106-112 | `x106`-`x112` | expression-statistics controls: no statistics row with and without a following `ANALYZE`, a never-analysed table, a key column at `SET STATISTICS 0` ([analyze.c#attstattarget-zero](../../../raw/postgres-17/src/backend/commands/analyze.c#L1024-L1032), [pg_attribute.h#attstattarget](../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L166-L177)), a mixed column-and-expression key, a narrow expression, and a partial expression index |

### Family 6 the drained queue and zero row counts

Tests 113 to 121: the shapes where a row count is zero, unknown, or written by the wrong command. Each of 113 to 120 is one table of a million rows, 115 reaching that size only during the churn.

| Test | Fixture | State it reaches |
|---|---|---|
| 113a-c | `p113a`, `p113b`, `p113c` | a queue drained to empty in three states: nothing run, `VACUUM` plus `ANALYZE`, and `ANALYZE` only |
| 114 | `p114` | drained to 1 %, then `VACUUM` and `ANALYZE` |
| 115 | `p115` | the index built on an analysed empty table, then a million rows loaded |
| 116 | `p116` | the subset empty from the start and measured empty |
| 117 | `p117` | drained, then `VACUUM` with no `ANALYZE` |
| 118 | `p118` | the subset measured empty, then 50,000 rows arrive with no `ANALYZE` |
| 119 | `p119` | fixture 118 after one `ANALYZE` |
| 120 | `p120` | a 2,000-row subset missed by a sample taken at `default_statistics_target = 1`, which sets the sample to `300 * attstattarget` rows ([analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895), [guc_tables.c#default_statistics_target](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2069-L2078)) |
| 121 | `nz_k`, `nzb_k`, `i_trunc` | three ways to leave a stale count: emptied, vacuumed and reloaded with no `ANALYZE`; the same plus a `REINDEX` while the table is empty; and `TRUNCATE` then reload, which assigns a new relfilenode and writes `reltuples = -1` ([tablecmds.c#truncate-relfilenumber](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2160-L2172), [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3945-L3953)) |

### Catalog forgeries run last

Fixture 84's forged index `reltuples`, and any forged `-1` counts or forged `indisvalid`, are applied *after* the rule 3 census, not during the churn. An `ANALYZE` of the table rewrites `reltuples` for the table and for every index on it ([analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660)), so a forgery written before the census is silently repaired and the fixture stops testing anything. Ordering the forgeries last is what keeps the stale-count shapes alive once rule 3 is in force.

### Feature gates that skip a fixture

A fixture that needs a feature the server under test does not have is recorded as skipped, never silently dropped and never rewritten. Three gates matter for a 12-through-17 claim: B-tree support function 4 and the `btequalimage`/`btvarstrequalimage` builtins, which tests 13 to 16 need; the `deduplicate_items` reloption, which tests 11 and 38 need ([reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)); and ICU collations, which tests 3, 4, 9, 51 and 52 need and which a build configured without ICU cannot create. A run reports how many fixtures were skipped and why, beside the fixtures it scored.

### What the suite does not cover

Named limits, so a consumer page does not claim more than the suite gives:

- **Dead-but-not-vacuumed entries are invisible to a density reading.** A fixture that deletes or moves rows out of an index with no following `VACUUM` leaves the entries physically present, so leaf pages stay dense while a rebuild would empty them. `VACUUM` itself can decline to remove them: index vacuuming is bypassed when fewer than 2 % of heap pages hold dead items ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L92), [vacuumlazy.c#bypass](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1925-L1940)).
- **One block size and one platform.** Every geometry constant derives from `BLCKSZ`; the suite fixes no second block size.
- **No concurrency.** Every phase runs alone. Two sessions racing on one index, or a rebuild against a concurrent `DROP INDEX`, is not a fixture.
- **No partitioned table end to end.** Leaf indexes can be scored, but no fixture drains a partition and then asks about the parent.
- **The engine's own regression suites validate the build, not the method.** `make check` and the contrib checks exercise PostgreSQL; nothing in them reads a bloat estimate ([regress.sgml#make-check](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59), [regress.sgml#contrib-suites](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195)).

## Where It Appears in Source

The suite has no code in the PostgreSQL tree. What it has in the tree is the behavior each family targets, and these are the files a reviewer reads to judge a fixture:

| Area | Files | What the suite takes from it |
|---|---|---|
| Deduplication safety | `src/backend/access/nbtree/nbtutils.c`, `src/include/access/nbtree.h` | `_bt_allequalimage`, the `INCLUDE` refusal, `BTEQUALIMAGE_PROC`, the `DEBUG1` oracle ([nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)) |
| Equal-image callbacks | `src/backend/utils/adt/datum.c`, `src/backend/utils/adt/varlena.c` | `btequalimage`, `btvarstrequalimage` and its collation test ([varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)) |
| Build geometry | `src/backend/access/nbtree/nbtsort.c` | the fillfactor targets and the build-time deduplication gate ([nbtsort.c#_bt_pagestate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L660-L666)) |
| Page reclamation | `src/backend/access/nbtree/nbtpage.c`, `src/backend/access/nbtree/nbtree.c` | `_bt_pagedel`, `btbulkdelete`, `btvacuumscan` and the cleanup-only estimate ([nbtree.c#btvacuumscan](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L939-L950)) |
| Row counts | `src/backend/catalog/index.c`, `src/backend/commands/analyze.c`, `src/backend/commands/vacuum.c`, `src/backend/access/heap/vacuumlazy.c`, `src/backend/utils/cache/relcache.c` | who writes `reltuples`, when it is exact, when it is sampled, and when it becomes `-1` |
| Auto-analyze | `src/backend/postmaster/autovacuum.c`, `src/backend/utils/misc/guc_tables.c`, `src/backend/access/common/reloptions.c` | the threshold rule 3 applies, and its GUC and reloption forms |
| Statistics reporting | `src/backend/catalog/system_views.sql`, `src/backend/utils/activity/pgstat_relation.c` | `n_mod_since_analyze` and its reset |
| Measurement | `contrib/pgstattuple/pgstatindex.c` | the page classes and density a physical reading reports ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L252)) |
| Write path | `src/backend/executor/execIndexing.c` | the partial-index predicate skip that makes a subset a subset |

## Related Structures and Functions

| Symbol | Role in the suite |
|---|---|
| [nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183) | the verdict family 1 is built around, and the `DEBUG1` line that makes the engine its own oracle |
| [nbtree.h#BTEQUALIMAGE_PROC](../../../raw/postgres-17/src/include/access/nbtree.h#L706-L712) | support function 4, the slot tests 12 to 16 fill with custom procedures |
| [nbtree.h#BTGetDeduplicateItems](../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150) | the reloption switch tests 11 and 38 turn off |
| [nbtree.h#BTGetTargetPageFreeSpace](../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145) | the leaf fill target behind the fillfactor controls 53 to 55 |
| [nbtree.h#P_ISDELETED](../../../raw/postgres-17/src/include/access/nbtree.h#L218-L225) | the page flags the drain and test 77 produce |
| [pg_index.h#indpred](../../../raw/postgres-17/src/include/catalog/pg_index.h#L58-L62) | the stored predicate that defines every family 2 fixture |
| [pg_index.h#indnkeyatts](../../../raw/postgres-17/src/include/catalog/pg_index.h#L33-L36) | the key-versus-`INCLUDE` split behind tests 10, 46, 47 and 100 to 105 |
| [pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L64-L66) | the count families 5 and 6 make stale, zero and unknown |
| [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953) | how a partial index's own population is estimated |
| [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3066-L3076) | the exact arithmetic rule 3 applies |
| [index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2830) | the exact count a build and the oracle rebuild leave behind |
| [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L252) | the physical reading, with its B-tree, other-session temp and invalid-index refusals |

## Interactions with Other Concepts

- **Deduplication.** Family 1 decides only whether an index *may* deduplicate. How much a page actually saves is a property of the insert-time pass and the write history ([nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)), which the suite treats as an outcome to measure, not an input to model.
- **VACUUM.** The suite depends on `VACUUM` to convert dead entries into reclaimable pages, and deliberately withholds it in the four fixtures where entries leave an index and nothing cleans up after them: 65, 67, 113a and 113c. A separate set - 69, 106, 117 and 121 - runs the `VACUUM` and withholds the `ANALYZE` instead. The boundary is that the suite never asserts *when* autovacuum would have run a `VACUUM`; rule 3 simulates the analyze side only.
- **ANALYZE and the planner's statistics.** Rule 3 and families 2, 5 and 6 lean on what `ANALYZE` writes. The suite does not check plan shape or selectivity; a claim about how the planner prices a bloated index is a different question and a different page.
- **REINDEX.** The oracle is the blocking `REINDEX INDEX` form. Comment survival, lock levels and the concurrent form belong to the method under test, not to this suite.
- **The measurement interface.** Whether a method reads `pgstatindex`, `bt_metap`, catalog columns only, or raw pages is out of scope here; the suite scores the decision, not the instrument. A method that cannot run on the fixture set says so and records the fixtures it skipped.

## Context Reviewed

- The nbtree deduplication path: `_bt_allequalimage` and its `INCLUDE` refusal, the per-key-column `BTEQUALIMAGE_PROC` lookup and call, the `DEBUG1` verdict, the build-time gate in `_bt_load`, and the insert-time `_bt_dedup_pass`.
- The two shipped equal-image callbacks and the collation determinism they read, plus `CREATE COLLATION`'s `deterministic` option and its catalog column.
- Internal-function resolution by `prosrc`, which decides tests 14 and 16.
- The B-tree build geometry: `_bt_pagestate`'s leaf and non-leaf fill targets, `BTGetFillFactor`, `BTGetTargetPageFreeSpace` and the fillfactor constants.
- Page reclamation: `btbulkdelete`, `btvacuumscan`, `_bt_pagedel`, the deleted and half-dead page flags, and how `pgstatindex` classifies them.
- The four writers of `reltuples`: a build, `ANALYZE` through `tupleFract`, `VACUUM` through `update_relstats_all_indexes` unless the AM marked the count estimated, and `RelationSetNewRelfilenumber`'s `-1`; plus `TRUNCATE`'s new relfilenode.
- The autovacuum analyze threshold, its two GUCs and their two per-table reloptions, `n_mod_since_analyze` in `pg_stat_all_tables`, and the `pgstat_report_analyze` reset.
- Sampling: `std_typanalyze`'s `300 * attstattarget`, `default_statistics_target`, and the `attstattarget = 0` skip.
- Extended `ndistinct` statistics and their catalog kind.
- `pgstatindex`'s entry checks, page classes, size arithmetic and density output.
- The executor's partial-index predicate skip, and `pg_index`'s `indpred`, `indnkeyatts` and `indisvalid`.
- The documentation of `make check` and the contrib test suites.

## Evidence Map

| Claim | Evidence |
|---|---|
| Deduplication safety is per index, refuses `INCLUDE`, and calls support function 4 per key column | [nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183) |
| A build deduplicates only when equal-image, not unique, and `deduplicate_items` is on | [nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152) |
| A nondeterministic collation makes a character key non-equal-image | [varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613) |
| An internal alias resolves by `prosrc`, so a rename still reaches the builtin | [fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240) |
| A partial index's population is estimated as `ceil(tupleFract * totalrows)` | [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953) |
| The executor skips an index whose predicate the row fails | [execIndexing.c#partial-predicate-skip](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L378-L387) |
| Rule 3's threshold is the engine's own analyze test | [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3066-L3076), [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095) |
| The defaults behind rule 3 are 50 and 0.1, both reloadable | [guc_tables.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375), [guc_tables.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914) |
| An `ANALYZE` rewrites index `reltuples`, which is why forgeries run last | [analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660) |
| `VACUUM` writes an exact index count unless the AM marked it estimated | [vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096), [nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894) |
| A new relfilenode leaves `reltuples = -1` | [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3945-L3953) |
| Index vacuuming is bypassed under 2 % of heap pages | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L92) |
| The drain's `ctid` block number is the item pointer's block id | [itemptr.h#ItemPointerData](../../../raw/postgres-17/src/include/storage/itemptr.h#L36-L40) |
| Test 120's sample size is `300 * attstattarget` | [analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895) |
| The engine's regression suites do not exercise a bloat method | [regress.sgml#make-check](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59) |

## Open Questions

- **The unvacuumed blind spot has no fixture that closes it.** Tests 65, 67, 113a and 113c leave their index entries dead but physically present, so a density reading sees a full file while a rebuild would write an almost empty one. The suite builds that state and scores what a method does with it, but pins no second input - an index entry count against leaf capacity, or `n_dead_tup` - that could catch the shape.
- **Only two of the six majors a 12-through-17 claim covers have fixture legs.** The 13, 14, 15 and 16 legs exist only as feature gates, not as runs, so a behavior that changed in one of them would not be caught.
- **No second block size.** Every geometry expectation follows from `BLCKSZ`, and the suite has no `--with-blocksize=16` leg to prove that a method's arithmetic is not tuned to 8192.
- **The drain is a shape, not a distribution.** One heap block in ten is kept, which produces a specific spread of surviving entries; nothing in the suite varies that ratio or its clustering, so a method sensitive to the *pattern* of deletion rather than its volume would pass unexamined.
- **`want_stage` predictions are not derivable.** A prediction filed before the run is a judgement, and two reviewers can file different ones for the same fixture. The suite records disagreement between prediction and measurement but has no rule for which is at fault.
- **Locale and encoding are fixed.** The ICU fixtures vary a collation on one column; no fixture varies the cluster's locale or encoding, so a character-key method is tested under one collation environment.

## Source References

- [nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)
- [nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5143-L5147)
- [nbtutils.c#allequalimage-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5149-L5170)
- [nbtutils.c#_bt_allequalimage-debug](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180)
- [nbtree.h#btm_allequalimage](../../../raw/postgres-17/src/include/access/nbtree.h#L113-L119)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-17/src/include/access/nbtree.h#L197-L202)
- [nbtree.h#P_ISDELETED](../../../raw/postgres-17/src/include/access/nbtree.h#L218-L225)
- [nbtree.h#BTEQUALIMAGE_PROC](../../../raw/postgres-17/src/include/access/nbtree.h#L706-L712)
- [nbtree.h#BTGetTargetPageFreeSpace](../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)
- [nbtree.h#BTGetDeduplicateItems](../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)
- [nbtsort.c#_bt_pagestate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L660-L666)
- [nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)
- [nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)
- [nbtpage.c#_bt_metaversion](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L739-L750)
- [nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802-L1815)
- [nbtree.c#btbulkdelete](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L821-L832)
- [nbtree.c#btvacuumscan](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L939-L950)
- [nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894)
- [datum.c#btequalimage](../../../raw/postgres-17/src/backend/utils/adt/datum.c#L432-L438)
- [varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)
- [fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)
- [pg_collation.h#collisdeterministic](../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40)
- [collationcmds.c#deterministic](../../../raw/postgres-17/src/backend/commands/collationcmds.c#L200-L207)
- [pg_index.h#indnkeyatts](../../../raw/postgres-17/src/include/catalog/pg_index.h#L33-L36)
- [pg_index.h#indpred](../../../raw/postgres-17/src/include/catalog/pg_index.h#L58-L62)
- [pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L64-L66)
- [pg_attribute.h#attstattarget](../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L166-L177)
- [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../../../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L84)
- [mvdistinct.c#statext_ndistinct_build](../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L86-L100)
- [execIndexing.c#partial-predicate-skip](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L378-L387)
- [analyze.c#tupleFract-default](../../../raw/postgres-17/src/backend/commands/analyze.c#L445-L450)
- [analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660)
- [analyze.c#partial-index-skip](../../../raw/postgres-17/src/backend/commands/analyze.c#L859-L863)
- [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)
- [analyze.c#attstattarget-zero](../../../raw/postgres-17/src/backend/commands/analyze.c#L1024-L1032)
- [analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895)
- [index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2830)
- [index.c#reindex_index](../../../raw/postgres-17/src/backend/catalog/index.c#L3583-L3600)
- [indexcmds.c#ReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2804-L2815)
- [vacuum.c#vac_update_relstats](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1425)
- [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L92)
- [vacuumlazy.c#bypass](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1925-L1940)
- [vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096)
- [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3945-L3953)
- [tablecmds.c#truncate-relfilenumber](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L2160-L2172)
- [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3066-L3076)
- [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095)
- [guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457)
- [guc_tables.c#default_statistics_target](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2069-L2078)
- [guc_tables.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375)
- [guc_tables.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914)
- [reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)
- [reloptions.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L243-L251)
- [reloptions.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L416-L425)
- [system_views.sql#n_mod_since_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L689)
- [pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L330-L338)
- [pgstat_relation.c#mod_since_analyze](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L855-L860)
- [itemptr.h#ItemPointerData](../../../raw/postgres-17/src/include/storage/itemptr.h#L36-L40)
- [sysattr.h#SelfItemPointerAttributeNumber](../../../raw/postgres-17/src/include/access/sysattr.h#L21)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L252)
- [pgstatindex.c#page-classes](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L295-L320)
- [regress.sgml#make-check](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59)
- [regress.sgml#contrib-suites](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195)

## Navigation

- [v17/index](../index.md)
- [wiki index](../../index.md)
- [versions](../../versions.md)
