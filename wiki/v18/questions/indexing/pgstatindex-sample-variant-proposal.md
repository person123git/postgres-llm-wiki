---
type: question
version: 18
pinned_commit: 6cb307251c5c6261286c1566496920976640108e
verified: false
verified_by_agent: not yet
---

# Proposing a Sampling pgstatindex Variant for PostgreSQL 18 (unverified)

## Question

In PostgreSQL 18, propose a `pgstatindex` variant that samples the index
instead of reading the whole index, and include a pros and cons section.

## Answer Up Front

Add a new, separately named function, `pgstatindex_approx`, instead of changing
`pgstatindex`. The design is straightforward because today's `pgstatindex`
already does a **physical block scan**, not a logical tree walk: it reads every
block from 1 to `nblocks - 1`, share-locks each, classifies it, and accumulates
leaf free space and fragment counts
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L275-L328)).
A sampler keeps that exact per-page logic but visits only a random subset of the
same block numbers.

Three things make this clean in v18:

- **Some columns cost nothing and stay exact.** `version`, `tree_level`, and
  `root_block_no` come from the single metapage read, and `index_size` is just
  `RelationGetNumberOfBlocks` times `BLCKSZ`. None of these need the page scan
  ([pgstatindex.c#metapage-read](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L252-L262),
  [pgstatindex.c#index_size](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L354)).
- **The two interesting metrics are ratios, so a sample estimates them
  directly.** `avg_leaf_density` is a ratio of two sums over leaf pages, and
  `leaf_fragmentation` is a per-leaf rate; both predicates are local to a single
  page, so they need no neighbor reads and no scaling
  ([pgstatindex.c#leaf-accumulation](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L320),
  [pgstatindex.c#result](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L360-L369)).
- **Only the page counts must be scaled up.** `internal_pages`, `leaf_pages`,
  `empty_pages`, and `deleted_pages` are extensive quantities, so the sampler
  multiplies sampled counts by `1 / sampling_fraction`.

This is different from the sibling question, [Why pgstatindex Cannot Use
pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](pgstatindex-approx-sampling.md):
that page is about reusing the heap visibility-map/free-space-map shortcut, which
does not transfer to B-trees. Pure **block sampling** sidesteps that entirely —
it does not touch the index FSM at all.

## What pgstatindex Reads Today

`pgstatindex_impl` is the baseline to preserve. It:

1. Rejects non-B-tree relations, other-session temp relations, and invalid
   indexes up front
   ([pgstatindex.c#guards](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L221-L247)).
2. Reads block 0 (the metapage, `BTREE_METAPAGE`) once and copies
   `btm_version`, `btm_level`, and `btm_root`
   ([pgstatindex.c#metapage-read](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L252-L262),
   [nbtree.h#BTMetaPageData](../../../../raw/postgres-18/src/include/access/nbtree.h#L104-L123),
   [nbtree.h#BTREE_METAPAGE](../../../../raw/postgres-18/src/include/access/nbtree.h#L149)).
3. Walks `blkno = 1 .. nblocks - 1` in physical order, using a `BAS_BULKREAD`
   ring buffer, share-locks each page, and classifies it from its
   `btpo_flags`: `P_ISDELETED` -> deleted, `P_IGNORE` -> empty/half-dead,
   `P_ISLEAF` -> leaf, else internal
   ([pgstatindex.c#scan-loop](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L328),
   [nbtree.h#btpo_flags](../../../../raw/postgres-18/src/include/access/nbtree.h#L76-L85),
   [nbtree.h#page-macros](../../../../raw/postgres-18/src/include/access/nbtree.h#L213-L226)).
4. For each live leaf page, it adds the usable space `max_avail` and the exact
   free space `PageGetExactFreeSpace` (= `pd_upper - pd_lower`), and counts a
   "fragment" when the page's right link `btpo_next` points to an earlier
   physical block
   ([pgstatindex.c#leaf-accumulation](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L320),
   [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L956-L972),
   [nbtree.h#btpo_next](../../../../raw/postgres-18/src/include/access/nbtree.h#L63-L70)).
5. Computes `avg_leaf_density = 100 - free_space / max_avail * 100` and
   `leaf_fragmentation = fragments / leaf_pages * 100`, returning `NaN` when the
   relevant denominator is zero
   ([pgstatindex.c#result](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L360-L369)).

The output columns are `version`, `tree_level`, `index_size`,
`root_block_no`, `internal_pages`, `leaf_pages`, `empty_pages`,
`deleted_pages`, `avg_leaf_density`, `leaf_fragmentation`
([pgstattuple--1.4--1.5.sql#pgstatindex](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).

The docs already warn that the result is accumulated page-by-page and is not an
instantaneous snapshot, so sampling does not weaken a guarantee that exists
today
([pgstattuple.sgml#pgstatindex-note](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L275-L279)).

## Proposed Function: pgstatindex_approx

### Field plan

| Output field | Source in proposal | Exact or estimated |
|---|---|---|
| `version`, `tree_level`, `root_block_no` | metapage read (block 0) | exact |
| `index_size` | `RelationGetNumberOfBlocks(rel) * BLCKSZ` | exact |
| `scanned_percent` | realized sample fraction `f` | exact (describes the sample) |
| `approx_internal_pages` / `approx_leaf_pages` / `approx_empty_pages` / `approx_deleted_pages` | sampled count `/ f` | estimated (scaled count) |
| `approx_avg_leaf_density` | `100 - (sum free_space) / (sum max_avail)` over sampled leaves | estimated (ratio, no scaling) |
| `approx_leaf_fragmentation` | `fragments / leaf_pages * 100` over sampled leaves | estimated (rate, no scaling) |

The estimator split is the key point. **Counts are extensive** and must be
multiplied by `1 / f`. **Density and fragmentation are intensive ratios** whose
predicates live entirely on the page being read (`pd_upper - pd_lower`,
`max_avail`, and `btpo_next < blkno` are all on the sampled leaf itself), so the
sample's own ratio is a direct estimate of the whole-index ratio with no scaling
and no neighbor reads
([pgstatindex.c#leaf-accumulation](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L320)).

### Sampling strategy

Draw a uniform random sample of distinct block numbers from `[1, nblocks - 1]`,
either Bernoulli (each block included with a target probability) or a fixed
target count without replacement. Then visit the selected block numbers in
ascending order. Selection randomness gives the estimator its statistical
meaning; ordered visitation keeps the access pattern closer to today's physical
scan, which already walks `blkno` upward with a `BAS_BULKREAD` strategy
([pgstatindex.c#scan-loop](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L328)).
Let `f = sampled_blocks / (nblocks - 1)` be the realized fraction reported as
`scanned_percent`. This mirrors how `pgstattuple_approx` reports
`scanned_percent = 100 * scanned / nblocks`
([pgstatapprox.c#scanned_percent](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L191-L197)).
Per sampled block, reuse the existing classification and accumulation verbatim
and keep `CHECK_FOR_INTERRUPTS`
([pgstatindex.c#scan-loop](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L286-L327)).

No tree traversal is required: B-tree leaves are reached by reading physical
blocks, exactly as the current physical loop does, so sampling never needs
sibling or downlink chasing.

### Output shape, modeled on pgstattuple_approx

`pgstattuple_approx` is the precedent for an explicitly approximate result: it
reports `table_len`, `scanned_percent`, and `approx_`-prefixed estimates, so
callers can never confuse it with the exact function
([pgstatapprox.c#output_type](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L32-L44),
[pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119)).
The proposed function copies that convention with `scanned_percent` and `approx_`
prefixes for every estimated column.

### How it plugs into the extension

The `pgstatindex` code already shows the registration pattern to copy: a
`PG_FUNCTION_INFO_V1` entry, and a `_v1_5` variant that drops the `superuser()`
check because the SQL `REVOKE EXECUTE ... FROM PUBLIC` controls access instead
([pgstatindex.c#PG_FUNCTION_INFO_V1](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L52-L64),
[pgstatindex.c#v1_5-note](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L159-L177)).
Concretely:

- Add `pgstatindex_approx` (C) in `pgstatindex.c`, sharing the classification
  body with `pgstatindex_impl`.
- Add the SQL in a new `pgstattuple--1.5--1.6.sql` upgrade script that follows
  the existing `CREATE OR REPLACE FUNCTION ... LANGUAGE C STRICT PARALLEL SAFE`
  plus `REVOKE EXECUTE ... FROM PUBLIC` / `GRANT ... TO pg_stat_scan_tables`
  shape used for every other function
  ([pgstattuple--1.4--1.5.sql#pgstathashindex](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L123-L136)).
- Bump `default_version` in the control file from `1.5` to `1.6`
  ([pgstattuple.control#default_version](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L3)).

> The DDL below is a **proposed** signature for a function that does not exist in
> v18; it is not runnable against the pinned checkout. It is shown to fix the
> column contract, following the verified shape of the existing scripts above.

```sql
-- proposed: contrib/pgstattuple/pgstattuple--1.5--1.6.sql
CREATE OR REPLACE FUNCTION pgstatindex_approx(IN relname regclass,
    IN sample_fraction FLOAT8 DEFAULT 0.01,
    OUT version INT,
    OUT tree_level INT,
    OUT index_size BIGINT,            -- exact
    OUT root_block_no BIGINT,         -- exact
    OUT scanned_percent FLOAT8,       -- realized sample fraction
    OUT approx_internal_pages BIGINT, -- estimated
    OUT approx_leaf_pages BIGINT,     -- estimated
    OUT approx_empty_pages BIGINT,    -- estimated
    OUT approx_deleted_pages BIGINT,  -- estimated
    OUT approx_avg_leaf_density FLOAT8,
    OUT approx_leaf_fragmentation FLOAT8)
AS 'MODULE_PATHNAME', 'pgstatindex_approx'
LANGUAGE C STRICT PARALLEL SAFE;

REVOKE EXECUTE ON FUNCTION pgstatindex_approx(regclass, FLOAT8) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION pgstatindex_approx(regclass, FLOAT8) TO pg_stat_scan_tables;
```

## SQL Prototype Using pageinspect

A useful prototype can be written in SQL on top of `pageinspect`, without adding
new C code. This should be treated as a diagnostic sketch, not the preferred
extension implementation: `pageinspect` is documented as a low-level debugging
module whose functions may be used only by superusers, and its B-tree page
helpers also perform their own page reads and share locks
([pageinspect.sgml#overview](../../../../raw/postgres-18/doc/src/sgml/pageinspect.sgml#L1-L13),
[btreefuncs.c#bt_page_stats_internal](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L260-L304)).

The building blocks are available in contrib:

- `bt_metap(text)` returns the metapage fields, including `version`, `root`, and
  `level`
  ([pageinspect--1.8--1.9.sql#bt_metap](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.8--1.9.sql#L51-L63),
  [btreefuncs.c#bt_metap](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L838-L930)).
- `bt_page_stats(text, bigint)` returns one B-tree data page's `type`,
  `btpo_next`, and related counters; pageinspect maps deleted pages to `d`/`D`,
  half-dead ignored pages to `e`, live leaf pages to `l`, root pages above the
  leaf level to `r`, and other internal pages to `i`
  ([pageinspect--1.8--1.9.sql#bt_page_stats](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.8--1.9.sql#L65-L78),
  [btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L91-L191)).
- `get_raw_page(text, bigint)` plus `page_header(bytea)` can expose `lower`,
  `upper`, and `special` for the same sampled block
  ([pageinspect--1.8--1.9.sql#get_raw_page](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.8--1.9.sql#L33-L41),
  [pageinspect--1.9--1.10.sql#page_header](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.9--1.10.sql#L7-L19)).

The raw page header matters because `bt_page_stats.free_size` uses
`PageGetFreeSpace`, which subtracts space for one new line pointer, while
`pgstatindex_impl` uses `PageGetExactFreeSpace`, which returns `pd_upper -
pd_lower` without that subtraction
([btreefuncs.c#free_size](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L184-L191),
[bufpage.c#free-space-functions](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L900-L972)).
For a closer SQL match to `pgstatindex`, compute sampled leaf free space as
`upper - lower` from `page_header`, and compute sampled leaf capacity as
`special - 24`, because v18's `SizeOfPageHeaderData` is the offset of
`pd_linp` in `PageHeaderData`
([bufpage.h#PageHeaderData](../../../../raw/postgres-18/src/include/storage/bufpage.h#L163-L185),
[bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-18/src/include/storage/bufpage.h#L235-L238)).

Run a production diagnostic with session timeouts, for example:

```sql
SET /* wiki_pgstatindex_pageinspect_timeout */ statement_timeout = '5min';
SET /* wiki_pgstatindex_pageinspect_timeout */ lock_timeout = '2s';
CREATE /* wiki_pgstatindex_pageinspect_setup */ EXTENSION IF NOT EXISTS pageinspect;
```

This fixed-count sample query chooses random non-metapage block numbers, then
computes the same estimator split as the proposed C function. It avoids reading
all index pages, but it still enumerates all block numbers in SQL and reads each
sampled block through both `bt_page_stats` and `get_raw_page`:

```sql
WITH /* wiki_pgstatindex_pageinspect_sample */ params AS (
  SELECT 'my_index'::regclass AS idx, 0.01::float8 AS sample_fraction
),
bounds AS (
  SELECT idx,
       sample_fraction,
       pg_relation_size(idx) AS index_size,
       pg_relation_size(idx) / current_setting('block_size')::int AS nblocks
  FROM params
  WHERE sample_fraction > 0 AND sample_fraction <= 1
),
target AS (
  SELECT *,
       CASE
         WHEN nblocks <= 1 THEN 0::bigint
         ELSE least(nblocks - 1,
              greatest(1::bigint,
                   ceil((nblocks - 1) * sample_fraction)::bigint))
       END AS sample_target
  FROM bounds
),
candidate_blocks AS MATERIALIZED (
  SELECT t.idx, t.nblocks, t.index_size, t.sample_target, b.blkno
  FROM target AS t
  CROSS JOIN LATERAL generate_series(1::bigint, t.nblocks - 1) AS b(blkno)
  ORDER BY random()
  LIMIT (SELECT sample_target FROM target)
),
sample_blocks AS MATERIALIZED (
  SELECT *
  FROM candidate_blocks
  ORDER BY blkno
),
page_sample AS (
  SELECT s.blkno,
       ps.type,
       ps.btpo_next,
       h.lower,
       h.upper,
       h.special
  FROM sample_blocks AS s
  CROSS JOIN LATERAL bt_page_stats(s.idx::text, s.blkno) AS ps
  CROSS JOIN LATERAL page_header(get_raw_page(s.idx::text, s.blkno)) AS h
),
agg AS (
  SELECT count(*)::bigint AS sampled_blocks,
       count(*) FILTER (WHERE type IN ('i', 'r'))::bigint AS internal_sample,
       count(*) FILTER (WHERE type = 'l')::bigint AS leaf_sample,
       count(*) FILTER (WHERE type = 'e')::bigint AS empty_sample,
       count(*) FILTER (WHERE type IN ('d', 'D'))::bigint AS deleted_sample,
       coalesce(sum(greatest(special - 24, 0)) FILTER (WHERE type = 'l'), 0)::numeric AS leaf_max_avail,
       coalesce(sum(greatest(upper - lower, 0)) FILTER (WHERE type = 'l'), 0)::numeric AS leaf_free_space,
       count(*) FILTER (WHERE type = 'l' AND btpo_next <> 0 AND btpo_next < blkno)::bigint AS fragments_sample
  FROM page_sample
)
SELECT m.version,
     m.level AS tree_level,
     t.index_size,
     m.root AS root_block_no,
     CASE WHEN t.nblocks > 1
      THEN 100.0 * a.sampled_blocks / (t.nblocks - 1)
      ELSE 0.0
     END AS scanned_percent,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.internal_sample::numeric * (t.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_internal_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.leaf_sample::numeric * (t.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_leaf_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.empty_sample::numeric * (t.nblocks - 1) / a.sampled_blocks)::bigint
      ELSE 0
     END AS approx_empty_pages,
     CASE WHEN a.sampled_blocks > 0
      THEN round(a.deleted_sample::numeric * (t.nblocks - 1) / a.sampled_blocks)::bigint
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
FROM target AS t
CROSS JOIN LATERAL bt_metap(t.idx::text) AS m
CROSS JOIN agg AS a;
```

`pg_relation_size(regclass)` supplies the exact main-fork byte size used here to
derive `nblocks` and `index_size`
([pg_proc.dat#pg_relation_size](../../../../raw/postgres-18/src/include/catalog/pg_proc.dat#L7743-L7751),
[system_functions.sql#pg_relation_size](../../../../raw/postgres-18/src/backend/catalog/system_functions.sql#L285-L289)).
If the prototype needs lower SQL-call overhead and can tolerate sampling
contiguous ranges instead of individual blocks, `bt_multi_page_stats(text,
bigint, bigint)` exposes the same per-page fields for a range, with negative
`blk_count` meaning all remaining pages
([pageinspect--1.11--1.12.sql#bt_multi_page_stats](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.11--1.12.sql#L7-L24),
[pageinspect.sgml#bt_multi_page_stats](../../../../raw/postgres-18/doc/src/sgml/pageinspect.sgml#L318-L349)).

The limits are important. This SQL sketch cannot share a single C helper for
classification and accumulation, cannot expose a polished extension API, and
cannot guarantee the same physical read scheduling that a C loop can. It is best
used to validate estimator behavior and expected error before implementing
`pgstatindex_approx` inside `pgstattuple`.

## Pros

- **It actually reads fewer pages.** At fraction `f`, it reads about `f *
  nblocks` blocks plus the metapage, versus every non-metapage block today
  ([pgstatindex.c#scan-loop](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L328)).
  This is a stronger saving than `pgstattuple_approx`, which still iterates over
  every heap block and only avoids *reading* the all-visible ones
  ([pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L74-L94)).
- **Several columns stay exact for free.** `version`, `tree_level`, and
  `root_block_no` come from one metapage read; `index_size` comes from the
  relation size with no read at all
  ([pgstatindex.c#metapage-read](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L252-L262),
  [pgstatindex.c#index_size](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L354)).
- **Density and fragmentation need no auxiliary structure.** Because both are
  page-local ratios, a random sample estimates them directly. This avoids the
  FSM/visibility-map problem that blocks a `pgstattuple_approx`-style shortcut
  for B-trees (see [the sibling page](pgstatindex-approx-sampling.md)).
- **Random sampling makes linear scaling defensible.**
  `pgstattuple_approx` explicitly does *not* extrapolate linearly because it is
  *not* a random sample, falling back to `vac_estimate_reltuples`
  ([pgstatapprox.c#extrapolation-note](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L174-L186)).
  A deliberate uniform random sample of index blocks removes that objection: the
  `count / f` estimator is unbiased, and `scanned_percent` lets callers reason
  about its error.
- **Small, low-risk code.** It reuses the exact per-page classification and
  accumulation already in `pgstatindex_impl`, keeping `CHECK_FOR_INTERRUPTS` and
  the `indisvalid` / other-temp guards
  ([pgstatindex.c#guards](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L221-L247)).
- **Shorter, time-bounded lock footprint.** It share-locks far fewer buffers and
  finishes sooner, so it is less likely to overlap concurrent page splits than a
  full scan.

## Cons

- **Page counts become noisy, and the noisiest ones matter most.**
  `approx_deleted_pages` and `approx_empty_pages` (half-dead) are exactly the
  bloat signals users want, yet they are usually rare. A small sample can miss
  them entirely (reporting 0 when the true count is small but nonzero) or
  over-extrapolate from a single hit. These scaled-count columns are the most
  likely to mislead
  ([pgstatindex.c#classification](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L301-L323)).
- **Spatial clustering raises variance.** Leaf density varies across an index —
  freshly split pages sit near half full while packed regions approach full. If
  density or fragmentation is clustered in the physical file (for example a
  bulk-loaded region next to a churned region), a small sample can be biased.
  Systematic (every k-th block) sampling would be worse under any periodic
  layout.
- **Sparse sampling still weakens sequential readahead.** Even if the sampler
  sorts selected blocks before reading them, it skips over most blocks instead
  of reading one dense ascending run like the current `BAS_BULKREAD` scan
  ([pgstatindex.c#scan-loop](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L328)).
  The break-even fraction depends on storage: sparse reads are costlier on
  spinning disks than on SSDs.
- **Still not a consistent snapshot.** Like `pgstatindex` today, it share-locks
  one page at a time, so concurrent splits, deletions, and page recycling during
  sampling perturb the estimate; a sampled block may be concurrently recycled
  and must be classified as found, just as the full scan does
  ([pgstattuple.sgml#pgstatindex-note](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L275-L279)).
- **`leaf_fragmentation` degrades to a bare rate.** The exact column counts
  backward right links across the whole physical ordering; a sample yields the
  rate but loses any sense of contiguous runs, and the metric is already a coarse
  proxy
  ([pgstatindex.c#fragments](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L315-L320)).
- **New surface area and a tuning knob.** It needs a new C function, an extension
  version bump (1.5 -> 1.6), new SQL with `REVOKE`/`GRANT`, documentation,
  regression tests, and a documented error model. A badly chosen
  `sample_fraction` produces false precision. It must not silently change
  `pgstatindex`, whose documented contract is exact page-by-page accumulation
  ([pgstattuple.control#default_version](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L3),
  [pgstattuple.sgml#pgstatindex-note](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L275-L279)).

## Why This Differs From pgstattuple_approx

`pgstattuple_approx` is not a random sampler at all. It iterates every heap
block, and its only shortcut is to skip *reading* a page when the visibility map
proves it is all-visible, taking that page's free space from the FSM instead
([pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L74-L94),
[pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L515-L539)).
B-trees have no equivalent per-leaf free-space side channel, which is why that
shortcut cannot be ported — the full argument is in
[Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL
18 (unverified)](pgstatindex-approx-sampling.md). The proposal here avoids the
problem by being a true random *block* sampler: it reads fewer pages outright and
never consults the index FSM.

## Tests To Add

The current coverage lives in the `pgstattuple` regression test and only checks
empty-index output, wrong-AM and unsupported-relation errors, and one
partition-index success case; it never populates a B-tree to check non-`NaN`
density
([pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L18-L131),
[pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L44-L299)).
A sampling function would need new tests:

- A `sample_fraction` of 1.0 must reproduce `pgstatindex` exactly on the same
  index (sampling everything equals the full scan).
- `version`, `tree_level`, `root_block_no`, and `index_size` must match
  `pgstatindex` for any fraction, since they are exact.
- A populated B-tree must yield `approx_avg_leaf_density` within tolerance of the
  exact value, with `scanned_percent` reflecting the requested fraction.
- An empty index must return `NaN` density like `pgstatindex`
  ([pgstatindex.c#result](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L360-L369)).
- Wrong AM (GIN/hash) and unsupported relation kinds must error identically to
  `pgstatindex`
  ([pgstatindex.c#guards](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L221-L247)).
- `sample_fraction` outside `(0, 1]` must be rejected with a clear error.

## Context Reviewed

- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L378)
- [pgstatindex.c#registration](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L52-L64)
- [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L48-L204)
- [pgstatapprox.c#output_type](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L32-L44)
- [pageinspect.sgml#pageinspect](../../../../raw/postgres-18/doc/src/sgml/pageinspect.sgml#L1-L13)
- [pageinspect.sgml#btree-functions](../../../../raw/postgres-18/doc/src/sgml/pageinspect.sgml#L284-L349)
- [pageinspect--1.8--1.9.sql](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.8--1.9.sql#L33-L78)
- [pageinspect--1.11--1.12.sql](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.11--1.12.sql#L7-L24)
- [btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L91-L191)
- [btreefuncs.c#bt_page_stats_internal](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L260-L304)
- [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L951-L972)
- [bufpage.c#free-space-functions](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L900-L972)
- [bufpage.h#PageHeaderData](../../../../raw/postgres-18/src/include/storage/bufpage.h#L163-L185)
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-18/src/include/access/nbtree.h#L63-L85)
- [nbtree.h#BTMetaPageData](../../../../raw/postgres-18/src/include/access/nbtree.h#L104-L123)
- [nbtree.h#page-macros](../../../../raw/postgres-18/src/include/access/nbtree.h#L149-L226)
- [pgstattuple--1.4--1.5.sql](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L136)
- [pgstattuple.control](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L161-L539)
- [pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138)
- [pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L1-L305)

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstatindex` is a physical block scan (1..nblocks-1), not a tree walk, with per-page classification | [pgstatindex.c#scan-loop](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L278-L328) |
| `version`/`tree_level`/`root_block_no` come from one metapage read | [pgstatindex.c#metapage-read](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L252-L262), [nbtree.h#BTMetaPageData](../../../../raw/postgres-18/src/include/access/nbtree.h#L104-L123) |
| `avg_leaf_density` and `leaf_fragmentation` are page-local ratios needing no neighbor reads | [pgstatindex.c#leaf-accumulation](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L320), [pgstatindex.c#result](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L360-L369), [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L956-L972) |
| Page counts are the columns that need `1/f` scaling and carry the most estimate risk | [pgstatindex.c#classification](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L301-L323) |
| `pgstattuple_approx` is the precedent output shape (`scanned_percent`, `approx_` columns) | [pgstatapprox.c#output_type](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L32-L44), [pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119) |
| `pgstattuple_approx` still iterates every block and avoids linear extrapolation because it is not a random sample | [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L74-L94), [pgstatapprox.c#extrapolation-note](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L174-L186) |
| Adding a function follows the `PG_FUNCTION_INFO_V1` + upgrade-script + control-bump pattern | [pgstatindex.c#registration](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L52-L64), [pgstattuple--1.4--1.5.sql#pgstathashindex](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L123-L136), [pgstattuple.control#default_version](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L3) |
| A SQL prototype can use pageinspect's B-tree metapage and page-stat functions, but those functions are superuser/debugging tools | [pageinspect.sgml#pageinspect](../../../../raw/postgres-18/doc/src/sgml/pageinspect.sgml#L1-L13), [pageinspect--1.8--1.9.sql#btree-functions](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.8--1.9.sql#L51-L78), [btreefuncs.c#bt_page_stats_internal](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L260-L304) |
| pageinspect's `free_size` is not the exact free-space value used by `pgstatindex`, so a closer SQL prototype reads the raw page header | [btreefuncs.c#free_size](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L184-L191), [bufpage.c#free-space-functions](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L900-L972), [pageinspect--1.9--1.10.sql#page_header](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.9--1.10.sql#L7-L19) |
| `pgstatindex` is already documented as non-instantaneous page-by-page accumulation | [pgstattuple.sgml#pgstatindex-note](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L275-L279) |
| Current regression coverage never populates a B-tree to check density | [pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L18-L131), [pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L44-L299) |

## Source References

- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L378) - baseline metapage read, full physical scan, classification, leaf accumulation, and result tuple.
- [pgstatindex.c#registration](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L52-L64) - `PG_FUNCTION_INFO_V1` registration pattern, including the `_v1_5` variants.
- [pgstatindex.c#v1_5-note](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L159-L177) - why v1.5 functions drop the `superuser()` check in favor of `REVOKE`/`GRANT`.
- [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L48-L204) - heap approximate scan, including the non-random-sample extrapolation note and `scanned_percent`.
- [pgstatapprox.c#output_type](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L32-L44) - the approximate result struct copied as a precedent.
- [pageinspect.sgml](../../../../raw/postgres-18/doc/src/sgml/pageinspect.sgml#L1-L349) - pageinspect superuser/debugging scope and B-tree helper documentation.
- [pageinspect--1.8--1.9.sql](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.8--1.9.sql#L33-L78) - current `get_raw_page`, `bt_metap`, and `bt_page_stats` SQL signatures after the int8 upgrade.
- [pageinspect--1.11--1.12.sql](../../../../raw/postgres-18/contrib/pageinspect/pageinspect--1.11--1.12.sql#L7-L24) - `bt_multi_page_stats` range helper signature.
- [btreefuncs.c#GetBTPageStatistics](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L91-L191) - pageinspect B-tree page type mapping and `free_size` calculation.
- [btreefuncs.c#bt_page_stats_internal](../../../../raw/postgres-18/contrib/pageinspect/btreefuncs.c#L260-L304) - pageinspect per-page read, share lock, and superuser check.
- [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L951-L972) - exact per-page free space used by leaf density.
- [bufpage.h#PageHeaderData](../../../../raw/postgres-18/src/include/storage/bufpage.h#L163-L238) - page header fields and `SizeOfPageHeaderData` used by the SQL prototype.
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-18/src/include/access/nbtree.h#L63-L85) - leaf right link `btpo_next` and page flag bits.
- [nbtree.h#BTMetaPageData](../../../../raw/postgres-18/src/include/access/nbtree.h#L104-L123) - metapage fields (`btm_version`, `btm_root`, `btm_level`) read exactly.
- [nbtree.h#page-macros](../../../../raw/postgres-18/src/include/access/nbtree.h#L149-L226) - `BTREE_METAPAGE`, `P_NONE`, `P_ISLEAF`, `P_ISDELETED`, `P_IGNORE`.
- [pgstattuple--1.4--1.5.sql](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L136) - `pgstatindex` and `pgstattuple_approx` SQL signatures and the `pgstathashindex` add-via-upgrade pattern.
- [pgstattuple.control](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L1-L5) - `default_version = '1.5'` to bump for a new function.
- [pgstattuple.sgml](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L275-L539) - documented `pgstatindex` accumulation caveat and `pgstattuple_approx` behavior.
- [pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138) and [pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L1-L305) - current regression coverage to extend.

## Open Questions

- Whether the PostgreSQL project would accept a `pgstatindex_approx` API and
  which error/confidence fields it should expose is a design question the v18
  source tree does not answer.
- The best default `sample_fraction` and whether to offer Bernoulli vs
  fixed-count sampling are policy choices not derivable from source.
- This page proposes a design; its claims about v18 behavior are cited, but the
  proposed function and its DDL are not present in the pinned checkout and remain
  unverified by construction.

## Related Pages

- [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](pgstatindex-approx-sampling.md)
- [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](avg-leaf-density-during-vacuum.md)
- [v18/index](../../index.md)
- [versions](../../../versions.md)
