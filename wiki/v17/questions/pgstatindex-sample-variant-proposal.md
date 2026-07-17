---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-17T19:25:39Z
---

# Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)

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
  - [Sampler](#sampler)
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

In PostgreSQL 17, propose a `pgstatindex` variant that samples the index
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
100 MiB, raise a requested fraction below 10% to an effective 10%. At exactly
100 MiB or above, preserve the request. A metapage-only index still samples no
ordinary pages. Report the realized sample through `sampled_pages`,
`sampled_leaf_pages`, and `scanned_percent`, and prefix every extrapolated field
with `approx_`.

PostgreSQL 17 already has the preferred first implementation building block:
`BlockSampler`. It implements Knuth's Algorithm S, samples without replacement,
and emits selected block numbers in ascending order. Its initialization now
takes a `uint32` seed, stores `pg_prng_state`, and returns the number of blocks
that iteration will produce
([sampling.h#BlockSamplerData](../../../raw/postgres-17/src/include/utils/sampling.h#L16-L42),
[sampling.c#BlockSampler](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L23-L116)).
`ANALYZE` uses the same sampler, supplies a seed from PostgreSQL's global PRNG,
and feeds the ascending block sequence to a read stream
([analyze.c#block-sampling](../../../raw/postgres-17/src/backend/commands/analyze.c#L1110-L1122),
[analyze.c#acquire_sample_rows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1167-L1204)).

Use `BlockSampler` as a starting point, not as an unqualified final choice. Its
sample-size and selected-count fields are `int`, while its population and
position fields are `BlockNumber`. Its skip loop can advance across many
candidate blocks without an interrupt check inside one `BlockSampler_Next`
call. A full-sample fast path, a wider partial-sample implementation or error,
and explicit seed/cancellation contracts are therefore still needed
([sampling.h#BlockSamplerData](../../../raw/postgres-17/src/include/utils/sampling.h#L27-L42),
[sampling.c#skip-loop](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116),
[block.h#BlockNumber](../../../raw/postgres-17/src/include/storage/block.h#L17-L35)).

The SQL implementation below is a diagnostic prototype. It proves that v17
`pageinspect` can reproduce the estimator, but it is not production-equivalent:
it is superuser-only, enumerates and random-sorts every candidate block number,
does not hold one relation lock across the whole run, and uses normal buffer
reads instead of `pgstatindex`'s bulk-read ring
([pageinspect.sgml#overview](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L13),
[btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L320),
[pgstatindex.c#buffer-strategy](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L223)).

### Current `pgstatindex` baseline

The v1.5 SQL definitions bind `pgstatindex(text)` and
`pgstatindex(regclass)` to separate wrappers. Both wrappers open the relation
with `AccessShareLock` and call `pgstatindex_impl`; SQL `REVOKE` and `GRANT`
statements, rather than inline checks in those wrappers, provide the current
privilege boundary
([pgstattuple--1.4--1.5.sql#pgstatindex-text](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstatindex.c#wrappers](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L144-L213)).

`pgstatindex_impl` then:

1. accepts only a physical `RELKIND_INDEX` whose access method is
   `BTREE_AM_OID`, rejects another session's temporary relation, and rejects any
   index with `indisvalid = false`
   ([pgstatindex.c#guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L250));
2. reads block 0 without a buffer content lock and copies `btm_version`,
   `btm_level`, and `btm_root` from `BTMetaPageData`
   ([pgstatindex.c#metapage-read](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L265),
   [nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L97-L122));
3. captures `RelationGetNumberOfBlocks(rel)` once, then visits every captured
   non-metapage block with `CHECK_FOR_INTERRUPTS`, `BAS_BULKREAD`, and a shared
   buffer content lock
   ([pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331),
   [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3971-L4003));
4. classifies each block as deleted, ignored/half-dead (`empty_pages`), live
   leaf, or internal from `BTPageOpaqueData` flags
   ([pgstatindex.c#classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L326),
   [nbtree.h#page-state-flags](../../../raw/postgres-17/src/include/access/nbtree.h#L62-L84)); and
5. closes the relation, formats ten strings, converts them through the caller's
   composite tuple descriptor, and returns the record
   ([pgstatindex.c#result-construction](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L333-L381)).

`BTIndexStat` holds the three metapage values, four page-class counters, summed
leaf capacity and free space, and the backward-link count
([pgstatindex.c#BTIndexStat](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L75-L95)).
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

A successful full loop assigns every captured ordinary page to exactly one
class, so the size formula equals the captured main-fork block count times
`BLCKSZ`. The code formats both ratios to two decimal places and emits `NaN`
when no live-leaf denominator exists
([pgstatindex.c#classification-and-results](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L372),
[bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L900-L916)).
The checked-in empty-index result has one metapage, zero page counters, and two
`NaN` values
([pgstattuple.out#empty-index](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

Do not call those outputs transactionally exact. The documentation says that
the function accumulates page-by-page rather than taking an instantaneous
whole-index snapshot. The implementation also reads the metapage without a
content lock, captures length once, and locks ordinary pages one at a time
([pgstattuple.sgml#snapshot-limit](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L279),
[pgstatindex.c#metapage-and-scan](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L331)).
For this proposal, **direct** means observed without sample extrapolation; it
does not imply one common observation time.

### Proposed API and field contract

The output contract should expose both the work performed and the number of
sampled leaves behind the ratio estimates.

| Output | Proposed source | Contract |
|---|---|---|
| `version`, `tree_level`, `root_block_no` | one share-locked, validated metapage observation | direct, not extrapolated |
| `index_size` | captured `nblocks * BLCKSZ` for the main fork | direct at length-capture time |
| `sampled_pages` | ordinary blocks actually inspected | direct |
| `sampled_leaf_pages` | sampled blocks classified as live leaves | direct |
| `scanned_percent` | `100 * sampled_pages / (nblocks - 1)` | direct description of the realized sample |
| `approx_internal_pages` | sampled internal count scaled by inverse inclusion fraction | estimate |
| `approx_leaf_pages` | sampled live-leaf count scaled by inverse inclusion fraction | estimate |
| `approx_empty_pages` | sampled half-dead count scaled by inverse inclusion fraction | estimate |
| `approx_deleted_pages` | sampled deleted count scaled by inverse inclusion fraction | estimate |
| `approx_avg_leaf_density` | ratio of sampled leaf free-space and capacity sums | estimate |
| `approx_leaf_fragmentation` | sampled backward-right-link leaves divided by sampled live leaves | estimate |

The input remains the **requested** fraction. The size rule can raise it, while
`sampled_pages` and `scanned_percent` expose the realized work. Whole-page
rounding can make the realized percentage exceed 10%; for example, 50 sampled
pages out of 495 ordinary pages is 10.1010%. The threshold compares captured
main-fork bytes, including the metapage, with integer `104857600`; it does not
compare formatted size text.

`pgstattuple_approx` provides the local naming precedent: it publishes
`scanned_percent` and `approx_` fields instead of weakening the full function's
result names
([pgstatapprox.c#output_type](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L38-L50),
[pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119)).

Proposed extension DDL:

```sql
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

This DDL is a proposal, not an object in the pinned checkout. The default is a
requested 1%; the size policy makes its effective value 10% below 100 MiB.
`STRICT` follows the existing null-input contract, and the grant follows the
v1.5 permission model
([pgstattuple--1.4--1.5.sql#function-attributes-and-grants](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119)).
`VOLATILE` is explicit because different calls can draw different samples; it
is also PostgreSQL's default when no volatility category is written
([create_function.sgml#volatility-default](../../../raw/postgres-17/doc/src/sgml/ref/create_function.sgml#L305-L343)).
`PARALLEL RESTRICTED` keeps execution in the parallel leader. This follows
v17's correction for relation-reading `pageinspect` functions, which otherwise
fail on temporary relations in parallel workers
([pageinspect--1.10--1.11.sql#parallel-restriction](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.10--1.11.sql#L6-L23),
[create_function.sgml#parallel-categories](../../../raw/postgres-17/doc/src/sgml/ref/create_function.sgml#L432-L439)).

### C call path and data structures

The proposed normal path is:

```text
pgstatindex_approx(regclass, float8)
  -> relation_open(oid, AccessShareLock)
  -> existing B-tree, temporary-relation, and indisvalid guards
  -> reject nonfinite or !(0 < sample_fraction <= 1)
  -> share-lock and validate block 0; copy true-root metadata
  -> RelationGetNumberOfBlocks(rel) once
  -> index_size = (uint64) nblocks * BLCKSZ
  -> effective_fraction = index_size < 100 MiB
       ? max(sample_fraction, 0.10) : sample_fraction
  -> compute sample_target from effective_fraction
  -> full ordinary scan when sample_target == nblocks - 1
     otherwise BlockSampler_Init(nblocks - 1, sample_target, uint32_seed)
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

The relation guard should remain aligned with v17 `pgstatindex_impl`. That
preserves its physical-B-tree-only boundary, permits a current-session temporary
B-tree, rejects another session's local-buffer relation, and rejects invalid
indexes before reading their pages
([pgstatindex.c#wrappers-and-guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L169-L250)).
The existing wrappers rely on SQL function `EXECUTE` privilege rather than
relation ownership or `SELECT` checks. Granting the proposal to
`pg_stat_scan_tables` deliberately preserves that diagnostic privilege boundary
([pgstattuple--1.4--1.5.sql#grant-model](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119),
[pgstatindex.c#v1.5-wrapper](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L203-L250)).

The implementation can refactor the current `BTIndexStat` page classifier and
add `BlockSamplerData`, captured population and sample counts, and the realized
leaf count. `BTMetaPageData` supplies metadata; `BTPageOpaqueData` supplies page
flags and `btpo_next`
([pgstatindex.c#BTIndexStat](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L75-L95),
[sampling.h#BlockSamplerData](../../../raw/postgres-17/src/include/utils/sampling.h#L25-L42),
[nbtree.h#page-structures](../../../raw/postgres-17/src/include/access/nbtree.h#L31-L122)).

Do not copy the exact function's weakest integrity behavior merely for code
reuse. The new function should content-lock block 0 and verify `BTP_META`,
`BTREE_MAGIC`, and the supported version range. Core's `_bt_getmeta` performs
those checks but is file-local, so a contrib patch must duplicate them or expose
a suitable core helper
([nbtpage.c#_bt_getmeta](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L134-L170)).
Apply `_bt_checkpage` to every sampled ordinary block before dereferencing its
special area. That catches all-zero pages and a wrong B-tree special-space size;
it does not validate unsampled pages or whole-tree invariants
([nbtpage.c#_bt_checkpage](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825)).

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

Reject any requested fraction that is nonfinite or does not satisfy
`0 < f <= 1`, before applying the floor. The 10% minimum is an explicit policy,
not a hidden replacement: the input retains the request, while sample counts and
`scanned_percent` report what was read. A request of at least 10% is unchanged.
Exactly 100 MiB is unchanged because the comparison is strict `<`, and the
`N = 0` branch takes precedence for an empty index.

The byte threshold maps to different page counts when PostgreSQL is built with a
nondefault `BLCKSZ`. PostgreSQL 17 accepts 1, 2, 4, 8, 16, or 32 KiB at configure
time, and exposes the compiled value as an internal setting
([configure.ac#block-size](../../../raw/postgres-17/configure.ac#L259-L289),
[guc_tables.c#block_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276)).

Initialize `BlockSampler` with population `N`; each returned value is in
`[0, N - 1]`, so add one before reading the index. Algorithm S samples without
replacement. `bs->t` only increases, so selected blocks arrive in ascending
physical order without a sampled-block array or sort
([sampling.c#BlockSampler-contract](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L23-L60),
[sampling.c#BlockSampler-iteration](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116)).

The native sampler uses fixed state and at most one PRNG draw per selected block,
but its inner loop still advances over skipped block numbers. One call can
therefore do work proportional to the population without reaching the caller's
next `CHECK_FOR_INTERRUPTS`. Also, `n` and `m` are signed `int` while a B-tree
population uses 32-bit unsigned block numbers. The implementation should bypass
the sampler for a full scan; a partial target above `INT_MAX` needs a wider
sampler, an explicit error, or a lower request
([sampling.h#sample-size-types](../../../raw/postgres-17/src/include/utils/sampling.h#L27-L42),
[sampling.c#skip-loop](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116),
[block.h#BlockNumber](../../../raw/postgres-17/src/include/storage/block.h#L17-L35)).

For each exhaustive page class `c`, use the unrounded estimator:

```text
estimated_pages(c) = sampled_pages(c) * N / k
```

Every captured ordinary block has the same inclusion probability. Integer
outputs require rounding, so four independently rounded class estimates can sum
a few pages above or below `N`; `index_size` must remain independent of those
rounded estimates.

For sampled live leaves, preserve the current ratio-of-sums definitions:

```text
approx_avg_leaf_density =
  100 - sum(PageGetFreeSpace(page))
        / sum(pd_special - SizeOfPageHeaderData) * 100

approx_leaf_fragmentation =
  sampled_backward_right_links / sampled_leaf_pages * 100
```

These are leaf-page estimates, not count-scaled values. Format non-`NaN` ratios
to two decimals to preserve the current convention. Return `NaN` for both when
no live leaf is sampled, and report `sampled_leaf_pages` so callers can see
whether a ratio rests on many leaves or very few
([pgstatindex.c#leaf-metrics-and-zero-cases](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L372)).

### Locking, concurrency, and integrity

The C implementation should hold one `AccessShareLock` from relation open to
relation close. In v17 that lock conflicts only with `AccessExclusiveLock`.
Ordinary DML and lazy VACUUM open indexes with `RowExclusiveLock`, so inserts,
deletes, splits, page deletion/reuse, and VACUUM can overlap the diagnostic
([lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L60-L103),
[execIndexing.c#ExecOpenIndices](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L188-L204),
[vacuumlazy.c#index-locks](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L356-L361)).

The result remains a mixed-time physical observation:

- The proposed metapage content lock protects that one observation, but is
  released before ordinary pages are read.
- `nblocks` is captured once. Blocks appended later are outside both the sample
  population and reported size.
- Each sampled page has a shared content lock only while it is validated and
  classified. It can change or be recycled before or after that observation.
- Sampling widens the statistical gap; it does not remove the current
  no-whole-index-snapshot boundary
  ([pgstattuple.sgml#snapshot-limit](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L279),
  [pgstatindex.c#one-time-length-and-page-locks](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).

Reuse `BAS_BULKREAD` for sampled page reads. In v17 it uses a nominal 256 KiB
ring capped at one eighth of `shared_buffers`; this limits replacement impact
even though sparse sampling does not become a dense sequential scan
([freelist.c#GetAccessStrategy](../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L536-L613)).

A validated sample is still not an integrity check. Unsampled blocks are never
inspected, and `_bt_checkpage` checks only for an all-zero page and the expected
special-space size. It does not verify parent links, sibling reciprocity, key
order, or tuple-to-heap correspondence
([nbtpage.c#_bt_checkpage](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825)).
With `sample_fraction = 1`, a healthy quiescent index should reproduce
`pgstatindex`; deliberate validation can make the new function stricter on
malformed storage.

### Extension, build, and generated-file implications

PostgreSQL 17's `pgstattuple` control file selects version 1.5. Both Make and
Meson install the 1.4 base script and the 1.4-to-1.5 update script. If no direct
install script exists for the target version, extension creation finds an
installable base and applies an update path
([pgstattuple.control#default-version](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5),
[Makefile#extension-scripts](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L3-L16),
[meson.build#extension-scripts](../../../raw/postgres-17/contrib/pgstattuple/meson.build#L15-L31),
[extension.c#install-path-selection](../../../raw/postgres-17/src/backend/commands/extension.c#L1614-L1668)).

A complete in-tree patch should:

1. add `PG_FUNCTION_INFO_V1(pgstatindex_approx)` and its C symbol, preferably by
   refactoring the current classifier;
2. add `pgstattuple--1.5--1.6.sql` to both `DATA` and Meson's `install_data`, and
   set `default_version = '1.6'`;
3. update `pgstattuple.sgml`;
4. extend `sql/pgstattuple.sql` and `expected/pgstattuple.out`; and
5. add an object to Make's `OBJS` and Meson's `pgstattuple_sources` only if the
   implementation uses a new C file
   ([pgstatindex.c#fmgr-registration](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L47-L68),
   [Makefile#module-and-tests](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L3-L26),
   [meson.build#module-and-tests](../../../raw/postgres-17/contrib/pgstattuple/meson.build#L3-L42)).

The existing C file has a generated-header dependency: `pgstatindex.c` includes
`pg_am.h`, which includes generated `pg_am_d.h`; `pg_am.dat` supplies
`BTREE_AM_OID`, and `genbki.pl` emits the macro. The proposal reuses that
dependency and adds the ordinary `utils/sampling.h` API. It needs no new core
catalog, parser rule, generated header, or index-AM callback
([pgstatindex.c#includes](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L28-L44),
[pg_am.h#generated-header](../../../raw/postgres-17/src/include/catalog/pg_am.h#L18-L29),
[pg_am.dat#BTREE_AM_OID](../../../raw/postgres-17/src/include/catalog/pg_am.dat#L15-L23),
[genbki.pl#OID-symbol-emission](../../../raw/postgres-17/src/backend/catalog/genbki.pl#L675-L686),
[sampling.h#API](../../../raw/postgres-17/src/include/utils/sampling.h#L13-L42)).

This remains a private B-tree-layout diagnostic. It reads
`BTMetaPageData` and `BTPageOpaqueData` directly rather than dispatching through
an access-method callback. The C implementation has no runtime dependency on
`pageinspect`; that separate contrib module is only for the SQL prototype
([pgstatindex.c#includes-and-AM-test](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L28-L73)).

## Pros

- **It reduces index-page reads.** The C path reads one metapage plus `k`
  selected ordinary pages instead of all captured ordinary pages
  ([pgstatindex.c#full-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).
- **It preserves direct metadata and size observations.** Metapage fields do not
  need extrapolation, and captured main-fork size comes from `nblocks * BLCKSZ`
  rather than rounded sample counts
  ([pgstatindex.c#metadata-and-length](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L283)).
- **It keeps the current metric definitions.** Density still uses
  `PageGetFreeSpace`; fragmentation still tests physical right-link direction
  ([pgstatindex.c#leaf-metrics](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L323)).
- **The native sampler needs no sampled-block array.** `BlockSamplerData` has
  fixed state, and Algorithm S emits selected blocks in ascending order
  ([sampling.h#BlockSamplerData](../../../raw/postgres-17/src/include/utils/sampling.h#L27-L42),
  [sampling.c#BlockSampler_Next](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116)).
- **The result advertises approximation.** `approx_` names, realized scan
  percentage, and direct sample counts reduce the chance that an estimate is
  mistaken for a full scan
  ([pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L102-L119)).
- **The 10% floor avoids extremely small draws below 100 MiB.** In the v17
  rerun, it raised the 3.88 MiB partial index from 5 to 50 pages for a requested
  1%, and raised the 0.66 MiB healthy index from 1 to 9 pages, eliminating the
  latter fixture's leaf-free draws. `scanned_percent` exposes that extra work.
- **The C path can retain bounded buffer replacement.** Reusing
  `BAS_BULKREAD` retains the 256 KiB-or-smaller ring
  ([freelist.c#GetAccessStrategy](../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L536-L613)).

## Cons

- **Rare classes are easy to miss.** Internal, half-dead, and deleted pages can
  be a small part of the population. A zero sample count yields zero even when
  the class exists; the full scanner distinguishes all four states by visiting
  every captured block
  ([pgstatindex.c#classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L326)).
- **Leaf ratios can be unsupported.** A sample with no live leaf returns `NaN`;
  a sample with very few leaves has little evidence. `sampled_leaf_pages` makes
  that boundary visible
  ([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
- **Physical clustering increases sampling risk.** The sampler chooses physical
  blocks, while the metrics classify those physical positions and right links.
  A small draw can miss a region containing unusual page states or backward
  links
  ([pgstatindex.c#physical-loop-and-links](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).
- **The floor overrides caller cost control.** Below 100 MiB, requested 1% and
  5% both read at least 10%. At exactly 100 MiB, the same 1% request reads about
  1%, creating a sharp cost discontinuity. A fixed byte threshold also maps to
  different page counts under nondefault `BLCKSZ`
  ([configure.ac#block-size](../../../raw/postgres-17/configure.ac#L259-L289)).
- **More pages do not force every finite seeded error summary to improve.** In
  this v17 run, the partial fixture's 1% leaf MAPE moved from 0.80% to 0.81%,
  even though the sample grew tenfold. The floor supplies more observations; it
  is not a deterministic guarantee for every fixed set of random draws.
- **Sparse ascending reads are not a dense sequential walk.** They retain
  ordered block numbers and a bulk-read ring, but gaps can reduce the locality
  of the current block-by-block scan
  ([pgstatindex.c#physical-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).
- **The native sampler has scale and cancellation limits.** Its sample count is
  `int`, and its skip loop can do work proportional to the population before
  returning control
  ([sampling.h#BlockSamplerData](../../../raw/postgres-17/src/include/utils/sampling.h#L27-L42),
  [sampling.c#skip-loop](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116)).
- **It remains a mixed-time observation.** DML and VACUUM can overlap an
  `AccessShareLock`, length is captured once, and page locks are not held
  together
  ([lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L60-L103),
  [pgstattuple.sgml#snapshot-limit](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L279)).
- **It cannot certify integrity.** Validation covers only sampled blocks and
  local metapage/page-shape checks
  ([nbtpage.c#metapage-checks](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L134-L170),
  [nbtpage.c#ordinary-page-checks](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825)).
- **It adds a long-lived API and maintenance surface.** The patch needs C, SQL
  upgrade, Make and Meson wiring, documentation, permissions, and regression
  changes; the current target contains no sampling test
  ([Makefile#module-and-tests](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L3-L26),
  [pgstattuple.sql#complete-test](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138)).

## SQL Prototype Using `contrib/pageinspect`

### Prototype boundaries

A fresh PostgreSQL 17 installation selects `pageinspect` 1.12. Its current
`bt_metap(text)` result uses 64-bit root/level fields, while
`bt_page_stats(text, int8)` accepts 64-bit block arguments
([pageinspect.control#default-version](../../../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5),
[pageinspect--1.8--1.9.sql#bt_metap](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L69-L84),
[pageinspect--1.8--1.9.sql#bt_page_stats](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L86-L103)).
No v12-style metapage compatibility wrapper is needed.

`bt_page_stats` reports `d` for a deleted leaf or old-format deleted page, `D`
for a deleted internal page with a full transaction ID, `e` for an ignored/
half-dead page, `l` for a live leaf, `r` for an internal root, and `i` for
another internal page. The prototype combines `d` and `D`, matching
`pgstatindex`'s single deleted-page class
([btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L101-L194),
[pgstatindex.c#classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L326)).

The page-stat result already includes `page_size` and `free_size`. A valid v17
B-tree page reserves a 24-byte fixed header and a 16-byte
`BTPageOpaqueData` special area, so `page_size - 40` equals the current density
capacity term. The sampler therefore reads each selected ordinary page once;
it does not need a second `get_raw_page`/`page_header` call
([nbtpage.c#_bt_pageinit](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1123-L1132),
[bufpage.c#PageInit](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L35-L60),
[bufpage.h#SizeOfPageHeaderData](../../../raw/postgres-17/src/include/storage/bufpage.h#L155-L214),
[nbtree.h#BTPageOpaqueData](../../../raw/postgres-17/src/include/access/nbtree.h#L62-L69)).

The remaining limits are deliberate:

- `ORDER BY random() LIMIT k` avoids unselected index-page reads, but still
  generates every candidate and performs a random-key sort. SQL `random()` is
  volatile and parallel-restricted
  ([pg_proc.dat#random](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L3377-L3398)).
- Below 100 MiB, requested fractions below 10% produce the same target after
  whole-page rounding. The floor increases sampled page reads but does not
  reduce the SQL prototype's O(`N`) candidate generation and sort.
- Every `bt_page_stats` call opens and closes the relation around one normal
  buffer read. `bt_metap` and `pg_relation_size` make separate opens. The SQL
  function therefore lacks one continuous relation lock and `BAS_BULKREAD`
  ([btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L320),
  [btreefuncs.c#bt_metap](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L839-L939),
  [dbsize.c#pg_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L370)).
- `bt_page_stats` does not call `_bt_checkpage`; the capacity derivation and
  classifier assume valid initialized B-tree pages
  ([btreefuncs.c#page-read](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L280-L295),
  [nbtpage.c#_bt_checkpage](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825)).
- The prototype permits at most 4,294,967,295 total blocks so its largest
  ordinary block is `MaxBlockNumber`. `bt_page_stats` performs the same
  `int64`-to-`BlockNumber` and relation-range checks
  ([btreefuncs.c#block-range](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L196-L250),
  [block.h#BlockNumber](../../../raw/postgres-17/src/include/storage/block.h#L17-L35)).
- Invalid fractions and over-limit indexes return no row from this SQL helper.
  The proposed C API must raise explicit errors.
- The helper is a user-created object, not an extension member. Remove it when
  the diagnostic is no longer needed.

### Setup

Both timeout settings are `PGC_USERSET`, so these changes are session-scoped and
need neither reload nor restart. The five-minute statement timeout bounds one
sampling statement; the two-second lock timeout bounds each relation-lock wait
([guc_tables.c#diagnostic-timeouts](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2610-L2629)).

```sql
SET /* wiki_pgstatindex_pageinspect_timeout */ statement_timeout = '5min';
SET /* wiki_pgstatindex_pageinspect_timeout */ lock_timeout = '2s';
CREATE /* wiki_pgstatindex_pageinspect_setup */ EXTENSION IF NOT EXISTS pageinspect;

SELECT /* wiki_pgstatindex_pageinspect_schema */
       n.nspname AS pageinspect_schema,
       e.extversion AS pageinspect_version
FROM pg_extension AS e
JOIN pg_namespace AS n ON n.oid = e.extnamespace
WHERE e.extname = 'pageinspect';

SET /* wiki_pgstatindex_pageinspect_path */ search_path = public, pg_catalog;
```

Require `pageinspect_version = 1.12`; `CREATE EXTENSION IF NOT EXISTS` does not
upgrade an older installed extension. Replace `public` with the returned
extension schema when needed. `pageinspect` is relocatable, and
`SET search_path FROM CURRENT` below saves this path for later calls
([pageinspect.control#relocatable](../../../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5),
[extension.c#IF-NOT-EXISTS](../../../raw/postgres-17/src/backend/commands/extension.c#L1908-L1926),
[create_function.sgml#configuration-parameter](../../../raw/postgres-17/doc/src/sgml/ref/create_function.sgml#L508-L531)).

### Sampler

The `_v17` suffix avoids colliding with the older prototype previously shown on
this page. PostgreSQL does not allow `CREATE OR REPLACE FUNCTION` to change a
record shape defined by `OUT` parameters
([pg_proc.c#record-return-replacement](../../../raw/postgres-17/src/backend/catalog/pg_proc.c#L397-L449)).

```sql
CREATE /* wiki_pgstatindex_pageinspect_function */ OR REPLACE FUNCTION pgstatindex_approx_pageinspect_v17(
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
  WHERE index_size / current_setting('block_size')::int
        <= 4294967295::bigint
),
sample_plan AS MATERIALIZED (
  SELECT *,
       CASE
         WHEN nblocks <= 1 THEN 0::bigint
         ELSE LEAST(nblocks - 1,
                    GREATEST(1::bigint,
                             ceil((nblocks - 1)::float8
                                  * effective_fraction)::bigint))
       END AS sample_target
  FROM bounds
),
candidate_blocks AS MATERIALIZED (
  SELECT p.idx, p.nblocks, p.index_size, p.sample_target, b.blkno
  FROM sample_plan AS p
  CROSS JOIN LATERAL
    generate_series(1::bigint, p.nblocks - 1) AS b(blkno)
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
  CROSS JOIN LATERAL bt_page_stats(s.idx::text, s.blkno) AS ps
),
agg AS (
  SELECT count(*)::bigint AS sampled_blocks,
       count(*) FILTER (WHERE type IN ('i', 'r'))::bigint
         AS internal_sample,
       count(*) FILTER (WHERE type = 'l')::bigint AS leaf_sample,
       count(*) FILTER (WHERE type = 'e')::bigint AS empty_sample,
       count(*) FILTER (WHERE type IN ('d', 'D'))::bigint
         AS deleted_sample,
       coalesce(sum(CASE WHEN page_size > 40
                         THEN page_size - 40 ELSE 0 END)
         FILTER (WHERE type = 'l'), 0)::numeric AS leaf_max_avail,
       coalesce(sum(free_size) FILTER (WHERE type = 'l'), 0)::numeric
         AS leaf_free_space,
       count(*) FILTER (WHERE type = 'l'
         AND btpo_next <> 0
         AND btpo_next < blkno)::bigint AS fragments_sample
  FROM page_sample
)
SELECT /* wiki_pgstatindex_pageinspect_result */ m.version,
     m.level::int AS tree_level,
     p.index_size,
     m.root::bigint AS root_block_no,
     a.sampled_blocks AS sampled_pages,
     a.leaf_sample AS sampled_leaf_pages,
     CASE WHEN p.nblocks > 1
      THEN 100.0::float8 * a.sampled_blocks::float8
           / (p.nblocks - 1)::float8
      ELSE 0.0::float8
     END AS scanned_percent,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.internal_sample::numeric * (p.nblocks - 1)
                 / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_internal_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.leaf_sample::numeric * (p.nblocks - 1)
                 / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_leaf_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.empty_sample::numeric * (p.nblocks - 1)
                 / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_empty_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.deleted_sample::numeric * (p.nblocks - 1)
                 / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_deleted_pages,
     CASE WHEN a.leaf_max_avail > 0
      THEN round((100.0 - a.leaf_free_space / a.leaf_max_avail
                  * 100.0)::numeric, 2)::float8
      ELSE 'NaN'::float8
     END AS approx_avg_leaf_density,
     CASE WHEN a.leaf_sample > 0
      THEN round((100.0 * a.fragments_sample
                  / a.leaf_sample)::numeric, 2)::float8
      ELSE 'NaN'::float8
     END AS approx_leaf_fragmentation
FROM sample_plan AS p
CROSS JOIN LATERAL bt_metap(p.idx::text) AS m
CROSS JOIN agg AS a;
$function$;
```

The materialized size CTE evaluates `pg_relation_size` once. Its one-argument
form selects the main fork
([pg_proc.dat#pg_relation_size](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7495),
[system_functions.sql#pg_relation_size](../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)).
The empty branch remains zero despite its size being below the floor. The output
reports the realized fraction, not a misleading copy of the request.

## Tests To Add

A production patch should add deterministic correctness tests and avoid brittle
assertions about one random 1% draw:

1. **Full-sample equivalence:** on a healthy, quiescent populated B-tree,
   `sample_fraction = 1` must match every shared `pgstatindex` field
   ([pgstatindex.c#full-loop-and-results](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L372)).
2. **Empty index:** a metapage-only index must return zero sample/count fields,
   zero `scanned_percent`, and `NaN` ratios
   ([pgstattuple.out#empty-index](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).
3. **Argument errors and null:** reject zero, negatives, values above one, NaN,
   and infinities before applying the floor; verify `STRICT` returns null for a
   null argument without entering C
   ([pgstattuple--1.4--1.5.sql#STRICT](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
4. **Size-floor boundaries:** below 100 MiB, a request below 10% must use
   `ceil(N * 0.10)` pages; a request of at least 10% must remain unchanged. At
   exactly 100 MiB and above, preserve the request. Cover whole-page rounding,
   an empty index, and real relations on both sides.
5. **Sampling invariants:** through a fixed-seed test path, selected blocks must
   be in range, unique, and ascending; direct sample fields must match the draw
   ([sampling.c#BlockSampler-iteration](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116)).
6. **No sampled leaf:** force a sample containing no live leaf and assert two
   `NaN` ratios plus `sampled_leaf_pages = 0`
   ([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
7. **Object and state boundaries:** cover a wrong access method, non-index and
   partitioned-index relation kinds, another-session and current-session
   temporary indexes, and v17's invalid-index rejection
   ([pgstatindex.c#guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L250)).
8. **Validation:** exercise an all-zero sampled page, wrong special-space size,
   bad metapage magic, and unsupported B-tree version; prove that an unsampled
   malformed page remains outside the claim
   ([nbtpage.c#metapage-checks](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L134-L170),
   [nbtpage.c#ordinary-page-checks](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825)).
9. **Concurrency and cancellation:** cover extension after length capture, page
   state changes during sampling, statement cancellation, and the long
   `BlockSampler_Next` skip-loop boundary
   ([pgstatindex.c#one-time-length-and-interrupts](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331),
   [sampling.c#skip-loop](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L63-L116)).
10. **Privileges and parallel placement:** test the `PUBLIC` revoke,
    `pg_stat_scan_tables` grant, and leader-only parallel marking
    ([pgstattuple--1.4--1.5.sql#grant-model](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119),
    [pageinspect--1.10--1.11.sql#parallel-restriction](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.10--1.11.sql#L6-L23)).
11. **Extension lifecycle:** test 1.5-to-1.6 update and a fresh default install
    through both Make and Meson builds
    ([extension.c#install-path-selection](../../../raw/postgres-17/src/backend/commands/extension.c#L1614-L1668),
    [meson.build#extension-data](../../../raw/postgres-17/contrib/pgstattuple/meson.build#L22-L31)).

The pinned tree declares one `pgstattuple` regression target. It checks the empty
`pgstatindex` result and several object/error paths, but contains no populated,
sampling, statistical, concurrency, malformed-page, or extension-upgrade test,
and no module isolation or TAP target
([Makefile#regression-target](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L10-L26),
[pgstattuple.sql#complete-test](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138)).
The comparison below exercises the SQL prototype, not a new C function or an
upstream test.

## Executed Comparison with Standard `pgstatindex`

The exact-pin PostgreSQL 17.10 run used six sub-50-MiB fixtures plus a
107.13 MiB healthy control. A full sample matched standard `pgstatindex` on
every shared field for all seven fixtures. Requested 1% and 5% samples both used
a 10% target below 100 MiB, while the larger control retained its requests.
Partial-sample accuracy still depended on page-class frequency and physical
distribution; standard `pgstatindex` obtains its counts by classifying every
captured ordinary block
([pgstatindex.c#full-scan-and-results](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L372)).

This tests the SQL estimator and random physical-block selection. It does not
benchmark the proposed C path, `BlockSampler`, the bulk-read ring, a continuous
relation lock, or stronger validation.

### Test scope and metrics

The run used the pinned PostgreSQL 17.10 checkout without concurrent fixture
DML. Quiescence matters because standard `pgstatindex` is a page-by-page
observation, not an instantaneous snapshot
([pgstattuple.sgml#snapshot-limit](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L279),
[pgstatindex.c#page-locking](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).

| Fixture | Construction | Boundary tested |
|---|---|---|
| `uniform_sparse` | Build 600,000 ordered keys; delete two of every three; `VACUUM` | Low leaf density spread across the index |
| `range_deleted` | Build 600,000 ordered keys; delete a contiguous 420,000-key middle range; `VACUUM` | Large deleted-page class next to dense survivors |
| `split_churn` | Create index empty; insert 600,000 deterministically permuted keys; delete one quarter; `VACUUM` | Low density plus backward-right-link fragmentation |
| `partial_sparse` | Build rows satisfying `WHERE active`; delete two thirds of indexed rows; `VACUUM` | Sparse partial B-tree |
| `small_dense` | Build 30,000 ordered keys without deletes | Healthy small-index control |
| `large_dense` | Build 5,000,000 ordered keys without deletes | Healthy control above 100 MiB |
| `empty` | Build an index on an empty table | Metapage-only edge |

`VACUUM (INDEX_CLEANUP ON)` requests index processing when dead tuples exist.
B-tree bulk deletion removes dead entries, then tries to delete a leaf that has
become empty; page deletion marks it half-dead before unlinking it, and physical
recycling can remain unsafe until later
([vacuum.sgml#INDEX_CLEANUP](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L186-L228),
[nbtree.c#btvacuumpage](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1229-L1385),
[nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1773-L1803)).
That path produced `range_deleted`'s deleted-page population; uniform deletes
left live tuples on each leaf and primarily lowered density.

A partial predicate filters heap tuples during index build and DML maintenance.
`pgstatindex` later scans the resulting physical B-tree without evaluating that
predicate
([heapam_handler.c#partial-build-filter](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1221-L1234),
[heapam_handler.c#partial-build-qual](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1631-L1644),
[execIndexing.c#partial-DML-filter](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L386),
[pgstatindex.c#guards-and-scan](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L331)).
The partial fixture therefore tests the same physical estimator over a smaller,
predicate-selected index. It does not estimate predicate selectivity or heap
bloat.

For every fixture, the run made one full-sample comparison and 100 draws at
requested fractions 0.01 and 0.05. It called `setseed()` before every partial
draw. PostgreSQL 17 documents that reissuing the same seed in one session
repeats subsequent `random()` results, and the implementation stores the PRNG
state in the backend
([func.sgml#setseed](../../../raw/postgres-17/doc/src/sgml/func.sgml#L1930-L1964),
[pseudorandomfuncs.c#setseed-and-random](../../../raw/postgres-17/src/backend/utils/adt/pseudorandomfuncs.c#L56-L93)).

The reported errors are:

```text
leaf MAPE (%) = mean(abs(approx_leaf_pages - leaf_pages) / leaf_pages) * 100
count MAPE (%) = mean(abs(estimate - full-scan count) / full-scan count) * 100
ratio MAE (pp) = mean(abs(sampled ratio - full-scan ratio))
```

`pp` means percentage points. Ratio errors omit draws with no sampled leaf; the
tables report those draws separately because both functions return `NaN` when
the leaf denominator is zero
([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).

### Disposable fixture

Run this only in a fresh disposable database. It intentionally creates and
bulk-deletes test data. The timeouts are session-scoped. Disabling routine
autovacuum prevents ordinary automatic maintenance from changing the fixtures
between manual `VACUUM` and measurement; anti-wraparound autovacuum remains an
exception
([create_table.sgml#autovacuum_enabled](../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1527-L1547)).
Run the prototype setup and sampler first.

```sql
SET /* wiki_pgstatindex_compare_timeout */ statement_timeout = '20min';
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
  (ANALYZE, INDEX_CLEANUP ON) samp_uniform_sparse;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_range_deleted
  (id int) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_range_deleted
SELECT i FROM generate_series(1, 600000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_range_deleted_idx
  ON samp_range_deleted (id);
DELETE /* wiki_pgstatindex_compare_bloat */ FROM samp_range_deleted
  WHERE id BETWEEN 90001 AND 510000;
VACUUM /* wiki_pgstatindex_compare_vacuum */
  (ANALYZE, INDEX_CLEANUP ON) samp_range_deleted;

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
  (ANALYZE, INDEX_CLEANUP ON) samp_split_churn;

CREATE /* wiki_pgstatindex_compare_table */ TABLE samp_partial_sparse
  (id int, active boolean) WITH (autovacuum_enabled = false);
INSERT /* wiki_pgstatindex_compare_load */ INTO samp_partial_sparse
SELECT i, i % 5 = 0 FROM generate_series(1, 900000) AS g(i);
CREATE /* wiki_pgstatindex_compare_index */ INDEX samp_partial_sparse_idx
  ON samp_partial_sparse (id) WHERE active;
DELETE /* wiki_pgstatindex_compare_bloat */ FROM samp_partial_sparse
  WHERE active AND id % 15 <> 0;
VACUUM /* wiki_pgstatindex_compare_vacuum */
  (ANALYZE, INDEX_CLEANUP ON) samp_partial_sparse;

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

The partial-index `WHERE` clause and parenthesized `VACUUM` option list are
present in the v17 grammar
([gram.y#CREATE-INDEX](../../../raw/postgres-17/src/backend/parser/gram.y#L8091-L8136),
[gram.y#VACUUM](../../../raw/postgres-17/src/backend/parser/gram.y#L11793-L11836)).

### Seeded comparison harness

The harness records a standard full-scan baseline, runs deterministic samples,
and asserts floor targets, strict synthetic threshold edges, and full-sample
equivalence. Its `sample_fraction` column stores the request;
`scanned_percent` stores the realized fraction.

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
       e.*
FROM samp_cases AS c
CROSS JOIN LATERAL pgstatindex(c.idx) AS e;

CREATE /* wiki_pgstatindex_compare_results */ TABLE samp_results AS
SELECT NULL::text AS case_name,
       NULL::float8 AS sample_fraction,
       NULL::int AS trial,
       s.*
FROM pgstatindex_approx_pageinspect_v17('samp_empty_idx', 1.0) AS s
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
        FROM pgstatindex_approx_pageinspect_v17(c.idx, f) AS s;
      END LOOP;
    END LOOP;
  END LOOP;
END
$block$;

SELECT /* wiki_pgstatindex_compare_floor_assertion */ bool_and(CASE
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
         abs(r.approx_leaf_fragmentation
             - e.leaf_fragmentation)::numeric END),
         2) AS fragmentation_mae_pp,
       round(max(CASE WHEN r.sampled_leaf_pages > 0 THEN
         abs(r.approx_leaf_fragmentation
             - e.leaf_fragmentation) END)::numeric,
         2) AS fragmentation_max_error_pp,
       count(*) FILTER (WHERE r.sampled_leaf_pages = 0) AS leaf_free_runs
FROM samp_exact AS e
JOIN samp_results AS r USING (case_name)
WHERE r.sample_fraction < 1.0
GROUP BY e.case_name, r.sample_fraction, e.leaf_pages, e.deleted_pages,
         e.avg_leaf_density, e.leaf_fragmentation
ORDER BY e.case_name, r.sample_fraction;
```

The before-floor comparison used the same function body with
`effective_fraction = sample_fraction`, the same fixture state, and the same 100
seeds. This isolates the policy's changed target from fixture and seed changes.

### Full-scan baseline

All six requested small fixtures were below both 50 MB and 50 MiB. The new
`large_dense` control was 107.13 MiB, so the rerun exercised both sides of the
100 MiB threshold. Standard `pgstatindex` returned:

| Fixture | Size (MiB) | Partial | Internal | Live leaf | Half-dead | Deleted | Density (%) | Fragmentation (%) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `empty` | 0.01 | no | 0 | 0 | 0 | 0 | `NaN` | `NaN` |
| `large_dense` | 107.13 | no | 50 | 13,662 | 0 | 0 | 90.08 | 0.00 |
| `partial_sparse` | 3.88 | yes | 3 | 492 | 0 | 0 | 30.21 | 0.00 |
| `range_deleted` | 12.88 | no | 4 | 494 | 0 | 1,149 | 89.69 | 0.00 |
| `small_dense` | 0.66 | no | 1 | 82 | 0 | 0 | 90.05 | 0.00 |
| `split_churn` | 18.25 | no | 12 | 2,323 | 0 | 0 | 47.82 | 49.98 |
| `uniform_sparse` | 12.88 | no | 7 | 1,640 | 0 | 0 | 30.21 | 0.00 |

The policy-target assertion and synthetic strict-edge assertion returned true.
The full-sample assertion returned `cases = 7` and
`all_full_samples_match = true`. For every fixture, the prototype matched the
standard version, level, size, root, four page-class counts, density, and
fragmentation. It reported every ordinary page and live leaf as sampled, with
100% scanned for populated indexes and 0% for the metapage-only index. The empty
result matches the checked-in v17 boundary
([pgstattuple.out#empty-index](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

This is equivalence on healthy, quiescent storage. It is not evidence that either
function returns a transactionally consistent snapshot, and it does not cover
the proposed C function's stricter malformed-page errors.

### Sampling results after the 10% floor

The following values summarize 100 seeded draws per fixture and requested
fraction. `Realized` is `scanned_percent`; whole-page rounding makes it slightly
exceed the target in most small fixtures. `mean / max` is absolute
percentage-point error. Deleted-count ranges and MAPE appear only where the full
count was nonzero.

| Fixture | Requested | Realized | Pages/draw | Leaf MAPE | Deleted estimate range (MAPE) | Density error, mean / max | Fragmentation error, mean / max | No-leaf draws |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `empty` | 1% | 0.0000% | 0 | n/a | n/a | n/a | n/a | 100 |
| `empty` | 5% | 0.0000% | 0 | n/a | n/a | n/a | n/a | 100 |
| `large_dense` | 1% | 1.0064% | 138 | 0.46% | n/a | 0.02 / 0.52 pp | 0.00 / 0.00 pp | 0 |
| `large_dense` | 5% | 5.0029% | 686 | 0.19% | n/a | 0.01 / 0.10 pp | 0.00 / 0.00 pp | 0 |
| `partial_sparse` | 1% | 10.1010% | 50 | 0.81% | n/a | 0.03 / 0.11 pp | 0.00 / 0.00 pp | 0 |
| `partial_sparse` | 5% | 10.1010% | 50 | 0.81% | n/a | 0.03 / 0.11 pp | 0.00 / 0.00 pp | 0 |
| `range_deleted` | 1% | 10.0182% | 165 | 9.31% | 1,018-1,288 (3.95%) | 0.58 / 2.05 pp | 0.00 / 0.00 pp | 0 |
| `range_deleted` | 5% | 10.0182% | 165 | 9.31% | 1,018-1,288 (3.95%) | 0.58 / 2.05 pp | 0.00 / 0.00 pp | 0 |
| `small_dense` | 1% | 10.8434% | 9 | 1.82% | n/a | 0.07 / 0.32 pp | 0.00 / 0.00 pp | 0 |
| `small_dense` | 5% | 10.8434% | 9 | 1.82% | n/a | 0.07 / 0.32 pp | 0.00 / 0.00 pp | 0 |
| `split_churn` | 1% | 10.0214% | 234 | 0.35% | n/a | 0.17 / 0.50 pp | 2.66 / 7.24 pp | 0 |
| `split_churn` | 5% | 10.0214% | 234 | 0.35% | n/a | 0.17 / 0.50 pp | 2.66 / 7.24 pp | 0 |
| `uniform_sparse` | 1% | 10.0182% | 165 | 0.39% | n/a | 0.03 / 0.11 pp | 0.00 / 0.00 pp | 0 |
| `uniform_sparse` | 5% | 10.0182% | 165 | 0.39% | n/a | 0.03 / 0.11 pp | 0.00 / 0.00 pp | 0 |

### Effect on the small partial index

The 3.88 MiB `partial_sparse` index had 495 ordinary pages: 3 internal and 492
live leaves. The policy therefore sampled `ceil(495 * 0.10) = 50` pages for
both requests below 10%. The same 100 seeds produced:

| Requested | Pages before -> after | Realized before -> after | Leaf MAPE before -> after | Density error mean / max before -> after | No-leaf draws before -> after |
|---:|---:|---:|---:|---:|---:|
| 1% | 5 -> 50 | 1.0101% -> 10.1010% | 0.80% -> 0.81% | 0.02 / 0.02 pp -> 0.03 / 0.11 pp | 0 -> 0 |
| 5% | 25 -> 50 | 5.0505% -> 10.1010% | 0.95% -> 0.81% | 0.03 / 0.23 pp -> 0.03 / 0.11 pp | 0 -> 0 |

The floor improved the 5% leaf-count summary and maximum density error, but the
1% leaf MAPE and density errors rose slightly in this fixed finite set. That is
not contradictory: the policy guarantees more sampled pages, not monotonic
improvement for every deterministic set of random draws. Both requested
fractions produce identical post-floor results because they use the same 50-page
target and reset the same seeds.

The floor does not change partial-index semantics. It samples more pages from
the predicate-selected physical B-tree and still says nothing about excluded
heap rows, predicate selectivity, or heap bloat
([heapam_handler.c#partial-build-qual](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1631-L1644),
[pgstatindex.c#physical-scan](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331)).

The 0.66 MiB `small_dense` control shows the clearer small-draw benefit. Its 1%
sample moved from one page to nine, leaf MAPE fell from 3.20% to 1.82%, maximum
density error fell from 3.15 to 0.32 points, and two leaf-free draws became
zero. At requested 5%, five pages became nine and leaf MAPE fell from 2.13% to
1.82%.

### What the comparison establishes

- **Full sampling is a strong equivalence check.** Across all tested shapes,
  sampling every ordinary block reproduced standard `pgstatindex`. This checks
  the SQL page-type mapping, count scaling, density denominator, and backward
  right-link test against the implementation
  ([pgstatindex.c#classification-and-ratios](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L372)).
- **The strict policy behaved as specified.** Every sub-100-MiB fixture used at
  least 10%; the 107.13 MiB control retained 1% and 5%, realized as 1.0064% and
  5.0029%. Synthetic cases confirmed that 104,857,599 bytes is floored while
  104,857,600 bytes is not.
- **The floor reduced but did not remove rare-class error.** The
  `range_deleted` fixture sampled 165 pages and had 3.95% deleted-count MAPE,
  while leaf-count MAPE remained 9.31%. The full scanner has no sampling error
  for these classes because it visits every block
  ([pgstatindex.c#classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L326)).
- **Uniform density was easier than fragmented-link frequency in this run.** On
  `split_churn`, density MAE was 0.17 points while fragmentation MAE was 2.66
  points with a 7.24-point maximum. The metrics use different observations:
  summed leaf free space versus the frequency of backward right links
  ([pgstatindex.c#leaf-metrics](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L323)).
- **The floor removed this small fixture's unsupported ratio draws.** The
  0.66 MiB control moved from one to nine pages at requested 1%, and two draws
  with no leaf became zero. This does not prove every sub-100-MiB index will
  contain a sampled leaf; page composition still controls that outcome.
- **Small partial indexes receive more evidence, not different semantics.** The
  partial fixture used 50 pages for either request, but its fixed-seed 1% error
  summary did not improve monotonically.
- **The empty edge is deterministic.** A metapage-only index has no ordinary
  sample population, so both requests drew zero pages and returned the standard
  zero-count/`NaN` boundary
  ([pgstattuple.out#empty-index](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).
- **Do not infer C performance from this SQL test.** It records accuracy and
  selected-page counts, not speedup. The prototype still enumerates and sorts
  every candidate and opens the relation once per selected `bt_page_stats` call;
  the C proposal has different sampling, locking, validation, and buffer access
  ([btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L320),
  [pgstatindex.c#buffer-strategy](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L223)).

## Context Reviewed

- Pinned checkout: `raw/postgres-17/` at
  `54eeefaedbee0385529f3edf321bb99e49232aaa` (`REL_17_STABLE`, PostgreSQL
  17.10).
- `pgstattuple`: control file, Makefile, Meson file, base/update SQL, all
  `pgstatindex` wrappers, `BTIndexStat`, complete `pgstatindex_impl`,
  `pgstatapprox.c`, documentation, regression input, and expected output.
- B-tree internals: metapage and opaque structures, page initialization,
  metapage and ordinary-page checks, page-state macros, relation length, buffer
  strategy, relation/content locks, DML and VACUUM locks, B-tree VACUUM tuple
  removal, half-dead and deleted states, and generated `BTREE_AM_OID` plumbing.
- Sampling: complete `sampling.h` and Algorithm-S implementation, reverse
  callers, the v17 `ANALYZE` read-stream path, PRNG state and seed types, sample
  count limits, ordering, and cancellation boundary.
- `pageinspect`: 1.12 control/update chain, 64-bit `bt_metap` and
  `bt_page_stats`, `d`/`D` page classification, `bt_multi_page_stats`, parallel
  restriction, per-call relation/page locks, docs, and eight regression targets.
- SQL and fixture surfaces: CREATE INDEX and VACUUM grammar,
  `INDEX_CLEANUP ON`, `autovacuum_enabled`, `pg_relation_size(regclass)`,
  build-time `BLCKSZ`, `block_size`, `random()`/`setseed()`, timeout contexts,
  and function return-shape replacement.
- Extension lifecycle: install-path selection, Make and Meson data lists, fmgr
  registration, SQL privileges, and generated catalog headers.
- Exact-pin execution under `.wiki-runtime/` built and installed PostgreSQL
  17.10, `pgstattuple`, and `pageinspect`. The one `pgstattuple` regression test
  and all eight `pageinspect` regression tests passed. The setup and sampler SQL
  blocks extracted from this page also parsed and executed; a full sample of a
  populated B-tree returned one row, while an invalid zero fraction returned no
  row as documented.
- The isolated comparison ran seven indexes from 0.01 MiB to 107.13 MiB. It
  executed 100 identical seeds at requested 1% and 5%, tested the policy against
  synthetic bytes immediately below, at, and above 100 MiB, and matched every
  shared standard-`pgstatindex` field at 100%. A test-only no-floor path supplied
  the focused before/after results for the partial and small healthy indexes.
  The server was stopped after the run.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Current v17 `pgstatindex` scans every captured ordinary block | [pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331) |
| It rejects invalid indexes and another session's temporary index | [pgstatindex.c#guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L250) |
| Metadata and pages do not form one instantaneous snapshot | [pgstattuple.sgml#snapshot-limit](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L268-L279), [pgstatindex.c#metapage-and-scan](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L252-L331) |
| `BlockSampler` samples without replacement and emits ascending blocks | [sampling.c#BlockSampler](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L23-L116) |
| v17 sampler state uses `pg_prng_state`/`uint32`, but sample counts remain `int` | [sampling.h#BlockSamplerData](../../../raw/postgres-17/src/include/utils/sampling.h#L16-L42) |
| Count estimates scale exhaustive classes; leaf ratios reuse current formulas | [pgstatindex.c#classification-and-ratios](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L372) |
| Threshold bytes use captured main-fork length while pages depend on `BLCKSZ` | [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3971-L4003), [configure.ac#block-size](../../../raw/postgres-17/configure.ac#L259-L289) |
| `AccessShareLock` permits concurrent DML and lazy VACUUM index maintenance | [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L60-L103), [execIndexing.c#ExecOpenIndices](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L188-L204), [vacuumlazy.c#index-locks](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L356-L361) |
| Proposed local validation is not whole-tree validation | [nbtpage.c#_bt_getmeta](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L134-L170), [nbtpage.c#_bt_checkpage](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825) |
| The C path can retain a bounded 256 KiB-or-smaller bulk-read ring | [freelist.c#GetAccessStrategy](../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L536-L613) |
| The extension needs SQL, control, Make, Meson, docs, and test changes | [Makefile#pgstattuple](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L3-L26), [meson.build#pgstattuple](../../../raw/postgres-17/contrib/pgstattuple/meson.build#L3-L42) |
| `BTREE_AM_OID` is an existing generated-header dependency | [pg_am.h#generated-header](../../../raw/postgres-17/src/include/catalog/pg_am.h#L18-L29), [pg_am.dat#BTREE_AM_OID](../../../raw/postgres-17/src/include/catalog/pg_am.dat#L15-L23), [genbki.pl#OID-symbol-emission](../../../raw/postgres-17/src/backend/catalog/genbki.pl#L675-L686) |
| v17 `pageinspect` uses 64-bit block arguments and `d`/`D` deleted types | [pageinspect--1.8--1.9.sql#bt_page_stats](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L86-L103), [btreefuncs.c#page-types](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L101-L163) |
| The one-read prototype derives valid leaf capacity from page initialization | [nbtpage.c#_bt_pageinit](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1123-L1132), [bufpage.c#PageInit](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L35-L60) |
| Partial predicates filter build/DML inputs, while `pgstatindex` scans physical pages | [heapam_handler.c#partial-build-qual](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1631-L1644), [execIndexing.c#partial-DML-filter](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L386), [pgstatindex.c#physical-scan](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L278-L331) |
| `setseed()` makes subsequent current-session random calls repeatable | [func.sgml#setseed](../../../raw/postgres-17/doc/src/sgml/func.sgml#L1930-L1964), [pseudorandomfuncs.c#setseed-and-random](../../../raw/postgres-17/src/backend/utils/adt/pseudorandomfuncs.c#L56-L93) |
| Existing tests contain no sampling function | [Makefile#regression-target](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L10-L26), [pgstattuple.sql#complete-test](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138) |

## Open Questions

- Would PostgreSQL accept this API and the fixed 100-MiB/10% policy, and should
  estimates include standard errors or confidence intervals? The pinned source
  cannot decide project policy.
- Should the production API expose a seed and explicit requested/effective
  fractions, even though sample counts and `scanned_percent` expose realized
  work?
- Is a fixed 10% floor below 100 MiB preferable to a minimum sampled-page or
  sampled-leaf count? The run removed leaf-free draws in one small fixture but
  did not make every partial-index finite-sample summary improve. The threshold
  also creates a cost discontinuity at exactly 100 MiB.
- Should independently rounded page-class estimates be allowed to miss `N` by a
  few pages, should the API return non-integer estimates, or should it use a
  total-preserving rounding rule?
- Is `BlockSampler`'s fixed state but potentially O(`N`) skip loop acceptable,
  or should v17 use a wider skip-ahead algorithm with explicit interrupt points?
- Should the C path use the v17 read-stream layer, as `ANALYZE` does, or stay
  close to current `pgstatindex` with `ReadBufferExtended` and a bulk-read ring?
- Should the function deliberately harden metapage and sampled-page validation,
  even though that makes a full sample stricter than current `pgstatindex` on
  malformed storage?
- The SQL helper remains intentionally non-equivalent: it returns no row for
  invalid fractions and over-limit indexes, uses superuser-only `pageinspect`,
  lacks one continuous lock and a bulk-read ring, and performs O(`N`) candidate
  generation plus a random-key sort.

## Source References

- [pgstatindex.c#entry-points-and-structures](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L47-L132) - fmgr registrations, AM tests, and accumulators.
- [pgstatindex.c#wrappers](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L134-L216) - legacy and v1.5 text/regclass wrappers.
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L381) - guards, metadata, full scan, formulas, and result construction.
- [pgstatapprox.c#statapprox_heap](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L38-L217) - approximate-output and scan-percentage precedent.
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L1-L136) - SQL bindings, attributes, and grants.
- [pgstattuple.control](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5), [Makefile#pgstattuple](../../../raw/postgres-17/contrib/pgstattuple/Makefile#L1-L27), and [meson.build#pgstattuple](../../../raw/postgres-17/contrib/pgstattuple/meson.build#L1-L42) - extension version, build data, objects, and tests.
- [pgstattuple.sgml#pgstatindex](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L161-L294) - output and page-by-page caveat.
- [pgstattuple.sql#complete-test](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138) and [pgstattuple.out#results](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L1-L305) - current regression surface.
- [sampling.h#sampling-API](../../../raw/postgres-17/src/include/utils/sampling.h#L13-L64) and [sampling.c#BlockSampler](../../../raw/postgres-17/src/backend/utils/misc/sampling.c#L23-L116) - sampler state and Algorithm S.
- [analyze.c#block-sampling](../../../raw/postgres-17/src/backend/commands/analyze.c#L1110-L1204) - in-core v17 caller and read-stream use.
- [block.h#BlockNumber](../../../raw/postgres-17/src/include/storage/block.h#L17-L35) - physical block-number type and range.
- [nbtree.h#page-structures](../../../raw/postgres-17/src/include/access/nbtree.h#L28-L137) - opaque/metapage layouts, flags, and versions.
- [nbtpage.c#metapage-checks](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L134-L170) and [nbtpage.c#ordinary-page-checks](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L794-L825) - proposed validation model.
- [nbtpage.c#_bt_pageinit](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1123-L1132) and [bufpage.c#PageInit](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L35-L60) - valid B-tree page capacity used by the one-read prototype.
- [freelist.c#GetAccessStrategy](../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L536-L613) - bulk-read ring size.
- [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L60-L103) - relation-lock compatibility.
- [extension.c#install-path-selection](../../../raw/postgres-17/src/backend/commands/extension.c#L1614-L1668) - extension install/update chain.
- [pg_am.h#generated-header](../../../raw/postgres-17/src/include/catalog/pg_am.h#L18-L29), [pg_am.dat#BTREE_AM_OID](../../../raw/postgres-17/src/include/catalog/pg_am.dat#L15-L23), and [genbki.pl#OID-symbol-emission](../../../raw/postgres-17/src/backend/catalog/genbki.pl#L675-L686) - generated AM OID boundary.
- [pageinspect.control](../../../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5), [pageinspect--1.8--1.9.sql#btree-signatures](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L69-L103), and [pageinspect--1.10--1.11.sql#parallel-restriction](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.10--1.11.sql#L6-L23) - current version, SQL shape, and parallel boundary.
- [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L101-L194), [btreefuncs.c#bt_page_stats](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L196-L333), and [btreefuncs.c#bt_metap](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L828-L939) - SQL prototype primitives.
- [nbtree.c#btvacuumpage](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1229-L1385) and [nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1773-L1803) - VACUUM entry removal and empty-leaf deletion.
- [heapam_handler.c#partial-build-qual](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1631-L1644) and [execIndexing.c#partial-DML-filter](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L368-L386) - partial-index filtering.
- [func.sgml#setseed](../../../raw/postgres-17/doc/src/sgml/func.sgml#L1930-L1964) and [pseudorandomfuncs.c#setseed-and-random](../../../raw/postgres-17/src/backend/utils/adt/pseudorandomfuncs.c#L56-L93) - repeatable current-session SQL sampling sequence.
- [dbsize.c#pg_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L370) and [system_functions.sql#pg_relation_size](../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289) - main-fork size implementation and wrapper.
- [configure.ac#block-size](../../../raw/postgres-17/configure.ac#L259-L289) and [guc_tables.c#block_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276) - build-time page sizes and reported value.
- [gram.y#CREATE-INDEX](../../../raw/postgres-17/src/backend/parser/gram.y#L8091-L8136) and [gram.y#VACUUM](../../../raw/postgres-17/src/backend/parser/gram.y#L11793-L11836) - fixture grammar.
- [vacuum.sgml#INDEX_CLEANUP](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L186-L228), [create_table.sgml#autovacuum_enabled](../../../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1527-L1538), and [guc_tables.c#timeouts](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2610-L2629) - fixture cleanup, routine-autovacuum, and timeout scopes.

## Navigation

- [PostgreSQL 17 index](../index.md)
- [PostgreSQL 17 Contrib Extensions](contrib-extensions.md)
- [PostgreSQL versions](../../versions.md)
