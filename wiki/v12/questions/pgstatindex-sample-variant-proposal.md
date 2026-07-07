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
- [Answer Up Front](#answer-up-front)
- [What pgstatindex Reads Today](#what-pgstatindex-reads-today)
- [Proposed Function: pgstatindex_approx](#proposed-function-pgstatindexapprox)
  - [Field plan](#field-plan)
  - [Extension wiring](#extension-wiring)
- [Pros](#pros)
- [Cons](#cons)
- [SQL Prototype Using contrib/pageinspect](#sql-prototype-using-contribpageinspect)
- [Tests To Add](#tests-to-add)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Source References](#source-references)
- [Open Questions](#open-questions)
- [Related Pages](#related-pages)

## Question

In PostgreSQL 12, propose a `pgstatindex` variant that samples the index
instead of reading the whole index, include a pros and cons section, and
provide a proposal for a SQL implementation using available extensions in
contrib.

## Answer Up Front

Add a new function, `pgstatindex_approx`, rather than changing
`pgstatindex`. In PostgreSQL 12, `pgstatindex_impl` reads the B-tree metapage,
then physically scans every non-metapage block from `1` through `nblocks - 1`,
share-locks each page, classifies it from the B-tree opaque flags, and
accumulates leaf free space plus backward-right-link fragments
([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)).
A sampled variant can reuse the same per-page classification and accumulation,
but visit only a uniform random subset of those non-metapage block numbers.

The useful output split is:

- `version`, `tree_level`, and `root_block_no` stay exact because they come from
  the single metapage read of `btm_version`, `btm_level`, and `btm_root`
  ([pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
  [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113)).
- `index_size` should stay exact by using the relation's block count or
  `pg_relation_size(idx)`, because v12's exact function derives `index_size`
  from the full page counts after it has scanned all non-metapage blocks
  ([pgstatindex.c#nblocks](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L271),
  [pgstatindex.c#index_size-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L342),
  [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)).
- `internal_pages`, `leaf_pages`, `empty_pages`, and `deleted_pages` are counts,
  so a random block sample scales them by `1 / f`, where `f` is the realized
  sample fraction over non-metapage blocks.
- `avg_leaf_density` and `leaf_fragmentation` are ratios over leaf pages, so the
  sample estimates those ratios directly. In v12, leaf density uses
  `PageGetFreeSpace(page)` and `max_avail`, while fragmentation increments when
  a live leaf page's `btpo_next` points to an earlier physical block
  ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307),
  [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L573-L596),
  [pgstatindex.c#result-ratios](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).

Use explicit `approx_` output names and report `scanned_percent`. That follows
the v12 `pgstattuple_approx` precedent, which exposes `scanned_percent` and
`approx_` fields instead of weakening the exact function's output contract
([pgstatapprox.c#output_type](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52),
[pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119)).

## What pgstatindex Reads Today

`pgstatindex_impl` is the baseline to preserve. It:

1. Rejects relations that are not B-tree indexes and rejects other-session
   temporary relations before reading index pages
   ([pgstatindex.c#guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238)).
2. Reads block 0, the B-tree metapage, and copies `btm_version`, `btm_level`,
   and `btm_root` into its result state
   ([pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
   [nbtree.h#BTREE_METAPAGE](../../../raw/postgres-12/src/include/access/nbtree.h#L131-L135)).
3. Reads every non-metapage block using a `BAS_BULKREAD` strategy, calls
   `CHECK_FOR_INTERRUPTS`, share-locks the buffer, and classifies the page as
   deleted, empty/half-dead, leaf, or internal
   ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
   [nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L187-L196)).
4. For each live leaf page, adds `max_avail`, adds `PageGetFreeSpace(page)`,
   counts the page as a leaf, and increments `fragments` if `btpo_next` is not
   `P_NONE` and is less than the current physical block number
   ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307),
   [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)).
5. Computes `avg_leaf_density = 100 - free_space / max_avail * 100` and
   `leaf_fragmentation = fragments / leaf_pages * 100`, returning `NaN` when a
   denominator is zero
   ([pgstatindex.c#result-ratios](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).

The published v12 output columns are `version`, `tree_level`, `index_size`,
`root_block_no`, `internal_pages`, `leaf_pages`, `empty_pages`,
`deleted_pages`, `avg_leaf_density`, and `leaf_fragmentation`
([pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L189-L265)).
The docs already warn that results are accumulated page-by-page and are not an
instantaneous whole-index snapshot
([pgstattuple.sgml#pgstatindex-note](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L278)).

## Proposed Function: pgstatindex_approx

### Field plan

| Output field | Source in proposal | Exact or estimated |
|---|---|---|
| `version`, `tree_level`, `root_block_no` | metapage read | exact |
| `index_size` | relation block count times `BLCKSZ`, or `pg_relation_size(idx)` | exact |
| `scanned_percent` | realized sample fraction over non-metapage blocks | exact description of the sample |
| `approx_internal_pages`, `approx_leaf_pages`, `approx_empty_pages`, `approx_deleted_pages` | sampled counts divided by `f` | estimated |
| `approx_avg_leaf_density` | `100 - sum(PageGetFreeSpace) / sum(max_avail) * 100` over sampled leaves | estimated ratio |
| `approx_leaf_fragmentation` | sampled leaf pages with `btpo_next < blkno` divided by sampled leaf pages | estimated rate |

Draw a uniform random sample of distinct block numbers from `[1, nblocks - 1]`,
then visit the selected blocks in ascending order. Random selection gives the
count estimators their meaning, and ascending visitation keeps the access pattern
closer to today's physical scan, which already walks block numbers upward with a
bulk-read strategy
([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
Report `scanned_percent = 100 * sampled_blocks / (nblocks - 1)`, mirroring the
way `pgstattuple_approx` reports how much of a relation it scanned
([pgstatapprox.c#scanned-percent](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L198-L207)).

The C patch should share the existing guard, metapage, page-classification,
leaf-accumulation, and `CHECK_FOR_INTERRUPTS` logic from `pgstatindex_impl`
([pgstatindex.c#guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238),
[pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
It should validate `sample_fraction` in `(0, 1]`, choose sampled block numbers
without replacement, and avoid deriving exact-size fields from scaled page
counts.

### Extension wiring

PostgreSQL 12's `pgstattuple` code keeps both text and regclass entry points for
backward compatibility, with v1.5 SQL grants replacing inline superuser checks
for extension-installed functions
([pgstatindex.c#entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L66),
[pgstatindex.c#v1_5-permissions](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212),
[pgstattuple--1.4--1.5.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
A v12 patch should add a new C symbol, add extension SQL in a
`pgstattuple--1.5--1.6.sql` upgrade script, revoke public execute, grant execute
to `pg_stat_scan_tables`, and bump the control file from `default_version =
'1.5'` to `1.6`
([pgstattuple--1.4--1.5.sql#grant-pattern](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L91-L119),
[pgstattuple.control#default_version](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5)).

Proposed extension DDL:

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

This DDL is a proposal, not an object defined by the v12 pinned checkout. It
follows the verified v12 extension script shape for C functions with OUT
columns, `STRICT`, `PARALLEL SAFE`, and SQL-managed privileges
([pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119)).

## Pros

- **It reads fewer index pages.** At sample fraction `f`, the C implementation
  reads the metapage plus roughly `f * (nblocks - 1)` data pages, instead of the
  current loop over every non-metapage block
  ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- **Several fields remain exact.** Metapage fields come from one block, and
  `index_size` can come from the relation size instead of scaled sampled counts
  ([pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
  [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)).
- **The ratio metrics are page-local.** Leaf density needs only sampled leaf
  free space and usable capacity; fragmentation needs only the sampled page's
  right link and its current block number
  ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)).
- **Random sampling makes count scaling more defensible.** The v12
  `pgstattuple_approx` code explicitly avoids simple linear extrapolation of
  live tuple count because that heap shortcut is not a random sample; a uniform
  index-block sample removes that specific objection for sampled page counts
  ([pgstatapprox.c#extrapolation-note](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L187-L196)).
- **The result contract is clear.** The existing `pgstattuple_approx` result
  uses `scanned_percent` and `approx_` fields, which gives a local naming
  precedent for an approximate `pgstatindex` API
  ([pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119)).
- **A SQL prototype is possible with contrib today.** v12 `pageinspect` exposes
  `bt_metap`, `bt_page_stats`, `get_raw_page`, and `page_header`, which are
  enough to sample selected B-tree blocks and compute the same estimator split
  ([pageinspect--1.5.sql#raw-page-header](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L7-L33),
  [pageinspect--1.5.sql#btree-functions](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L146-L175)).

## Cons

- **Rare page classes become noisy.** `approx_deleted_pages` and
  `approx_empty_pages` are scaled from sampled hits, so a small sample can miss
  rare deleted or half-dead pages, or overestimate them from one sampled page
  ([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).
- **Physical clustering can raise variance.** B-tree splits, deletes, and page
  reuse can concentrate density or fragmentation problems in parts of the file;
  a small physical block sample can miss such clustered regions even when the
  estimator is unbiased in expectation. The source metric is page-local, but it
  is still collected from physical block positions
  ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- **Sparse ordered reads lose some sequential-scan benefit.** Sorting sampled
  block numbers before reading them is better than visiting random order, but it
  still skips around the file instead of doing the current dense ascending scan
  with `BAS_BULKREAD`
  ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- **It is still not a consistent snapshot.** Current `pgstatindex` is already
  accumulated page-by-page, and the docs say concurrent changes can affect the
  result; a sampler inherits that limitation
  ([pgstattuple.sgml#pgstatindex-note](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L278)).
- **`leaf_fragmentation` loses run structure.** The exact function counts every
  leaf page whose right link moves backward in physical block order; a sample
  estimates only the rate of backward links among sampled leaf pages
  ([pgstatindex.c#fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307)).
- **The SQL prototype is operationally clunky.** v12 `pageinspect` is a
  superuser-only low-level debugging module, and the prototype enumerates block
  numbers in SQL, sorts by `random()`, and reads each sampled block through both
  `bt_page_stats` and `get_raw_page`/`page_header`
  ([pageinspect.sgml#overview](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L13),
  [btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L213),
  [rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L94-L171)).
- **The v12 pageinspect SQL API uses `int4` block numbers.** The prototype must
  cast sampled block numbers to `int4`, while backend `BlockNumber` is wider;
  a production C function should not inherit that SQL wrapper limit
  ([pageinspect--1.5.sql#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L175),
  [btreefuncs.c#bt_page_stats-args](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L165)).
- **It adds extension and test surface.** A real patch needs a new C function,
  extension version bump, SQL upgrade script, docs, permissions, sample-fraction
  validation, and regression tests. Existing v12 tests exercise exact
  `pgstatindex` entry points and error cases, but no sampled index statistic
  exists in the checked-out tests
  ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L45),
  [pgstattuple.sql#pgstatindex-errors](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L55-L113)).

## SQL Prototype Using contrib/pageinspect

The SQL prototype should use `pageinspect`, not `pgstatindex`, because
`pgstatindex` itself reads the whole index. `pageinspect` is a contrib module for
low-level page debugging and its functions are superuser-only in v12
([pageinspect.sgml#overview](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L13)).
The required building blocks are available:

- The `bt_metap` C symbol reads B-tree metapage fields including `version`,
  `root`, and `level`, but the v12 pageinspect 1.7 SQL wrapper also declares
  `oldest_xact` as `int4` while `btreefuncs.c` formats the underlying
  `TransactionId` as an unsigned decimal string. The prototype below uses a
  six-column compatibility wrapper around the same C symbol so tuple conversion
  stops before `oldest_xact`
  ([pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26),
  [btreefuncs.c#bt_metap-output](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L542-L575),
  [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2144)).
- `bt_page_stats(text, int4)` reads one non-metapage B-tree block, share-locks
  it, returns its `type`, `free_size`, `btpo_next`, and other fields, and errors
  on block 0
  ([pageinspect--1.5.sql#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L175),
  [btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L213)).
- v12 `bt_page_stats.free_size` is computed with `PageGetFreeSpace(page)`, the
  same routine used by v12 `pgstatindex_impl` for leaf density
  ([btreefuncs.c#free-size](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L131-L148),
  [pgstatindex.c#leaf-free-space](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L296-L299)).
- `get_raw_page(text, int4)` plus `page_header(bytea)` exposes the page header's
  `special` offset. The SQL wrapper uses that to compute `max_avail = special -
  24`, because v12's `SizeOfPageHeaderData` is the offset of `pd_linp` in
  `PageHeaderData`
  ([pageinspect--1.5.sql#raw-page-header](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L7-L33),
  [rawpage.c#page_header-output](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L256-L278),
  [bufpage.h#SizeOfPageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L163-L216)).

The setup statements below use session/transaction-scope timeouts: v12 defines
both `statement_timeout` and `lock_timeout` as `PGC_USERSET`, so no restart or
reload is required for a session-local diagnostic setting
([guc.c#diagnostic-timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2397)).
`CREATE EXTENSION IF NOT EXISTS` is v12 syntax, and loading an extension runs
its script to create SQL objects
([create_extension.sgml#syntax](../../../raw/postgres-12/doc/src/sgml/ref/create_extension.sgml#L24-L29),
[create_extension.sgml#description](../../../raw/postgres-12/doc/src/sgml/ref/create_extension.sgml#L35-L55)).
If the wrapper errors with `function bt_page_stats(text, integer) does not
exist`, that is not a v12 argument-type mismatch: v12 documents
`bt_page_stats(relname text, blkno int)` and the extension script creates
`bt_page_stats(text, int4)`. The usual cause is that `pageinspect` is not
installed in the current database, or its schema is not visible through the
wrapper's `search_path`
([pageinspect.sgml#bt_page_stats](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L270-L293),
[pageinspect--1.5.sql#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L175)).
`pg_extension.extnamespace` records the extension schema, and the v12
`CREATE FUNCTION` syntax can save the current `search_path` for execution with
`SET search_path FROM CURRENT`
([pg_extension.h#FormData_pg_extension](../../../raw/postgres-12/src/include/catalog/pg_extension.h#L30-L45),
[create_function.sgml#configuration-parameter](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L494-L505)).

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

If the wrapper errors with `value "4145147631" is out of range for type
integer`, the likely v12-specific cause is `bt_metap`, not the estimator math:
pageinspect 1.7 declares `oldest_xact int4`, but the C function supplies the
unsigned `btm_oldest_btpo_xact` string. PostgreSQL converts every declared
output column in the tuple descriptor, so the conversion can fail even though
the outer query only needs `version`, `level`, and `root`
([pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26),
[btreefuncs.c#bt_metap-output](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L564-L575),
[nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113),
[execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2144)).

The SQL wrapper uses v12-supported `CREATE FUNCTION` syntax for default input
arguments, `RETURNS TABLE`, and `PARALLEL` marking; its query uses v12 `WITH ...
MATERIALIZED`, aggregate `FILTER`, `pg_relation_size(regclass)`,
`current_setting(text)`, `generate_series(int8, int8)`, `random()`, and
`ceil(float8)`
([create_function.sgml#syntax](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L23-L39),
[create_function.sgml#defaults](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L183-L188),
[select.sgml#with-materialized](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L75-L98),
[select.sgml#materialized-note](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L292-L328),
[syntax.sgml#aggregate-filter](../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L1563-L1580),
[syntax.sgml#filter-semantics](../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L1727-L1738),
[pg_proc.dat#prototype-builtins](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5758-L5764),
[pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891),
[pg_proc.dat#generate_series-int8](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L7634-L7645),
[pg_proc.dat#random](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3223-L3225),
[pg_proc.dat#ceil](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L637-L642)).

This is diagnostic SQL, not the preferred extension implementation. It avoids
reading all index pages, but it still enumerates all candidate block numbers and
sorts them by `random()` before reading the sampled blocks.

The prototype also floors the effective sample fraction at `0.10` when
`pg_relation_size(idx) < 100 MiB`. The default `sample_fraction = 0.01` would
otherwise draw only a handful of blocks from a small index, which makes
`approx_leaf_pages`, `approx_avg_leaf_density`, and `approx_leaf_fragmentation`
very noisy and can produce `NaN` ratios when the sample happens to miss every
leaf page. The 100 MiB threshold and the 10% floor are policy choices, not
derivations from the v12 source; they are applied only inside this prototype's
`bounds` CTE and do not change the user-supplied `sample_fraction` argument or
the validation in the same CTE's `WHERE` clause.

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
WITH params AS (
  SELECT $1 AS idx, $2::float8 AS sample_fraction
),
bounds AS (
  SELECT idx,
       sample_fraction,
       pg_relation_size(idx) AS index_size,
       pg_relation_size(idx) / current_setting('block_size')::int AS nblocks,
       CASE
         WHEN pg_relation_size(idx) < (100 * 1024 * 1024)::bigint
              THEN GREATEST(sample_fraction, 0.10::float8)
         ELSE sample_fraction
       END AS effective_sample_fraction
  FROM params
  WHERE sample_fraction > 0
    AND sample_fraction <= 1
    AND pg_relation_size(idx) / current_setting('block_size')::int <= 2147483648::bigint
),
target AS (
  SELECT *,
       CASE
         WHEN nblocks <= 1 THEN 0::bigint
         ELSE ceil((nblocks - 1)::float8 * effective_sample_fraction)::bigint
       END AS raw_sample_target
  FROM bounds
),
sample_plan AS (
  SELECT *,
       CASE
         WHEN nblocks <= 1 THEN 0::bigint
         WHEN raw_sample_target < 1 THEN 1::bigint
         WHEN raw_sample_target > nblocks - 1 THEN nblocks - 1
         ELSE raw_sample_target
       END AS sample_target
  FROM target
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
       ps.free_size,
       ps.btpo_next,
       h.special
  FROM sample_blocks AS s
  CROSS JOIN LATERAL bt_page_stats(s.idx::text, s.blkno::int4) AS ps
  CROSS JOIN LATERAL page_header(get_raw_page(s.idx::text, s.blkno::int4)) AS h
),
agg AS (
  SELECT count(*)::bigint AS sampled_blocks,
       count(*) FILTER (WHERE type IN ('i', 'r'))::bigint AS internal_sample,
       count(*) FILTER (WHERE type = 'l')::bigint AS leaf_sample,
       count(*) FILTER (WHERE type = 'e')::bigint AS empty_sample,
       count(*) FILTER (WHERE type = 'd')::bigint AS deleted_sample,
       coalesce(sum(CASE WHEN special::int > 24 THEN special::int - 24 ELSE 0 END)
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

## Tests To Add

- `sample_fraction = 1.0` should match `pgstatindex` on page counts, density,
  and fragmentation because it samples every non-metapage block
  ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- `version`, `tree_level`, `root_block_no`, and `index_size` should match
  `pgstatindex` for every valid sample fraction because they are exact fields
  in the proposal
  ([pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
  [pgstatindex.c#index_size-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L342)).
- Empty B-trees should return `NaN` for both ratio fields, following the current
  denominator guards
  ([pgstatindex.c#result-ratios](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356),
  [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L45)).
- Wrong access methods and unsupported relation kinds should error like current
  `pgstatindex`
  ([pgstatindex.c#guards](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238),
  [pgstattuple.sql#pgstatindex-errors](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L55-L113)).
- `sample_fraction` outside `(0, 1]` should raise a clear C-extension error.
  The SQL prototype above filters invalid fractions out, so that is a prototype
  limitation rather than the desired API behavior.

## Context Reviewed

- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)
- [pgstatindex.c#entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L66)
- [pgstatapprox.c#statapprox_heap](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L64-L208)
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L136)
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L278)
- [pageinspect.sgml#overview](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L13)
- [pageinspect.sgml#btree-functions](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L242-L305)
- [pageinspect--1.5.sql#raw-page-header](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L7-L33)
- [pageinspect--1.5.sql#btree-functions](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L146-L175)
- [pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26)
- [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2144)
- [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153)
- [btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L237)
- [btreefuncs.c#bt_metap](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L505-L575)
- [rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L94-L171)
- [rawpage.c#page_header](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L218-L287)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L573-L596)
- [bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L163-L216)
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)
- [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113)
- [nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L131-L196)
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)

## Evidence Map

| Claim | Evidence |
|---|---|
| Current `pgstatindex` scans every non-metapage block physically | [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315) |
| Metapage fields are exact | [pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253), [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113) |
| v12 leaf density uses `PageGetFreeSpace`, and fragmentation uses `btpo_next < blkno` | [pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L573-L596) |
| `pgstattuple_approx` is the local approximate-output precedent | [pgstatapprox.c#output_type](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52), [pgstattuple--1.4--1.5.sql#pgstattuple_approx](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L104-L119) |
| `pageinspect` can prototype sampled B-tree page reads, but it is superuser/debugging-oriented | [pageinspect.sgml#overview](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L13), [pageinspect--1.5.sql#btree-functions](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L146-L175), [btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L213) |
| The v12 `bt_metap` SQL wrapper can raise `integer` overflow on unsigned `oldest_xact`; the compatibility helper avoids converting that output column | [pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26), [btreefuncs.c#bt_metap-output](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L564-L575), [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2144) |
| The `bt_page_stats(text, integer)` error is a setup or `search_path` problem, not a v12 signature mismatch | [pageinspect.sgml#bt_page_stats](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L270-L293), [pageinspect--1.5.sql#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L175), [pg_extension.h#FormData_pg_extension](../../../raw/postgres-12/src/include/catalog/pg_extension.h#L30-L45) |
| v12 SQL syntax and built-ins used by the wrapper exist in the pinned checkout | [create_function.sgml#syntax](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L23-L39), [select.sgml#with-materialized](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L75-L98), [syntax.sgml#aggregate-filter](../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L1563-L1580), [pg_proc.dat#prototype-builtins](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5758-L5764) |

## Source References

- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365) - baseline metapage read, full physical scan, page classification, leaf accumulation, and result tuple.
- [pgstatindex.c#entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L66) - v12 registration pattern and text/regclass compatibility note.
- [pgstatindex.c#v1_5-permissions](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212) - v1.5 permission model relying on SQL `REVOKE`/`GRANT`.
- [pgstatapprox.c#statapprox_heap](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L64-L208) - heap approximate scan, VM skip behavior, non-random-sample note, and `scanned_percent` calculation.
- [pgstatapprox.c#output_type](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L38-L52) - approximate result struct used as output-shape precedent.
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L136) - existing `pgstatindex`, `pgstattuple_approx`, grants, and extension SQL style.
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) - current `default_version = '1.5'`.
- [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L278) - documented `pgstatindex` columns and page-by-page accumulation caveat.
- [pageinspect.sgml](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L10-L305) - pageinspect superuser/debugging scope and B-tree helper documentation.
- [pageinspect--1.5.sql](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L7-L175) - `get_raw_page`, `page_header`, `bt_metap`, and `bt_page_stats` SQL signatures.
- [pageinspect--1.6--1.7.sql#bt_metap](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.6--1.7.sql#L16-L26) - v12 pageinspect 1.7 `bt_metap` output shape.
- [execTuples.c#BuildTupleFromCStrings](../../../raw/postgres-12/src/backend/executor/execTuples.c#L2111-L2144) - tuple conversion loop for C-string result fields.
- [btreefuncs.c#GetBTPageStatistics](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L90-L153) - pageinspect B-tree page type mapping, `max_avail`, and `free_size` calculation.
- [btreefuncs.c#bt_page_stats](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L161-L237) - single-page pageinspect read, share lock, superuser check, and output tuple.
- [btreefuncs.c#bt_metap](../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L505-L575) - pageinspect metapage readout.
- [rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L94-L171) - raw page copy path and share lock.
- [rawpage.c#page_header](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L218-L287) - raw page header fields exposed to SQL.
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L573-L596) - index-page free-space routine used by v12 `pgstatindex`.
- [bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L163-L216) - page header layout and `SizeOfPageHeaderData`.
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68) - B-tree right-link and flag fields.
- [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113) - metapage fields read exactly.
- [nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L131-L196) - metapage constant and B-tree page-state macros.
- [guc.c#diagnostic-timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2397) - `statement_timeout` and `lock_timeout` contexts.
- [create_extension.sgml#syntax](../../../raw/postgres-12/doc/src/sgml/ref/create_extension.sgml#L24-L55) - `CREATE EXTENSION IF NOT EXISTS` syntax and extension loading behavior.
- [pg_extension.h#FormData_pg_extension](../../../raw/postgres-12/src/include/catalog/pg_extension.h#L30-L45) - extension schema catalog field used to find where `pageinspect` installed its helper functions.
- [create_function.sgml#syntax](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L23-L39) and [create_function.sgml#configuration-parameter](../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L494-L505) - SQL function syntax and `SET search_path FROM CURRENT` behavior used by the wrapper.
- [select.sgml#with-materialized](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L75-L98) and [select.sgml#materialized-note](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L292-L328) - v12 CTE materialization syntax and semantics.
- [syntax.sgml#aggregate-filter](../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L1563-L1580) and [syntax.sgml#filter-semantics](../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L1727-L1738) - aggregate `FILTER` syntax and behavior.
- [pg_proc.dat#prototype-builtins](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5758-L5764), [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [pg_proc.dat#generate_series-int8](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L7634-L7645), [pg_proc.dat#random](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3223-L3225), and [pg_proc.dat#ceil](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L637-L642) - built-in functions used by the SQL wrapper.
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113) - current exact `pgstatindex` regression coverage to extend.

## Open Questions

- Whether PostgreSQL would accept a `pgstatindex_approx` API, and whether it
  should expose confidence/error fields, is a project design question not
  answered by the v12 source tree.
- The best default `sample_fraction`, and whether to use Bernoulli sampling,
  fixed-count sampling, or range sampling, are policy choices not derivable from
  the pinned source.
- The SQL prototype is intentionally diagnostic. It depends on superuser-only
  `pageinspect` functions, filters invalid arguments by returning no rows, and
  does not implement the same validation or read scheduling that a C extension
  function should provide.

## Related Pages

- [v12/index](../index.md)
- [versions](../../versions.md)
