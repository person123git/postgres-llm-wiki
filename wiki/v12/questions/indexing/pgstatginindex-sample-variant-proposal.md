---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Proposing a Sampling pgstatginindex Variant for PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Recommendation](#recommendation)
  - [Why GIN needs a different function, not a reused one](#why-gin-needs-a-different-function-not-a-reused-one)
  - [The v12 GIN physical page taxonomy](#the-v12-gin-physical-page-taxonomy)
  - [What the metapage already knows, and what it hides](#what-the-metapage-already-knows-and-what-it-hides)
  - [Proposed API and field contract](#proposed-api-and-field-contract)
  - [C call path and data structures](#c-call-path-and-data-structures)
  - [Sampling policy: three GIN-specific rules](#sampling-policy-three-gin-specific-rules)
  - [Estimator math](#estimator-math)
  - [Per-class free space and density](#per-class-free-space-and-density)
  - [Locking, concurrency, and integrity](#locking-concurrency-and-integrity)
  - [Extension, build, and generated-file implications](#extension-build-and-generated-file-implications)
- [Pros](#pros)
- [Cons](#cons)
- [SQL Prototype Using `contrib/pageinspect`](#sql-prototype-using-contribpageinspect)
  - [Prototype boundaries](#prototype-boundaries)
  - [Setup](#setup)
  - [Exact page census](#exact-page-census)
  - [Sampling function](#sampling-function)
  - [Exact pending-list walk](#exact-pending-list-walk)
- [Tests To Add](#tests-to-add)
- [Executed Comparison With the Exact Census](#executed-comparison-with-the-exact-census)
  - [Test scope and fixtures](#test-scope-and-fixtures)
  - [Exact full-scan truth](#exact-full-scan-truth)
  - [Full-sample equivalence](#full-sample-equivalence)
  - [Cost at each requested fraction](#cost-at-each-requested-fraction)
  - [Post-stratification measured](#post-stratification-measured)
  - [The floor: the absolute floor lost](#the-floor-the-absolute-floor-lost)
  - [Rare-class detection at the derived floor](#rare-class-detection-at-the-derived-floor)
  - [Density estimates](#density-estimates)
  - [Metapage drift the exact fields expose](#metapage-drift-the-exact-fields-expose)
  - [Error paths](#error-paths)
  - [What the comparison establishes](#what-the-comparison-establishes)
- [Follow-Up: Bloat and Wasted Space](#follow-up-bloat-and-wasted-space)
  - [Short answer](#short-answer)
  - [Why free space is not wasted space in a GIN index](#why-free-space-is-not-wasted-space-in-a-gin-index)
  - [The three waste shapes it does measure](#the-three-waste-shapes-it-does-measure)
  - [The waste it cannot measure](#the-waste-it-cannot-measure)
  - [Proposed additional output fields](#proposed-additional-output-fields)
  - [Prototype for the new fields](#prototype-for-the-new-fields)
  - [How the new fields behave under sampling](#how-the-new-fields-behave-under-sampling)
  - [Turning free_percent into a rebuild estimate](#turning-free_percent-into-a-rebuild-estimate)
  - [A bloat verdict for GIN](#a-bloat-verdict-for-gin)
  - [What the follow-up measurements establish](#what-the-follow-up-measurements-establish)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, based on the question document "Proposing a Sampling
pgstatindex Variant for PostgreSQL 12 (unverified)", create a question document
with a new sampling pgstatindex function only for GIN indexes.

(The prompt wording was corrected for spelling and grammar at the user's
request; meaning preserved. The user also chose a GIN-native page-class output
contract, exact-pin empirical tests in addition to source, and a GIN-specific
sample policy rather than carrying over the B-tree page's 100 MiB / 10% floor.)

Follow-up:

Can this function measure bloat? Can it measure wasted space?

(The follow-up wording was corrected for capitalization and sentence structure
at the user's request; meaning preserved. The user also chose exact-pin
empirical tests in addition to source, and asked that the answer propose
concrete wasted-space output fields rather than staying diagnostic-only.
Answered in [Follow-Up: Bloat and Wasted
Space](#follow-up-bloat-and-wasted-space).)

## Answer

### Recommendation

Add a new `pgstattuple` function named `pgstatginindex_approx`. Do not change
the existing `pgstatginindex` contract. The new function reads the metapage
exactly, then draws a simple random sample of physical blocks and reports
GIN-native page classes: entry-tree internal and leaf, posting-tree (data)
internal and leaf, pending-list, deleted, and never-initialized pages.

Three things make this a genuinely different design from the B-tree
`pgstatindex_approx` proposal, not a copy with renamed columns:

1. **There is no full-scan GIN baseline to approximate.** v12
   `pgstatginindex` reads only block 0 and returns three columns
   ([pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573)),
   and `pgstattuple()` rejects GIN outright
   ([pgstattuple.c#AM-dispatch](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L262-L286)).
   So this function adds physical page accounting that v12 has no way to
   produce, rather than making an existing expensive scan cheaper.
2. **One page class is already known exactly.** The metapage carries
   `nPendingPages` and `nPendingHeapTuples`, and they are maintained in real
   time, not at VACUUM
   ([ginblock.h#GinMetaPageData](../../../../raw/postgres-12/src/include/access/ginblock.h#L54-L100),
   [ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657)).
   That turns the sample into a two-stratum problem with one stratum size known
   for free, which permits a post-stratified estimator. Measured below: it cut
   the estimate's standard deviation by 85% on an index whose pending list was
   31% of its blocks.
3. **The floor can be derived instead of guessed.** The metapage also carries
   `nDataPages`, so the function can predict how rare the posting tree is
   *before* choosing a sample size. The measured result forced a design change:
   the absolute floor this page first proposed was worse than the B-tree
   page's percentage floor, and only a metapage-derived floor performed well.

Every extrapolated field is prefixed `approx_`, following the local
`pgstattuple_approx` precedent
([pgstattuple--1.4.sql#pgstattuple_approx](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L81-L95)).

### Why GIN needs a different function, not a reused one

v12 offers no physical page accounting for a GIN index at all:

| Tool | GIN behavior in v12 | Evidence |
|---|---|---|
| `pgstatindex(regclass)` | rejects: "is not a btree index" | [pgstatindex.c#guards](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L238) |
| `pgstathashindex(regclass)` | rejects: "is not a hash index" | [pgstatindex.c#pgstathashindex-guard](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L599-L606) |
| `pgstattuple(regclass)` | rejects: `"%s" (gin index) is not supported` | [pgstattuple.c#GIN-reject](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276), [pgstattuple.c#error](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L308-L311) |
| `pgstatginindex(regclass)` | returns 3 metapage columns, reads 1 block | [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573) |
| `pageinspect` GIN functions | one caller-supplied raw page each, superuser only | [ginfuncs.c#gin_page_opaque_info](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154), [pageinspect.sgml#superuser](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14) |

The closest in-tree model for the new function is `pgstathashindex`, which is
an AM-specific full physical scan that classifies every page and reports class
counts plus a free-space percentage
([pgstatindex.c#pgstathashindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L581-L727)).
The proposal keeps its shape and replaces its dense loop with a sampler.

### The v12 GIN physical page taxonomy

GIN stores an 8-byte `GinPageOpaqueData` in the special area of every page:
a `rightlink`, a `maxoff` whose meaning depends on page kind, and a `flags`
word
([ginblock.h#GinPageOpaqueData](../../../../raw/postgres-12/src/include/access/ginblock.h#L18-L38)).
Eight flag bits exist
([ginblock.h#flag-bits](../../../../raw/postgres-12/src/include/access/ginblock.h#L40-L48)),
and the predicate macros that read them are the classification vocabulary
([ginblock.h#page-macros](../../../../raw/postgres-12/src/include/access/ginblock.h#L107-L138)).

| Class | On-disk flags | Where set |
|---|---|---|
| metapage (block 0) | `GIN_META` only | [ginutil.c#GinInitMetabuffer](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L356-L383) |
| entry-tree leaf | `GIN_LEAF` | [gininsert.c#GinInitBuffer](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L345-L349) |
| entry-tree internal | no bits set | [ginbtree.c#root-split](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L507-L514) |
| posting-tree (data) leaf | `GIN_DATA\|GIN_LEAF\|GIN_COMPRESSED` | [gindatapage.c#dataSplitPageLeaf](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1043-L1045) |
| posting-tree internal | `GIN_DATA` | [ginbtree.c#root-split](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L507-L514) |
| pending-list page | `GIN_LIST`, plus `GIN_LIST_FULLROW` | [ginfast.c#writeListPage](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L57-L141) |
| deleted posting page | previous flags OR `GIN_DELETED`, plus a delete XID | [ginvacuum.c#ginDeletePage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L187-L192) |
| deleted ex-pending page | flags overwritten to exactly `GIN_DELETED` | [ginfast.c#shiftList-flags](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632) |
| never-initialized page | `pd_upper == 0` | [bufpage.h#PageIsNew](../../../../raw/postgres-12/src/include/storage/bufpage.h#L225-L229) |

Two ordering consequences matter for a classifier:

- `GIN_DELETED` is OR'ed onto a posting page's existing bits, so `GIN_DATA` and
  `GIN_LEAF` survive deletion. A classifier must test deleted **before** data,
  or deleted pages disappear into the data counts
  ([ginvacuum.c#ginDeletePage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L129-L235)).
- Reuse is XID-gated. `GinPageIsRecyclable` requires `PageIsNew` or a delete
  XID older than `RecentGlobalXmin`
  ([ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138)),
  and `GinNewBuffer` only reuses a page the FSM offers and the macro accepts
  ([ginutil.c#GinNewBuffer](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)).

GIN never truncates the main fork: nothing under
`src/backend/access/gin/` references `RelationTruncate`, `smgrtruncate`, or
`pages_removed`, and `ginvacuumcleanup` reports `num_pages` as the current
block count rather than shrinking it
([ginvacuum.c#ginvacuumcleanup-tail](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L783-L794)).
So a deleted-page share is a durable bloat signal, and no v12 view reports it.

### What the metapage already knows, and what it hides

`ginvacuumcleanup` scans every block from `GIN_ROOT_BLKNO` to the end on every
real VACUUM and writes four counters into the metapage
([ginvacuum.c#classification-loop](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L746-L781),
[ginutil.c#ginUpdateStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L659-L716),
[gin.h#GinStatsData](../../../../raw/postgres-12/src/include/access/gin.h#L38-L49)).
Its classification is deliberately coarse:

```c
if (GinPageIsRecyclable(page))        { RecordFreeIndexPage(...); totFreePages++; }
else if (GinPageIsData(page))         { idxStat.nDataPages++; }
else if (!GinPageIsList(page))        { idxStat.nEntryPages++;
                                        if (GinPageIsLeaf(page))
                                          idxStat.nEntries += PageGetMaxOffsetNumber(page); }
```

Four consequences, all measured below:

- **Deleted-but-not-yet-recyclable posting pages are counted as data pages.**
  They still carry `GIN_DATA`, and the recyclable test fails while their delete
  XID is recent
  ([ginvacuum.c#classification-loop](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L746-L777)).
- **Deleted-and-recyclable pages are counted in no metapage class at all.**
  They fall into the first branch, which only increments a local
  `totFreePages` and records an FSM entry
  ([ginvacuum.c:758-763](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L758-L763),
  [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55)).
- **Pending-list pages are counted in no metapage class either.** They are
  excluded by `!GinPageIsList(page)` and appear only in `nPendingPages`, which
  `ginUpdateStats` explicitly does not write
  ([ginutil.c#ginUpdateStats-comment](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L659-L663)).
- **`nTotalPages` is a snapshot, not a live length.** It is set to the
  `RelationGetNumberOfBlocks` value read during that VACUUM
  ([ginvacuum.c:779-781](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L781)).

The planner lives with exactly this staleness: `gincostestimate` trusts
`nPendingPages`, scales the other counters by live-size growth, and abandons
them entirely past 4X growth in favor of an invented 90%-entry/10%-data split
([selfuncs.c#metapage-trust](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6730),
[selfuncs.c#4X-scaling](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6732-L6750),
[selfuncs.c#invented-stats](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6751-L6770)).
A sampled physical census is the missing measurement that would let an operator
see that drift instead of inferring it.

### Proposed API and field contract

| Output | Source | Contract |
|---|---|---|
| `version` | metapage `ginVersion` | direct |
| `index_size` | captured `nblocks * BLCKSZ` | direct at length-capture time |
| `total_pages` | captured `nblocks` | direct |
| `pending_pages` | metapage `nPendingPages` | **direct, never estimated** |
| `pending_tuples` | metapage `nPendingHeapTuples` | **direct, never estimated** |
| `meta_total_pages`, `meta_entry_pages`, `meta_data_pages`, `meta_entries` | metapage VACUUM counters | direct, as of last VACUUM |
| `sampled_pages` | ordinary blocks actually read | direct |
| `sampled_nonpending_pages` | sampled blocks not in the pending list | direct |
| `scanned_percent` | `100.0 * sampled_pages / (nblocks - 1)` | direct |
| `approx_entry_internal_pages` | post-stratified scale-up | estimate |
| `approx_entry_leaf_pages` | post-stratified scale-up | estimate |
| `approx_data_internal_pages` | post-stratified scale-up | estimate |
| `approx_data_leaf_pages` | post-stratified scale-up | estimate |
| `approx_deleted_pages` | post-stratified scale-up | estimate |
| `approx_new_pages` | post-stratified scale-up | estimate |
| `approx_entry_leaf_density` | ratio of sums over sampled entry leaves | estimate |
| `approx_data_leaf_density` | ratio of sums over sampled data leaves | estimate |

Reporting the metapage counters next to the sampled estimates is the point of
the design: the pair is what exposes drift, and neither number alone does.

Six further fields -- `pending_space`, `total_space`, `approx_free_space`,
`approx_free_percent`, `approx_recyclable_pages` and `approx_recyclable_space`
-- are proposed in [Proposed additional output
fields](#proposed-additional-output-fields), which answers the filed follow-up
on bloat and wasted space.

Proposed extension DDL:

```sql
-- proposed: contrib/pgstattuple/pgstattuple--1.5--1.6.sql
CREATE /* wiki_pgstatginindex_approx_extension */ FUNCTION pgstatginindex_approx(
    IN relname regclass,
    IN sample_fraction FLOAT8 DEFAULT 0.01,
    IN target_data_pages INTEGER DEFAULT 20,
    IN max_sample_pages BIGINT DEFAULT -1,
    OUT version INTEGER,
    OUT index_size BIGINT,
    OUT total_pages BIGINT,
    OUT pending_pages BIGINT,
    OUT pending_tuples BIGINT,
    OUT meta_total_pages BIGINT,
    OUT meta_entry_pages BIGINT,
    OUT meta_data_pages BIGINT,
    OUT meta_entries BIGINT,
    OUT sampled_pages BIGINT,
    OUT sampled_nonpending_pages BIGINT,
    OUT scanned_percent FLOAT8,
    OUT approx_entry_internal_pages BIGINT,
    OUT approx_entry_leaf_pages BIGINT,
    OUT approx_data_internal_pages BIGINT,
    OUT approx_data_leaf_pages BIGINT,
    OUT approx_deleted_pages BIGINT,
    OUT approx_new_pages BIGINT,
    OUT approx_entry_leaf_density FLOAT8,
    OUT approx_data_leaf_density FLOAT8)
AS 'MODULE_PATHNAME', 'pgstatginindex_approx'
LANGUAGE C STRICT VOLATILE PARALLEL RESTRICTED;

REVOKE /* wiki_pgstatginindex_approx_permissions */ EXECUTE
ON FUNCTION pgstatginindex_approx(regclass, FLOAT8, INTEGER, BIGINT) FROM PUBLIC;
GRANT /* wiki_pgstatginindex_approx_permissions */ EXECUTE
ON FUNCTION pgstatginindex_approx(regclass, FLOAT8, INTEGER, BIGINT) TO pg_stat_scan_tables;
```

This is proposed DDL, not an object in the pinned checkout. The `regclass`-only
signature matches `pgstatginindex`, which has never had a `text` overload
([pgstattuple--1.0--1.1.sql#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.0--1.1.sql#L6-L11),
[pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L47-L57)).
The revoke/grant pair copies the `pgstathashindex` template exactly
([pgstattuple--1.4--1.5.sql#pgstathashindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L121-L136)),
and the documented privilege model is `pg_stat_scan_tables` plus explicit
grants
([pgstattuple.sgml#privileges](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L15-L24)).
No C-level `superuser()` check belongs in a new function; since 1.5 the module
relies on `REVOKE`
([pgstatindex.c#v1.5-comment](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L168)).

`PARALLEL RESTRICTED` is deliberately more conservative than the existing GIN
function's `PARALLEL SAFE`
([pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L47-L57))
because an unseeded sampler is not deterministic. A final patch could relax it
after auditing its seeding.

### C call path and data structures

```text
pgstatginindex_approx(regclass, float8, int, bigint)
  -> relation_open(oid, AccessShareLock)
  -> IS_INDEX(rel) && IS_GIN(rel) guard
  -> RELATION_IS_OTHER_TEMP(rel) rejection
  -> ReadBuffer(rel, GIN_METAPAGE_BLKNO); LockBuffer(GIN_SHARE)
  -> validate flags == GIN_META and ginVersion
  -> copy the full GinMetaPageData; UnlockReleaseBuffer
  -> RelationGetNumberOfBlocks(rel) once
  -> n_ord   = nblocks - 1
  -> n_pend  = Min(metadata.nPendingPages, n_ord)      /* exact stratum */
  -> n_np    = n_ord - n_pend
  -> k = sample size from the three rules below
  -> BlockSampler_Init(&bs, n_ord, k, seed)
  -> bstrategy = GetAccessStrategy(BAS_BULKREAD)
  -> while BlockSampler_HasMore(&bs)
       -> blkno = BlockSampler_Next(&bs) + 1
       -> CHECK_FOR_INTERRUPTS()
       -> ReadBufferExtended(rel, MAIN_FORKNUM, blkno, RBM_NORMAL, bstrategy)
       -> LockBuffer(BUFFER_LOCK_SHARE)
       -> PageGetSpecialSize check, then classify and accumulate
       -> UnlockReleaseBuffer
  -> relation_close(rel, AccessShareLock)
  -> build the composite result
```

The guard, metapage read, and lock pattern come from
`pgstatginindex_internal`
([pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573)),
using `relation_open` plus `IS_INDEX`/`IS_GIN`
([pgstatindex.c#IS-macros](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L73))
so the existing error text `relation "%s" is not a GIN index` is preserved —
eight regression lines already depend on that wording
([pgstattuple.out#GIN-errors](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L140-L152)).
The other-session temp rejection is unchanged
([rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-12/src/include/utils/rel.h#L541-L549)).
The length capture, `BAS_BULKREAD` strategy, `CHECK_FOR_INTERRUPTS`, and
per-page share lock come from `pgstathashindex`
([pgstatindex.c#length-and-strategy](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L626-L630),
[pgstatindex.c#scan-loop](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L632-L643)).

`BlockSampler` is the same Knuth Algorithm S machinery `ANALYZE` uses
([sampling.h#BlockSampler](../../../../raw/postgres-12/src/include/utils/sampling.h#L26-L43),
[sampling.c#BlockSampler_Init](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L51),
[sampling.c#BlockSampler_Next](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L59-L112),
[analyze.c#block-sampling](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1019-L1049)).
Initialize it with population `n_ord` and add one to every returned value so
block 0 is never drawn. Two properties matter: it returns block numbers in
ascending order, so `BAS_BULKREAD` stays useful, and if the population is
smaller than the requested sample it selects every block, which makes the
function degrade into an exact census
([sampling.c#Algorithm-S-contract](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L35)).

Two limits are real. `BlockSamplerData.n` and `.m` are `int` while `N` and `t`
are 32-bit unsigned `BlockNumber`
([sampling.h#BlockSamplerData](../../../../raw/postgres-12/src/include/utils/sampling.h#L28-L36),
[block.h#BlockNumber](../../../../raw/postgres-12/src/include/storage/block.h#L17-L35)),
and `sampler_random_init_state` keeps only the low 32 bits of the seed with a
hard-coded first state word
([sampling.c#sampler_random_init_state](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L228-L234)).
Nothing in v12 samples an index: the only `BlockSampler` call site is the
table-AM-bound `acquire_sample_rows`
([analyze.c#acquire_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1009-L1049)),
and `IndexAmRoutine` has no analyze or sample callback
([amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L216)).

Reuse `GinIndexStat` for the three existing fields and add a page-class
accumulator shaped like `HashIndexStat`, which already pairs `BlockNumber`
class counters with `int64` item counts and a `uint64` free-space sum
([pgstatindex.c#GinIndexStat](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L97-L108),
[pgstatindex.c#HashIndexStat](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L110-L128)).

### Sampling policy: three GIN-specific rules

**R1 — the pending list is an exact stratum, never sampled.** Block 0 is always
read; `pending_pages` and `pending_tuples` are reported verbatim from the
metapage. Those two fields are maintained transactionally by
`ginHeapTupleFastInsert` and `shiftList`, not by VACUUM
([ginfast.c#pending-counters](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L368-L397),
[ginfast.c#shiftList-counters](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L596-L613)),
which is why `ginGetStats` documents them as trustworthy while the rest are not
([ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L636)).
No B-tree metapage field has this property.

**R2 — post-stratified expansion.** Because `n_pend` is exact, the non-pending
stratum size `n_np = n_ord - n_pend` is exact too. Scale non-pending class
counts by `n_np / k_np`, where `k_np` is the number of sampled pages that were
not pending, instead of by `n_ord / k`. This removes the pending list's
contribution to sampling variance.

**R3 — a metapage-derived floor, not a size-based percentage floor.** The
B-tree proposal raises a sub-10% request to 10% when the index is under
100 MiB. For GIN that rule is keyed to the wrong quantity: what determines
whether a bloat signal is visible is the absolute number of sampled pages
landing in the posting tree, and the metapage's `nDataPages` already estimates
how rare that is. So:

```text
k_floor = 0                                              when nDataPages = 0
k_floor = ceil(target_data_pages * nTotalPages / nDataPages)   otherwise
k       = min(n_ord, max(ceil(n_ord * f), k_floor, 50))
k       = min(k, max_sample_pages)                       when max_sample_pages > 0
```

`max_sample_pages` exists because the derived floor is data-dependent and can
escalate: on the measured `f_mixed_gin` fixture it produced 1,727 pages, 18.52%
of the index, against a requested 1%. The caller must be able to cap that, and
`scanned_percent` must report what actually happened.

This page originally proposed a flat absolute floor of 50 pages and rejected
the B-tree percentage rule. Measurement contradicted that: see
[The floor: the absolute floor lost](#the-floor-the-absolute-floor-lost).

### Estimator math

```text
N     = nblocks - 1                    -- ordinary blocks (population)
P     = min(nPendingPages, N)          -- exact pending stratum
N_np  = N - P                          -- exact non-pending stratum
k     = realized sample size over the N ordinary blocks
k_np  = sampled blocks not classified pending
```

For each non-pending class `c`:

```text
approx_pages(c) = sampled(c) * N_np / k_np      (R2, post-stratified)
```

and `pending_pages = P` directly. The plain alternative,
`sampled(c) * N / k`, is what a B-tree-style sampler would use; the measured
difference is in
[Post-stratification measured](#post-stratification-measured).

Degenerate cases:

- `N = 0` (metapage-only index): `k = 0`, every class estimate 0, both
  densities `NaN`. An empty GIN index actually has two blocks — metapage and
  root — so `N = 1` and the sample is a full census
  ([ginblock.h#fixed-blocks](../../../../raw/postgres-12/src/include/access/ginblock.h#L50-L52),
  [gin.out#n_total_pages](../../../../raw/postgres-12/contrib/pageinspect/expected/gin.out#L12)).
- `k_np = 0` (every sampled page was pending): return 0 for all non-pending
  classes and flag it through `sampled_nonpending_pages`.
- Four independently rounded class estimates need not sum to `N_np`.
  `index_size` and `total_pages` must stay independent of them.

Emit `scanned_percent` as `double` computed in floating point. Do not repeat
`pgstatapprox.c`'s wart, where `scanned_percent` is declared `uint64`,
computed with integer division, then returned through `Float8GetDatum`
([pgstatapprox.c#output_type](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52),
[pgstatapprox.c#integer-division](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L201-L207),
[pgstatapprox.c#values-fill](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L300-L311)).
Do copy its `natts != NUM_OUTPUT_COLUMNS` guard, which neither
`pgstathashindex` nor `pgstatginindex_internal` has
([pgstatapprox.c#natts-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L262-L266)).

### Per-class free space and density

GIN has no single free-space formula. `pd_upper - pd_lower`, i.e.
`PageGetExactFreeSpace`, is meaningful on every GIN page kind because GIN never
moves `pd_upper` off `pd_special` on data pages and uses ordinary line
pointers elsewhere
([bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L626-L647)).
The **denominator** differs by class:

| Class | Capacity | Evidence |
|---|---|---|
| entry leaf / internal | `BLCKSZ - MAXALIGN(SizeOfPageHeaderData) - MAXALIGN(sizeof(GinPageOpaqueData))` = 8160 at 8 kB | [ginentrypage.c#fullness-test](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L459-L483) |
| data leaf / internal | `GinDataPageMaxDataSize` = 8152 at 8 kB | [ginblock.h#GinDataPageMaxDataSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L319-L326) |
| pending list | `GinListPageSize` = 8160 at 8 kB | [ginblock.h#GinListPageSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L328-L332) |

Data pages reserve `MAXALIGN(sizeof(ItemPointerData))` after the header for the
page's right bound, which is why their capacity is 8 bytes smaller
([ginblock.h#data-page-layout](../../../../raw/postgres-12/src/include/access/ginblock.h#L265-L303)).
GIN's own accessors agree: `GinDataLeafPageGetFreeSpace` is literally
`PageGetExactFreeSpace`, while an internal data page uses
`GinDataPageMaxDataSize - maxoff * sizeof(PostingItem)`
([ginblock.h#GinDataLeafPageGetFreeSpace](../../../../raw/postgres-12/src/include/access/ginblock.h#L287-L291),
[ginblock.h#GinNonLeafDataPageGetFreeSpace](../../../../raw/postgres-12/src/include/access/ginblock.h#L319-L321)).

Three traps a classifier must avoid:

1. **Never call `PageGetMaxOffsetNumber` on a `GIN_DATA` page.** On data pages
   `pd_lower` encodes a payload byte count, not a line-pointer count
   ([ginblock.h#GinDataPageSetDataSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L305-L317)).
2. **`maxoff` is forced to `InvalidOffsetNumber` on compressed data leaves**,
   so it is not a TID count there
   ([gindatapage.c#compressed-maxoff](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L989-L996)).
3. **A pre-9.4 uncompressed data leaf has untrustworthy `pd_lower`.** GIN
   itself substitutes zero free space for such a page
   ([gindatapage.c#compressed-only-freespace](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L525-L528)).
   Report it as a separate legacy class rather than folding it into density.

Densities are ratios of sums over sampled pages of one class, matching
`pgstatindex`'s existing `avg_leaf_density` shape, and return `NaN` when the
class denominator is zero, matching its existing degenerate output
([pgstatindex.c#NaN-cases](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).

### Locking, concurrency, and integrity

One `AccessShareLock` from open to close. That conflicts only with
`AccessExclusiveLock`, so DML, pending-list cleanup, and VACUUM all overlap the
diagnostic. Specifically, `ginInsertCleanup` can fire from an ordinary insert
whenever the pending list exceeds its limit
([ginfast.c#cleanup-trigger](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L437-L461)),
which means `pending_pages` can be exact at the instant block 0 was read and
already wrong by the time the sample finishes.

The result is therefore a mixed-time observation, and more so than
`pgstatindex`'s:

- The metapage lock covers its fields as one observation, then is released
  ([pgstatindex.c#metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L541-L554)).
- `nblocks` is captured once; later-appended blocks are outside both the sample
  population and `index_size`.
- Each sampled page is share-locked only while classified.
- Sampling widens the statistical gap but does not create the
  no-whole-index-snapshot boundary, which the documentation already states for
  `pgstatindex`
  ([pgstattuple.sgml#snapshot-caveat](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L275-L279)).

`pgstatginindex` has no such caveat paragraph today because it reads one page
under one lock
([pgstattuple.sgml#pgstatginindex](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L298-L356)).
The new function needs a stronger one.

`BAS_BULKREAD` gives a 256 KiB ring capped at one eighth of `shared_buffers`
([freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L536-L588)).
A sparse sample reads few pages, but the ring still bounds replacement.

Validation must be explicit, not accidental. GIN has no page-ID word; the only
structural signal is the 8-byte special size
([ginblock.h#no-page-id](../../../../raw/postgres-12/src/include/access/ginblock.h#L18-L28)),
which `gin_leafpage_items` checks
([ginfuncs.c#special-size-check](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L187-L193)).
Check it per sampled page. Unlike `pgstathashindex`, which raises
`ERRCODE_INDEX_CORRUPTED` on an unrecognized page type
([pgstatindex.c#hash-unexpected-type](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L676-L682)),
prefer an `other_pages` bucket: aborting a sampled scan on one odd page
destroys the whole answer. A sample is not an integrity check either way —
unsampled blocks are never looked at.

### Extension, build, and generated-file implications

`pgstattuple` ships a 1.4 base script plus a 1.4-to-1.5 update, and
`default_version` is 1.5; there is no `pgstattuple--1.5.sql`
([pgstattuple.control#default-version](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5),
[Makefile#DATA](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L6-L13)).
So a fresh install runs `1.4.sql` then `1.4--1.5.sql`, using the extension
manager's install-then-update path
([extension.c#install-path-selection](../../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400),
[extension.c#install-then-update](../../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550)).

A complete in-tree patch would:

1. add `PG_FUNCTION_INFO_V1(pgstatginindex_approx)` and the C symbol, next to
   the existing GIN entry points
   ([pgstatindex.c#PG_FUNCTION_INFO_V1](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L55-L68),
   [fmgr.h#PG_FUNCTION_INFO_V1](../../../../raw/postgres-12/src/include/fmgr.h#L392-L413));
2. add `pgstattuple--1.5--1.6.sql` to `DATA` and bump `default_version` to 1.6
   ([Makefile#DATA](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L6-L13));
3. add the varlistentry and a caveat paragraph to `pgstattuple.sgml` after the
   existing GIN entry
   ([pgstattuple.sgml#pgstatginindex](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L298-L356));
4. add a new regression file, because `REGRESS = pgstattuple` is a single test
   whose stated design keeps every relation empty
   ([Makefile#REGRESS](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L13),
   [pgstattuple.sql#preamble](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L3-L7));
   `pageinspect` already demonstrates a per-AM `REGRESS` list
   ([pageinspect/Makefile#REGRESS](../../../../raw/postgres-12/contrib/pageinspect/Makefile#L15)); and
5. add an `OBJS` entry only if a new C file is introduced
   ([Makefile#OBJS](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L3-L4)).

Generated-header impact is small. `pgstatindex.c` already includes
`access/gin_private.h` and `catalog/pg_am.h`, and `GIN_AM_OID` comes from the
generated `pg_am_d.h` emitted by `genbki.pl` from `pg_am.dat`
([pgstatindex.c#includes](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L28-L44),
[pg_am.dat#GIN_AM_OID](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29),
[genbki.pl#OID-symbols](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603)).
The only new include is `utils/sampling.h`. No new catalog, parser rule, or
index-AM callback is needed, and `ginblock.h` reaches the module through
`gin_private.h`.

## Pros

- **It measures something v12 cannot measure at all.** Every other
  `pgstattuple` entry point refuses a GIN index
  ([pgstattuple.c#GIN-reject](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276)),
  and the existing GIN function reads one block
  ([pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L541-L554)).
- **Deleted pages become visible.** VACUUM's own classifier merges
  deleted-but-unrecyclable pages into `nDataPages` and drops recyclable ones
  entirely
  ([ginvacuum.c#classification-loop](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L746-L777)).
  Measured: 168 of 409 ordinary blocks (41.1%) invisible on one fixture.
- **Two fields stay exact for free.** `pending_pages` and `pending_tuples`
  come from the metapage, which maintains them in real time
  ([ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L636)).
- **Post-stratification measurably beats plain expansion.** Standard deviation
  fell 85% and the low bias vanished on a 31%-pending index; see
  [Post-stratification measured](#post-stratification-measured).
- **Full sample is an exact census.** `BlockSampler` selects every block when
  the population is not larger than the request
  ([sampling.c#Algorithm-S-contract](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L35)).
  Verified on all seven fixtures.
- **Entry-leaf density is cheap and accurate.** 137 of 13,672 pages (1.00%)
  reproduced a 50.44% density with 0.00 maximum error over 100 seeds.
- **Ascending block order plus `BAS_BULKREAD` bounds the buffer footprint**
  ([sampling.c#BlockSampler_Next](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L59-L112),
  [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L536-L588)).
- **`approx_` naming keeps estimates labelled**, following
  `pgstattuple_approx`
  ([pgstattuple--1.4.sql#pgstattuple_approx](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L81-L95)).

## Cons

- **Rare classes stay unreliable at any affordable sample size.** A 12-page
  posting-tree internal class in an 8,776-block index was missed by 87 of 100
  runs at 1%, 32 of 100 at 9.42%, and still 6 of 100 at 18.52%. No floor fixes
  this; only a near-full scan does.
- **The derived floor makes cost data-dependent.** It produced 1,727 pages
  (18.52%) against a requested 1% on one fixture, and 50 on another. Without
  `max_sample_pages` a caller cannot bound the work.
- **The derived floor trusts stale counters.** It is computed from
  `nDataPages`/`nTotalPages`, which are as of the last VACUUM
  ([ginblock.h#GinMetaPageData](../../../../raw/postgres-12/src/include/access/ginblock.h#L76-L82)).
  Measured drift: `nTotalPages` read 548 while the index held 794 blocks.
- **A single-page class yields `NaN` almost always.** A fixture with exactly
  one entry leaf returned `NaN` for entry-leaf density in 88 of 100 runs at
  12.22% coverage, matching `pgstatindex`'s zero-denominator behavior
  ([pgstatindex.c#NaN-cases](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).
- **`pending_pages` is exact only at metapage-read time.** An ordinary insert
  can trigger `ginInsertCleanup` mid-sample
  ([ginfast.c#cleanup-trigger](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L437-L461)),
  which also invalidates R2's stratum size.
- **Per-class capacity denominators are an extra correctness surface.** Entry,
  data, and list pages have three different capacities, and pre-9.4
  uncompressed data leaves have untrustworthy `pd_lower`
  ([ginblock.h#pre-9.4-caveat](../../../../raw/postgres-12/src/include/access/ginblock.h#L305-L312)).
- **The native sampler has scale limits**: an `int` sample count against a
  32-bit unsigned population, and a skip loop that can do work proportional to
  the population between two returned blocks, during which the caller's
  `CHECK_FOR_INTERRUPTS` cannot fire
  ([sampling.h#BlockSamplerData](../../../../raw/postgres-12/src/include/utils/sampling.h#L28-L36),
  [sampling.c#skip-loop](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L97-L107)).
- **It cannot certify integrity.** Unsampled blocks are never read, and the
  only available structural check is the 8-byte special size
  ([ginblock.h#no-page-id](../../../../raw/postgres-12/src/include/access/ginblock.h#L18-L28)).
- **New long-lived API and test surface.** C, SQL upgrade script, control
  version, Makefile, docs, permissions, and a new regression file — and v12's
  GIN regression coverage is a single empty index today
  ([pgstattuple.out#GIN-result](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L126-L131)).

## SQL Prototype Using `contrib/pageinspect`

### Prototype boundaries

The prototype exists to prove that the estimator and the classification are
reproducible from v12 interfaces. It is not a substitute for the C function.

- `pageinspect` is superuser-only by module-level design
  ([pageinspect.sgml#superuser](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14)),
  and every entry point re-checks
  ([ginfuncs.c#superuser-check](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L46-L49),
  [rawpage.c#superuser-check](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106)).
- `get_raw_page` re-opens and re-closes the relation per call, uses a `NULL`
  buffer strategy, and holds the share lock only across one `memcpy`
  ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L94-L172)).
  So the prototype holds no continuous relation lock and gets no
  `BAS_BULKREAD` protection.
- Its block number is SQL `int4`
  ([pageinspect--1.5.sql#get_raw_page](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L9-L17)),
  capping addressable blocks well below `BlockNumber`
  ([block.h#BlockNumber](../../../../raw/postgres-12/src/include/storage/block.h#L17-L35)).
- `ORDER BY random() LIMIT k` enumerates every candidate block number and sorts
  it, so the prototype is O(N) even when it reads k pages. The C proposal is
  not.
- `gin_page_opaque_info` performs no page-kind validation at all
  ([ginfuncs.c#gin_page_opaque_info](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154)).
  The prototype survives a non-GIN relation only because
  `gin_metapage_info` demands `flags == GIN_META` on block 0
  ([ginfuncs.c#META-check](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L53-L59)).
  That is luck, not a design; the C function must test `relam`.
- The helpers below are user objects, not extension members. Drop them when
  finished.

### Setup

`statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so these `SET`
commands take effect for the session only and need neither reload nor restart
([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386),
[guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)).

```sql
SET /* wiki_pgstatginindex_sample_timeout */ statement_timeout = '5min';
SET /* wiki_pgstatginindex_sample_timeout */ lock_timeout = '2s';
CREATE /* wiki_pgstatginindex_sample_setup */ EXTENSION IF NOT EXISTS pageinspect;

SELECT /* wiki_pgstatginindex_sample_schema */ n.nspname AS pageinspect_schema
FROM pg_extension AS e
JOIN pg_namespace AS n ON n.oid = e.extnamespace
WHERE e.extname = 'pageinspect';
```

`pageinspect` is relocatable, so confirm its schema is on `search_path`
([pageinspect.control#relocatable](../../../../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5)).

### Exact page census

This is the reference the sampler is measured against. The capacity expressions
are derived from `page_header`'s `special` offset: 24 bytes of page header for
entry and list pages, and 8 more for a data page's right bound
([pageinspect--1.5.sql#page_header](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L19-L33),
[ginblock.h#GinDataPageMaxDataSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L319-L326)).

```sql
CREATE /* wiki_pgstatginindex_census */ FUNCTION gin_census(idx regclass)
RETURNS TABLE (blkno bigint, flags text[], page_class text,
               rightlink bigint, maxoff int, lower_off int, upper_off int,
               special_off int, pagesize int, free_space int, capacity int)
LANGUAGE sql AS $$
WITH n AS (
  SELECT (pg_relation_size(idx) / current_setting('block_size')::bigint) AS nblocks
), b AS (
  SELECT g.i AS blkno, get_raw_page(idx::text, g.i::int) AS pg
  FROM n, generate_series(1, n.nblocks - 1) AS g(i)
), c AS (
  SELECT b.blkno, o.rightlink, o.maxoff, o.flags,
         h.lower::int AS lower_off, h.upper::int AS upper_off,
         h.special::int AS special_off, h.pagesize::int AS pagesize
  FROM b,
       LATERAL gin_page_opaque_info(b.pg) AS o,
       LATERAL page_header(b.pg) AS h
)
SELECT c.blkno, c.flags,
       CASE
         WHEN c.upper_off = 0                         THEN 'new'
         WHEN 'deleted'  = ANY (c.flags)              THEN 'deleted'
         WHEN 'list'     = ANY (c.flags)              THEN 'pending'
         WHEN 'data'     = ANY (c.flags)
              AND 'leaf' = ANY (c.flags)              THEN 'data_leaf'
         WHEN 'data'     = ANY (c.flags)              THEN 'data_internal'
         WHEN 'leaf'     = ANY (c.flags)              THEN 'entry_leaf'
         ELSE                                             'entry_internal'
       END AS page_class,
       c.rightlink, c.maxoff, c.lower_off, c.upper_off, c.special_off, c.pagesize,
       (c.upper_off - c.lower_off) AS free_space,
       CASE WHEN 'data' = ANY (c.flags) THEN c.special_off - 32
            ELSE c.special_off - 24 END AS capacity
FROM c;
$$;
```

The `CASE` order mirrors the flag-setting order in source: `deleted` before
`data` because `GinPageSetDeleted` ORs the bit onto existing flags
([ginvacuum.c#ginDeletePage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L187-L192)),
and `new` first because an uninitialized page has no valid special area
([bufpage.h#PageIsNew](../../../../raw/postgres-12/src/include/storage/bufpage.h#L225-L229)).
The flag names come from `gin_page_opaque_info`'s own mapping
([ginfuncs.c#flag-names](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L117-L141)).

### Sampling function

```sql
CREATE /* wiki_pgstatginindex_sampler */ FUNCTION gin_sample_stats(
                 idx regclass,
                 sample_fraction float8 DEFAULT 0.01,
                 min_sample_pages int DEFAULT 50,
                 poststratify boolean DEFAULT true)
RETURNS TABLE (version int, index_size bigint, total_pages bigint,
               pending_pages bigint, pending_tuples bigint,
               meta_n_total_pages bigint, meta_n_entry_pages bigint,
               meta_n_data_pages bigint,
               sampled_pages bigint, sampled_nonpending_pages bigint,
               scanned_percent float8,
               approx_entry_internal_pages bigint, approx_entry_leaf_pages bigint,
               approx_data_internal_pages bigint, approx_data_leaf_pages bigint,
               approx_deleted_pages bigint, approx_new_pages bigint,
               approx_entry_leaf_density float8, approx_data_leaf_density float8)
LANGUAGE plpgsql AS $$
DECLARE
  blksz   bigint := current_setting('block_size')::bigint;
  isize   bigint := pg_relation_size(idx);
  nblocks bigint := isize / blksz;
  n_ord   bigint; n_pend bigint; n_np bigint;
  k       bigint; k_np bigint;
  exp_num bigint; exp_den bigint;
  m       record; f_eff float8;
BEGIN
  SELECT * INTO m FROM gin_metapage_info(get_raw_page(idx::text, 0));   -- R1

  n_ord  := GREATEST(nblocks - 1, 0);
  n_pend := m.n_pending_pages;
  n_np   := GREATEST(n_ord - n_pend, 0);

  IF sample_fraction IS NULL OR sample_fraction <= 0 OR sample_fraction > 1
     OR sample_fraction <> sample_fraction THEN
    RAISE EXCEPTION 'sample_fraction must satisfy 0 < f <= 1, got %',
      sample_fraction USING ERRCODE = 'invalid_parameter_value';
  END IF;

  IF n_ord = 0 THEN
    k := 0;
  ELSE
    k := LEAST(n_ord, GREATEST(ceil(n_ord * sample_fraction)::bigint,
                               min_sample_pages::bigint));            -- R3
  END IF;

  CREATE TEMP TABLE IF NOT EXISTS gin_sample_scratch (
    blkno bigint, page_class text, free_space int, capacity int) ON COMMIT DROP;
  TRUNCATE gin_sample_scratch;

  IF k > 0 THEN
    INSERT INTO gin_sample_scratch
    WITH picked AS (
      SELECT g.i AS blkno FROM generate_series(1, n_ord) AS g(i)
      ORDER BY random() LIMIT k
    ), raw AS (
      SELECT p.blkno, get_raw_page(idx::text, p.blkno::int) AS pg FROM picked AS p
    )
    SELECT r.blkno,
           CASE
             WHEN h.upper = 0                 THEN 'new'
             WHEN 'deleted' = ANY (o.flags)   THEN 'deleted'
             WHEN 'list'    = ANY (o.flags)   THEN 'pending'
             WHEN 'data'    = ANY (o.flags)
                  AND 'leaf' = ANY (o.flags)  THEN 'data_leaf'
             WHEN 'data'    = ANY (o.flags)   THEN 'data_internal'
             WHEN 'leaf'    = ANY (o.flags)   THEN 'entry_leaf'
             ELSE                                  'entry_internal'
           END,
           (h.upper - h.lower)::int,
           CASE WHEN 'data' = ANY (o.flags) THEN (h.special - 32)::int
                ELSE (h.special - 24)::int END
    FROM raw AS r,
         LATERAL gin_page_opaque_info(r.pg) AS o,
         LATERAL page_header(r.pg) AS h;
  END IF;

  SELECT count(*), count(*) FILTER (WHERE page_class <> 'pending')
    INTO k, k_np FROM gin_sample_scratch;

  f_eff := CASE WHEN n_ord = 0 THEN 0 ELSE 100.0 * k / n_ord END;

  IF poststratify THEN exp_num := n_np;  exp_den := k_np;   -- R2
  ELSE                 exp_num := n_ord; exp_den := k;
  END IF;

  RETURN QUERY
  SELECT m.version, isize, nblocks,
         n_pend, m.n_pending_tuples,
         m.n_total_pages, m.n_entry_pages, m.n_data_pages,
         k, k_np, f_eff,
         CASE WHEN exp_den = 0 THEN 0 ELSE round(count(*) FILTER
           (WHERE s.page_class = 'entry_internal') * exp_num::numeric / exp_den)::bigint END,
         CASE WHEN exp_den = 0 THEN 0 ELSE round(count(*) FILTER
           (WHERE s.page_class = 'entry_leaf')     * exp_num::numeric / exp_den)::bigint END,
         CASE WHEN exp_den = 0 THEN 0 ELSE round(count(*) FILTER
           (WHERE s.page_class = 'data_internal')  * exp_num::numeric / exp_den)::bigint END,
         CASE WHEN exp_den = 0 THEN 0 ELSE round(count(*) FILTER
           (WHERE s.page_class = 'data_leaf')      * exp_num::numeric / exp_den)::bigint END,
         CASE WHEN exp_den = 0 THEN 0 ELSE round(count(*) FILTER
           (WHERE s.page_class = 'deleted')        * exp_num::numeric / exp_den)::bigint END,
         CASE WHEN exp_den = 0 THEN 0 ELSE round(count(*) FILTER
           (WHERE s.page_class = 'new')            * exp_num::numeric / exp_den)::bigint END,
         CASE WHEN sum(s.capacity) FILTER (WHERE s.page_class = 'entry_leaf') > 0
              THEN 100.0 - 100.0
                   * sum(s.free_space) FILTER (WHERE s.page_class = 'entry_leaf')
                   / sum(s.capacity)   FILTER (WHERE s.page_class = 'entry_leaf')
              ELSE 'NaN'::float8 END,
         CASE WHEN sum(s.capacity) FILTER (WHERE s.page_class = 'data_leaf') > 0
              THEN 100.0 - 100.0
                   * sum(s.free_space) FILTER (WHERE s.page_class = 'data_leaf')
                   / sum(s.capacity)   FILTER (WHERE s.page_class = 'data_leaf')
              ELSE 'NaN'::float8 END
  FROM gin_sample_scratch AS s;
END;
$$;
```

`min_sample_pages` stands in for R3's derived floor so the two floor policies
can be compared directly; the derived value is computed separately below.
`poststratify` exists only to measure R2 against plain expansion.

### Exact pending-list walk

R1's optional extension: the pending list is a singly linked chain from
`metadata->head` through each page's `rightlink`, so it can be enumerated
exactly, and its cost is known in advance from `nPendingPages`
([ginblock.h#pending-head-tail](../../../../raw/postgres-12/src/include/access/ginblock.h#L56-L74),
[ginfast.c#writeListPage](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L57-L141)).
On a `GIN_LIST` page, `maxoff` is a heap-tuple count, not an item-pointer count
([ginblock.h#maxoff-meaning](../../../../raw/postgres-12/src/include/access/ginblock.h#L29-L36)).

```sql
CREATE /* wiki_pgstatginindex_pending_walk */ FUNCTION gin_walk_pending(idx regclass)
RETURNS TABLE (walked_pages bigint, walked_tuples bigint,
               fullrow_pages bigint, free_space bigint, pages_read bigint)
LANGUAGE plpgsql AS $$
DECLARE
  blk bigint; pg bytea; o record; h record;
  n bigint := 0; t bigint := 0; fr bigint := 0; fs bigint := 0;
BEGIN
  SELECT pending_head INTO blk FROM gin_metapage_info(get_raw_page(idx::text, 0));
  WHILE blk <> 4294967295 LOOP          -- InvalidBlockNumber
    pg := get_raw_page(idx::text, blk::int);
    SELECT * INTO o FROM gin_page_opaque_info(pg);
    SELECT * INTO h FROM page_header(pg);
    n := n + 1;
    t := t + o.maxoff;
    IF 'list_fullrow' = ANY (o.flags) THEN fr := fr + 1; END IF;
    fs := fs + (h.upper - h.lower);
    blk := o.rightlink;
  END LOOP;
  RETURN QUERY SELECT n, t, fr, fs, n + 1;
END;
$$;
```

`4294967295` is `InvalidBlockNumber` as `gin_page_opaque_info` renders it in
`bigint`, which the in-tree expected output also shows
([gin.out#InvalidBlockNumber](../../../../raw/postgres-12/contrib/pageinspect/expected/gin.out#L7-L8)).

## Tests To Add

v12's `pgstattuple` regression suite creates a GIN index on a table that never
receives a row, by explicit design
([pgstattuple.sql#preamble](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L3-L7),
[pgstattuple.sql#GIN-test](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L44-L49)),
so the only recorded GIN result is `(2, 0, 0)`
([pgstattuple.out#GIN-result](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L126-L131)).
There is no coverage anywhere in `contrib/pgstattuple` of a non-empty GIN
index, a populated pending list, a posting tree, or a deleted page.

A patch should add, in a separate regression file:

1. Empty-index output: `total_pages = 2`, `scanned_percent = 100`, both
   densities `NaN`.
2. Wrong-AM rejection for a B-tree and a hash index, matching the existing
   `relation "%s" is not a GIN index` text
   ([pgstattuple.out#GIN-errors](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L140-L152)).
3. `sample_fraction` validation: `0`, `1.5`, `NaN`, `NULL`.
4. `sample_fraction = 1.0` equality against a full census on a populated index.
5. A `fastupdate = on` index with inserts, asserting `pending_pages > 0` and
   that it equals a pending-list walk.
6. A posting-tree index after `DELETE` plus `VACUUM`, asserting
   `approx_deleted_pages > 0`.

Platform independence requires the boolean/normalized idioms already used
in-tree: `COUNT(*) > 0` and a block count derived from
`pg_relation_size / current_setting('block_size')`
([gin.sql#platform-independent-idiom](../../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql#L16-L19)),
and `WITH (fastupdate = off)` where deterministic structure is needed
([gin.sql#fastupdate-off](../../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql#L1-L3)).

## Executed Comparison With the Exact Census

### Test scope and fixtures

The pinned 12.2 source was built out of tree under `.wiki-runtime/`, and one
isolated server (`autovacuum = off`, 8 kB `block_size`) ran `pgstattuple` 1.5
and `pageinspect` 1.7. Seven GIN fixtures on `int[]` columns were built, then
6,100 seeded sampler runs: 5,600 in the main grid (7 fixtures x {1%, 5%} x
{floor 0, floor 50} x {post-stratified, plain} x 100 seeds), 200 for the floor
comparison, and 300 for the derived floor. Reproducibility used
`setseed(seed / 1000.0)` before each call.

`fastupdate` and `gin_pending_list_limit` were set per index to keep pending
lists from self-cleaning; both are GIN reloptions requiring
`AccessExclusiveLock` to change
([reloptions.c#fastupdate](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133),
[reloptions.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L322-L330)),
and the reloption overrides the `PGC_USERSET` GUC of the same name, which
applies at session/transaction scope and needs neither reload nor restart
([guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184),
[gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-12/src/include/access/gin_private.h#L24-L39),
[config.sgml#gin_pending_list_limit](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7972-L7992)).

First, the derived capacity constants were confirmed against the running
server. Every GIN page reported `pagesize = 8192` and `special = 8184`, giving
8160 for entry and list pages and 8152 for data pages — matching
`GinDataPageMaxDataSize` and `GinListPageSize` exactly
([ginblock.h#GinDataPageMaxDataSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L319-L326),
[ginblock.h#GinListPageSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L328-L332)).

### Exact full-scan truth

| Index | ordinary pages | entry int. | entry leaf | data int. | data leaf | pending | deleted | new | entry density | data density |
|---|---|---|---|---|---|---|---|---|---|---|
| `f_big_gin` | 13672 | 67 | 13605 | 0 | 0 | 0 | 0 | 0 | 50.44 | — |
| `f_mixed_gin` | 8776 | 40 | 8163 | 12 | 96 | 393 | 72 | 0 | 36.75 | 14.06 |
| `f_pending_gin` | 793 | 3 | 544 | 0 | 0 | 246 | 0 | 0 | 50.46 | — |
| `f_small_gin` | 411 | 3 | 408 | 0 | 0 | 0 | 0 | 0 | 50.46 | — |
| `f_bloat_gin` | 409 | 0 | 1 | 24 | 216 | 0 | 168 | 0 | 5.88 | 12.15 |
| `f_partial_gin` | 164 | 1 | 163 | 0 | 0 | 0 | 0 | 0 | 50.52 | — |
| `f_empty_gin` | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0.00 | — |

`f_big_gin` was 112,009,216 bytes (106.82 MiB), above 100 MiB;
`f_mixed_gin` was 71,901,184 bytes (68.57 MiB). No fixture produced a `new`
page, so that class is untested here.

Only `f_bloat_gin` and `f_mixed_gin` contain posting trees. Building one
requires a key whose compressed posting list no longer fits inside one entry
tuple, bounded by `GinMaxItemSize`
([ginblock.h#GinMaxItemSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L245-L256));
a first attempt using 1,500 distinct keys over 600,000 rows produced entry
pages only, and 8 distinct keys over 900,000 rows was needed.

### Full-sample equivalence

At `sample_fraction = 1.0`, on all seven fixtures, `sampled_pages` equalled
`ordinary_pages`, `scanned_percent` was 100, and all six class estimates plus
`pending_pages` matched the census exactly. This is the behavior
`BlockSampler_Init`'s contract predicts when the population does not exceed
the requested sample
([sampling.c#Algorithm-S-contract](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L35)).

### Cost at each requested fraction

| Index | ordinary pages | k at 1% (floor 50) | k at 5% (floor 50) | full scan | 1% read as % of full |
|---|---|---|---|---|---|
| `f_big_gin` | 13672 | 137 | 684 | 13673 | 1.01 |
| `f_mixed_gin` | 8776 | 88 | 439 | 8777 | 1.01 |
| `f_pending_gin` | 793 | 50 | 50 | 794 | 6.42 |
| `f_small_gin` | 411 | 50 | 50 | 412 | 12.38 |
| `f_bloat_gin` | 409 | 50 | 50 | 410 | 12.44 |
| `f_partial_gin` | 164 | 50 | 50 | 165 | 30.91 |
| `f_empty_gin` | 1 | 1 | 1 | 2 | 100.00 |

A 50-page floor binds only below roughly 5,000 ordinary pages at 1%, and is
capped by the population on the empty index.

### Post-stratification measured

R2 against plain `n_ord / k` expansion, requested 1%, 100 seeds, estimating
`entry_leaf_pages`:

| Index | pending share | estimator | mean | stddev | mean abs err | max abs err |
|---|---|---|---|---|---|---|
| `f_pending_gin` (truth 544) | 246/793 = 31.0% | post-stratified | **544.0** | **6.8** | **5.0** | **27** |
| `f_pending_gin` (truth 544) | 246/793 = 31.0% | plain | 537.1 | 46.6 | 39.2 | 116 |
| `f_mixed_gin` (truth 8163) | 393/8776 = 4.5% | post-stratified | **8148.0** | **149.0** | **119.6** | **379** |
| `f_mixed_gin` (truth 8163) | 393/8776 = 4.5% | plain | 8111.8 | 227.8 | 187.0 | 584 |

Post-stratification cut the standard deviation by 85% at a 31% pending share
and by 35% at 4.5%, and removed a consistent low bias (544.0 versus 537.1
against a truth of 544). This is the clearest measured argument for a
GIN-specific estimator, and it works only because `nPendingPages` is exact.

### The floor: the absolute floor lost

This test was designed to confirm that a GIN absolute floor beats the B-tree
proposal's size-based percentage floor. It showed the opposite.
`f_mixed_gin` is 68.57 MiB, so the B-tree rule (under 100 MiB, raise to 10%)
would sample 878 of 8,776 pages; the absolute floor of 50 does not bind at all
because 1% is already 88.

| Sample size | % of ordinary | data int. missed (truth 12) | max err | deleted missed (truth 72) | max err | data leaf max err | data density `NaN` runs |
|---|---|---|---|---|---|---|---|
| 88 (1%, floor 50) | 1.01 | **87/100** | 90 | **33/100** | 220 | 308 | **36/100** |
| 878 (B-tree 10% rule) | 9.42 | 32/100 | 28 | **0/100** | 79 | 85 | **0/100** |

The percentage floor was better on every measure. Entry-leaf density was
already excellent at 88 pages (max error 0.11), so the loss is confined to the
posting-tree and deleted classes — exactly the classes a GIN bloat diagnostic
exists to report. The absolute floor was therefore replaced by the
metapage-derived floor in R3.

### Rare-class detection at the derived floor

Derived floor `ceil(20 * nTotalPages / nDataPages)` per fixture:

| Index | ordinary | `nTotalPages` | `nDataPages` | stale data share | derived floor | final k at 1% |
|---|---|---|---|---|---|---|
| `f_big_gin` | 13672 | 13673 | 0 | 0.000% | — | 137 |
| `f_mixed_gin` | 9324 | 9325 | 108 | 1.158% | 1727 | 1727 |
| `f_pending_gin` | 793 | 548 | 0 | 0.000% | — | 50 |
| `f_small_gin` | 411 | 412 | 0 | 0.000% | — | 50 |
| `f_bloat_gin` | 409 | 410 | 240 | 58.537% | 35 | 50 |
| `f_partial_gin` | 164 | 165 | 0 | 0.000% | — | 50 |
| `f_empty_gin` | 1 | 2 | 0 | 0.000% | — | 1 |

`f_mixed_gin` was re-measured after a VACUUM flushed its pending list; its
truth became 9,324 ordinary pages with 43 entry internal, 8,708 entry leaf,
12 data internal, 96 data leaf, 0 pending, and 465 deleted pages, with
densities 37.60% and 24.82%. Three sample sizes, 100 seeds each:

| pages read | % of ordinary | data int. mean (truth 12) | data int. missed | data leaf max err (truth 96) | deleted max err (truth 465) | data density `NaN` |
|---|---|---|---|---|---|---|
| 94 (1%) | 1.01 | 12.9 | **87/100** | 301 | 527 | 37/100 |
| 878 (10%) | 9.42 | 10.9 | 32/100 | 85 | 193 | 0/100 |
| **1727 (derived)** | **18.52** | **11.7** | **6/100** | **47** | **173** | **0/100** |

The derived floor works, at a real cost: 18.52% of the index against a
requested 1%. Two limits are visible. It cannot make a 12-page class fully
reliable even at 18.52% coverage, and it depends on stale counters —
`f_pending_gin`'s `nTotalPages` read 548 against 794 live blocks, so a similar
fixture with a stale data share would derive the wrong floor.

### Density estimates

Requested 1%, floor 50, post-stratified, 100 seeds. `NaN` had to be detected
with an explicit comparison against the `NaN` literal because PostgreSQL
`float8` treats `NaN = NaN` as true.

| Index | entry leaves | truth | mean | max err | `NaN` runs | data leaves | truth | mean | max err | `NaN` runs |
|---|---|---|---|---|---|---|---|---|---|---|
| `f_big_gin` | 13605 | 50.44 | 50.44 | 0.00 | 0 | 0 | — | — | — | 100 |
| `f_mixed_gin` | 8163 | 36.75 | 36.75 | 0.11 | 0 | 96 | 14.06 | 14.50 | 14.06 | 36 |
| `f_pending_gin` | 544 | 50.46 | 50.46 | 0.34 | 0 | 0 | — | — | — | 100 |
| `f_small_gin` | 408 | 50.46 | 50.47 | 0.15 | 0 | 0 | — | — | — | 100 |
| `f_bloat_gin` | 1 | 5.88 | 5.88 | 0.00 | **88** | 216 | 12.15 | 12.20 | 2.00 | 0 |
| `f_partial_gin` | 163 | 50.52 | 50.53 | 0.19 | 0 | 0 | — | — | — | 100 |
| `f_empty_gin` | 1 | 0.00 | 0.00 | 0.00 | 0 | 0 | — | — | — | 100 |

Density for a dominant class is nearly free: 137 of 13,672 pages reproduced
50.44% with zero error across 100 seeds. Density for a one-page class is
`NaN` in 88 of 100 runs, and the 12 runs that did sample it returned the exact
value. A 96-page class produced a 14.06-point maximum error, meaning at least
one run sampled only an empty data leaf.

### Metapage drift the exact fields expose

These figures involve no sampling; they come from block 0 and the census.

| Index | live blocks | `nTotalPages` | `nEntryPages` | `nDataPages` | true entry | true data | true deleted | true pending |
|---|---|---|---|---|---|---|---|---|
| `f_big_gin` | 13673 | 13673 | 13672 | 0 | 13672 | 0 | 0 | 0 |
| `f_mixed_gin` | 8777 | 8384 | 8203 | 180 | 8203 | 108 | 72 | 393 |
| `f_pending_gin` | 794 | 548 | 547 | 0 | 547 | 0 | 0 | 246 |
| `f_small_gin` | 412 | 412 | 411 | 0 | 411 | 0 | 0 | 0 |
| `f_bloat_gin` | 410 | 410 | 1 | 240 | 1 | 240 | 168 | 0 |
| `f_partial_gin` | 165 | 165 | 164 | 0 | 164 | 0 | 0 | 0 |
| `f_empty_gin` | 2 | 2 | 1 | 0 | 1 | 0 | 0 | 0 |

Four separate failures of the metapage view, each traceable to the
classification loop:

- **`f_mixed_gin`: `nDataPages = 180` conflates 108 real data pages with 72
  deleted ones.** They still carried `GIN_DATA` and their delete XID was too
  recent for `GinPageIsRecyclable`, so they took the `GinPageIsData` branch
  ([ginvacuum.c#classification-loop](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L758-L767)).
- **`f_bloat_gin`: after a second VACUUM, `nDataPages` fell 408 -> 240 and the
  168 deleted pages vanished from every metapage class.** Their delete XID had
  aged past `RecentGlobalXmin`, so they took the recyclable branch, which only
  records an FSM entry
  ([ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138),
  [ginvacuum.c:758-763](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L758-L763)).
  The metapage then accounted for 241 of 409 ordinary pages; **168 blocks
  (41.1%) were invisible**, and the fork was never shortened.
- **Pending pages appear in no metapage class.** `f_pending_gin` shows
  `nEntryPages = 547` against 793 ordinary blocks, with the 246-page pending
  list excluded by `!GinPageIsList(page)`
  ([ginvacuum.c:768-774](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L768-L774)).
- **`nTotalPages` lags the live length.** It read 8384 against 8777 live blocks
  and 548 against 794, because it is the length captured during the last
  VACUUM
  ([ginvacuum.c:779-781](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L781)).

The pending-list walk (R1's optional extension) matched the metapage exactly on
both pending fixtures: 393 walked pages / 40,000 tuples and 246 / 50,000,
every page `list_fullrow`, densities 99.79% and 99.63%, and cost exactly
`nPendingPages + 1` reads. Both lists also happened to be contiguous physical
tails (`pending_tail - pending_head + 1 = nPendingPages`), but nothing in
source guarantees this, because `GinNewBuffer` may hand back a recycled block
from the FSM
([ginutil.c#GinNewBuffer](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)).

One further measurement is worth recording because it is counter-intuitive:
VACUUM-ing `f_mixed_gin` to flush its 393-page pending list **grew** the index
from 8,777 to 9,325 blocks and left 465 deleted pages. `nEntryPages` rose
8,203 -> 8,751 while `nDataPages` fell 180 -> 108, and after the run the
metapage accounted for 8,860 of 9,325 blocks, leaving 465 (4.99%)
unrepresented. The flush is a bulk insert into an entry tree from which VACUUM
never deletes pages
([README#page-deletion](../../../../raw/postgres-12/src/backend/access/gin/README#L389-L396)),
and the drained list pages become `GIN_DELETED`
([ginfast.c#shiftList-flags](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632)).

### Error paths

| Input | Prototype result |
|---|---|
| `sample_fraction = 0` | `ERROR: sample_fraction must satisfy 0 < f <= 1, got 0` |
| `sample_fraction = 1.5` | rejected |
| `sample_fraction = 'NaN'` | rejected |
| `sample_fraction = NULL` | rejected (the C function would be `STRICT` instead) |
| a B-tree index | `ERROR: input page is not a GIN metapage / DETAIL: Flags 0000, expected 0008` |
| `pgstatginindex` on the same B-tree index | `ERROR: relation "f_small_btree" is not a GIN index` |

The B-tree case is the important one. The prototype has no access-method check;
it survived only because `gin_metapage_info` requires `flags == GIN_META`
([ginfuncs.c#META-check](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L53-L59))
and a B-tree metapage's bytes at that offset happened to read as `0000`. The
proposed C function must test `IS_GIN(rel)` explicitly, as
`pgstatginindex_internal` does
([pgstatindex.c#GIN-guard](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L525-L529)).

### What the comparison establishes

1. Full-sample equivalence holds on all seven fixtures.
2. Post-stratification on the exactly-known pending stratum is a real,
   measured improvement, unique to GIN among v12 index AMs.
3. The absolute floor this page first proposed was worse than the B-tree
   percentage floor; the metapage-derived floor is better but costs 18.52% of
   one fixture.
4. Dominant-class page counts and densities are accurate at 1%; rare classes
   are not reliable at any affordable fraction.
5. The metapage misreports GIN physical structure in four distinct ways, up to
   41.1% of one index's blocks, and only a physical scan or sample sees it.

Test objects and the temporary server are disposable artifacts under
`.wiki-runtime/`; `raw/postgres-12/` was not modified.

## Follow-Up: Bloat and Wasted Space

### Short answer

Partly, and only if two extra fields are added and the output is read against a
structural baseline instead of against zero.

| Question | Answer |
|---|---|
| Can it measure bloat? | Yes as a **classifier**, once `approx_free_percent` is compared against GIN's ~50% structural baseline. At a requested 1%, an alarm threshold of 60% separated four bloated fixtures from four healthy ones in 800 of 800 seeded runs. |
| Can it measure wasted space? | It can measure **free space**, not wasted space. The two differ by GIN's structural baseline, which is large: four healthy, never-deleted indexes each reported 49.5-49.7% free. |
| Can it measure bloat *before* VACUUM? | No. Deleting 360,000 of 400,000 rows changed nothing at all: every page class, both densities, and `free_space` were byte-identical before and after the `DELETE`. |
| Can it price a `REINDEX`? | Only with a baseline correction. Of the bytes raw `free_space` reported, a real rebuild returned 97.7%, 91.4%, 70.0% and 0.0% on four fixtures; subtracting the structural baseline first predicted 96.2-98.5% of the recovered bytes on the three bloated ones and exactly 0 on the healthy one. |

Two new exact fields and four new estimated fields close most of the gap; see
[Proposed additional output fields](#proposed-additional-output-fields). The
irreducible blind spot is dead heap pointers that VACUUM has not yet removed,
because GIN stores them as ordinary payload bytes.

### Why free space is not wasted space in a GIN index

GIN has no `fillfactor`. `ginoptions` parses exactly two reloptions,
`fastupdate` and `gin_pending_list_limit`
([ginutil.c#ginoptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L629)),
so there is no configured target density to compare a measured density against.
That is the structural difference from `pgstatindex`, whose `avg_leaf_density`
is read against the B-tree's fillfactor
([pgstatindex.c#avg_leaf_density](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351)).

The baseline is set instead by the entry-tree split rule. `entrySplitPage`
copies every tuple into a workspace and switches to the right page as soon as
`lsize > totalsize / 2`, deliberately equalizing **data size**, not tuple count
([ginentrypage.c#entrySplitPage](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L623-L689)).
There is no rightmost-page or sorted-insertion special case, so a freshly built
entry tree lands near half full and stays there.

Measured on the pinned build, four indexes that had never had a row deleted:

| Index | ordinary pages | entry-leaf density | `free_percent` |
|---|---|---|---|
| `f_big_gin` | 13672 | 50.44 | **49.56** |
| `f_pending_gin` | 793 | 50.46 | **49.57** |
| `f_small_gin` | 411 | 50.46 | **49.66** |
| `f_partial_gin` | 164 | 50.52 | **49.54** |

The decisive control is a sibling index built over identical rows. Building
`f_small_gin2` on the same column of the same table produced a byte-identical
result: 412 blocks, 3 entry internal, 408 entry leaf, density 50.46,
`free_percent` 49.66, and the same 1,688,200 used bytes. A rebuild of
`f_small_gin` would therefore recover **zero bytes** while the function reports
**49.66% free**. Any tool that equates free space with recoverable space is
wrong by half the index here.

Posting-tree leaves behave differently, and the difference is also in source:
`leafRepackItems` fills the left page as full as it can, and
`dataBeginPlaceToPageLeaf` keeps that packing only when `btree->isBuild`,
otherwise rebalancing toward 50/50 or 75% when appending
([gindatapage.c#build-versus-split-packing](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667)).
Measured fresh-build data-leaf densities were 83.54% (`f_waste_gin2`) and
93.81% (`f_bloat_gin2`), against 20.15% and 12.15% for their bloated
counterparts. So a low data-leaf density is a real signal, while a ~50%
entry-leaf density is not.

### The three waste shapes it does measure

**1. Whole deleted pages, split by reusability.** GIN stores a deleted page's
delete XID in the page header's `pd_prune_xid`, and `GinPageIsRecyclable`
accepts a page when it is new, or deleted with a delete XID preceding
`RecentGlobalXmin`
([ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138)).
`pd_prune_xid` is in the standard page header
([bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164)),
so one page read yields both the class and its reusability. Two deletion paths
write it differently, and the difference is visible on disk:

| Path | Flags written | Delete XID |
|---|---|---|
| posting-page deletion | `GIN_DELETED` OR'ed onto existing bits | `ReadNewTransactionId()` ([ginvacuum.c#ginDeletePage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L187-L192)) |
| pending-page drain | flags overwritten to exactly `GIN_DELETED` | never set, so it stays 0 ([ginfast.c#shiftList-flags](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632)) |

A zero delete XID makes the page immediately recyclable, because
`TransactionIdPrecedes` falls back to unsigned comparison when either argument
is not a normal XID
([transam.c#TransactionIdPrecedes](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L313)).
`shiftList` also records each drained page in the FSM directly
([ginfast.c#shiftList-fsm](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L664-L665)).
The census found exactly that split on `f_mixed_gin`: 393 deleted pages with
`prune_xid = 0` (drained pending pages) and 72 with `prune_xid = 522` (deleted
posting pages), plus 168 pages at `prune_xid = 518` on `f_bloat_gin`.

**2. Intra-page free space, per class.** `pd_upper - pd_lower`, i.e.
`PageGetExactFreeSpace`
([bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L626-L647)),
is the same quantity `pgstathashindex` accumulates
([pgstatindex.c#GetHashPageStats](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L735-L752)).

**3. The pending list, exactly.** This is the shape intra-page free space
cannot see at all. `f_pending_gin` carries 246 pending pages, 2,015,232 bytes,
**30.98% of the index**, yet its `free_percent` is 49.57 -- statistically
indistinguishable from `f_big_gin`'s 49.56. The pending-list walk explains why:
all 246 pages together hold only 7,360 free bytes, so they are 99.6% dense.
Pending bloat is invisible in any density ratio and visible only in the exact
metapage counters that rule R1 already reports.

### The waste it cannot measure

**1. Dead heap pointers before VACUUM.** This is the hard limit. Before
VACUUM, a dead TID is ordinary payload inside a posting list, so no page-header
arithmetic can see it. Measured on `f_waste_gin`, 400,000 rows over 16 keys,
deleting 360,000 of them without vacuuming:

| Stage | blocks | data leaf | deleted | entry density | data density | `free_space` | `free_percent` |
|---|---|---|---|---|---|---|---|
| built | 130 | 112 | 0 | 3.92 | 95.90 | 174560 | 16.58 |
| 90% deleted, no VACUUM | 130 | 112 | 0 | 3.92 | 95.90 | **174560** | **16.58** |
| after VACUUM | 130 | 72 | 40 | 3.92 | 20.15 | 932624 | 88.60 |

Not one field moved at the middle stage, while `pgstattuple` on the heap
reported 360,000 dead tuples, 75.13% dead. The waste only becomes measurable
once `ginVacuumItemPointers` rewrites the posting lists
([ginvacuum.c#ginVacuumItemPointers](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L47-L84)).
This is not a sampling limitation: a full `pgstathashindex`-style scan would be
equally blind, because GIN pages carry no `LP_DEAD` line pointers for a
dead-item counter to read, which is why `pgstathashindex` can report
`dead_items` and a GIN function cannot
([pgstatindex.c#dead_items](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L742-L750)).

**2. Entry tuples for keys that lost every TID.** `ginVacuumEntryPage` rebuilds
such a tuple with `plist = NULL`, `plistsize = 0` and `nitems = 0` and puts it
back on the page; it never deletes the tuple
([ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)),
and `GinFormTuple` accepts `nipd = 0`
([ginentrypage.c#GinFormTuple](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L44-L95)).
Measured on `f_keys_gin`, 4,000 distinct keys, then every row of 2,000 of them
deleted and vacuumed:

| Stage | blocks | entry leaves | entry tuples | metapage `n_entries` | density | `free_percent` |
|---|---|---|---|---|---|---|
| built | 309 | 307 | 4000 | 4000 | 53.01 | 46.92 |
| half the keys dead, vacuumed | 309 | 307 | **4000** | **4000** | 28.10 | 71.74 |
| fresh sibling build | 155 | 153 | **2000** | **2000** | 53.18 | 46.92 |

VACUUM freed the posting bytes but kept all 4,000 key tuples, and the fork never
shrank. Only a rebuild dropped the 2,000 dead keys. A page-class census counts
those tuples as used space, so this waste is reported as *occupancy*, not as
free space.

**3. Alignment padding inside an entry tuple.** `GinFormTuple` notes that
`index_form_tuple` MAXALIGNs the tuple, so "there may well be some wasted pad
space"
([ginentrypage.c#GinFormTuple-padding](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L71-L80)).
Padding lies inside the item, below `pd_lower`, so it is invisible to
`PageGetExactFreeSpace`.

**4. Whether free space is reachable.** Free space on an entry page is only
usable by keys that route to that page, and a recyclable page is only handed
back when the FSM offers it and `GinPageIsRecyclable` still accepts it
([ginutil.c#GinNewBuffer](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)).
Nothing in the output distinguishes reachable from stranded free space.

**5. Recyclability is a moment, not a property.** The census's verdict agreed
with VACUUM's own accounting, but only after the horizon advanced. The first
VACUUM of `f_waste` reported "40 index pages have been deleted, 0 are currently
reusable"; a second VACUUM reported "0 index pages have been deleted, 40 are
currently reusable", after which `pg_freespace` offered exactly 40 blocks. Those
two numbers are `pages_deleted`, incremented in `ginDeletePage`
([ginvacuum.c:234](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L234))
and by `shiftList`
([ginfast.c:588](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L588)),
and `pages_free`, which counts only the pages that were recyclable during that
one scan
([ginvacuum.c:786](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L786),
[vacuumlazy.c#index-cleanup-report](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827)).

### Proposed additional output fields

Names follow the local precedent: `pgstattuple` reports `free_space` and
`free_percent`
([pgstattuple.sgml#free_space](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L108-L117)),
and `pgstattuple_approx` prefixes its estimated versions
([pgstattuple.sgml#approx_free_space](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L593-L602)).

| Output | Derivation | Contract |
|---|---|---|
| `pending_space` | `pending_pages * BLCKSZ` | **exact**, from the metapage |
| `total_space` | `8160 * (total_pages - 1 - pending_pages)` | **exact** |
| `approx_free_space` | post-stratified sum of `pd_upper - pd_lower` over sampled live pages, plus `8160` per estimated deleted or new page | estimate |
| `approx_free_percent` | `100 * approx_free_space / total_space` | estimate |
| `approx_recyclable_pages` | sampled deleted pages passing `GinPageIsRecyclable`, post-stratified | estimate |
| `approx_recyclable_space` | `approx_recyclable_pages * BLCKSZ` | estimate |

```sql
-- proposed additions to the pgstatginindex_approx signature
    OUT pending_space BIGINT,
    OUT total_space BIGINT,
    OUT approx_free_space BIGINT,
    OUT approx_free_percent FLOAT8,
    OUT approx_recyclable_pages BIGINT,
    OUT approx_recyclable_space BIGINT
```

Four design decisions, each with an in-tree precedent:

- **Whole deleted and never-initialized pages count as free space.**
  `pgstathashindex` does exactly this for unused pages
  ([pgstatindex.c#unused-as-free](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L689-L690)).
- **The denominator excludes bookkeeping pages.** `pgstathashindex` excludes
  the metapage and bitmap pages
  ([pgstatindex.c#total_space](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L692-L702));
  the GIN analogue excludes the metapage and the pending list, which makes
  `total_space` exact because `nPendingPages` is exact.
- **The recyclability horizon is available to a contrib function.**
  `RecentGlobalXmin` is `PGDLLIMPORT`
  ([snapmgr.h#RecentGlobalXmin](../../../../raw/postgres-12/src/include/utils/snapmgr.h#L59-L62)),
  `contrib/amcheck` reads it directly
  ([verify_nbtree.c#RecentGlobalXmin](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L396-L400)),
  and `pgstatapprox.c` in this very module already computes a vacuum horizon
  with `GetOldestXmin`
  ([pgstatapprox.c#GetOldestXmin](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L72-L75)).
- **One capacity constant, with a stated error.** `total_space` uses 8160 for
  every page, but a data page's capacity is `GinDataPageMaxDataSize` = 8152
  ([ginblock.h#GinDataPageMaxDataSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L319-L326)),
  so the denominator overstates by 8 bytes per data page, at most 0.098%.

The two multipliers are deliberately different. `approx_free_space` and
`total_space` count **usable capacity** at 8160 so a whole free page is
commensurable with the intra-page free bytes added to it, which is what
`pgstathashindex` does when it multiplies unused pages by `hashm_bsize` rather
than `BLCKSZ`
([pgstatindex.c#space_per_page](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L618-L624)).
`pending_space` and `approx_recyclable_space` count **file bytes** at `BLCKSZ`,
because an operator comparing them against `pg_relation_size` wants whole
blocks.

### Prototype for the new fields

The census helper gains `pd_prune_xid` and a recyclability verdict.
`page_header` already exposes `prune_xid`
([pageinspect--1.5.sql#page_header](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L19-L33),
[rawpage.c#page_header](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L216-L278)),
and `age()` returns `INT_MAX` for a non-normal XID
([xid.c#xid_age](../../../../raw/postgres-12/src/backend/utils/adt/xid.c#L99-L113)),
which reproduces `TransactionIdPrecedes`'s treatment of a zero delete XID.

```sql
CREATE /* wiki_pgstatginindex_waste_census */ FUNCTION gin_waste_census(idx regclass)
RETURNS TABLE (blkno bigint, page_class text, live boolean,
               free_space int, capacity int,
               prune_xid xid, recyclable boolean, entry_tuples int)
LANGUAGE sql AS $$
WITH n AS (
  SELECT (pg_relation_size(idx) / current_setting('block_size')::bigint) AS nblocks,
         age((txid_snapshot_xmin(txid_current_snapshot()) % 4294967296)::text::xid)
           AS horizon_age
), b AS (
  SELECT g.i AS blkno, get_raw_page(idx::text, g.i::int) AS pg
  FROM n, generate_series(1, n.nblocks - 1) AS g(i)
), c AS (
  SELECT b.blkno, o.flags, h.lower::int AS lower_off, h.upper::int AS upper_off,
         h.special::int AS special_off, h.prune_xid, n.horizon_age
  FROM b, n,
       LATERAL gin_page_opaque_info(b.pg) AS o,
       LATERAL page_header(b.pg) AS h
)
SELECT c.blkno,
       CASE
         WHEN c.upper_off = 0                    THEN 'new'
         WHEN 'deleted' = ANY (c.flags)          THEN 'deleted'
         WHEN 'list'    = ANY (c.flags)          THEN 'pending'
         WHEN 'data'    = ANY (c.flags)
              AND 'leaf' = ANY (c.flags)         THEN 'data_leaf'
         WHEN 'data'    = ANY (c.flags)          THEN 'data_internal'
         WHEN 'leaf'    = ANY (c.flags)          THEN 'entry_leaf'
         ELSE                                        'entry_internal'
       END,
       (c.upper_off <> 0
        AND NOT 'deleted' = ANY (c.flags)
        AND NOT 'list'    = ANY (c.flags)),
       (c.upper_off - c.lower_off),
       CASE WHEN 'data' = ANY (c.flags) THEN c.special_off - 32
            ELSE c.special_off - 24 END,
       c.prune_xid,
       (c.upper_off = 0
        OR ('deleted' = ANY (c.flags) AND age(c.prune_xid) > c.horizon_age)),
       CASE WHEN c.upper_off <> 0
                 AND NOT 'data' = ANY (c.flags)
                 AND NOT 'list' = ANY (c.flags)
                 AND NOT 'deleted' = ANY (c.flags)
            THEN (c.lower_off - 24) / 4 ELSE NULL END
FROM c;
$$;
```

Three prototype-only caveats. The SQL horizon is
`txid_snapshot_xmin(txid_current_snapshot())` rather than `RecentGlobalXmin`,
which is a proxy, valid here only because the test server ran with
`autovacuum = off` and one session. `entry_tuples` is derived as
`(pd_lower - 24) / 4`, the line-pointer count, because v12 `pageinspect` has no
entry-page item decoder; the opaque `maxoff` field is not a tuple count on an
entry page
([ginblock.h#maxoff-meaning](../../../../raw/postgres-12/src/include/access/ginblock.h#L29-L36)).
The independent check on recyclability came from `pg_freespace`, which needs the
separate `pg_freespacemap` extension
([pg_freespacemap--1.1.sql#pg_freespace](../../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L20));
`RecordFreeIndexPage` records `BLCKSZ - 1`
([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55)),
which reads back as `avail = 8160`. It matched the census exactly: 465 blocks
for `f_mixed_gin`, 168 for `f_bloat_gin`, 40 for `f_waste_gin`, 0 for the four
healthy indexes and for `f_keys_gin`, whose VACUUM freed intra-page space
without deleting a single page.

### How the new fields behave under sampling

Nine fixtures at `sample_fraction = 1.0` reproduced the exact census on every
new field, including `approx_free_space`, `approx_free_percent` and
`approx_recyclable_pages`, as `BlockSampler`'s contract predicts
([sampling.c#Algorithm-S-contract](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L35)).

Then 1,800 seeded runs, 100 seeds per fixture per requested fraction, floor 50,
post-stratified:

| Index | ordinary | truth `free_percent` | pages read at 1% | mean | stddev | max abs err | `free_space` MAPE |
|---|---|---|---|---|---|---|---|
| `f_big_gin` | 13672 | 49.56 | 137 | 49.56 | 0.04 | 0.25 | 0.02% |
| `f_mixed_gin` | 9324 | 64.39 | 94 | 64.59 | 0.90 | 2.52 | 1.15% |
| `f_pending_gin` | 793 | 49.57 | 50 | 49.63 | 0.45 | 1.65 | 0.35% |
| `f_small_gin` | 411 | 49.66 | 50 | 49.66 | 0.33 | 0.91 | 0.43% |
| `f_bloat_gin` | 409 | 93.45 | 50 | 93.45 | 0.88 | 2.55 | 0.78% |
| `f_keys_gin` | 308 | 71.74 | 50 | 71.63 | 3.53 | 9.72 | 3.98% |
| `f_partial_gin` | 164 | 49.54 | 50 | 49.54 | 0.16 | 0.25 | 0.25% |
| `f_waste_gin` | 129 | 88.60 | 50 | 88.82 | 1.58 | 5.22 | 1.42% |
| `f_empty_gin` | 1 | 100.00 | 1 | 100.00 | 0.00 | 0.00 | 0.00% |

A ratio over a dominant class is cheap and accurate: 137 of 13,672 pages fixed
`free_percent` to within 0.25 points across 100 seeds. The worst case, 9.72
points on a 308-page index at a forced 16% coverage, is a small-index
finite-sample effect, not a large-index risk.

The page-count estimates are noisier than the ratio, exactly as the main
comparison found for rare classes:

| Index | truth deleted | mean | min | max | missed entirely | `recyclable` disagreed with `deleted` |
|---|---|---|---|---|---|---|
| `f_bloat_gin` (1%) | 168 | 169.0 | 98 | 213 | 0/100 | 0/100 |
| `f_mixed_gin` (1%) | 465 | 511.9 | 99 | 992 | 0/100 | 0/100 |
| `f_mixed_gin` (5%) | 465 | 463.8 | 260 | 699 | 0/100 | 0/100 |
| `f_waste_gin` (1%) | 40 | 40.2 | 21 | 62 | 0/100 | 0/100 |

The recyclable split cost nothing in accuracy here, because every deleted page
in every fixture was recyclable by measurement time. A fixture that mixes fresh
and aged delete XIDs was not built; see
[Open Questions](#open-questions).

The classifier result is the strongest one. Over 800 runs at a requested 1%,
excluding the empty index:

| | value |
|---|---|
| highest `free_percent` drawn on any healthy index | **51.22** |
| lowest `free_percent` drawn on any bloated index | **62.02** |
| runs where a 60% threshold classified correctly | **800 / 800** |

`f_empty_gin` is the one false positive: a two-block index reports 100% free in
every run, so any threshold needs a minimum-size guard.

### Turning free_percent into a rebuild estimate

Raw free space is not the bytes a rebuild returns, because the rebuilt index
keeps GIN's structural free space. Measured against sibling indexes built over
identical live rows:

| Index | current bytes | fresh bytes | actually recovered | `free_space` reported | naive accuracy | baseline-corrected prediction | corrected accuracy |
|---|---|---|---|---|---|---|---|
| `f_small_gin` | 3375104 | 3375104 | **0** | 1665560 | **0.0%** | **0** | exact |
| `f_bloat_gin` | 3358720 | 311296 | 3047424 | 3118968 | 97.7% | 3001344 | **98.5%** |
| `f_keys_gin` | 2531328 | 1269760 | 1261568 | 1803140 | **70.0%** | 1213560 | **96.2%** |
| `f_waste_gin` | 1064960 | 212992 | 851968 | 932624 | 91.4% | 838096 | **98.4%** |

The correction is `reported_free_space - baseline_free_space`, where the
baseline is the free space a fresh build of the same data carries. The naive
reading is worst exactly where the entry tree dominates, because that is where
the ~50% baseline is largest: 70.0% on the entry-tree-only `f_keys_gin`, 97.7%
on the posting-tree-dominated `f_bloat_gin`, and meaningless on the healthy
index. The corrected form landed within 1.5-3.8% on all three bloated
fixtures and was exactly right on the healthy one.

The baseline is not free to obtain -- it comes from a build -- so the practical
form is a per-class density comparison, not a byte subtraction. Fresh-build
baselines measured here were 50.4-53.2% for entry leaves and 83.5-93.8% for
posting-tree leaves, but only where the class held more than one page. On a
one-page entry tree the number is meaningless: fresh `f_bloat_gin2` measured
2.94% and fresh `f_waste_gin2` 1.96% on their single entry leaf, so the bloated
`f_bloat_gin`'s 5.88% is *higher* than its own rebuild's.

### A bloat verdict for GIN

Read the output in this order. Every rule below rests on a measurement above.

1. **`pending_space` first.** It is exact, and 30.98% of one fixture's bytes
   were pending pages that no density ratio revealed.
2. **`approx_recyclable_space` next.** Whole pages, already reusable, that no
   v12 view reports; the metapage hides them entirely
   ([ginvacuum.c#classification-loop](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L746-L777)).
3. **`approx_data_leaf_density` against ~85-95%, not against 100%.** Fresh
   builds measured 83.54% and 93.81%; bloated ones 20.15% and 12.15%.
4. **`approx_entry_leaf_density` against ~50%, not against 100%,** and only when
   `approx_entry_leaf_pages > 1`. Four healthy multi-leaf indexes measured
   50.44-50.52% and a rebuild of one recovered zero bytes, while on a one-page
   entry tree a rebuild moved the number the wrong way (5.88% bloated versus
   2.94% fresh).
5. **`approx_free_percent` above 60% as the single-number alarm**, with a
   minimum-size guard. It separated bloated from healthy in 800 of 800 runs.
6. **Never conclude "no bloat" from a healthy reading**, because a pre-VACUUM
   index with 90% of its entries dead reported numbers identical to its own
   freshly built state.

The corresponding session settings are unchanged from the prototype's:
`statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so they apply to
the session only and need neither reload nor restart
([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386),
[guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)).

### What the follow-up measurements establish

1. Free space is not wasted space in GIN: four healthy, never-deleted indexes
   sat at 49.5-49.7% free, and one of them was byte-identical to a fresh build
   of the same rows.
2. The function can classify bloat reliably at a 1% sample -- 800 of 800 runs
   -- once the threshold sits above the structural baseline.
3. It cannot see dead entries before VACUUM at all: deleting 90% of the rows
   changed no output field by even one byte.
4. Entry tuples for keys that lost every TID survive VACUUM and are reported as
   used space, not as waste: 4,000 tuples retained where a fresh build has
   2,000.
5. The two new exact fields, `pending_space` and `total_space`, and the four new
   estimates reproduce the exact census at full sample and stay within 0.25
   points of truth at 1% on a 13,672-page index.
6. Priced against real rebuilds, only 97.7%, 91.4% and 70.0% of the reported
   free space materialized as recovered bytes on three bloated fixtures, and
   0.0% on a healthy one; the baseline-corrected form predicted 96.2-98.5% of
   the recovered bytes and was exact on the healthy index.

New test objects created for this follow-up (the `f_waste` and `f_keys` tables
and the four sibling indexes `f_waste_gin2`, `f_keys_gin2`, `f_bloat_gin2` and
`f_small_gin2`) were dropped afterwards, so the fixture set described in
[Executed Comparison With the Exact
Census](#executed-comparison-with-the-exact-census) is unchanged. The temporary
server was stopped and `raw/postgres-12/` was not modified.

## Context Reviewed

- GIN on-disk layout: `ginblock.h` in full, plus `gin_private.h`, `gin.h`.
- GIN implementation: `ginutil.c`, `ginvacuum.c`, `ginfast.c`, `gindatapage.c`,
  `ginentrypage.c`, `ginbtree.c`, `gininsert.c`, `ginxlog.c`, and
  `src/backend/access/gin/README`.
- `contrib/pgstattuple`: `pgstatindex.c`, `pgstattuple.c`, `pgstatapprox.c`,
  every SQL script, `pgstattuple.control`, `Makefile`, `sql/pgstattuple.sql`,
  `expected/pgstattuple.out`.
- `contrib/pageinspect`: `ginfuncs.c`, `rawpage.c`, `pageinspect--1.5.sql`,
  `pageinspect.control`, `Makefile`, `sql/gin.sql`, `expected/gin.out`.
- Sampling: `utils/sampling.h`, `sampling.c`, and the sole call site in
  `analyze.c`; `amapi.h` checked for an absent index-sample callback.
- Buffers and pages: `bufpage.c`, `bufpage.h`, `freelist.c`, `block.h`,
  `indexfsm.c`.
- Planner consumption: `gincostestimate` in `selfuncs.c`.
- Settings: `guc.c` for `gin_pending_list_limit`, `statement_timeout`,
  `lock_timeout`; `reloptions.c` for `fastupdate` and the per-index limit.
- Docs: `pgstattuple.sgml`, `pageinspect.sgml`, `config.sgml`, `gin.sgml`.
- Catalog/build: `pg_am.dat`, `genbki.pl`, `extension.c`, `fmgr.h`.
- Related wiki pages: the v12 B-tree sampling proposal and the v12 GIN bloat
  page, used for scope comparison only, not as factual support.

Added for the bloat / wasted-space follow-up:

- Page-header storage of the GIN delete XID: `bufpage.h` `PageHeaderData` and
  the `pd_prune_xid` comment, `storage.sgml`'s page-layout table.
- XID comparison: `transam.c` `TransactionIdPrecedes`, `xid.c` `xid_age`,
  `snapmgr.h` for `RecentGlobalXmin`'s linkage.
- Fill policy: `ginutil.c` `ginoptions` for the absent `fillfactor`,
  `ginentrypage.c` `entrySplitPage`, `gindatapage.c` build-versus-split
  packing.
- VACUUM accounting: `ginvacuum.c` `ginVacuumItemPointers` /
  `ginVacuumEntryPage` / `pages_deleted` / `pages_free`, `ginfast.c`
  `shiftList` FSM recording, `vacuumlazy.c`'s index cleanup report.
- Free-space precedent: `pgstatindex.c` `GetHashPageStats` and
  `pgstathashindex`'s `free_percent`/`dead_items`, `pgstatapprox.c`
  `GetOldestXmin`, `pgstattuple.sgml`'s `free_space` and `approx_free_space`
  column definitions.
- FSM cross-check: `contrib/pg_freespacemap` SQL and C, `indexfsm.c`.
- Horizon precedent outside this module: `contrib/amcheck` `verify_nbtree.c`.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstatginindex` reads only block 0 and returns 3 columns | [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573) |
| `pgstattuple()` rejects GIN | [pgstattuple.c#GIN-reject](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276) |
| GIN page flags and classification macros | [ginblock.h#flag-bits](../../../../raw/postgres-12/src/include/access/ginblock.h#L40-L48), [ginblock.h#page-macros](../../../../raw/postgres-12/src/include/access/ginblock.h#L107-L138) |
| `GIN_DELETED` is OR'ed onto existing flags | [ginvacuum.c#ginDeletePage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L187-L192) |
| Deleted ex-pending page flags overwritten to `GIN_DELETED` | [ginfast.c#shiftList-flags](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632) |
| VACUUM's three-way classification and metapage write | [ginvacuum.c#classification-loop](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L746-L781) |
| `nPendingPages` trustworthy, other counters as of last VACUUM | [ginutil.c#ginGetStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657) |
| `ginUpdateStats` does not write pending counters | [ginutil.c#ginUpdateStats](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L659-L716) |
| Planner scales metapage stats, abandons past 4X growth | [selfuncs.c#4X-scaling](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6732-L6770) |
| XID-gated page reuse | [ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [ginutil.c#GinNewBuffer](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335) |
| VACUUM never deletes entry-tree pages | [README#page-deletion](../../../../raw/postgres-12/src/backend/access/gin/README#L389-L396) |
| Leftmost/rightmost posting pages never deleted | [ginvacuum.c#ginScanToDelete](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L302-L317) |
| Per-class capacity constants | [ginblock.h#GinDataPageMaxDataSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L319-L326), [ginblock.h#GinListPageSize](../../../../raw/postgres-12/src/include/access/ginblock.h#L328-L332) |
| `PageGetExactFreeSpace` is `pd_upper - pd_lower` | [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L626-L647) |
| `maxoff` invalid on compressed data leaves | [gindatapage.c#compressed-maxoff](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L989-L996) |
| Pre-9.4 data leaves have untrustworthy `pd_lower` | [ginblock.h#pre-9.4-caveat](../../../../raw/postgres-12/src/include/access/ginblock.h#L305-L312) |
| `pgstathashindex` is the AM-specific full-scan model | [pgstatindex.c#pgstathashindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L581-L727) |
| `BAS_BULKREAD` ring size | [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L536-L588) |
| `BlockSampler` contract, ascending order, full-selection case | [sampling.c#BlockSampler_Init](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L23-L51), [sampling.c#BlockSampler_Next](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L59-L112) |
| `int`/`BlockNumber` width mismatch | [sampling.h#BlockSamplerData](../../../../raw/postgres-12/src/include/utils/sampling.h#L28-L36) |
| Only `analyze.c` uses `BlockSampler`; no index-AM sample callback | [analyze.c#acquire_sample_rows](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1009-L1049), [amapi.h#IndexAmRoutine](../../../../raw/postgres-12/src/include/access/amapi.h#L163-L216) |
| `pgstatapprox` naming and its integer-division wart | [pgstatapprox.c#output_type](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52), [pgstatapprox.c#integer-division](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L201-L207) |
| `pgstathashindex` revoke/grant template | [pgstattuple--1.4--1.5.sql#pgstathashindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L121-L136) |
| v1.5 relies on `REVOKE`, not `superuser()` | [pgstatindex.c#v1.5-comment](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L168) |
| Extension version wiring and install-then-update | [pgstattuple.control#default-version](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5), [extension.c#install-then-update](../../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550) |
| `GIN_AM_OID` comes from a generated header | [pg_am.dat#GIN_AM_OID](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29), [genbki.pl#OID-symbols](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603) |
| GIN regression coverage is one empty index | [pgstattuple.sql#preamble](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L3-L7), [pgstattuple.out#GIN-result](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L126-L131) |
| `pgstatginindex` has no snapshot caveat; `pgstatindex` does | [pgstattuple.sgml#pgstatginindex](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L298-L356), [pgstattuple.sgml#snapshot-caveat](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L275-L279) |
| `pageinspect` is superuser-only and re-opens per call | [pageinspect.sgml#superuser](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L14), [rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L94-L172) |
| `gin_pending_list_limit` is `PGC_USERSET`; reloption overrides it | [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [gin_private.h#GinGetPendingListCleanupSize](../../../../raw/postgres-12/src/include/access/gin_private.h#L24-L39) |
| Pending cleanup can fire from an ordinary insert | [ginfast.c#cleanup-trigger](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L437-L461) |
| Prototype timeouts are session-scoped | [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397) |

Measured claims (seven fixtures, 6,100 seeded runs, exact-pin 12.2 server) are
reported in [Executed Comparison With the Exact
Census](#executed-comparison-with-the-exact-census) and are not source claims.

Added for the bloat / wasted-space follow-up:

| Claim | Evidence |
|---|---|
| GIN has no `fillfactor`; only two reloptions exist | [ginutil.c#ginoptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L629) |
| Entry pages split by equal data size, with no sorted-insert case | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L623-L689) |
| Data leaves are packed tight only during a build | [gindatapage.c#build-versus-split-packing](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667) |
| The GIN delete XID lives in the page header's `pd_prune_xid` | [ginblock.h#GinPageGetDeleteXid](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164) |
| A zero delete XID compares as older than any normal XID | [transam.c#TransactionIdPrecedes](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L313) |
| `age()` reports a non-normal XID as `INT_MAX` old | [xid.c#xid_age](../../../../raw/postgres-12/src/backend/utils/adt/xid.c#L99-L113) |
| Drained pending pages are put in the FSM immediately | [ginfast.c#shiftList-fsm](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L664-L665) |
| Dead TIDs leave a posting list only when VACUUM rewrites it | [ginvacuum.c#ginVacuumItemPointers](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L47-L84) |
| An entry tuple with zero remaining TIDs is rebuilt, not deleted | [ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556), [ginentrypage.c#GinFormTuple](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L44-L95) |
| `GinFormTuple` can leave MAXALIGN pad space inside a tuple | [ginentrypage.c#GinFormTuple-padding](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L71-L80) |
| `pgstathashindex` counts whole unused pages as free space | [pgstatindex.c#unused-as-free](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L689-L690) |
| Its `free_percent` denominator excludes bookkeeping pages | [pgstatindex.c#total_space](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L692-L702) |
| Hash can count `dead_items` from line pointers; GIN has none | [pgstatindex.c#dead_items](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L742-L750) |
| `free_space` / `approx_free_space` naming precedent | [pgstattuple.sgml#free_space](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L108-L117), [pgstattuple.sgml#approx_free_space](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L593-L602) |
| A vacuum horizon is already computed inside this module | [pgstatapprox.c#GetOldestXmin](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L72-L75) |
| `RecentGlobalXmin` is reachable from a contrib module | [snapmgr.h#RecentGlobalXmin](../../../../raw/postgres-12/src/include/utils/snapmgr.h#L59-L62), [verify_nbtree.c#RecentGlobalXmin](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L396-L400) |
| VACUUM's two index numbers are `pages_deleted` and `pages_free` | [ginvacuum.c:234](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L234), [ginvacuum.c:786](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L786), [ginfast.c:588](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L588), [vacuumlazy.c#index-cleanup-report](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827) |
| `page_header` exposes `prune_xid` to SQL | [pageinspect--1.5.sql#page_header](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L19-L33), [rawpage.c#page_header](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L216-L278) |
| A free index page reads back from the FSM as 8160 | [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55), [pg_freespacemap--1.1.sql#pg_freespace](../../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L20) |

Follow-up measured claims (nine fixtures, 1,800 further seeded runs, four
sibling-index rebuilds, same exact-pin 12.2 server) are reported in
[Follow-Up: Bloat and Wasted Space](#follow-up-bloat-and-wasted-space) and are
not source claims.

## Open Questions

- **No `new` page was produced by any fixture.** The `approx_new_pages` class
  and the `pd_upper == 0` branch are therefore untested here. `pgstathashindex`
  counts `PageIsNew` pages as unused
  ([pgstatindex.c:645-646](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L645-L646)),
  but whether a GIN index can present one to a reader between extension and
  initialization was not established.
- **No pre-9.4 uncompressed data leaf was tested.** The pinned build creates
  `GIN_CURRENT_VERSION = 2` indexes
  ([ginblock.h#GIN_CURRENT_VERSION](../../../../raw/postgres-12/src/include/access/ginblock.h#L102)),
  so the legacy `pd_lower` path is a source-only claim.
- **`target_data_pages = 20` is not derived from anything.** It reproduced
  6/100 misses on a 12-page class; no principled value was established, and
  the right default may depend on which class the operator cares about.
- **The derived floor's behavior when `nDataPages` is badly stale is
  untested.** `f_pending_gin` showed `nTotalPages` can lag by 31%, but no
  fixture combined a stale `nDataPages` with a grown data tree.
- **Whether the C function should reuse `pgstatginindex_internal` or duplicate
  it is unresolved.** Refactoring changes a function that eight regression
  lines depend on
  ([pgstattuple.out#GIN-errors](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L140-L152)).
- **Seeding is unresolved for reproducible regression output.** Nothing in v12
  exposes a `BlockSampler` seed, and `sampler_random_init_state` keeps only 32
  seed bits with a fixed first state word
  ([sampling.c#sampler_random_init_state](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c#L228-L234)).
  The prototype used `setseed()`, which the C function cannot rely on.
- **`PARALLEL RESTRICTED` versus `PARALLEL SAFE` was not settled.** The
  existing GIN function is `PARALLEL SAFE`
  ([pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L47-L57));
  whether a sampler can be too depends on per-worker seeding not analyzed here.
- **The `f_mixed_gin` 14.06-point data-density error implies at least one run
  sampled only an empty data leaf**, but which page and why was not traced.
- **Pending-list contiguity was observed but not explained.** Both fixtures
  gave `pending_tail - pending_head + 1 = nPendingPages`; source permits
  non-contiguity via FSM reuse
  ([ginutil.c#GinNewBuffer](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)),
  so no design may assume it.
- **Whether upstream ever added GIN support to `pgstattuple` or an index
  sampler after v12 was not investigated**, since cross-version evidence is out
  of scope for this page.

From the bloat / wasted-space follow-up:

- **`bufpage.h` says `pd_prune_xid` "is currently unused in index pages"**
  ([bufpage.h#pd_prune_xid-comment](../../../../raw/postgres-12/src/include/storage/bufpage.h#L132-L136)),
  and `storage.sgml` describes it as the "Oldest unpruned XMAX on page"
  ([storage.sgml#pd_prune_xid](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L872-L877)),
  yet GIN stores a deleted page's delete XID there
  ([ginblock.h#GinPageGetDeleteXid](../../../../raw/postgres-12/src/include/access/ginblock.h#L135-L136)).
  Source wins over both comments, but whether any other index AM relies on the
  field being untouched was not established, so a new reader of the field
  should not assume the comment.
- **No fixture mixed fresh and aged delete XIDs.** Every deleted page in every
  fixture was recyclable by measurement time, so
  `approx_recyclable_pages` never disagreed with `approx_deleted_pages` in
  1,800 runs. The split's accuracy when the two differ is untested.
- **The prototype's horizon is a proxy.** It uses
  `txid_snapshot_xmin(txid_current_snapshot())` rather than
  `RecentGlobalXmin`, which coincided only because the test server ran
  `autovacuum = off` with one session. A C implementation should use
  `GetOldestXmin` as `pgstatapprox.c` does
  ([pgstatapprox.c#GetOldestXmin](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L72-L75)),
  and the two horizons' divergence under concurrency was not measured.
- **The 60% alarm threshold is fitted, not derived.** It sits inside a measured
  gap between 51.22 and 62.02 on nine fixtures. Nothing establishes that gap on
  other opclasses, on multicolumn GIN indexes, or on `jsonb`/`tsvector` data,
  where key-size distribution differs and the entry-tree baseline may not be
  near 50%.
- **The baseline correction needs a baseline.** Turning
  `approx_free_percent` into recoverable bytes required a fresh sibling index.
  No way to derive the per-class baseline density from the index itself was
  established, and the measured entry-leaf baselines already spanned
  50.4-53.2%.
- **`f_empty_gin` is a false positive at any threshold.** A two-block index
  reports 100% free space. The minimum-size guard was not calibrated.
- **Entry-leaf density is uninterpretable on a one-page entry tree**, and the
  proposed output gives no direct warning. Fresh `f_bloat_gin2` measured 2.94%
  against the bloated `f_bloat_gin`'s 5.88%, so the metric moved the wrong way.
  An `approx_entry_leaf_pages > 1` precondition is proposed but was not turned
  into an output flag.
- **Whether `total_space` should use per-class capacities** rather than a flat
  8160, accepting a 0.098% overstatement per data page, was decided for
  simplicity and not tested against an index whose blocks are almost all data
  pages.
- **The pending list's contribution to a bloat verdict is unresolved.** It is
  reported exactly and excluded from `total_space`, but a large pending list is
  itself a cost, and this page's own earlier measurement showed flushing one
  can grow the index. No combined score was proposed.

## Source References

- [ginblock.h](../../../../raw/postgres-12/src/include/access/ginblock.h)
- [gin.h](../../../../raw/postgres-12/src/include/access/gin.h)
- [gin_private.h](../../../../raw/postgres-12/src/include/access/gin_private.h)
- [ginutil.c](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c)
- [ginvacuum.c](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c)
- [ginfast.c](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c)
- [gindatapage.c](../../../../raw/postgres-12/src/backend/access/gin/gindatapage.c)
- [ginentrypage.c](../../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c)
- [ginbtree.c](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c)
- [gininsert.c](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c)
- [gin/README](../../../../raw/postgres-12/src/backend/access/gin/README)
- [pgstatindex.c](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c)
- [pgstattuple.c](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c)
- [pgstatapprox.c](../../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c)
- [pgstattuple--1.4.sql](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql)
- [pgstattuple--1.4--1.5.sql](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql)
- [pgstattuple--1.0--1.1.sql](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.0--1.1.sql)
- [pgstattuple.control](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control)
- [pgstattuple/Makefile](../../../../raw/postgres-12/contrib/pgstattuple/Makefile)
- [pgstattuple.sql](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql)
- [pgstattuple.out](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out)
- [ginfuncs.c](../../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c)
- [rawpage.c](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c)
- [pageinspect--1.5.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql)
- [pageinspect.control](../../../../raw/postgres-12/contrib/pageinspect/pageinspect.control)
- [pageinspect/Makefile](../../../../raw/postgres-12/contrib/pageinspect/Makefile)
- [gin.sql](../../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql)
- [gin.out](../../../../raw/postgres-12/contrib/pageinspect/expected/gin.out)
- [sampling.h](../../../../raw/postgres-12/src/include/utils/sampling.h)
- [sampling.c](../../../../raw/postgres-12/src/backend/utils/misc/sampling.c)
- [analyze.c](../../../../raw/postgres-12/src/backend/commands/analyze.c)
- [amapi.h](../../../../raw/postgres-12/src/include/access/amapi.h)
- [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c)
- [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h)
- [freelist.c](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c)
- [block.h](../../../../raw/postgres-12/src/include/storage/block.h)
- [indexfsm.c](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c)
- [selfuncs.c](../../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c)
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c)
- [reloptions.c](../../../../raw/postgres-12/src/backend/access/common/reloptions.c)
- [rel.h](../../../../raw/postgres-12/src/include/utils/rel.h)
- [extension.c](../../../../raw/postgres-12/src/backend/commands/extension.c)
- [fmgr.h](../../../../raw/postgres-12/src/include/fmgr.h)
- [pg_am.dat](../../../../raw/postgres-12/src/include/catalog/pg_am.dat)
- [genbki.pl](../../../../raw/postgres-12/src/backend/catalog/genbki.pl)
- [transam.c](../../../../raw/postgres-12/src/backend/access/transam/transam.c)
- [xid.c](../../../../raw/postgres-12/src/backend/utils/adt/xid.c)
- [snapmgr.h](../../../../raw/postgres-12/src/include/utils/snapmgr.h)
- [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c)
- [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c)
- [pg_freespacemap--1.1.sql](../../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap--1.1.sql)
- [pgstattuple.sgml](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml)
- [pageinspect.sgml](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml)
- [config.sgml](../../../../raw/postgres-12/doc/src/sgml/config.sgml)
- [gin.sgml](../../../../raw/postgres-12/doc/src/sgml/gin.sgml)
- [storage.sgml](../../../../raw/postgres-12/doc/src/sgml/storage.sgml)

## Navigation

- [v12/index](../../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](pgstatindex-sample-variant-proposal.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
