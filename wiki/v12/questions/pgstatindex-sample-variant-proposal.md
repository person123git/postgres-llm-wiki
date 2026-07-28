---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Recommendation](#recommendation)
  - [Current `pgstatindex` baseline](#current-pgstatindex-baseline)
  - [Proposed API and field contract](#proposed-api-and-field-contract)
  - [C call path and data structures](#c-call-path-and-data-structures)
  - [Sampling strategy and estimator math](#sampling-strategy-and-estimator-math)
  - [Locking, concurrency, and integrity](#locking-concurrency-and-integrity)
  - [Extension, build, and generated-file implications](#extension-build-and-generated-file-implications)
- [Pros](#pros)
- [Cons](#cons)
- [SQL Prototype Using `contrib/pageinspect`](#sql-prototype-using-contribpageinspect)
  - [Prototype boundaries](#prototype-boundaries)
  - [Setup](#setup)
  - [Compatibility helper and sampler](#compatibility-helper-and-sampler)
- [Tests To Add](#tests-to-add)
- [Executed Comparison with Standard `pgstatindex`](#executed-comparison-with-standard-pgstatindex)
  - [Test scope and metrics](#test-scope-and-metrics)
  - [Disposable fixture](#disposable-fixture)
  - [Seeded comparison harness](#seeded-comparison-harness)
  - [Full-scan baseline](#full-scan-baseline)
  - [Sampling results after the 10% floor](#sampling-results-after-the-10-floor)
  - [Effect on the small partial index](#effect-on-the-small-partial-index)
  - [What the comparison establishes](#what-the-comparison-establishes)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, propose a `pgstatindex` variant that samples the index
instead of reading the whole index, include a pros and cons section, and
provide a proposal for a SQL implementation using available extensions in
contrib.

Follow-up (2026-07-17): Add a section with tests comparing the proposed sampling pgstatindex variant with the standard pgstatindex across multiple types of bloated indexes, empty indexes, partial indexes, and indexes under 50 MB.

(The follow-up wording was corrected for grammar and clarity at the user's request; meaning preserved.)

Follow-up (2026-07-17): Add an exception for indexes under 100 MB so the effective sample ratio has a minimum of 10%. Rerun the tests and explain how this changes the results for small partial indexes.

(The follow-up wording was corrected at the user's request. The user chose to interpret `100 MB` as 100 MiB, or 104,857,600 bytes; “under” is a strict comparison.)

## Answer

### Recommendation

Add a new `pgstattuple` function named `pgstatindex_approx`. Do not change the
existing `pgstatindex` contract. Capture the B-tree main-fork length once, then
draw a simple random sample without replacement from physical blocks `1`
through `nblocks - 1`. When the captured main-fork size is strictly below
100 MiB, raise a requested fraction below 10% to an effective 10%; at exactly
100 MiB or above, preserve the requested fraction. An empty metapage-only index
still samples zero ordinary pages. Keep metadata and sample diagnostics as
direct observations, report the realized fraction through `scanned_percent`,
and prefix every extrapolated metric with `approx_`.

PostgreSQL 12 already has a suitable first implementation building block:
`BlockSampler`. It implements Knuth's Algorithm S, selects a requested number
of blocks from a known population, and returns selected block numbers in
ascending order. The C function can initialize it over `nblocks - 1` candidates
and add one to each returned block number to skip the metapage
([sampling.c#BlockSampler_Init](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L51),
[sampling.c#BlockSampler_Next](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L53-L112)).

Use `BlockSampler` as a starting point, not as an unqualified final choice. Its
sample count is an `int`, and its skip loop can advance through many candidate
block numbers before returning a sampled page. Those limits need explicit API,
cancellation, and very-large-index decisions
([sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43),
[sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111),
[block.h#BlockNumber](../../../raw/postgres-12/src/include/storage/block.h#L17-L35)).

The SQL implementation below is a diagnostic prototype. It proves that the v12
`pageinspect` interfaces can reproduce the estimator, but it is not a substitute
for the C function: it is superuser-only, enumerates and random-sorts every
candidate block number, does not hold one relation lock across the run, and uses
normal buffer access instead of `pgstatindex`'s bulk-read ring
([pageinspect.sgml#overview](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14),
[btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L213),
[pgstatindex.c#buffer-strategy](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L223)).

### Current `pgstatindex` baseline

The v1.5 SQL overloads bind `pgstatindex(text)` and `pgstatindex(regclass)` to
separate wrappers. Both wrappers open the relation with `AccessShareLock` and
call the shared `pgstatindex_impl`. SQL privileges replace the inline superuser
check for these v1.5 entry points
([pgstattuple--1.4--1.5.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstatindex.c#wrappers](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L213)).

`pgstatindex_impl` then:

1. accepts only a physical `RELKIND_INDEX` whose access method is
   `BTREE_AM_OID`, and rejects another session's temporary relation
   ([pgstatindex.c#guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L238));
2. reads block 0 and copies `btm_version`, `btm_level`, and `btm_root` from
   `BTMetaPageData`
   ([pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
   [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110));
3. captures `RelationGetNumberOfBlocks(rel)` once, then visits every captured
   non-metapage block with `CHECK_FOR_INTERRUPTS`, `BAS_BULKREAD`, and a shared
   buffer content lock
   ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L255-L315),
   [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2810));
4. classifies each block as deleted, half-dead (`empty_pages`), live leaf, or
   internal from `BTPageOpaqueData` flags
   ([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L283-L310),
   [nbtree.h#page-state-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L181-L196)); and
5. closes the relation, builds ten C strings, converts them through the caller's
   composite tuple descriptor, and returns the record
   ([pgstatindex.c#result-construction](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L317-L365)).

`BTIndexStat` holds the three metapage values, four page-class counters, summed
leaf capacity and free space, and the backward-link count
([pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95)).
The existing formulas are:

```text
index_size = (1 + internal_pages + leaf_pages
                + empty_pages + deleted_pages) * BLCKSZ

avg_leaf_density = 100
                   - sum(PageGetFreeSpace(page))
                     / sum(pd_special - SizeOfPageHeaderData) * 100

leaf_fragmentation = count(live leaf with btpo_next != P_NONE
                            and btpo_next < current block)
                     / leaf_pages * 100
```

The code formats both percentages to two decimals and emits `NaN` when the
relevant leaf denominator is zero
([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356),
[bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)).
The checked-in empty-index result has only the metapage, zero page counters, and
two `NaN` values
([pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

Do not call the existing outputs transactionally exact. The documentation says
the function accumulates page-by-page rather than taking an instantaneous
whole-index snapshot. It also reads the metapage without a content lock, captures
relation length once, and locks ordinary pages one at a time
([pgstattuple.sgml#snapshot-limit](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279),
[pgstatindex.c#metapage-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315)).
For the proposal, “direct” means read without sampling extrapolation. It does not
mean that all direct fields describe one common instant.

### Proposed API and field contract

The following output contract adds two direct sample-size fields. They make a
small or leaf-free sample visible without requiring the caller to reverse the
percentage calculation.

| Output | Proposed source | Contract |
|---|---|---|
| `version`, `tree_level`, `root_block_no` | one share-locked, validated metapage observation | direct, not extrapolated |
| `index_size` | captured `nblocks * BLCKSZ` for the main fork | direct at length-capture time |
| `sampled_pages` | number of ordinary blocks actually inspected | direct |
| `sampled_leaf_pages` | sampled blocks classified as live leaves | direct |
| `scanned_percent` | `100 * sampled_pages / (nblocks - 1)` | direct description of the realized sample |
| `approx_internal_pages` | sampled internal count scaled by the inverse inclusion fraction | estimate |
| `approx_leaf_pages` | sampled live-leaf count scaled by the inverse inclusion fraction | estimate |
| `approx_empty_pages` | sampled half-dead count scaled by the inverse inclusion fraction | estimate |
| `approx_deleted_pages` | sampled deleted count scaled by the inverse inclusion fraction | estimate |
| `approx_avg_leaf_density` | ratio of sampled leaf free-space and capacity sums | estimate |
| `approx_leaf_fragmentation` | sampled backward-right-link leaves divided by sampled live leaves | estimate |

The input remains the **requested** fraction. The size rule can raise it, but
`sampled_pages` and `scanned_percent` expose the realized sample. Rounding to a
whole page can make `scanned_percent` exceed 10%; for example, 50 sampled pages
out of 495 ordinary pages is 10.1010%. The threshold compares the captured main-
fork byte size, including the metapage, with `104857600`; it does not compare
formatted size text.

`pgstattuple_approx` provides the local naming precedent: it publishes
`scanned_percent` and `approx_` fields rather than weakening `pgstattuple`'s
result names
([pgstatapprox.c#output_type](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52),
[pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119)).

Proposed extension DDL:

```sql
-- proposed: contrib/pgstattuple/pgstattuple--1.5--1.6.sql
CREATE /* wiki_pgstatindex_approx_extension */ FUNCTION pgstatindex_approx(
    IN relname regclass,
    IN sample_fraction FLOAT8 DEFAULT 0.01,
    OUT version INT,
    OUT tree_level INT,
    OUT index_size BIGINT,
    OUT root_block_no BIGINT,
    OUT sampled_pages BIGINT,
    OUT sampled_leaf_pages BIGINT,
    OUT scanned_percent FLOAT8,
    OUT approx_internal_pages BIGINT,
    OUT approx_leaf_pages BIGINT,
    OUT approx_empty_pages BIGINT,
    OUT approx_deleted_pages BIGINT,
    OUT approx_avg_leaf_density FLOAT8,
    OUT approx_leaf_fragmentation FLOAT8)
AS 'MODULE_PATHNAME', 'pgstatindex_approx'
LANGUAGE C STRICT VOLATILE PARALLEL RESTRICTED;

REVOKE /* wiki_pgstatindex_approx_permissions */ EXECUTE
ON FUNCTION pgstatindex_approx(regclass, FLOAT8) FROM PUBLIC;
GRANT /* wiki_pgstatindex_approx_permissions */ EXECUTE
ON FUNCTION pgstatindex_approx(regclass, FLOAT8) TO pg_stat_scan_tables;
```

This is proposed DDL, not an object in the pinned checkout. The default remains
a requested 1%; the size policy makes its effective value 10% below 100 MiB.
`STRICT` follows the existing null-input behavior. The revoke/grant follows the
v1.5 permission model. A new API needs only the `regclass` entry point; the
existing text entry point is retained for historical compatibility, not as a
requirement for new functions
([pgstatindex.c#compatibility-comment](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L66),
[pgstattuple--1.4--1.5.sql#grant-model](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119)).

`PARALLEL RESTRICTED` is a conservative proposal for an unseeded random
sampler. The existing non-random full scanner is declared `PARALLEL SAFE`,
while v12 marks SQL `random()` volatile and parallel-restricted. A final patch
could use `PARALLEL SAFE` only after its per-call seeding and worker execution
behavior are audited
([pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pg_proc.dat#random](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3223-L3225)).

### C call path and data structures

The proposed normal path is:

```text
pgstatindex_approx(regclass, float8)
  -> relation_open(oid, AccessShareLock)
  -> existing B-tree and other-session-temp guards
  -> share-lock and validate block 0; copy true-root metadata
  -> RelationGetNumberOfBlocks(rel) once
  -> index_size = (int64) nblocks * BLCKSZ
  -> effective_fraction = index_size < 100 MiB
       ? max(sample_fraction, 0.10) : sample_fraction
  -> compute sample_target from effective_fraction
  -> BlockSampler_Init(nblocks - 1, sample_target, seed)
  -> while BlockSampler_HasMore
       -> blkno = BlockSampler_Next(...) + 1
       -> CHECK_FOR_INTERRUPTS
       -> ReadBufferExtended(..., BAS_BULKREAD)
       -> BUFFER_LOCK_SHARE
       -> _bt_checkpage
       -> classify and accumulate
       -> unlock and release
  -> relation_close(rel, AccessShareLock)
  -> construct the composite result
```

The relation-opening and guard behavior should remain aligned with
`pgstatindex_impl`. That preserves its physical-B-tree-only boundary, permits a
current-session temporary B-tree, rejects another session's local-buffer
relation, and continues to describe readable physical storage without consulting
`pg_index.indisvalid`, `indisready`, or `indislive`
([pgstatindex.c#wrappers-and-guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L169-L238),
[rel.h#RELATION_IS_OTHER_TEMP](../../../raw/postgres-12/src/include/utils/rel.h#L541-L549),
[pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L44)).

The existing v1.5 wrappers rely on SQL function `EXECUTE` privilege and perform
no relation-level ownership or `SELECT` check. Granting the proposed function to
`pg_stat_scan_tables` preserves that existing diagnostic privilege boundary; it
must be a deliberate choice, not an accidental consequence of copying the SQL
script
([pgstattuple--1.4--1.5.sql#grant-model](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119),
[pgstatindex.c#v1.5-wrapper-and-worker](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L203-L238)).

The patch should reuse or refactor the existing `BTIndexStat` accumulator, then
add `BlockSamplerData`, the captured population size, the realized sample size,
and sampled-leaf count. `BTMetaPageData` supplies the reported metadata;
`BTPageOpaqueData` supplies page flags and `btpo_next`
([pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95),
[sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43),
[nbtree.h#page-structures](../../../raw/postgres-12/src/include/access/nbtree.h#L29-L110)).

Do not copy the exact function's weakest integrity details merely for code reuse.
The new function should content-lock block 0 and apply the ordinary B-tree page
check before reading it. It should also verify `BTP_META`, `BTREE_MAGIC`, and the
supported version range. Core's `_bt_getmeta` performs those metadata checks but
is file-local, so an in-tree contrib patch must duplicate the checks or expose a
suitable core helper
([nbtpage.c#_bt_getmeta](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148),
[nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).
Apply `_bt_checkpage` to every sampled ordinary block before dereferencing its
special area. That catches all-zero pages and a wrong B-tree special-space size;
it does not validate unsampled pages or the full tree
([nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720),
[nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L747-L769)).

### Sampling strategy and estimator math

Let:

```text
S = nblocks * BLCKSZ                     -- captured main-fork bytes
N = max(nblocks - 1, 0)                  -- ordinary physical blocks
f_effective = max(f_requested, 0.10)     when S < 100 * 1024 * 1024
f_effective = f_requested                when S >= 100 * 1024 * 1024
k = 0                                    when N = 0
k = min(N, max(1, ceil(N*f_effective)))  when N > 0
f_realized = k / N                       when N > 0
```

Reject any requested `sample_fraction` that does not satisfy `0 < f <= 1`,
including NaN and infinities, with `ERRCODE_INVALID_PARAMETER_VALUE`, before
applying the floor. The 10% minimum is a documented policy rather than a hidden
replacement: the input name and call retain the requested value, while
`sampled_pages` and `scanned_percent` report what was actually read. A request
of 10% or more is unchanged. Exactly 100 MiB is also unchanged because the test
is strict `<`, and the `N = 0` branch takes precedence for an empty index.
The byte threshold corresponds to different page counts on builds with different
`BLCKSZ`: v12 permits 1, 2, 4, 8, 16, or 32 KiB at build time, and exposes the
compiled value through the internal `block_size` setting
([configure.in#block-size](../../../raw/postgres-12/configure.in#L247-L265),
[guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888),
[config.sgml#block_size](../../../raw/postgres-12/doc/src/sgml/config.sgml#L9070-L9085)).

Initialize `BlockSampler` with population `N`; each returned value is in
`[0, N - 1]`, so add one before reading the index. Algorithm S samples without
replacement, and `bs->t` only increases, so output arrives in ascending physical
order without an array or a sort
([sampling.c#BlockSampler-contract](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L51),
[sampling.c#BlockSampler-iteration](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L53-L112)).
PostgreSQL 12's `ANALYZE` path supplies `random()` as the seed and consumes
`BlockSampler_Next` in a loop; that is an implementation precedent, not a
requirement that this API hide its seed forever
([analyze.c#block-sampling](../../../raw/postgres-12/src/backend/commands/analyze.c#L1019-L1049)).

The native sampler uses fixed memory and at most one random draw per selected
block, but its inner loop still advances across skipped block numbers. It can therefore do
O(`N`) arithmetic in the worst case even though it performs only `k` index-page
reads. `CHECK_FOR_INTERRUPTS` in the caller cannot fire while one long
`BlockSampler_Next` call is inside that skip loop
([sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111)).
Also, `BlockSamplerData.n` and `.m` are `int` while the block population is a
32-bit unsigned `BlockNumber`. A patch can bypass the sampler for `f = 1` and run
the ordinary full loop, but a partial sample whose target exceeds `INT_MAX`
needs a wider sampler, a documented error, or a lower requested fraction
([sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43),
[block.h#BlockNumber](../../../raw/postgres-12/src/include/storage/block.h#L17-L35)).

For each exhaustive page class `c`, use the unrounded estimator:

```text
estimated_pages(c) = sampled_pages(c) * N / k
```

The physical-block sample gives every captured ordinary block the same inclusion
probability. Integer output requires rounding, so the four independently rounded
counts can differ from `N` by a small number of pages. `index_size` must remain
independent of those rounded estimates.

For sampled live leaves, preserve the current ratio-of-sums definition:

```text
approx_avg_leaf_density =
  100 - sum(PageGetFreeSpace(page))
        / sum(pd_special - SizeOfPageHeaderData) * 100

approx_leaf_fragmentation =
  sampled_backward_right_links / sampled_leaf_pages * 100
```

These are estimates over sampled live leaves, not count-scaled values. Format
non-`NaN` ratios to two decimal places to preserve the current output convention.
Return `NaN` for both when no live leaf is sampled, exactly as the current
function does when the full scan has no live leaf
([pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307),
[pgstatindex.c#ratio-zero-cases](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).
Report `sampled_leaf_pages` so callers can distinguish a well-supported ratio
from a ratio based on very few leaves.

### Locking, concurrency, and integrity

The C implementation should hold one `AccessShareLock` from relation open to
relation close. In v12 that lock conflicts only with `AccessExclusiveLock`.
Ordinary DML and lazy VACUUM open indexes with `RowExclusiveLock`, so inserts,
deletes, splits, page deletion/reuse, and VACUUM can overlap the diagnostic
([lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105),
[execIndexing.c#ExecOpenIndices](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L141-L213),
[vacuumlazy.c#index-locks](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L275-L292)).

The result remains a mixed-time physical observation:

- The metapage lock makes its fields one protected observation, but the lock is
  released before ordinary pages are read.
- `nblocks` is captured once. Blocks appended afterward are outside both the
  sample population and reported `index_size`.
- Every sampled page has a shared content lock only while it is classified. Its
  contents can change or the physical block can be recycled before or after that
  observation.
- Sampling fewer pages widens the statistical gap, but it does not remove the
  existing no-whole-index-snapshot boundary
  ([pgstattuple.sgml#snapshot-limit](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279),
  [pgstatindex.c#one-time-length-and-page-locks](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).

Use the same `BAS_BULKREAD` strategy for sampled page reads. In v12 it uses a
nominal 256 KiB ring capped at one eighth of `shared_buffers`; the sparse sample
still reads only selected pages, but this limits its replacement footprint
([freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L587)).

A share-locked and B-tree-validated sample is still not an integrity check.
Unsampled blocks are never inspected, and `_bt_checkpage` does not verify parent
links, sibling reciprocity, key order, or tuple-to-heap correspondence
([nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).
With `sample_fraction = 1`, a healthy quiescent index should produce the same
page counts and ratios as `pgstatindex`; the proposed validation can deliberately
turn malformed-page misclassification into an error instead.

### Extension, build, and generated-file implications

PostgreSQL 12's `pgstattuple` control file selects version 1.5. The Makefile
ships a 1.4 base script and a 1.4-to-1.5 update script, and the extension manager
can install a base version then follow update scripts when no direct target
install script exists
([pgstattuple.control#default-version](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5),
[Makefile#extension-scripts](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L3-L13),
[extension.c#install-path-selection](../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400),
[extension.c#install-then-update](../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550)).

A complete in-tree patch should:

1. add `PG_FUNCTION_INFO_V1(pgstatindex_approx)` and the C symbol, preferably by
   refactoring the existing page classifier in `pgstatindex.c`;
2. add `pgstattuple--1.5--1.6.sql` to `DATA` and bump `default_version` to 1.6;
3. update `pgstattuple.sgml`;
4. extend `sql/pgstattuple.sql` and `expected/pgstattuple.out`; and
5. add a separate object to `OBJS` only if the implementation uses a new C file
   ([Makefile#module-and-tests](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24),
   [fmgr.h#PG_FUNCTION_INFO_V1](../../../raw/postgres-12/src/include/fmgr.h#L383-L413),
   [fmgr.c#fetch_finfo_record](../../../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L458-L511)).

The existing code has one generated-header dependency: `pgstatindex.c` includes
`pg_am.h`, which includes generated `pg_am_d.h`; `pg_am.dat` supplies
`BTREE_AM_OID`, and `genbki.pl` emits that symbol. The proposal reuses this
existing dependency and adds the ordinary core header `utils/sampling.h`; it
does not require a new catalog, parser rule, generated header, or index-AM
callback
([pg_am.h#generated-header](../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29),
[pg_am.dat#BTREE_AM_OID](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23),
[catalog/Makefile#generated-headers](../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L100),
[genbki.pl#OID-symbol-emission](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603),
[sampling.h#API](../../../raw/postgres-12/src/include/utils/sampling.h#L13-L43)).

This remains a private B-tree-layout diagnostic. It reads `BTMetaPageData` and
`BTPageOpaqueData` directly instead of dispatching through an access-method
callback. The C implementation has no runtime dependency on `pageinspect`; that
separate contrib module is needed only for the SQL prototype
([pgstatindex.c#includes-and-AM-test](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L28-L73)).

## Pros

- **It reduces index-page reads.** The C path reads one metapage plus `k`
  selected ordinary pages instead of all captured ordinary pages
  ([pgstatindex.c#full-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- **It preserves direct metadata and size observations.** Metapage fields do not
  need extrapolation, and captured main-fork size can be computed from
  `nblocks * BLCKSZ` rather than rounded sampled counts
  ([pgstatindex.c#metadata-and-size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L271),
  [pgstatindex.c#current-size-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L342)).
- **It uses the current metric definitions.** Sampled density still uses
  `PageGetFreeSpace`; sampled fragmentation still tests the physical right-link
  direction
  ([pgstatindex.c#leaf-metrics](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)).
- **The native sampler needs no sampled-block array.** `BlockSamplerData` has
  fixed state, and Algorithm S emits selected blocks in ascending order
  ([sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43),
  [sampling.c#BlockSampler_Next](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L53-L112)).
- **The result advertises approximation.** `approx_` names, realized scan
  percentage, and direct sample counts reduce the chance that estimated counts
  are mistaken for a complete physical scan
  ([pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119)).
- **The 10% floor avoids extremely small draws below 100 MiB.** In the rerun,
  it raised the 3.88 MiB partial index from 5 pages at a requested 1% to 50
  pages, and it eliminated leaf-free draws for the 0.66 MiB healthy index
  ([floor rerun](#sampling-results-after-the-10-floor)). `scanned_percent`
  exposes the increase instead of presenting the requested fraction as the
  realized one.
- **The preferred C path can retain bounded buffer replacement.** Reusing
  `BAS_BULKREAD` keeps the v12 256 KiB-or-smaller ring behavior
  ([freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L587)).

## Cons

- **Rare classes are easy to miss.** Internal, half-dead, and deleted pages can
  be a small part of the physical population. A zero sample count yields a zero
  estimate even when that class exists; the full scanner distinguishes all four
  states only because it visits every block
  ([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).
- **Leaf ratios can be unsupported.** A sample with no live leaf returns `NaN`;
  a sample with very few leaves has little information. `sampled_leaf_pages`
  exposes this boundary
  ([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).
- **Physical clustering increases sampling risk.** The sampler chooses physical
  block numbers, while the source metrics classify and measure those physical
  positions. A small sample can miss a region containing unusual page states or
  backward right-links
  ([pgstatindex.c#physical-loop-and-links](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- **The size floor overrides caller cost control.** Below 100 MiB, requested 1%
  and 5% both read at least 10% of ordinary pages. At exactly 100 MiB, the same
  1% request reads about 1%, creating a sharp policy boundary. The fixed byte
  threshold also maps to different page counts when PostgreSQL is built with a
  nondefault `BLCKSZ`
  ([configure.in#block-size](../../../raw/postgres-12/configure.in#L247-L265)).
- **Sparse ascending reads are not the same as a dense sequential walk.** They
  retain ordered block numbers and the bulk-read ring, but gaps can reduce the
  locality of the existing block-by-block scan
  ([pgstatindex.c#physical-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- **The native v12 sampler has scale limits.** Its sample count is `int`, and its
  skip loop can do work proportional to the block population even when page I/O
  is small
  ([sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43),
  [sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111)).
- **It remains a mixed-time observation.** DML and VACUUM can overlap an
  `AccessShareLock`, relation length is captured once, and page locks are not
  held together
  ([lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105),
  [pgstattuple.sgml#snapshot-limit](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279)).
- **It cannot certify integrity.** Validation covers only sampled blocks and
  only the local checks implemented by `_bt_checkpage` and the metapage checks
  ([nbtpage.c#B-tree-page-checks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148),
  [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).
- **It adds a long-lived API and maintenance surface.** The patch needs C, SQL
  upgrade, control, Makefile, documentation, permission, and regression changes;
  v12's existing test target has no sampled function
  ([Makefile#module-and-tests](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24),
  [pgstattuple.sql#complete-test](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119)).

## SQL Prototype Using `contrib/pageinspect`

### Prototype boundaries

The prototype uses v12 `pageinspect` because calling `pgstatindex` would perform
the full scan. `pageinspect` is a low-level debugging extension whose functions
are superuser-only
([pageinspect.sgml#overview](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14)).

A fresh v12 install selects pageinspect 1.7. That SQL definition declares
`bt_metap.oldest_xact` as signed `int4`, while the C symbol formats the underlying
`TransactionId` with `%u`. A sufficiently high value can therefore fail tuple
conversion even when the outer query needs only earlier columns. The six-column
compatibility helper below calls the same C symbol with a shorter declared tuple
descriptor; `BuildTupleFromCStrings` converts only `tupdesc->natts` values
([pageinspect.control#default-version](../../../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5),
[pageinspect/Makefile#extension-scripts](../../../raw/postgres-12/contrib/pageinspect/Makefile#L7-L15),
[pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26),
[btreefuncs.c#bt_metap-output](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L505-L575),
[execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2152)).

`bt_page_stats(text, int4)` returns page type, page size, free size, and sibling
links from one share-locked page read. Its type mapping is `d` for deleted, `e`
for half-dead, `l` for leaf, `r` for an internal root, and `i` for another
internal page, matching `pgstatindex`'s classification order
([pageinspect--1.5.sql#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L159-L175),
[btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153),
[pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).

The earlier prototype read every sampled page again through
`get_raw_page`/`page_header` to obtain `pd_special`. That is unnecessary for a
valid initialized v12 B-tree page. `_bt_pageinit` reserves
`sizeof(BTPageOpaqueData)` through `PageInit`; the fixed header is 24 bytes and
the opaque structure is 16 bytes, so the usable capacity reported by the exact
formula is `page_size - 40`. The revised prototype therefore reads each sampled
ordinary page once
([nbtpage.c#_bt_pageinit](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L920-L929),
[bufpage.c#PageInit](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L35-L60),
[bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164),
[bufpage.h#SizeOfPageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L207-L217),
[nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)).

The remaining limits are deliberate and visible:

- `ORDER BY random() LIMIT k` avoids reading unselected index pages, but still
  generates every candidate number and performs a random-key sort. SQL
  `random()` is volatile and parallel-restricted
  ([pg_proc.dat#random](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3223-L3225)).
- Below 100 MiB, requested fractions below 10% all produce the same target after
  whole-page rounding. The floor increases sampled page reads, but it does not
  reduce this SQL prototype's O(`N`) candidate generation and random-key sort.
- Each `bt_page_stats` call opens and closes the relation around one normal
  buffer read. The SQL function does not hold one continuous `AccessShareLock`
  or use `BAS_BULKREAD`
  ([btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L213),
  [relation.c#relation_close](../../../raw/postgres-12/src/backend/access/common/relation.c#L196-L217)).
- `bt_page_stats` does not call `_bt_checkpage`; the `page_size - 40` derivation
  and the page classifier assume valid initialized B-tree pages. The prototype
  is not safe evidence about malformed storage
  ([btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153),
  [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).
- The `int4` pageinspect API limits the prototype to indexes whose largest
  sampled block fits signed `int4`. The CTE allows at most `2^31` total blocks,
  because its largest ordinary block is then `nblocks - 1 = 2^31 - 1`. The C
  proposal uses `BlockNumber` and should not inherit this SQL-only bound
  ([pageinspect--1.5.sql#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L159-L175),
  [datatype.sgml#integer-range](../../../raw/postgres-12/doc/src/sgml/datatype.sgml#L356-L371),
  [block.h#BlockNumber](../../../raw/postgres-12/src/include/storage/block.h#L17-L35)).
- Invalid fractions and over-limit indexes produce no row in this SQL prototype.
  The proposed C API must raise explicit errors.
- The helpers are user-created objects, not members of the `pageinspect` or
  `pgstattuple` extension. Remove them manually when the diagnostic is no longer
  needed.

### Setup

Both timeout settings are `PGC_USERSET`, so these `SET` commands affect only the
session and require neither reload nor restart
([guc.c#diagnostic-timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2397)).
The five-minute statement timeout bounds the whole sampling query; the two-second
lock timeout bounds each relation-lock acquisition.

```sql
SET /* wiki_pgstatindex_pageinspect_timeout */ statement_timeout = '5min';
SET /* wiki_pgstatindex_pageinspect_timeout */ lock_timeout = '2s';
CREATE /* wiki_pgstatindex_pageinspect_setup */ EXTENSION IF NOT EXISTS pageinspect;

SELECT /* wiki_pgstatindex_pageinspect_schema */ n.nspname AS pageinspect_schema
FROM pg_extension AS e
JOIN pg_namespace AS n ON n.oid = e.extnamespace
WHERE e.extname = 'pageinspect';

-- Replace public with the schema returned above if pageinspect lives elsewhere.
SET /* wiki_pgstatindex_pageinspect_path */ search_path = public, pg_catalog;
```

`pageinspect` is relocatable, and `pg_extension.extnamespace` records its schema.
`SET search_path FROM CURRENT` on the function below saves the configured path
for later executions
([pageinspect.control#relocatable](../../../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5),
[pg_extension.h#FormData_pg_extension](../../../raw/postgres-12/src/include/catalog/pg_extension.h#L30-L45),
[create_function.sgml#configuration-parameter](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L494-L505)).

### Compatibility helper and sampler

This revision adds output columns to the earlier prototype. PostgreSQL 12 does
not let `CREATE OR REPLACE FUNCTION` change the anonymous composite result of an
existing function with the same input signature. If that older prototype is
already installed, use a new function name or explicitly remove the old helper
before creating this one
([pg_proc.c#ProcedureCreate-result-type-check](../../../raw/postgres-12/src/backend/catalog/pg_proc.c#L411-L460),
[create_function.sgml#replacement-restrictions](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L638-L648)).

```sql
CREATE /* wiki_pgstatindex_pageinspect_metap_compat */ OR REPLACE FUNCTION pgstatindex_pageinspect_bt_metap_compat(
  relname text,
  OUT magic int4,
  OUT version int4,
  OUT root int4,
  OUT level int4,
  OUT fastroot int4,
  OUT fastlevel int4)
AS '$libdir/pageinspect', 'bt_metap'
LANGUAGE C STRICT PARALLEL SAFE;

CREATE /* wiki_pgstatindex_pageinspect_function */ OR REPLACE FUNCTION pgstatindex_approx_pageinspect(
    idx regclass,
    sample_fraction float8 DEFAULT 0.01)
RETURNS TABLE (
    version int,
    tree_level int,
    index_size bigint,
    root_block_no bigint,
    sampled_pages bigint,
    sampled_leaf_pages bigint,
    scanned_percent float8,
    approx_internal_pages bigint,
    approx_leaf_pages bigint,
    approx_empty_pages bigint,
    approx_deleted_pages bigint,
    approx_avg_leaf_density float8,
    approx_leaf_fragmentation float8)
LANGUAGE sql
VOLATILE
PARALLEL RESTRICTED
SET search_path FROM CURRENT
AS $function$
WITH /* wiki_pgstatindex_pageinspect_sample */ params AS (
  SELECT $1 AS idx, $2::float8 AS sample_fraction
),
size_once AS MATERIALIZED (
  SELECT idx, sample_fraction, pg_relation_size(idx) AS index_size
  FROM params
  WHERE sample_fraction > 0
    AND sample_fraction <= 1
),
bounds AS (
  SELECT idx,
       sample_fraction,
       CASE
         WHEN index_size < 104857600::bigint
           THEN GREATEST(sample_fraction, 0.10::float8)
         ELSE sample_fraction
       END AS effective_fraction,
       index_size,
       index_size / current_setting('block_size')::int AS nblocks
  FROM size_once
  WHERE index_size / current_setting('block_size')::int <= 2147483648::bigint
),
sample_plan AS MATERIALIZED (
  SELECT *,
       CASE
         WHEN nblocks <= 1 THEN 0::bigint
         ELSE LEAST(nblocks - 1,
                    GREATEST(1::bigint,
                             ceil((nblocks - 1)::float8 * effective_fraction)::bigint))
       END AS sample_target
  FROM bounds
),
candidate_blocks AS MATERIALIZED (
  SELECT p.idx, p.nblocks, p.index_size, p.sample_target, b.blkno
  FROM sample_plan AS p
  CROSS JOIN LATERAL generate_series(1::bigint, p.nblocks - 1) AS b(blkno)
  ORDER BY random()
  LIMIT (SELECT sample_target FROM sample_plan)
),
sample_blocks AS MATERIALIZED (
  SELECT *
  FROM candidate_blocks
  ORDER BY blkno
),
page_sample AS (
  SELECT s.blkno,
       ps.type,
       ps.page_size,
       ps.free_size,
       ps.btpo_next
  FROM sample_blocks AS s
  CROSS JOIN LATERAL bt_page_stats(s.idx::text, s.blkno::int4) AS ps
),
agg AS (
  SELECT count(*)::bigint AS sampled_blocks,
       count(*) FILTER (WHERE type IN ('i', 'r'))::bigint AS internal_sample,
       count(*) FILTER (WHERE type = 'l')::bigint AS leaf_sample,
       count(*) FILTER (WHERE type = 'e')::bigint AS empty_sample,
       count(*) FILTER (WHERE type = 'd')::bigint AS deleted_sample,
       coalesce(sum(CASE WHEN page_size > 40 THEN page_size - 40 ELSE 0 END)
         FILTER (WHERE type = 'l'), 0)::numeric AS leaf_max_avail,
       coalesce(sum(free_size) FILTER (WHERE type = 'l'), 0)::numeric AS leaf_free_space,
       count(*) FILTER (WHERE type = 'l'
         AND btpo_next <> 0
         AND btpo_next::bigint < blkno)::bigint AS fragments_sample
  FROM page_sample
)
SELECT m.version,
     m.level::int AS tree_level,
     p.index_size,
     m.root::bigint AS root_block_no,
     a.sampled_blocks AS sampled_pages,
     a.leaf_sample AS sampled_leaf_pages,
     CASE WHEN p.nblocks > 1
      THEN 100.0::float8 * a.sampled_blocks::float8 / (p.nblocks - 1)::float8
      ELSE 0.0::float8
     END AS scanned_percent,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.internal_sample::numeric * (p.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_internal_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.leaf_sample::numeric * (p.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_leaf_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.empty_sample::numeric * (p.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_empty_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.deleted_sample::numeric * (p.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_deleted_pages,
     CASE WHEN a.leaf_max_avail > 0
      THEN round((100.0 - a.leaf_free_space / a.leaf_max_avail * 100.0)::numeric, 2)::float8
      ELSE 'NaN'::float8
     END AS approx_avg_leaf_density,
     CASE WHEN a.leaf_sample > 0
      THEN round((100.0 * a.fragments_sample / a.leaf_sample)::numeric, 2)::float8
      ELSE 'NaN'::float8
     END AS approx_leaf_fragmentation
FROM sample_plan AS p
CROSS JOIN LATERAL pgstatindex_pageinspect_bt_metap_compat(p.idx::text) AS m
CROSS JOIN agg AS a;
$function$;
```

The `bounds` CTE evaluates `pg_relation_size` once, applies the strict
104,857,600-byte threshold, and carries both requested and effective fractions.
The output intentionally reports only the realized `scanned_percent`; the
comparison harness retains the requested value beside each result. The empty
branch remains zero even though its metapage size is below the threshold.

The syntax used here exists in v12: materialized CTEs, aggregate `FILTER`,
`pg_relation_size(regclass)`, `generate_series(int8, int8)`, `random()`, and
`ceil(float8)` are all defined in the pinned checkout
([select.sgml#WITH-MATERIALIZED](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L75-L98),
[syntax.sgml#aggregate-FILTER](../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L1563-L1580),
[pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891),
[pg_proc.dat#generate_series-int8](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L7634-L7645),
[pg_proc.dat#random](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3223-L3225),
[pg_proc.dat#ceil](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L637-L642)).

## Tests To Add

The patch should add deterministic correctness tests and avoid brittle assertions
about one random 1% draw:

1. **Full-sample equivalence:** on a healthy, quiescent populated B-tree,
   `sample_fraction = 1` must match `pgstatindex` for page counts and both
   ratios; direct fields must also match
   ([pgstatindex.c#full-loop-and-results](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L356)).
2. **Empty index:** a metapage-only index must return zero sampled/count fields,
   zero `scanned_percent`, and `NaN` ratios, matching the existing empty-index
   boundary
   ([pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).
3. **Argument errors and null:** reject zero, negative, values above one, NaN,
   and infinities before applying the floor; verify that `STRICT` returns null
   for a null argument without entering the C function
   ([pgstattuple--1.4--1.5.sql#STRICT](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
4. **Size-floor boundaries:** below 100 MiB, requests below 10% must use
   `ceil(N * 0.10)` pages; requests of 10% or more must remain unchanged. At
   exactly 100 MiB and above, the requested fraction must remain unchanged.
   Verify zero ordinary pages for an empty index, whole-page rounding in
   `scanned_percent`, and both real relations below and above the threshold.
5. **Sampling invariants:** through an exposed or test-only fixed-seed path,
   sampled blocks must be in range, unique, and ascending; the reported sample
   counts and percentage must match the realized draw
   ([sampling.c#BlockSampler-iteration](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L53-L112)).
6. **No sampled leaf:** force a sample containing no live leaf and assert two
   `NaN` ratio fields plus `sampled_leaf_pages = 0`
   ([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).
7. **Object and state boundaries:** cover a wrong access method, non-index
   relation kinds, partitioned index, another-session temporary index, a
   current-session temporary index, and readable invalid/not-ready B-trees
   ([pgstatindex.c#guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L238),
   [pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L44)).
8. **Validation:** exercise an all-zero sampled page, wrong special-space size,
   bad metapage magic, and unsupported B-tree version; also prove that an
   unsampled malformed page is outside the function's integrity claim
   ([nbtpage.c#B-tree-page-checks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148),
   [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).
9. **Concurrency and cancellation:** cover extension after length capture, page
   state changes during sampling, statement cancellation, and the long
   `BlockSampler_Next` skip-loop boundary
   ([pgstatindex.c#one-time-length-and-interrupts](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
   [sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111)).
10. **Privileges and parallel placement:** test `PUBLIC` revoke,
    `pg_stat_scan_tables` grant, and the chosen parallel marking
    ([pgstattuple--1.4--1.5.sql#grant-model](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119)).
11. **Extension lifecycle:** test upgrade from 1.5 to 1.6 and a fresh default
    install that follows the base-plus-update path
    ([extension.c#install-path-selection](../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400),
    [extension.c#install-then-update](../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550)).

The pinned tree currently declares one `pgstattuple` regression target. It tests
empty `pgstatindex` output and object/error paths, but has no populated,
sampling, concurrency, malformed-page, permissions, or statistical test and no
module isolation or TAP target
([Makefile#regression-target](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24),
[pgstattuple.sql#complete-test](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119)).
The executed SQL-prototype comparison below exercises populated, empty, partial,
and statistical cases, but it does not add the proposed C function or an
upstream regression test.

## Executed Comparison with Standard `pgstatindex`

The floor-policy rerun used the same six quiescent fixtures plus a 107.13 MiB
healthy index above the threshold. A full sample matched standard `pgstatindex`
on every shared output field for all seven fixtures. Requested 1% and 5% samples
both used a 10% effective target below 100 MiB, while the above-threshold
fixture retained its requested fractions. Partial samples still did not have one
uniform error rate; accuracy depended on the number and physical distribution
of leaf, internal, and deleted pages. Standard `pgstatindex` obtains its counts
only by classifying every captured ordinary block
([pgstatindex.c#full-scan-and-results](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L356)).

This test exercises the `pageinspect` SQL prototype above, not the unimplemented
C proposal. It tests the estimator and random physical-block selection on these
fixtures. It does not benchmark the proposed C path, its `BlockSampler`, its
bulk-read ring, its continuous relation lock, or its stronger page validation.

### Test scope and metrics

The test ran on the pinned PostgreSQL 12.2 checkout with no concurrent fixture
DML. Quiescence matters because standard `pgstatindex` is a page-by-page
physical observation, not one instantaneous whole-index snapshot
([pgstattuple.sgml#snapshot-limit](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279),
[pgstatindex.c#page-locking](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).

The seven fixtures isolate different shapes and both policy sides:

| Fixture | Construction | Boundary tested |
|---|---|---|
| `uniform_sparse` | Build 600,000 ordered keys; delete two of every three; `VACUUM` | Low live-leaf density spread uniformly across the index |
| `range_deleted` | Build 600,000 ordered keys; delete a contiguous 420,000-key middle range; `VACUUM` | A large deleted-page class next to dense surviving leaves |
| `split_churn` | Create the index empty; insert 600,000 keys in a deterministic permuted order; delete one quarter; `VACUUM` | Low density plus backward-right-link fragmentation |
| `partial_sparse` | Build only rows satisfying `WHERE active`; delete two thirds of those indexed rows; `VACUUM` | A sparse partial B-tree |
| `small_dense` | Build 30,000 ordered keys without deletes | Healthy small-index control |
| `large_dense` | Build 5,000,000 ordered keys without deletes | Healthy 107.13 MiB control above the floor threshold |
| `empty` | Build an index on an empty table | Metapage-only edge |

With index cleanup enabled, lazy VACUUM opens the indexes, removes dead index
entries, and asks B-tree VACUUM to delete an empty leaf when legal. B-tree page
deletion first makes the page half-dead, unlinks it, and leaves a deleted page
that cannot necessarily be reclaimed immediately
([vacuumlazy.c#index-cleanup](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L289),
[vacuumlazy.c#remove-index-entries](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L761-L772),
[nbtree.c#empty-page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1327-L1359),
[nbtpage.c#_bt_pagedel](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1285-L1517)).
That path produced the `range_deleted` fixture's deleted-page population. The
uniform deletes left live tuples on each leaf instead, so they primarily reduced
live-leaf density.

A partial-index predicate controls which heap tuples enter the index during a
build and which DML operations maintain it. Once the physical B-tree exists,
`pgstatindex` checks the relation and access-method kinds and scans its pages; it
does not evaluate the partial predicate
([heapam_handler.c#partial-build-predicate](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1198-L1210),
[heapam_handler.c#partial-build-filter](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626),
[execIndexing.c#partial-DML-filter](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L363),
[pgstatindex.c#guards-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L315)).
The partial fixture therefore tests the same physical estimator over a smaller,
predicate-selected B-tree. It does not estimate predicate selectivity or parent-
table bloat.

For each fixture, the rerun made one requested `sample_fraction = 1.0`
comparison and 100 draws at requested fractions 0.01 and 0.05. The updated
prototype floored both partial requests to 0.10 for the six sub-100-MiB fixtures
and preserved 0.01 and 0.05 for `large_dense`. It reset `setseed()` before every
partial draw; PostgreSQL 12 documents that reseeding the current session repeats
subsequent `random()` values, and the implementation stores the seed in backend-
local state
([func.sgml#setseed](../../../raw/postgres-12/doc/src/sgml/func.sgml#L1130-L1159),
[float.c#setseed](../../../raw/postgres-12/src/backend/utils/adt/float.c#L2585-L2644)).
The reported errors are:

```text
leaf MAPE (%) = mean(abs(approx_leaf_pages - leaf_pages) / leaf_pages) * 100
count MAPE (%) = mean(abs(estimate - full-scan count) / full-scan count) * 100
ratio MAE (pp) = mean(abs(sampled ratio - full-scan ratio))
```

`pp` means percentage points. Ratio errors exclude draws with no sampled live
leaf; the table reports those draws separately because the prototype returns
`NaN` when the leaf denominator is zero, following standard `pgstatindex`
([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).

### Disposable fixture

Run this only in a fresh disposable database. It intentionally creates test
relations, bulk-deletes generated rows, and leaves the objects in place for
inspection. First run the `pageinspect` setup, compatibility helper, and sampler
from the preceding section. The timeout changes are session-scoped and require
neither reload nor restart
([guc.c#diagnostic-timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2397)).
The table reloption disables routine autovacuum so it cannot change a fixture
between manual VACUUM and measurement; anti-wraparound autovacuum remains an
exception
([create_table.sgml#autovacuum_enabled](../../../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L1383-L1403)).
`INDEX_CLEANUP TRUE` explicitly requests dead-index-entry removal
([vacuum.sgml#INDEX_CLEANUP](../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L186-L203)).

```sql
SET /* wiki_pgstatindex_compare_timeout */ statement_timeout = '15min';
SET /* wiki_pgstatindex_compare_timeout */ lock_timeout = '2s';
CREATE /* wiki_pgstatindex_compare_extension */ EXTENSION IF NOT EXISTS pgstattuple;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_uniform_sparse
  (id int) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_uniform_sparse
SELECT i FROM generate_series(1, 600000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_uniform_sparse_idx
  ON samp_uniform_sparse (id);
DELETE /* wiki_pgstatindex_compare_bloat */ FROM samp_uniform_sparse
  WHERE id % 3 <> 0;
VACUUM /* wiki_pgstatindex_compare_vacuum */
  (ANALYZE, INDEX_CLEANUP TRUE) samp_uniform_sparse;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_range_deleted
  (id int) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_range_deleted
SELECT i FROM generate_series(1, 600000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_range_deleted_idx
  ON samp_range_deleted (id);
DELETE /* wiki_pgstatindex_compare_bloat */ FROM samp_range_deleted
  WHERE id BETWEEN 90001 AND 510000;
VACUUM /* wiki_pgstatindex_compare_vacuum */
  (ANALYZE, INDEX_CLEANUP TRUE) samp_range_deleted;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_split_churn
  (id int) WITH (autovacuum_enabled = false);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_split_churn_idx
  ON samp_split_churn (id);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_split_churn
SELECT ((i * 104729::bigint) % 1000003)::int
FROM generate_series(1, 600000) AS g(i);
DELETE /* wiki_pgstatindex_compare_bloat */ FROM samp_split_churn
  WHERE id % 4 = 0;
VACUUM /* wiki_pgstatindex_compare_vacuum */
  (ANALYZE, INDEX_CLEANUP TRUE) samp_split_churn;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_partial_sparse
  (id int, active boolean) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_partial_sparse
SELECT i, i % 5 = 0 FROM generate_series(1, 900000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_partial_sparse_idx
  ON samp_partial_sparse (id) WHERE active;
DELETE /* wiki_pgstatindex_compare_bloat */ FROM samp_partial_sparse
  WHERE active AND id % 15 <> 0;
VACUUM /* wiki_pgstatindex_compare_vacuum */
  (ANALYZE, INDEX_CLEANUP TRUE) samp_partial_sparse;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_small_dense
  (id int) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_small_dense
SELECT i FROM generate_series(1, 30000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_small_dense_idx
  ON samp_small_dense (id);

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_large_dense
  (id int) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_large_dense
SELECT i FROM generate_series(1, 5000000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_large_dense_idx
  ON samp_large_dense (id);

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_empty
  (id int) WITH (autovacuum_enabled = false);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_empty_idx
  ON samp_empty (id);
```

The partial-index `WHERE` clause and parenthesized VACUUM options are present in
v12's grammar
([gram.y#CREATE-INDEX](../../../raw/postgres-12/src/backend/parser/gram.y#L7335-L7397),
[gram.y#VACUUM](../../../raw/postgres-12/src/backend/parser/gram.y#L10497-L10533)).

### Seeded comparison harness

The harness records one standard full-scan baseline, runs deterministic samples,
then asserts policy targets, strict threshold edges, full-sample equivalence, and
partial-sample errors. Its `sample_fraction` column stores the request;
`scanned_percent` stores the realized fraction after the floor and rounding.

```sql
CREATE /* wiki_pgstatindex_compare_cases */ TABLE samp_cases (
  case_name text PRIMARY KEY,
  idx regclass NOT NULL,
  shape text NOT NULL,
  is_partial boolean NOT NULL
);
INSERT /* wiki_pgstatindex_compare_cases */ INTO samp_cases VALUES
  ('empty', 'samp_empty_idx', 'metapage only', false),
  ('partial_sparse', 'samp_partial_sparse_idx',
   'partial index; uniform post-delete sparsity', true),
  ('range_deleted', 'samp_range_deleted_idx',
   'contiguous delete; deleted-page population', false),
  ('small_dense', 'samp_small_dense_idx',
   'healthy small-index baseline', false),
  ('large_dense', 'samp_large_dense_idx',
   'healthy above-threshold baseline', false),
  ('split_churn', 'samp_split_churn_idx',
   'random-order splits plus uniform delete', false),
  ('uniform_sparse', 'samp_uniform_sparse_idx',
   'uniform post-delete sparsity', false);

CREATE /* wiki_pgstatindex_compare_exact */ TABLE samp_exact AS
SELECT c.case_name,
       c.idx,
       c.shape,
       c.is_partial,
       pg_relation_size(c.idx) AS index_bytes,
       e.version,
       e.tree_level,
       e.index_size,
       e.root_block_no,
       e.internal_pages,
       e.leaf_pages,
       e.empty_pages,
       e.deleted_pages,
       e.avg_leaf_density,
       e.leaf_fragmentation
FROM samp_cases AS c
CROSS JOIN LATERAL pgstatindex(c.idx) AS e;

CREATE /* wiki_pgstatindex_compare_results */ TABLE samp_results AS
SELECT NULL::text AS case_name,
       NULL::float8 AS sample_fraction,
       NULL::int AS trial,
       s.*
FROM pgstatindex_approx_pageinspect('samp_empty_idx', 1.0) AS s
WITH NO DATA;

DO /* wiki_pgstatindex_compare_draws */ $block$
DECLARE
  c record;
  f float8;
  run int;
  runs int;
  seed float8;
BEGIN
  FOR c IN SELECT case_name, idx FROM samp_cases ORDER BY case_name LOOP
    FOREACH f IN ARRAY ARRAY[0.01::float8, 0.05::float8, 1.0::float8] LOOP
      runs := CASE WHEN f = 1.0 THEN 1 ELSE 100 END;
      FOR run IN 1..runs LOOP
        seed := 2.0 * run / (runs + 1.0) - 1.0;
        PERFORM /* wiki_pgstatindex_compare_seed */ setseed(seed);
        INSERT /* wiki_pgstatindex_compare_draw */ INTO samp_results
        SELECT c.case_name, f, run, s.*
        FROM pgstatindex_approx_pageinspect(c.idx, f) AS s;
      END LOOP;
    END LOOP;
  END LOOP;
END
$block$;

SELECT /* wiki_pgstatindex_compare_floor_assertion */
       bool_and(CASE
         WHEN e.index_bytes < 104857600::bigint
           THEN r.sampled_pages = CASE
             WHEN e.index_size / current_setting('block_size')::int <= 1 THEN 0
             ELSE ceil((e.index_size
               / current_setting('block_size')::int - 1) * 0.10)::bigint
           END
         ELSE r.sampled_pages = ceil(
           (e.index_size / current_setting('block_size')::int - 1)
           * r.sample_fraction)::bigint
       END) AS all_targets_match_policy
FROM samp_exact AS e
JOIN samp_results AS r USING (case_name)
WHERE r.sample_fraction IN (0.01, 0.05);

WITH /* wiki_pgstatindex_compare_threshold_edges */ cases(
  index_size, requested, expected) AS (
  VALUES
    (104857599::bigint, 0.01::float8, 0.10::float8),
    (104857599::bigint, 0.10::float8, 0.10::float8),
    (104857599::bigint, 0.20::float8, 0.20::float8),
    (104857600::bigint, 0.01::float8, 0.01::float8),
    (104857601::bigint, 0.01::float8, 0.01::float8)
)
SELECT /* wiki_pgstatindex_compare_threshold_edges */ bool_and(
  (CASE WHEN index_size < 104857600::bigint
    THEN greatest(requested, 0.10::float8)
    ELSE requested END) = expected) AS strict_threshold_cases_match
FROM cases;

SELECT /* wiki_pgstatindex_compare_full_assertion */
       count(*) AS cases,
       bool_and(r.version = e.version
         AND r.tree_level = e.tree_level
         AND r.index_size = e.index_size
         AND r.root_block_no = e.root_block_no
         AND r.sampled_pages = GREATEST(
           e.index_size / current_setting('block_size')::int - 1, 0)
         AND r.sampled_leaf_pages = e.leaf_pages
         AND r.scanned_percent = CASE
           WHEN e.index_size > current_setting('block_size')::int
             THEN 100.0 ELSE 0.0 END
         AND r.approx_internal_pages = e.internal_pages
         AND r.approx_leaf_pages = e.leaf_pages
         AND r.approx_empty_pages = e.empty_pages
         AND r.approx_deleted_pages = e.deleted_pages
         AND r.approx_avg_leaf_density
             IS NOT DISTINCT FROM e.avg_leaf_density
         AND r.approx_leaf_fragmentation
             IS NOT DISTINCT FROM e.leaf_fragmentation)
         AS all_full_samples_match
FROM samp_exact AS e
JOIN samp_results AS r USING (case_name)
WHERE r.sample_fraction = 1.0;

SELECT /* wiki_pgstatindex_compare_sample_results */
       e.case_name,
       r.sample_fraction,
       min(r.sampled_pages) AS sampled_pages,
       round(min(r.scanned_percent)::numeric, 4) AS realized_percent,
       round(avg(abs(r.approx_leaf_pages - e.leaf_pages)::numeric
         / nullif(e.leaf_pages, 0) * 100), 2) AS leaf_mape_pct,
       min(r.approx_deleted_pages) AS deleted_est_min,
       max(r.approx_deleted_pages) AS deleted_est_max,
       round(avg(abs(r.approx_deleted_pages - e.deleted_pages)::numeric
         / nullif(e.deleted_pages, 0) * 100), 2) AS deleted_mape_pct,
       round(avg(CASE WHEN r.sampled_leaf_pages > 0 THEN
         abs(r.approx_avg_leaf_density - e.avg_leaf_density)::numeric END),
         2) AS density_mae_pp,
       round(max(CASE WHEN r.sampled_leaf_pages > 0 THEN
         abs(r.approx_avg_leaf_density - e.avg_leaf_density) END)::numeric,
         2) AS density_max_error_pp,
       round(avg(CASE WHEN r.sampled_leaf_pages > 0 THEN
         abs(r.approx_leaf_fragmentation - e.leaf_fragmentation)::numeric END),
         2) AS fragmentation_mae_pp,
       round(max(CASE WHEN r.sampled_leaf_pages > 0 THEN
         abs(r.approx_leaf_fragmentation - e.leaf_fragmentation) END)::numeric,
         2) AS fragmentation_max_error_pp,
       count(*) FILTER (WHERE r.sampled_leaf_pages = 0) AS leaf_free_runs
FROM samp_exact AS e
JOIN samp_results AS r USING (case_name)
WHERE r.sample_fraction < 1.0
GROUP BY e.case_name, r.sample_fraction, e.leaf_pages, e.deleted_pages,
         e.avg_leaf_density, e.leaf_fragmentation
ORDER BY e.case_name, r.sample_fraction;
```

`pg_relation_size(regclass)` opens the relation with `AccessShareLock` and
calculates the selected fork's physical size; the one-argument SQL overload used
here supplies the main fork
([dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L335),
[pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)).

### Full-scan baseline

The six original indexes remained below both 50 MB and 50 MiB. The new
`large_dense` control was 107.13 MiB, so the rerun exercised both sides of the
100 MiB policy threshold. Standard `pgstatindex` returned this baseline:

| Fixture | Size (MiB) | Partial | Internal | Live leaf | Half-dead | Deleted | Density (%) | Fragmentation (%) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `empty` | 0.01 | no | 0 | 0 | 0 | 0 | `NaN` | `NaN` |
| `large_dense` | 107.13 | no | 50 | 13,662 | 0 | 0 | 90.08 | 0.00 |
| `partial_sparse` | 3.88 | yes | 3 | 492 | 0 | 0 | 30.21 | 0.00 |
| `range_deleted` | 12.88 | no | 4 | 494 | 0 | 1,149 | 89.69 | 0.00 |
| `small_dense` | 0.66 | no | 1 | 82 | 0 | 0 | 90.05 | 0.00 |
| `split_churn` | 18.25 | no | 12 | 2,323 | 0 | 0 | 47.82 | 49.98 |
| `uniform_sparse` | 12.88 | no | 7 | 1,640 | 0 | 0 | 30.21 | 0.00 |

The policy-target assertion and strict synthetic threshold-edge assertion both
returned true. The 100% assertion returned `cases = 7` and
`all_full_samples_match = true`. For every fixture, the prototype matched the
standard version, level, size, root, four page-class counts, density, and
fragmentation. It also reported every ordinary page and every live leaf as
sampled, with 100% scanned for populated indexes and 0% for the metapage-only
index. The empty result agrees with the checked-in v12 regression boundary
([pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

This is equivalence on healthy, quiescent storage. It is not a claim that either
function returns a transactionally consistent snapshot, and it does not cover
the proposal's deliberately stricter malformed-page errors.

### Sampling results after the 10% floor

The following values summarize 100 seeded draws per fixture and requested
fraction. `Realized` is the actual `scanned_percent`; whole-page rounding makes
it slightly exceed the target in most small fixtures. `mean / max` reports
absolute percentage-point error. Deleted-page estimate ranges and MAPE are shown
only where the standard count was nonzero.

| Fixture | Requested | Realized | Pages/draw | Leaf MAPE | Deleted estimate range (MAPE) | Density error, mean / max | Fragmentation error, mean / max | No-leaf draws |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `empty` | 1% | 0.0000% | 0 | n/a | n/a | n/a | n/a | 100 |
| `empty` | 5% | 0.0000% | 0 | n/a | n/a | n/a | n/a | 100 |
| `large_dense` | 1% | 1.0064% | 138 | 0.41% | n/a | 0.02 / 0.52 pp | 0.00 / 0.00 pp | 0 |
| `large_dense` | 5% | 5.0029% | 686 | 0.18% | n/a | 0.02 / 0.10 pp | 0.00 / 0.00 pp | 0 |
| `partial_sparse` | 1% | 10.1010% | 50 | 0.86% | n/a | 0.02 / 0.11 pp | 0.00 / 0.00 pp | 0 |
| `partial_sparse` | 5% | 10.1010% | 50 | 0.86% | n/a | 0.02 / 0.11 pp | 0.00 / 0.00 pp | 0 |
| `range_deleted` | 1% | 10.0182% | 165 | 9.17% | 988-1,308 (3.92%) | 0.52 / 1.47 pp | 0.00 / 0.00 pp | 0 |
| `range_deleted` | 5% | 10.0182% | 165 | 9.17% | 988-1,308 (3.92%) | 0.52 / 1.47 pp | 0.00 / 0.00 pp | 0 |
| `small_dense` | 1% | 10.8434% | 9 | 1.99% | n/a | 0.08 / 0.36 pp | 0.00 / 0.00 pp | 0 |
| `small_dense` | 5% | 10.8434% | 9 | 1.99% | n/a | 0.08 / 0.36 pp | 0.00 / 0.00 pp | 0 |
| `split_churn` | 1% | 10.0214% | 234 | 0.38% | n/a | 0.18 / 0.72 pp | 2.54 / 7.28 pp | 0 |
| `split_churn` | 5% | 10.0214% | 234 | 0.38% | n/a | 0.18 / 0.72 pp | 2.54 / 7.28 pp | 0 |
| `uniform_sparse` | 1% | 10.0182% | 165 | 0.43% | n/a | 0.03 / 0.11 pp | 0.00 / 0.00 pp | 0 |
| `uniform_sparse` | 5% | 10.0182% | 165 | 0.43% | n/a | 0.03 / 0.11 pp | 0.00 / 0.00 pp | 0 |

### Effect on the small partial index

The 3.88 MiB `partial_sparse` index has 495 ordinary pages: 3 internal and 492
live leaves. The policy therefore samples `ceil(495 * 0.10) = 50` pages for both
requested fractions below 10%. The same 100 seeds produced this before/after
comparison:

| Requested | Pages before -> after | Realized before -> after | Leaf MAPE before -> after | Density error mean / max before -> after | No-leaf draws before -> after |
|---:|---:|---:|---:|---:|---:|
| 1% | 5 -> 50 | 1.0101% -> 10.1010% | 1.18% -> 0.86% | 0.02 / 0.02 pp -> 0.02 / 0.11 pp | 0 -> 0 |
| 5% | 25 -> 50 | 5.0505% -> 10.1010% | 1.08% -> 0.86% | 0.02 / 0.23 pp -> 0.02 / 0.11 pp | 0 -> 0 |

The floor modestly improved the estimated leaf-page count. It did not improve
mean density error because live-leaf occupancy was already highly uniform, and
finite seeded draws do not make every maximum error monotonic: the 1% row's
maximum density error rose from 0.02 to 0.11 points, while the 5% row's maximum
fell from 0.23 to 0.11. Both requested fractions now produce identical results
because they use the same effective target and the harness resets the same seed
before each draw. The floor does not change partial-index semantics: it samples
the predicate-selected physical B-tree and still says nothing about excluded
heap rows or parent-table bloat
([heapam_handler.c#partial-build-filter](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626),
[pgstatindex.c#guards-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L315)).

### What the comparison establishes

- **Full sampling is a strong equivalence check.** On all tested page shapes,
  sampling every ordinary block reproduced standard `pgstatindex`. This checks
  the SQL prototype's page-type mapping, count scaling, density denominator,
  and backward-link test against the current implementation
  ([pgstatindex.c#classification-and-ratios](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L356)).
- **The strict size policy behaved as specified.** All six sub-100-MiB fixtures
  used at least 10%, including requested 1% and 5%. The 107.13 MiB control kept
  its requested fractions and realized 1.0064% and 5.0029% after page rounding.
  Synthetic byte-size cases confirmed that 104,857,599 bytes is floored while
  104,857,600 bytes is not.
- **The floor reduced, but did not remove, rare-class error.** On the
  range-deleted index, moving both requests to 165 pages reduced leaf-count MAPE
  to 9.17% and deleted-count MAPE to 3.92%. Standard `pgstatindex` can separate
  the page classes without sampling error only because it visits every block
  ([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).
- **Uniform density remained easier than fragmented-link frequency.** On
  `split_churn`, the floored draw had 0.18-point density MAE but 2.54-point
  fragmentation MAE and a 7.28-point maximum. The estimators use different page
  observations: density is a ratio of summed free space to capacity, while
  fragmentation counts sampled leaves whose physical right link points backward
  ([pgstatindex.c#leaf-metrics](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)).
- **The floor removed this fixture's unsupported small-index ratio.** The 0.66
  MiB `small_dense` index moved from one sampled page and two leaf-free 1% draws
  to nine pages and no leaf-free draws. This result does not prove that every
  sub-100-MiB index will contain a sampled leaf; physical page composition still
  controls that outcome.
- **Small partial indexes receive more evidence, not different semantics.** The
  3.88 MiB partial fixture used 50 pages for either request and improved leaf-
  count MAPE to 0.86%. The result still says nothing about rows excluded by
  `WHERE active` or the parent table.
- **The empty edge is deterministic.** A metapage-only index has no ordinary
  sample population, so both 1% and 5% drew zero pages and returned the standard
  zero-count/`NaN` boundary
  ([pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).
- **Do not use this SQL test to predict C performance.** The comparison records
  accuracy and pages inspected, not a speedup. The SQL prototype still
  enumerates and random-sorts all candidate block numbers and performs one
  normal-buffer `pageinspect` call per selected page; the C proposal has a
  different sampling, locking, validation, and buffer-access path
  ([btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L213),
  [pgstatindex.c#buffer-strategy](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L223)).

## Context Reviewed

- Pinned checkout: `raw/postgres-12/` at
  `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- `pgstattuple`: control file, Makefile, base/update SQL, every `pgstatindex`
  wrapper, `BTIndexStat`, all of `pgstatindex_impl`, `pgstatapprox.c`, docs,
  complete regression input, and expected output.
- B-tree internals: metapage and opaque structures, root metadata, page
  initialization, generic and B-tree-specific page checks, page-state macros,
  relation length, buffer access strategy, relation and content locks, DML and
  VACUUM index locks, B-tree VACUUM tuple removal, half-dead marking, page
  unlink/deletion/recyclability, and generated `BTREE_AM_OID` plumbing.
- Partial indexes: predicate catalog storage, serial build filtering, executor
  DML filtering, and the physical-page boundary in `pgstatindex`.
- Sampling: complete `sampling.h` and `sampling.c`, reverse callers, the v12
  `ANALYZE` use of `BlockSampler`, and backend-local `random()`/`setseed()`
  implementation and documentation.
- `pageinspect`: control/Makefile/update SQL, `bt_metap`, `bt_page_stats`,
  `get_raw_page`, `page_header`, tuple conversion, docs, and regression wiring.
- SQL and sizing surfaces: CREATE INDEX and VACUUM grammar, `INDEX_CLEANUP`,
  `autovacuum_enabled`, `pg_relation_size(regclass)`, build-time `BLCKSZ` and
  internal `block_size`, and diagnostic timeout contexts.
- Extension lifecycle: target-version path selection, install-then-update,
  fmgr V1 symbol metadata, SQL privileges, and generated catalog headers.
- Earlier exact-pin execution under `.wiki-runtime/` built and installed
  `pageinspect`, compared the pre-floor and one-read/no-floor prototypes on
  populated and metapage-only B-trees, and confirmed full-sample equivalence.
  Invalid fractions returned no rows as documented. The `pgstattuple` regression
  target and all five `pageinspect` regression targets passed.
- The first comparative run used six indexes from 0.01 MiB through 18.25 MiB:
  uniform sparse, range-deleted, split/churn, partial sparse, small healthy, and
  empty. It retained requested 1% and 5% fractions and supplied the before-floor
  values used in the focused comparison.
- The floor-policy rerun reused those fixtures and added a 107.13 MiB healthy
  index. It executed 100 identical seeds at requested 1% and 5%, confirmed the
  10% target on all six sub-threshold fixtures, preserved both requests above
  the threshold, validated strict synthetic cases immediately below, at, and
  above 100 MiB, and matched every shared standard-`pgstatindex` field at 100%
  for all seven indexes. The isolated server was stopped after the run.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Current `pgstatindex` scans every captured ordinary block | [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315) |
| Current metadata and pages do not form one instantaneous snapshot | [pgstattuple.sgml#snapshot-limit](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279), [pgstatindex.c#metapage-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315) |
| `BlockSampler` samples without replacement and emits ascending block numbers | [sampling.c#BlockSampler](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L112) |
| The native sampler has an `int` sample count and a potentially long skip loop | [sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43), [sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111) |
| Count estimates scale sampled exhaustive classes; ratio estimates reuse current leaf formulas | [pgstatindex.c#classification-and-leaf-metrics](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356) |
| The floor can use captured main-fork bytes, while page counts depend on build-time `BLCKSZ` | [pgstatindex.c#captured-length](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L271), [configure.in#block-size](../../../raw/postgres-12/configure.in#L247-L265), [guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888) |
| `AccessShareLock` permits concurrent DML and lazy VACUUM index maintenance | [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105), [execIndexing.c#ExecOpenIndices](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L141-L213), [vacuumlazy.c#index-locks](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L275-L292) |
| Proposed local validation checks page shape, not whole-tree integrity | [nbtpage.c#_bt_getmeta](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148), [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720) |
| The C implementation can retain a bounded bulk-read ring | [freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L587) |
| The extension needs a new update script, control bump, Makefile data entry, docs, and tests | [Makefile#module-and-tests](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24), [pgstattuple.control#default-version](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) |
| `BTREE_AM_OID` is an existing generated-header dependency; no new catalog is needed | [pg_am.h#generated-header](../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29), [pg_am.dat#BTREE_AM_OID](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23), [genbki.pl#OID-symbol-emission](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603) |
| The pageinspect compatibility helper avoids v1.7 `oldest_xact` conversion | [pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26), [btreefuncs.c#bt_metap-output](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L505-L575), [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2152) |
| The revised SQL derives valid-page capacity from one `bt_page_stats` read | [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153), [nbtpage.c#_bt_pageinit](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L920-L929), [bufpage.c#PageInit](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L35-L60) |
| VACUUM removes dead entries and can turn empty leaves into deleted physical pages | [vacuumlazy.c#remove-index-entries](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L761-L772), [nbtree.c#empty-page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1327-L1359), [nbtpage.c#_bt_pagedel](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1285-L1517) |
| A partial predicate filters build and DML inputs, while `pgstatindex` reads the resulting physical B-tree | [heapam_handler.c#partial-build-filter](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626), [execIndexing.c#partial-DML-filter](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L363), [pgstatindex.c#guards-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L315) |
| `setseed()` makes subsequent current-session `random()` calls repeatable | [func.sgml#setseed](../../../raw/postgres-12/doc/src/sgml/func.sgml#L1130-L1159), [float.c#setseed](../../../raw/postgres-12/src/backend/utils/adt/float.c#L2585-L2644) |
| The existing empty-index boundary is zero ordinary pages and `NaN` ratios | [pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82) |
| Existing tests do not cover sampling | [Makefile#regression-target](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24), [pgstattuple.sql#complete-test](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119) |

## Open Questions

- Would PostgreSQL accept this API and the fixed 100-MiB/10% policy, and should
  estimated fields include standard errors or confidence intervals? The pinned
  source cannot decide project policy.
- Should the production API expose a seed for repeatability and add explicit
  requested/effective-fraction fields, even though `sampled_pages` and
  `scanned_percent` already expose the realized work?
- Is a fixed 10% floor below 100 MiB preferable to a minimum sampled-page or
  sampled-leaf count? The rerun removed leaf-free draws in the tested small
  index and reduced several errors, but these fixtures do not establish a
  universal minimum. The strict threshold also creates a cost discontinuity at
  exactly 100 MiB.
- Should independently rounded page-class estimates be allowed to sum a few
  pages above or below `N`, should the API return non-integer estimates, or
  should it apply a total-preserving rounding rule?
- Is v12 `BlockSampler`'s fixed-state but potentially O(`N`) skip loop acceptable
  for the intended fractions and index sizes, or should the patch use a wider
  skip-ahead algorithm with explicit interrupt points?
- Should the new function deliberately harden metapage and sampled-page
  validation, as proposed here, even though that makes `sample_fraction = 1`
  stricter than current `pgstatindex` on malformed storage?
- The SQL prototype is intentionally not production-equivalent. It returns no
  row for invalid fractions and over-limit indexes, uses superuser-only
  `pageinspect`, lacks one continuous relation lock and the bulk-read ring, and
  performs O(`N`) candidate generation plus a random-key sort.

## Source References

- [pgstatindex.c#entry-points-and-structures](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L132) - fmgr registrations, AM tests, and accumulators.
- [pgstatindex.c#wrappers](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L216) - legacy and v1.5 text/regclass wrappers.
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365) - guards, metadata, full scan, formulas, and result construction.
- [pgstatapprox.c#statapprox_heap](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L208) - local approximate-output and scan-percentage precedent.
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L136) - v1.5 SQL bindings, attributes, and grants.
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) and [Makefile#pgstattuple](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24) - current extension version, scripts, objects, and test target.
- [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L295) - output and page-by-page caveat.
- [pgstattuple.sql#complete-test](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119) and [pgstattuple.out#results](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L1-L248) - complete current regression surface.
- [sampling.h#sampling-API](../../../raw/postgres-12/src/include/utils/sampling.h#L13-L65) and [sampling.c#BlockSampler](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L112) - sampler state and Algorithm S.
- [analyze.c#block-sampling](../../../raw/postgres-12/src/backend/commands/analyze.c#L1019-L1049) - in-core v12 caller.
- [block.h#BlockNumber](../../../raw/postgres-12/src/include/storage/block.h#L17-L35) - physical block-number type and range.
- [nbtree.h#page-structures](../../../raw/postgres-12/src/include/access/nbtree.h#L29-L135) - opaque/metapage layouts, flags, and versions.
- [nbtree.h#page-state-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L181-L196) - classification macros.
- [nbtpage.c#metapage-checks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148) and [nbtpage.c#ordinary-page-checks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720) - proposed validation model.
- [nbtpage.c#_bt_pageinit](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L920-L929) and [bufpage.c#PageInit](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L35-L60) - valid B-tree page capacity used by the one-read SQL prototype.
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597) - density free-space term.
- [freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L587) - bulk-read ring size.
- [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105) - relation-lock compatibility.
- [extension.c#install-path-selection](../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400) and [extension.c#install-then-update](../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550) - extension installation chain.
- [pg_am.h#generated-header](../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29), [pg_am.dat#BTREE_AM_OID](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23), and [genbki.pl#OID-symbol-emission](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603) - generated AM OID boundary.
- [pageinspect.control](../../../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5) and [pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26) - default pageinspect version and metapage SQL shape.
- [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153), [btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L237), and [btreefuncs.c#bt_metap](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L505-L584) - SQL prototype primitives.
- [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2152) - compatibility-wrapper tuple conversion.
- [vacuumlazy.c#index-cleanup](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L289), [vacuumlazy.c#remove-index-entries](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L761-L772), and [nbtree.c#btvacuumpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1359) - manual index cleanup, tuple removal, and empty-leaf handling.
- [nbtpage.c#_bt_pagedel](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1285-L1517) - B-tree half-dead and deleted-page transition.
- [heapam_handler.c#partial-build-filter](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1198-L1210) and [heapam_handler.c#partial-build-scan](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) - partial-index build predicate setup and filtering.
- [execIndexing.c#partial-DML-filter](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L363) - partial-index maintenance predicate.
- [func.sgml#setseed](../../../raw/postgres-12/doc/src/sgml/func.sgml#L1130-L1159) and [float.c#random-and-setseed](../../../raw/postgres-12/src/backend/utils/adt/float.c#L2585-L2644) - repeatable current-session SQL sampling sequence.
- [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L335) and [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891) - relation-size implementation and main-fork overload.
- [configure.in#block-size](../../../raw/postgres-12/configure.in#L247-L265), [guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888), and [config.sgml#block_size](../../../raw/postgres-12/doc/src/sgml/config.sgml#L9070-L9085) - build-time page-size choices and reported compiled value.
- [gram.y#CREATE-INDEX](../../../raw/postgres-12/src/backend/parser/gram.y#L7335-L7397) and [gram.y#VACUUM](../../../raw/postgres-12/src/backend/parser/gram.y#L10497-L10533) - fixture DDL grammar.
- [vacuum.sgml#INDEX_CLEANUP](../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L186-L203) and [create_table.sgml#autovacuum_enabled](../../../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L1383-L1403) - fixture cleanup and routine-autovacuum controls.
- [guc.c#diagnostic-timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2397) - session/transaction timeout contexts.

## Navigation

- [PostgreSQL 12 index](../index.md)
- [How `pgstatindex` Calculates B-Tree Index Statistics in PostgreSQL 12](how-pgstatindex-calculates-information.md)
- [PostgreSQL versions](../../versions.md)
