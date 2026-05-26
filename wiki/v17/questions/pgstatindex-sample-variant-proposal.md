---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: not yet
---

# Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)

## Question

In PostgreSQL 17, propose a `pgstatindex` variant that samples the index
instead of reading the whole index, include a pros and cons section, and
provide a proposal for a SQL implementation using available extensions in
contrib.

## Answer Up Front

Add a new function, `pgstatindex_approx`, rather than changing
`pgstatindex`. In PostgreSQL 17, `pgstatindex_impl` already reads B-tree pages
by physical block number, not by walking the logical tree: it reads block 0 for
metadata, then scans every non-metapage block from `1` through `nblocks - 1`,
share-locks each page, classifies the page from the B-tree opaque flags, and
accumulates leaf free space and backward-right-link fragments
([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L378)).
A sampled variant can keep the same per-page classification and accumulation,
but visit only a random subset of those non-metapage block numbers.

The useful split is:

- `version`, `tree_level`, and `root_block_no` stay exact because they come
  from the single metapage read of `btm_version`, `btm_level`, and `btm_root`
  ([pgstatindex.c#metapage-read](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L253-L264),
  [nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L104-L123)).
- `index_size` stays exact if it is computed from `RelationGetNumberOfBlocks`
  or the equivalent relation size instead of from sampled counts
  ([pgstatindex.c#nblocks](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L283),
  [pg_proc.dat#pg_relation_size](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7495),
  [system_functions.sql#pg_relation_size](../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)).
- `internal_pages`, `leaf_pages`, `empty_pages`, and `deleted_pages` are counts,
  so the sample must scale them by `1 / f`, where `f` is the realized sample
  fraction of non-metapage blocks.
- `avg_leaf_density` and `leaf_fragmentation` are ratios over leaf pages, so the
  sample estimates them directly. In v17, the density numerator uses
  `PageGetFreeSpace(page)`, not `PageGetExactFreeSpace(page)`, and
  fragmentation is counted when a leaf page's `btpo_next` points to an earlier
  physical block
  ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L323),
  [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L900-L916),
  [pgstatindex.c#result](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L350-L373)).

This should be a separate approximate API with explicit `approx_` output names.
`pgstattuple_approx` is the local precedent for exposing `scanned_percent` and
`approx_` columns rather than weakening the exact function's contract
([pgstatapprox.c#output_type](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L32-L45),
[pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119)).

## What pgstatindex Reads Today

`pgstatindex_impl` is the baseline to preserve. It:

1. Rejects non-B-tree relations, other-session temporary relations, and invalid
   indexes before reading index pages
   ([pgstatindex.c#guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L226-L250)).
2. Reads the B-tree metapage once and copies `btm_version`, `btm_level`, and
   `btm_root` into the result state
   ([pgstatindex.c#metapage-read](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L253-L264),
   [nbtree.h#BTREE_METAPAGE](../../../raw/postgres-17/src/include/access/nbtree.h#L149-L152)).
3. Reads every non-metapage block using a `BAS_BULKREAD` access strategy,
   calls `CHECK_FOR_INTERRUPTS`, share-locks the buffer, and classifies the
   page as deleted, empty/half-dead, leaf, or internal
   ([pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L330),
   [nbtree.h#page-macros](../../../raw/postgres-17/src/include/access/nbtree.h#L213-L226)).
4. For live leaf pages, adds `max_avail`, adds `PageGetFreeSpace(page)`, counts
   the page as a leaf, and increments `fragments` if `btpo_next` is nonzero and
   less than the current block number
   ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L323),
   [nbtree.h#BTPageOpaqueData](../../../raw/postgres-17/src/include/access/nbtree.h#L63-L85)).
5. Computes `avg_leaf_density = 100 - free_space / max_avail * 100` and
   `leaf_fragmentation = fragments / leaf_pages * 100`, returning `NaN` when
   the relevant denominator is zero
   ([pgstatindex.c#result](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L350-L373)).

The published output columns are `version`, `tree_level`, `index_size`,
`root_block_no`, `internal_pages`, `leaf_pages`, `empty_pages`,
`deleted_pages`, `avg_leaf_density`, and `leaf_fragmentation`
([pgstattuple--1.4--1.5.sql#pgstatindex](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L23-L38),
[pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L161-L279)).
The documentation already warns that results are accumulated page-by-page and
are not an instantaneous whole-index snapshot
([pgstattuple.sgml#pgstatindex-note](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L274-L279)).

## Proposed Function: pgstatindex_approx

### Field plan

| Output field | Source in proposal | Exact or estimated |
|---|---|---|
| `version`, `tree_level`, `root_block_no` | metapage read | exact |
| `index_size` | relation block count or `pg_relation_size(idx)` | exact |
| `scanned_percent` | realized sample fraction over non-metapage blocks | exact description of the sample |
| `approx_internal_pages`, `approx_leaf_pages`, `approx_empty_pages`, `approx_deleted_pages` | sampled counts divided by `f` | estimated |
| `approx_avg_leaf_density` | `100 - sum(PageGetFreeSpace) / sum(max_avail) * 100` over sampled leaves | estimated ratio |
| `approx_leaf_fragmentation` | sampled leaf pages with `btpo_next < blkno` divided by sampled leaf pages | estimated rate |

Use a uniform random sample of distinct block numbers from `[1, nblocks - 1]`,
then visit the selected blocks in ascending block order. Random selection gives
the count estimators their statistical meaning; ascending visitation keeps the
I/O pattern closer to the existing physical scan, which already advances
`blkno` upward with `BAS_BULKREAD`
([pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L330)).
The function should report `scanned_percent = 100 * sampled_blocks / (nblocks -
1)`, mirroring the way `pgstattuple_approx` reports the fraction of heap pages
it actually scanned
([pgstatapprox.c#scanned_percent](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L203-L208)).

The C extension wiring would follow existing `pgstattuple` patterns: add a
`PG_FUNCTION_INFO_V1` entry, add a v1.5-style entry point that relies on SQL
`REVOKE`/`GRANT` instead of an inline `superuser()` check, add a
`pgstattuple--1.5--1.6.sql` upgrade script, and bump `default_version` from
`1.5` to `1.6`
([pgstatindex.c#registration](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L55-L65),
[pgstatindex.c#v1_5-note](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L163-L179),
[pgstattuple--1.4--1.5.sql#pgstathashindex](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L123-L136),
[pgstattuple.control#default_version](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5)).

The proposed SQL signature would be:

```sql
-- proposed: contrib/pgstattuple/pgstattuple--1.5--1.6.sql
CREATE OR REPLACE FUNCTION pgstatindex_approx(IN relname regclass,
    IN sample_fraction FLOAT8 DEFAULT 0.01,
    OUT version INT,
    OUT tree_level INT,
    OUT index_size BIGINT,
    OUT root_block_no BIGINT,
    OUT scanned_percent FLOAT8,
    OUT approx_internal_pages BIGINT,
    OUT approx_leaf_pages BIGINT,
    OUT approx_empty_pages BIGINT,
    OUT approx_deleted_pages BIGINT,
    OUT approx_avg_leaf_density FLOAT8,
    OUT approx_leaf_fragmentation FLOAT8)
AS 'MODULE_PATHNAME', 'pgstatindex_approx'
LANGUAGE C STRICT PARALLEL SAFE;

REVOKE EXECUTE ON FUNCTION pgstatindex_approx(regclass, FLOAT8) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION pgstatindex_approx(regclass, FLOAT8) TO pg_stat_scan_tables;
```

This DDL is a proposal. It is not present in the PostgreSQL 17 pinned checkout.

## SQL Prototype Using contrib/pageinspect

A SQL prototype can be built with the contrib `pageinspect` extension. This is a
diagnostic implementation, not a replacement for a new `pgstattuple` C API:
`pageinspect` is documented as a low-level page debugging module whose functions
may be used only by superusers, and its B-tree helpers perform their own page
reads and share locks
([pageinspect.sgml#overview](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L13),
[btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L304)).

The v17 building blocks are available in contrib:

- `bt_metap(text)` returns B-tree metapage fields including `version`, `root`,
  and `level`
  ([pageinspect--1.8--1.9.sql#bt_metap](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L69-L84),
  [btreefuncs.c#bt_metap](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L828-L938)).
- `bt_page_stats(text, bigint)` returns the sampled page's `type`, `free_size`,
  `btpo_next`, and related counters. Its type mapping is compatible with
  `pgstatindex`: deleted pages become `d` or `D`, half-dead pages become `e`,
  live leaf pages become `l`, non-leaf root pages become `r`, and other internal
  pages become `i`
  ([pageinspect--1.8--1.9.sql#bt_page_stats](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L86-L102),
  [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L91-L191)).
- `get_raw_page(text, bigint)` plus `page_header(bytea)` exposes the sampled
  page's `special` offset, which the SQL prototype uses to compute
  `max_avail = special - 24`. The `24` is v17's `SizeOfPageHeaderData`, the
  offset of `pd_linp` in `PageHeaderData`
  ([pageinspect--1.8--1.9.sql#get_raw_page](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L33-L47),
  [pageinspect--1.9--1.10.sql#page_header](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L7-L19),
  [bufpage.h#PageHeaderData](../../../raw/postgres-17/src/include/storage/bufpage.h#L163-L238)).

The important v17 detail is that `bt_page_stats.free_size` uses
`PageGetFreeSpace(page)`, the same free-space routine used by
`pgstatindex_impl`, so the SQL prototype can use `free_size` directly for the
sampled leaf density numerator
([btreefuncs.c#free_size](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L184-L191),
[pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L315),
[bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L900-L916)).

Run diagnostic SQL with session timeouts:

```sql
SET /* wiki_pgstatindex_pageinspect_timeout */ statement_timeout = '5min';
SET /* wiki_pgstatindex_pageinspect_timeout */ lock_timeout = '2s';
CREATE /* wiki_pgstatindex_pageinspect_setup */ EXTENSION IF NOT EXISTS pageinspect;
```

The following SQL-language wrapper samples random non-metapage blocks, visits
them in block order, and computes the same exact-vs-estimated split as the C
proposal. It still enumerates candidate block numbers in SQL and reads each
sampled block through both `bt_page_stats` and `get_raw_page`, so it is best for
validation and one-off diagnostics, not a polished extension API.

```sql
CREATE /* wiki_pgstatindex_pageinspect_function */ OR REPLACE FUNCTION pgstatindex_approx_pageinspect(
    idx regclass,
    sample_fraction float8 DEFAULT 0.01)
RETURNS TABLE (
    version int,
    tree_level int,
    index_size bigint,
    root_block_no bigint,
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
AS $function$
WITH params AS (
  SELECT $1 AS idx, $2::float8 AS sample_fraction
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
                   ceil((nblocks - 1)::float8 * sample_fraction)::bigint))
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
       ps.free_size,
       ps.btpo_next,
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
       coalesce(sum(free_size) FILTER (WHERE type = 'l'), 0)::numeric AS leaf_free_space,
       count(*) FILTER (WHERE type = 'l' AND btpo_next <> 0 AND btpo_next < blkno)::bigint AS fragments_sample
  FROM page_sample
)
SELECT m.version,
     m.level::int AS tree_level,
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
$function$;
```

For lower SQL-call overhead, a prototype can sample contiguous ranges and use
`bt_multi_page_stats(text, bigint, bigint)`, which reports the same per-page
fields for a range and accepts a negative `blk_count` to mean all remaining
pages. That changes the estimator because it clusters the sampled pages, so it
is better for operational convenience than for low-variance random sampling
([pageinspect--1.11--1.12.sql#bt_multi_page_stats](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.11--1.12.sql#L7-L24),
[pageinspect.sgml#bt_multi_page_stats](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L351-L367),
[btreefuncs.c#bt_multi_page_stats](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L337-L472)).

## Pros

- **It reads fewer index pages.** At sample fraction `f`, the C version reads
  roughly `f * (nblocks - 1)` data pages plus the metapage, instead of every
  non-metapage block read by current `pgstatindex`
  ([pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L330)).
- **Several fields remain exact.** The metapage fields come from one block, and
  `index_size` can come from the relation size rather than scaled page counts
  ([pgstatindex.c#metapage-read](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L253-L264),
  [pg_proc.dat#pg_relation_size](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7495)).
- **The ratio metrics are page-local.** Leaf density needs the sampled page's
  free space and usable capacity; leaf fragmentation needs the sampled page's
  right link and current block number. Neither estimate needs sibling reads or
  a logical tree traversal
  ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L323)).
- **Random sampling makes count scaling defensible.** `pgstattuple_approx`
  explicitly avoids simple linear extrapolation because it is not taking a
  random sample; a deliberate uniform random block sample removes that specific
  objection for page counts
  ([pgstatapprox.c#extrapolation-note](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L188-L199)).
- **The implementation can stay small.** It can share the existing guard logic,
  page classification, `CHECK_FOR_INTERRUPTS`, and leaf accumulation from
  `pgstatindex_impl`
  ([pgstatindex.c#guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L226-L250),
  [pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L330)).
- **The pageinspect SQL prototype is available now.** It can validate estimator
  behavior with contrib functions before adding a new C function, while making
  the superuser/debugging limitation explicit
  ([pageinspect.sgml#overview](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L13)).

## Cons

- **Rare page classes become noisy.** `approx_deleted_pages` and
  `approx_empty_pages` are scaled from sampled hits. If deleted or half-dead
  pages are rare, a small sample can report zero when they exist, or
  overestimate from one sampled page
  ([pgstatindex.c#classification](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L299-L325)).
- **Physical clustering can raise variance.** Leaf density and fragmentation
  can vary by file region after splits, deletes, bulk loads, and page reuse; a
  small block sample may miss a clustered problem even when the estimator is
  unbiased in expectation. A contiguous-range SQL variant is especially exposed
  to this problem.
- **Sparse ordered reads lose some sequential-scan advantage.** Sorting selected
  blocks before reading them is better than reading in random order, but it
  still skips most blocks rather than making the current dense ascending scan
  ([pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L330)).
- **It is still not a consistent snapshot.** Current `pgstatindex` already
  accumulates page-by-page; a sampler inherits that behavior and can observe
  concurrent splits, deletion states, or page recycling at different moments
  ([pgstattuple.sgml#pgstatindex-note](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L274-L279)).
- **`leaf_fragmentation` loses run structure.** The exact function counts every
  backward right link across the full physical file; a sample estimates only the
  rate of backward links among sampled leaf pages
  ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L316-L323)).
- **The SQL prototype is operationally clunky.** It needs `pageinspect`, uses
  superuser-only functions, enumerates block numbers in SQL, sorts by
  `random()`, and reads each sampled block twice to get both B-tree page stats
  and the page header
  ([pageinspect.sgml#overview](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L13),
  [btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L304)).
- **It adds API and testing surface.** A real extension patch needs a new C
  function, extension version bump, SQL upgrade script, docs, permissions,
  sample-fraction validation, and regression tests. Existing pgstattuple tests
  cover empty-index output, wrong-AM errors, unsupported relation kinds, and a
  partition-index success case, but not populated B-tree density or approximate
  sampling behavior
  ([pgstattuple.sql](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L131),
  [pgstattuple.out](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L299)).

## Why This Differs From pgstattuple_approx

`pgstattuple_approx` is not random block sampling. It iterates over every heap
block, skips reading blocks only when the visibility map proves all tuples are
visible, gets skipped-page free space from the free space map, and estimates
live tuple count with `vac_estimate_reltuples` because the scan is not a random
sample
([pgstatapprox.c#statapprox_heap](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L64-L208),
[pgstattuple.sgml#pgstattuple_approx](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L489-L539)).

For a B-tree `pgstatindex_approx`, the proposal is simpler and more explicit:
sample physical index blocks, read only those blocks plus the metapage, and
expose estimate columns so callers know which values are approximate.

## Tests To Add

- `sample_fraction = 1.0` must match `pgstatindex` on the same index for page
  counts, density, and fragmentation.
- `version`, `tree_level`, `root_block_no`, and `index_size` must match
  `pgstatindex` for every sample fraction because they are exact fields.
- A populated B-tree should compare `approx_avg_leaf_density` with the exact
  `avg_leaf_density` using a tolerance, and should verify `scanned_percent`.
- An empty B-tree must return `NaN` for the two ratio fields, following the
  current denominator guards
  ([pgstatindex.c#result](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L360-L373)).
- Wrong access methods and unsupported relation kinds should error like current
  `pgstatindex`
  ([pgstatindex.c#guards](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L226-L250),
  [pgstattuple.out](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L122-L299)).
- `sample_fraction` outside `(0, 1]` should raise a clear error in the C
  implementation. The SQL prototype above only filters invalid fractions out,
  so that is an implementation gap.

## Context Reviewed

- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L378)
- [pgstatindex.c#registration](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L55-L65)
- [pgstatapprox.c#statapprox_heap](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L64-L208)
- [pgstatapprox.c#output_type](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L32-L45)
- [pageinspect.sgml#overview](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L13)
- [pageinspect.sgml#btree-functions](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L284-L367)
- [pageinspect--1.8--1.9.sql](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L33-L102)
- [pageinspect--1.9--1.10.sql](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L7-L19)
- [pageinspect--1.11--1.12.sql](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.11--1.12.sql#L7-L24)
- [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L91-L191)
- [btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L304)
- [btreefuncs.c#bt_multi_page_stats](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L337-L472)
- [btreefuncs.c#bt_metap](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L828-L938)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L900-L916)
- [bufpage.h#PageHeaderData](../../../raw/postgres-17/src/include/storage/bufpage.h#L163-L238)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-17/src/include/access/nbtree.h#L63-L85)
- [nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L104-L123)
- [nbtree.h#page-macros](../../../raw/postgres-17/src/include/access/nbtree.h#L149-L226)
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L23-L136)
- [pgstattuple.control](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L161-L539)
- [pgstattuple.sql](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138)
- [pgstattuple.out](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L1-L305)

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstatindex` performs a physical scan of non-metapage blocks and classifies each page locally | [pgstatindex.c#scan-loop](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L279-L330) |
| Metapage fields are exact | [pgstatindex.c#metapage-read](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L253-L264), [nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L104-L123) |
| v17 leaf density uses `PageGetFreeSpace`, and fragmentation uses `btpo_next < blkno` | [pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L323), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L900-L916) |
| `pgstattuple_approx` is the output-shape precedent but is not random sampling | [pgstatapprox.c#output_type](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L32-L45), [pgstatapprox.c#extrapolation-note](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L188-L199) |
| `pageinspect` can prototype the idea with metapage and page-stat helpers, but it is superuser/debugging-oriented | [pageinspect.sgml#overview](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L13), [pageinspect--1.8--1.9.sql#btree-functions](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L69-L102), [btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L304) |
| `bt_page_stats.free_size` matches the v17 `pgstatindex` free-space routine | [btreefuncs.c#free_size](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L184-L191), [pgstatindex.c#leaf-accumulation](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L315) |
| Current pgstattuple tests do not cover populated B-tree sampling behavior | [pgstattuple.sql](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L131), [pgstattuple.out](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L299) |

## Source References

- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L378) - baseline metapage read, full physical scan, classification, leaf accumulation, and result tuple.
- [pgstatindex.c#registration](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L55-L65) - `PG_FUNCTION_INFO_V1` registration pattern.
- [pgstatindex.c#v1_5-note](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L163-L179) - v1.5 permission model relying on SQL `REVOKE`/`GRANT`.
- [pgstatapprox.c#statapprox_heap](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L64-L208) - heap approximate scan, VM skip behavior, non-random-sample note, and `scanned_percent` calculation.
- [pgstatapprox.c#output_type](../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L32-L45) - approximate result struct used as output-shape precedent.
- [pageinspect.sgml](../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L1-L367) - pageinspect superuser/debugging scope and B-tree helper documentation.
- [pageinspect--1.8--1.9.sql](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L33-L102) - `get_raw_page`, `bt_metap`, and `bt_page_stats` SQL signatures.
- [pageinspect--1.9--1.10.sql](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L7-L19) - `page_header(bytea)` SQL signature.
- [pageinspect--1.11--1.12.sql](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.11--1.12.sql#L7-L24) - `bt_multi_page_stats` SQL signature.
- [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L91-L191) - pageinspect B-tree page type mapping, `max_avail`, and `free_size` calculation.
- [btreefuncs.c#bt_page_stats_internal](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L260-L304) - per-page pageinspect read, share lock, and superuser check.
- [btreefuncs.c#bt_multi_page_stats](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L337-L472) - range page-stat helper behavior.
- [btreefuncs.c#bt_metap](../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L828-L938) - pageinspect metapage readout.
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L900-L916) - index-page free-space routine used by v17 `pgstatindex`.
- [bufpage.h#PageHeaderData](../../../raw/postgres-17/src/include/storage/bufpage.h#L163-L238) - page header fields and `SizeOfPageHeaderData` used by the SQL prototype.
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-17/src/include/access/nbtree.h#L63-L85) - B-tree right-link and flag fields.
- [nbtree.h#BTMetaPageData](../../../raw/postgres-17/src/include/access/nbtree.h#L104-L123) - metapage fields read exactly.
- [nbtree.h#page-macros](../../../raw/postgres-17/src/include/access/nbtree.h#L149-L226) - metapage constant and B-tree page-state macros.
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L23-L136) - existing `pgstatindex`, `pgstattuple_approx`, and `pgstathashindex` extension SQL patterns.
- [pgstattuple.control](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5) - current `default_version = '1.5'`.
- [pgstattuple.sgml](../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L161-L539) - documented `pgstatindex` columns, page-by-page accumulation caveat, and `pgstattuple_approx` behavior.
- [pgstattuple.sql](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138) and [pgstattuple.out](../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L1-L305) - current regression coverage to extend.

## Open Questions

- Whether PostgreSQL would accept a `pgstatindex_approx` API, and which error
  or confidence fields it should expose, is a project design question not
  answered by the v17 source tree.
- The best default `sample_fraction` and whether to use Bernoulli sampling,
  fixed-count sampling, or range sampling are policy choices not derivable from
  source.
- The SQL prototype is intentionally diagnostic. It depends on `pageinspect`,
  needs superuser-only functions, and does not implement the same validation or
  read scheduling that a C extension function should provide.

## Related Pages

- [PostgreSQL 17 Contrib Extensions (unverified)](contrib-extensions.md)
- [v17/index](../index.md)
- [versions](../../versions.md)
