---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# A Heuristic to Detect B-Tree Index Bloat in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Statistics that are available](#statistics-that-are-available)
  - [The heuristic](#the-heuristic)
  - [SQL example](#sql-example)
  - [How the thresholds work](#how-the-thresholds-work)
  - [Limits and caveats](#limits-and-caveats)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

based on PostgreSQL's usually available statistics and execution of pgstatindex, propose a heuristic to tell if an index is bloated or not.

## Answer

A practical B-tree bloat heuristic compares the current physical B-tree state from `pgstatindex` with the index's configured `fillfactor` and with usage data from `pg_stat_user_indexes`. The index is flagged as bloated when the live-leaf density is far below the target fillfactor and/or when a large share of the index size is tied up in empty (half-dead) or deleted pages.

### Statistics that are available

The `pgstatindex` function in `contrib/pgstattuple` returns one row per B-tree index with the fields `version`, `tree_level`, `index_size`, `root_block_no`, `internal_pages`, `leaf_pages`, `empty_pages`, `deleted_pages`, `avg_leaf_density`, and `leaf_fragmentation` ([pgstattuple--1.4.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31)). The `regclass` overload is the one SQL callers normally use after the 1.4-to-1.5 update ([pgstattuple--1.4.sql#pgstatindexbyid](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L74)).

`pgstatindex` scans every main-fork block after the metapage. For each live leaf page it adds `pd_special - SizeOfPageHeaderData` to `max_avail` and `PageGetFreeSpace(page)` to `free_space`, then reports `avg_leaf_density = 100 - free_space / max_avail * 100` ([pgstatindex.c#density-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351)). It classifies `P_ISDELETED` pages as `deleted_pages`, `P_IGNORE` (half-dead) pages as `empty_pages`, and the remaining live leaf pages as `leaf_pages` ([pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)). The reported `index_size` is `(1 + internal_pages + leaf_pages + empty_pages + deleted_pages) * BLCKSZ` ([pgstatindex.c#index-size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L336-L342)). `leaf_fragmentation` is `fragments / leaf_pages * 100`, where a fragment is a live leaf whose `btpo_next` points to an earlier block ([pgstatindex.c#fragmentation-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).

`pg_stat_user_indexes` is a built-in view that exposes `idx_scan`, `idx_tup_read`, and `idx_tup_fetch` for user indexes, collected from cumulative statistics ([system_views.sql#pg_stat_user_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L679-L682)). `pg_class` stores the index `reloptions` (which is where `fillfactor` lives) and `relpages` for a cheap size filter ([pg_class.h#relpages-reltuples](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [pg_class.h#reloptions](../../../raw/postgres-12/src/include/catalog/pg_class.h#L133-L134)). The `pg_options_to_table` function turns a `reloptions` array into `(option_name, option_value)` rows ([pg_proc.dat#pg_options_to_table](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3603-L3608)).

### The heuristic

1. Fetch the configured leaf `fillfactor`. The default is 90 and the minimum allowed is 10 for B-tree indexes ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L168-L170), [reloptions.c#btree-fillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186)). If no `fillfactor` option is set, use 90.
2. Run `pgstatindex(index_oid::regclass)` to get `leaf_pages`, `empty_pages`, `deleted_pages`, `avg_leaf_density`, `index_size`, and `leaf_fragmentation`.
3. Estimate how many leaf pages the live data would occupy if packed to the target fillfactor:

```
ideal_leaf_pages = leaf_pages * (avg_leaf_density / fillfactor)
wasted_leaf_pages = leaf_pages - ideal_leaf_pages
wasted_leaf_bytes = wasted_leaf_pages * block_size
```

4. Add the space consumed by pages that are not live leaves at all:

```
wasted_dead_bytes = (empty_pages + deleted_pages) * block_size
```

5. Total recoverable bytes:

```
wasted_bytes = wasted_leaf_bytes + wasted_dead_bytes
bloat_ratio = wasted_bytes / index_size
```

6. Treat an index as a bloat candidate when `bloat_ratio > 0.30` and `index_size > 1 MB`. Add an additional `leaf_fragmentation > 30 %` warning when range scans are expected to suffer from non-sequential leaf ordering. Use `idx_scan` from `pg_stat_user_indexes` to avoid spending `REINDEX` work on an index that is never used.

The `1 MB` minimum and `0.30` ratio are operational guardrails; they are not inferred from the core code. A site with tight storage may want a lower ratio, while a site with abundant sequential I/O may tolerate a higher one.

### SQL example

```sql
SET /* wiki_bloat_timeout_guard */ statement_timeout = '30s';
SET /* wiki_bloat_timeout_guard */ lock_timeout = '5s';

WITH idx AS (
    SELECT
        s.schemaname,
        s.indexrelname,
        s.relname,
        s.idx_scan,
        i.oid AS indexrelid,
        COALESCE(f.fillfactor, 90) AS fillfactor
    FROM pg_stat_user_indexes s
    JOIN pg_class i ON i.oid = s.indexrelid
    JOIN pg_am a ON a.oid = i.relam
    LEFT JOIN LATERAL (
        SELECT option_value::int AS fillfactor
        FROM pg_options_to_table(i.reloptions)
        WHERE option_name = 'fillfactor'
    ) f ON true
    WHERE a.amname = 'btree'
      AND i.relpages > 64
)
SELECT /* wiki_btree_bloat_candidates */
    schemaname,
    indexrelname,
    relname,
    st.index_size,
    st.leaf_pages,
    st.empty_pages,
    st.deleted_pages,
    st.avg_leaf_density,
    st.leaf_fragmentation,
    fillfactor,
    GREATEST(0,
        (st.leaf_pages * current_setting('block_size')::int
         * (1 - (st.avg_leaf_density / fillfactor)))
    )::bigint AS wasted_leaf_bytes,
    ((st.empty_pages + st.deleted_pages)
        * current_setting('block_size')::int) AS wasted_dead_pages_bytes,
    (GREATEST(0,
        (st.leaf_pages * current_setting('block_size')::int
         * (1 - (st.avg_leaf_density / fillfactor)))
     ) + ((st.empty_pages + st.deleted_pages)
        * current_setting('block_size')::int))::bigint AS wasted_bytes,
    round(
        (GREATEST(0,
            (st.leaf_pages * current_setting('block_size')::int
             * (1 - (st.avg_leaf_density / fillfactor)))
         ) + ((st.empty_pages + st.deleted_pages)
            * current_setting('block_size')::int))::numeric
        / NULLIF(st.index_size, 0),
        3
    ) AS bloat_ratio,
    (st.leaf_fragmentation > 30.0) AS high_fragmentation,
    (round(
        (GREATEST(0,
            (st.leaf_pages * current_setting('block_size')::int
             * (1 - (st.avg_leaf_density / fillfactor)))
         ) + ((st.empty_pages + st.deleted_pages)
            * current_setting('block_size')::int))::numeric
        / NULLIF(st.index_size, 0),
        3
     ) > 0.30
     AND st.index_size > 1048576) AS is_bloated
FROM idx
CROSS JOIN LATERAL pgstatindex(idx.indexrelid::regclass) st
ORDER BY wasted_bytes DESC NULLS LAST;
```

The `i.relpages > 64` predicate keeps the expensive `pgstatindex` scan away from tiny indexes where the result is mostly noise. `pg_class.relpages` may be stale, so it is only a pre-filter. The query parses and executes on the pinned PostgreSQL 12 checkout; it was tested against a freshly built B-tree index on an isolated server and returned a `bloat_ratio` near zero for a healthy default-fillfactor index.

### How the thresholds work

`avg_leaf_density` is a percentage of the usable leaf-page space. A newly built default-fillfactor B-tree should be close to 90 % because `nbtsort.c` builds rightmost leaf pages to `BTREE_DEFAULT_FILLFACTOR` full and `nbtsplitloc.c` applies the same fillfactor to rightmost and localized-monotonic splits ([nbtsort.c#rightmost-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L727-L730), [nbtsplitloc.c#leaffillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186)). If `avg_leaf_density` has drifted far below that, the leaves are underfilled; the formula converts that gap into an estimated page count that could be recovered by repacking the live data to the target density.

`empty_pages` and `deleted_pages` are not included in `avg_leaf_density`, so they are counted as fully wasted space. `leaf_fragmentation` measures logical right-to-left jumps in the leaf chain; it is not a space metric, but high fragmentation can hurt range-scan locality. The `bloat_ratio` is therefore the primary space signal, while `leaf_fragmentation` is a secondary I/O signal.

`idx_scan` from `pg_stat_user_indexes` is not part of the bloat ratio. It is used to order the work: an index with high `wasted_bytes` and high `idx_scan` is a better `REINDEX` target than an equally bloated index that is never scanned.

### Limits and caveats

- `pgstatindex` is B-tree only. It rejects non-B-tree indexes at the start of `pgstatindex_impl` ([pgstatindex.c#AM-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228)).
- `pgstatindex` acquires an `AccessShareLock` and scans the whole main fork; on a large index it can be slow and is not an instantaneous snapshot ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- `avg_leaf_density` counts `LP_DEAD` items as occupied space because `PageGetFreeSpace(page)` does not distinguish dead from live line pointers. After `VACUUM` reclaims dead tuples, the free space grows and density drops; this is the intended bloat signal.
- The formula ignores internal-page savings and non-leaf fillfactor (`BTREE_NONLEAF_FILLFACTOR` is 70 %). A bloated tree may also have a smaller-than-expected height after page deletion, so the heuristic deliberately omits internal-page estimates.
- `pg_options_to_table` requires `pg_class` read access. The query is already constrained to `pg_stat_user_indexes`, which shows only user schemas.
- The 30 % bloat ratio, the 1 MB size floor, and the 30 % fragmentation flag are operational choices, not PostgreSQL defaults.

## Context Reviewed

- `contrib/pgstattuple/pgstatindex.c` - `pgstatindex` implementation, page classification, and output calculations.
- `contrib/pgstattuple/pgstattuple--1.4.sql` - SQL function definitions for `pgstatindex` overloads.
- `doc/src/sgml/pgstattuple.sgml` - Documented `pgstatindex` output columns.
- `src/include/access/nbtree.h` - B-tree fillfactor defaults.
- `src/backend/access/common/reloptions.c` - `fillfactor` reloption registration for B-tree.
- `src/backend/catalog/system_views.sql` - Definition of `pg_stat_user_indexes`.
- `src/include/catalog/pg_proc.dat` - `pg_options_to_table` function catalog.
- `src/include/catalog/pg_class.h` - `pg_class` columns (`relpages`, `reloptions`).
- `src/backend/access/nbtree/nbtsort.c` and `nbtsplitloc.c` - How leaf fillfactor is applied during build and split.

## Evidence Map

| Claim | Source |
|---|---|
| `pgstatindex` returns `avg_leaf_density` and `leaf_fragmentation` as `float8` percentages | [pgstattuple--1.4.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31) |
| `avg_leaf_density` formula is `100 - free_space / max_avail * 100` | [pgstatindex.c#density-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351) |
| `leaf_fragmentation` is `fragments / leaf_pages * 100` | [pgstatindex.c#fragmentation-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356) |
| `empty_pages` are `P_IGNORE` (half-dead) and `deleted_pages` are `P_ISDELETED` | [pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310) |
| `index_size` counts metapage + classified pages times `BLCKSZ` | [pgstatindex.c#index-size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L336-L342) |
| B-tree default `fillfactor` is 90 and minimum is 10 | [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L168-L170) |
| `fillfactor` reloption is registered for B-tree with those bounds | [reloptions.c#btree-fillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186) |
| `pg_stat_user_indexes` columns come from `pg_stat_get_*` on `I.oid` | [system_views.sql#pg_stat_user_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L679-L682) |
| `pg_options_to_table` converts a `reloptions` array to name/value rows | [pg_proc.dat#pg_options_to_table](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3603-L3608) |
| `pg_class` carries `relpages` and `reloptions` | [pg_class.h#relpages-reltuples](../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66), [pg_class.h#reloptions](../../../raw/postgres-12/src/include/catalog/pg_class.h#L133-L134) |
| Rightmost leaf pages are built/split to the leaf fillfactor | [nbtsort.c#rightmost-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L727-L730) |

## Open Questions

- The heuristic uses a fixed 30 % bloat ratio and 1 MB minimum. Workload-specific calibration (for example, indexes on random- vs sequential-access tables) is left to the operator.
- The formula does not attempt to estimate internal-page savings from a `REINDEX`. A more aggressive heuristic could compare `tree_level` before and after a trial `REINDEX INDEX CONCURRENTLY`, but that requires a second pass and additional locking.
- `pgstatindex` must read every block. A production deployment may want to call it only for indexes that already look suspicious from `pg_class.relpages` and `pg_statio_user_indexes` read patterns.

## Source References

- [pgstatindex.c](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c)
- [pgstattuple--1.4.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql)
- [pgstattuple.sgml](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml)
- [nbtree.h](../../../raw/postgres-12/src/include/access/nbtree.h)
- [reloptions.c](../../../raw/postgres-12/src/backend/access/common/reloptions.c)
- [system_views.sql](../../../raw/postgres-12/src/backend/catalog/system_views.sql)
- [pg_proc.dat](../../../raw/postgres-12/src/include/catalog/pg_proc.dat)
- [pg_class.h](../../../raw/postgres-12/src/include/catalog/pg_class.h)
- [nbtsort.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c)
- [nbtsplitloc.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c)

## Navigation

- [PostgreSQL 12.2 landing page](../index.md)
- [PostgreSQL 12 Codebase Navigation Guide](../codebase-navigation-guide.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12](./how-pgstatindex-calculates-information.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12](./bloated-indexes-query-planner.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12](./pgstatindex-sample-variant-proposal.md)
