---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-17T18:12:12Z
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

## Answer

### Recommendation

Add a new `pgstattuple` function named `pgstatindex_approx`. Do not change the
existing `pgstatindex` contract. Capture the B-tree main-fork length once, then
draw a simple random sample without replacement from physical blocks `1`
through `nblocks - 1`. Keep metadata and sample diagnostics as direct
observations. Prefix every extrapolated metric with `approx_`.

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

This is proposed DDL, not an object in the pinned checkout. `STRICT` follows the
existing null-input behavior. The revoke/grant follows the v1.5 permission
model. A new API needs only the `regclass` entry point; the existing text entry
point is retained for historical compatibility, not as a requirement for new
functions
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
N = max(nblocks - 1, 0)       -- ordinary physical blocks
k = 0                          when N = 0
k = min(N, max(1, ceil(N*f)))  when N > 0
f_realized = k / N             when N > 0
```

Reject any `sample_fraction` that does not satisfy `0 < f <= 1`, including NaN
and infinities, with `ERRCODE_INVALID_PARAMETER_VALUE`. Unlike the former SQL
prototype, do not silently replace a requested fraction with an undocumented
larger fraction. A minimum-page policy can be added only if the API names and
reports it as such.

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
                             ceil((nblocks - 1)::float8 * sample_fraction)::bigint))
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
   and infinities; verify that `STRICT` returns null for a null argument without
   entering the C function
   ([pgstattuple--1.4--1.5.sql#STRICT](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
4. **Sampling invariants:** through an exposed or test-only fixed-seed path,
   sampled blocks must be in range, unique, and ascending; the reported sample
   counts and percentage must match the realized draw
   ([sampling.c#BlockSampler-iteration](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L53-L112)).
5. **No sampled leaf:** force a sample containing no live leaf and assert two
   `NaN` ratio fields plus `sampled_leaf_pages = 0`
   ([pgstatindex.c#ratio-zero-cases](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).
6. **Object and state boundaries:** cover a wrong access method, non-index
   relation kinds, partitioned index, another-session temporary index, a
   current-session temporary index, and readable invalid/not-ready B-trees
   ([pgstatindex.c#guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L238),
   [pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L44)).
7. **Validation:** exercise an all-zero sampled page, wrong special-space size,
   bad metapage magic, and unsupported B-tree version; also prove that an
   unsampled malformed page is outside the function's integrity claim
   ([nbtpage.c#B-tree-page-checks](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148),
   [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).
8. **Concurrency and cancellation:** cover extension after length capture, page
   state changes during sampling, statement cancellation, and the long
   `BlockSampler_Next` skip-loop boundary
   ([pgstatindex.c#one-time-length-and-interrupts](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
   [sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111)).
9. **Privileges and parallel placement:** test `PUBLIC` revoke,
   `pg_stat_scan_tables` grant, and the chosen parallel marking
   ([pgstattuple--1.4--1.5.sql#grant-model](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L119)).
10. **Extension lifecycle:** test upgrade from 1.5 to 1.6 and a fresh default
    install that follows the base-plus-update path
    ([extension.c#install-path-selection](../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400),
    [extension.c#install-then-update](../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550)).

The pinned tree currently declares one `pgstattuple` regression target. It tests
empty `pgstatindex` output and object/error paths, but has no populated,
sampling, concurrency, malformed-page, permissions, or statistical test and no
module isolation or TAP target
([Makefile#regression-target](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24),
[pgstattuple.sql#complete-test](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119)).

## Context Reviewed

- Pinned checkout: `raw/postgres-12/` at
  `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- `pgstattuple`: control file, Makefile, base/update SQL, every `pgstatindex`
  wrapper, `BTIndexStat`, all of `pgstatindex_impl`, `pgstatapprox.c`, docs,
  complete regression input, and expected output.
- B-tree internals: metapage and opaque structures, root metadata, page
  initialization, generic and B-tree-specific page checks, page-state macros,
  relation length, buffer access strategy, relation and content locks, DML and
  VACUUM index locks, and generated `BTREE_AM_OID` plumbing.
- Sampling: complete `sampling.h` and `sampling.c`, reverse callers, and the
  v12 `ANALYZE` use of `BlockSampler`.
- `pageinspect`: control/Makefile/update SQL, `bt_metap`, `bt_page_stats`,
  `get_raw_page`, `page_header`, tuple conversion, docs, and regression wiring.
- Extension lifecycle: target-version path selection, install-then-update,
  fmgr V1 symbol metadata, SQL privileges, and generated catalog headers.
- Exact-pin execution under `.wiki-runtime/`: built and installed pageinspect,
  ran the original prototype, then ran the revised one-read/no-hidden-floor
  prototype on populated and metapage-only B-trees. A full sample matched
  `pgstatindex`; invalid fractions returned no rows as documented for the SQL
  prototype. The `pgstattuple` regression target and all five `pageinspect`
  regression targets passed. The temporary server was stopped after review.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Current `pgstatindex` scans every captured ordinary block | [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315) |
| Current metadata and pages do not form one instantaneous snapshot | [pgstattuple.sgml#snapshot-limit](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279), [pgstatindex.c#metapage-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315) |
| `BlockSampler` samples without replacement and emits ascending block numbers | [sampling.c#BlockSampler](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L112) |
| The native sampler has an `int` sample count and a potentially long skip loop | [sampling.h#BlockSamplerData](../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43), [sampling.c#skip-loop](../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L76-L111) |
| Count estimates scale sampled exhaustive classes; ratio estimates reuse current leaf formulas | [pgstatindex.c#classification-and-leaf-metrics](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356) |
| `AccessShareLock` permits concurrent DML and lazy VACUUM index maintenance | [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105), [execIndexing.c#ExecOpenIndices](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L141-L213), [vacuumlazy.c#index-locks](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L275-L292) |
| Proposed local validation checks page shape, not whole-tree integrity | [nbtpage.c#_bt_getmeta](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148), [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720) |
| The C implementation can retain a bounded bulk-read ring | [freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L587) |
| The extension needs a new update script, control bump, Makefile data entry, docs, and tests | [Makefile#module-and-tests](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24), [pgstattuple.control#default-version](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) |
| `BTREE_AM_OID` is an existing generated-header dependency; no new catalog is needed | [pg_am.h#generated-header](../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29), [pg_am.dat#BTREE_AM_OID](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23), [genbki.pl#OID-symbol-emission](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603) |
| The pageinspect compatibility helper avoids v1.7 `oldest_xact` conversion | [pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26), [btreefuncs.c#bt_metap-output](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L505-L575), [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2152) |
| The revised SQL derives valid-page capacity from one `bt_page_stats` read | [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153), [nbtpage.c#_bt_pageinit](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L920-L929), [bufpage.c#PageInit](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L35-L60) |
| Existing tests do not cover sampling | [Makefile#regression-target](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24), [pgstattuple.sql#complete-test](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119) |

## Open Questions

- Would PostgreSQL accept this API, and should estimated fields include standard
  errors or confidence intervals? The pinned source cannot decide project policy.
- Should the production API expose a seed for repeatability, and should it accept
  a minimum sampled-page count instead of silently imposing one?
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
- [guc.c#diagnostic-timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2397) - session/transaction timeout contexts.

## Navigation

- [PostgreSQL 12 index](../index.md)
- [How `pgstatindex` Calculates B-Tree Index Statistics in PostgreSQL 12](how-pgstatindex-calculates-information.md)
- [Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12](index-bloat-reindex-heuristic.md)
- [PostgreSQL versions](../../versions.md)
