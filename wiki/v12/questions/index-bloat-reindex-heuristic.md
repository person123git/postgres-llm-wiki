---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12: based on PostgreSQL's usually available statistics and
execution of pgstatindex, propose a heuristic to find bloated indexes and
prioritize them for reindexing.

## Answer

Use a three-stage funnel. Stage 1 shortlists candidate B-tree indexes from
statistics that PostgreSQL 12 always maintains (`pg_class`, the cumulative
`pg_stat_*` / `pg_statio_*` views, and `pg_relation_size()`); this stage reads
no index pages. Stage 2 runs `pgstatindex()` only on the shortlist; this is
the expensive, exact step because `pgstatindex` physically reads every
non-metapage block of the index
([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
Stage 3 ranks confirmed candidates by estimated wasted bytes times scan
frequency, and executes `REINDEX` — normally `REINDEX INDEX CONCURRENTLY`,
which PostgreSQL 12 supports
([ref/reindex.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L153-L171)).

The split exists because the two evidence sources have opposite cost/accuracy
profiles:

| Source | Cost | What it tells you |
|---|---|---|
| `pg_class.relpages` / `reltuples`, `pg_stat_*` views, `pg_relation_size()` | Catalog lookups, collector counters, and per-segment-file `stat()` calls; no index page reads ([dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)) | Size, growth, churn, and usage. Cannot directly measure leaf density. |
| `pgstatindex()` | Reads every block of the index from block 1 to the end ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)) | Exact `avg_leaf_density`, `leaf_fragmentation`, page-class counts ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)). |

Bloat here means what the v12 manual calls a bloated index: many empty or
nearly-empty pages that `REINDEX` can remove by writing a fresh copy
([ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)).
VACUUM cannot repair low-density pages: B-tree page deletion applies only to
completely empty pages, and partly-full pages are never merged
([nbtree-README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L200-L212),
[maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874)).

### Prerequisites

- The `pgstattuple` extension (default version 1.5) must be installed;
  `pgstatindex` execution is granted to `pg_stat_scan_tables` and revoked from
  `PUBLIC`, and superusers bypass the check
  ([pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5),
  [pgstattuple--1.4--1.5.sql#grants](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
  [pgstattuple.sgml#access](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L24)).
- The cumulative counters used in Stage 1 require `track_counts`, which
  defaults to on (`PGC_SUSET`, so changing it is session-scoped for superusers
  or a config change)
  ([guc.c#track_counts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1400)).
- Counters are cumulative since the last stats reset and lag live activity:
  backends flush to the collector when going idle, and the collector
  republishes at most every 500 ms
  ([monitoring.sgml#stats-lag](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L240)).
  For rates (scans/day, churn/day), snapshot the views twice and subtract.

### Stage 1 - Shortlist From Always-Available Statistics

Scope first, then score churn. The scope filter keeps only indexes that
`pgstatindex` can inspect and that are worth a full read:

- B-tree only: `pgstatindex` raises an error for any non-B-tree relation
  ([pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238),
  [pg_am.dat#btree](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L20)).
- Valid indexes only (`pg_index.indisvalid`); invalid leftovers such as a
  failed concurrent rebuild's `*_ccnew` index are a DROP-and-retry case, not a
  REINDEX-priority case
  ([pg_index.h#indisvalid](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L40),
  [ref/reindex.sgml#failed-concurrent](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)).
- Skip temporary relations; `pgstatindex` rejects other sessions' temp
  relations outright
  ([pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238),
  [pg_class.h#relpersistence](../../../raw/postgres-12/src/include/catalog/pg_class.h#L49-L81)).
- Apply a size floor (the example below uses 64 MB) using
  `pg_relation_size()`, which sums `stat()` file sizes per segment and reads
  no blocks
  ([dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308),
  [pg_proc.dat#pg_relation_size-1arg](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6888)).

Then read these signals:

| Signal | Where | Why it indicates bloat risk or priority |
|---|---|---|
| Non-HOT update volume `n_tup_upd - n_tup_hot_upd`, plus `n_tup_del` | `pg_stat_user_tables` ([system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)) | Only HOT updates avoid new index entries; the docs define `n_tup_hot_upd` as rows updated "with no separate index update required" ([monitoring.sgml#n_tup_hot_upd](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2766-L2771), [README.HOT:6](../../../raw/postgres-12/src/backend/access/heap/README.HOT#L6)). High non-HOT churn means many dead index entries get created and later removed, the pattern that strands sparse leaf pages. |
| `n_dead_tup`, `last_vacuum`, `last_autovacuum` | `pg_stat_user_tables` ([monitoring.sgml#n_dead_tup](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2777-L2792)) | Tables with chronic dead-tuple backlogs feed index churn; recent VACUUM activity tells you whether index entry removal already happened. |
| `idx_scan` per index | `pg_stat_user_indexes` ([system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L682)) | Priority weight: rebuilding an unscanned index buys nothing. `idx_scan` counts scan starts ([monitoring.sgml#idx_scan](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2884-L2899)). |
| `idx_blks_read` vs `idx_blks_hit` | `pg_statio_user_indexes` ([system_views.sql#pg_statio_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L684-L708), [monitoring.sgml#idx_blks](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3024-L3073)) | A bloated index that also misses cache is paying real disk reads; that raises its priority over an equally bloated but fully cached one. |
| Bytes per live row: `pg_relation_size(index) / reltuples` | `pg_class` + `pg_relation_size()` ([catalogs.sgml#reltuples](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1772-L1782)) | Trend signal. Capture it after each rebuild as a baseline; growth at stable row counts means falling density. |

Two `idx_scan` interpretation rules matter for prioritization:

- Bitmap index scans do count: `btgetbitmap` starts its scan through
  `_bt_first`, and `_bt_first` is where the B-tree increments the counter via
  `pgstat_count_index_scan`
  ([nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335),
  [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L745-L768),
  [pgstat.h#pgstat_count_index_scan](../../../raw/postgres-12/src/include/pgstat.h#L1374-L1378)).
- `idx_scan = 0` does not prove the index is useless. Insert-time uniqueness
  enforcement descends the tree through `_bt_search` and `_bt_check_unique`,
  not `_bt_first`, so it never increments `idx_scan`
  ([nbtinsert.c#_bt_doinsert](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L213-L260),
  [nbtinsert.c#_bt_check_unique](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L320-L343)).
  Check `pg_index.indisunique` before treating a zero-scan index as a drop
  candidate instead of a reindex candidate
  ([pg_index.h#indisunique](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L40)).

Freshness caveat for `pg_class`: index `relpages`/`reltuples` are estimates
updated by VACUUM, ANALYZE, and index-creating DDL
([catalogs.sgml#relpages](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1782),
[vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1117-L1191),
[index.c#index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680)).
VACUUM can skip the index cleanup pass entirely when
`_bt_vacuum_needs_cleanup` decides nothing requires it; `btvacuumcleanup` then
returns `NULL` and `lazy_cleanup_index` returns before calling
`vac_update_relstats`, leaving the index's `pg_class` row stale
([nbtree.c#_bt_vacuum_needs_cleanup](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L783-L846),
[nbtree.c#btvacuumcleanup-skip](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L942),
[vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1771-L1830)).
The skip threshold is `vacuum_cleanup_index_scale_factor` (default 0.1,
`PGC_USERSET`, so session/transaction scope via `SET`; also settable per index
as a storage parameter)
([guc.c#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3424-L3432),
[reloptions.c#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1417-L1418),
[config.sgml#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7852-L7891)).
A standalone ANALYZE (not VACUUM ANALYZE) refreshes index `relpages` from the
live block count and estimates index `reltuples`, so scheduled ANALYZE keeps
Stage 1 inputs usable
([analyze.c#index-relstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629),
[bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199)).
Prefer `pg_relation_size()` over `relpages` for current size; it reads the
file system directly.

Shortlist query, verified against the pinned v12 catalogs and views. Both
timeout GUCs are `PGC_USERSET`, so plain `SET` gives them session scope
([guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2386),
[guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2389-L2397)):

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

SELECT /* wiki_index_bloat_shortlist_v12 */
       n.nspname                                AS schema_name,
       ct.relname                               AS table_name,
       ci.relname                               AS index_name,
       pg_relation_size(ci.oid)                 AS index_bytes,
       ci.reltuples                             AS index_reltuples,
       pg_relation_size(ci.oid)
         / GREATEST(ci.reltuples, 1)            AS bytes_per_row,
       s.idx_scan,
       io.idx_blks_read,
       io.idx_blks_hit,
       t.n_tup_upd - t.n_tup_hot_upd            AS non_hot_updates,
       t.n_tup_del,
       t.n_dead_tup,
       t.last_vacuum,
       t.last_autovacuum
FROM pg_index x
JOIN pg_class ci            ON ci.oid = x.indexrelid
JOIN pg_class ct            ON ct.oid = x.indrelid
JOIN pg_namespace n         ON n.oid = ci.relnamespace
JOIN pg_am am               ON am.oid = ci.relam AND am.amname = 'btree'
JOIN pg_stat_user_indexes s ON s.indexrelid = ci.oid
JOIN pg_statio_user_indexes io ON io.indexrelid = ci.oid
JOIN pg_stat_user_tables t  ON t.relid = ct.oid
WHERE x.indisvalid
  AND ci.relpersistence <> 't'
  AND pg_relation_size(ci.oid) >= 64 * 1024 * 1024
ORDER BY (t.n_tup_upd - t.n_tup_hot_upd) + t.n_tup_del DESC,
         pg_relation_size(ci.oid) DESC;
```

Optional absolute estimator (proposal, not a PostgreSQL formula): expected
leaf pages for a compact index are roughly
`reltuples * entry_bytes / (leaf_capacity * fillfactor/100)`, where
`entry_bytes ~= MAXALIGN(8-byte IndexTupleData header + key data) + 4-byte
line pointer`. The components are citable — the index tuple header is a
6-byte `ItemPointerData` plus a 2-byte `t_info`, attribute data starts at a
MAXALIGN boundary, each item costs one 4-byte `ItemIdData`, and key widths can
be approximated from `pg_statistic.stawidth` for plain-column keys
([itup.h#IndexTupleData](../../../raw/postgres-12/src/include/access/itup.h#L25-L53),
[itemptr.h#ItemPointerData](../../../raw/postgres-12/src/include/storage/itemptr.h#L22-L47),
[itemid.h#ItemIdData](../../../raw/postgres-12/src/include/storage/itemid.h#L17-L32),
[pg_statistic.h#stawidth](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L39-L49)).
Treat the result only as a shortlist hint: expression indexes have no
`pg_statistic` rows to read widths from, and Stage 2 measures density exactly
anyway.

### Stage 2 - Confirm With pgstatindex

What Stage 2 costs and locks, so you can schedule it:

- `pgstatindex` opens the index with `AccessShareLock` and reads every block
  from 1 to `RelationGetNumberOfBlocks() - 1` — I/O proportional to the full
  current index size
  ([pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212),
  [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- It reads through a `BAS_BULKREAD` strategy ring of `256 kB / BLCKSZ`
  buffers, so it does not flood `shared_buffers`, but the disk reads are real
  ([pgstatindex.c#bulkread](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L218-L223),
  [freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L547-L576)).
- Results accumulate page by page and are not a single snapshot; concurrent
  changes can shift the numbers
  ([pgstattuple.sgml#non-snapshot](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L278)).

Interpret the output against the rebuild target, not against 100%:

| Output | Reading it |
|---|---|
| `avg_leaf_density` | Compare to the index's fillfactor (default 90; a sorted rebuild fills leaves to exactly that target). Density near fillfactor means REINDEX gains nothing. Below ~70% is a watch candidate, below ~50-60% a strong one — operational bands, not PostgreSQL-defined cutoffs ([nbtsort.c#leaf-fill](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L713-L734), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)). |
| `deleted_pages`, `empty_pages` | Space VACUUM has reclaimed (or half-reclaimed) but the file still holds. Under continued inserts these pages get recycled through the FSM (`RecordFreeIndexPage` at VACUUM, reuse at page split), so a large deleted count on a still-growing index may self-absorb without REINDEX ([nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1177), [nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)). On a shrinking or static index they are dead weight only REINDEX removes. |
| `leaf_fragmentation` | Physical-order signal: percentage of live leaf pages whose right sibling sits at a lower block number ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)). It matters for large range/ordered scans on cold, seek-sensitive storage; a fresh rebuild lays logically adjacent pages physically adjacent ([nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12), [maintenance.sgml#adjacency](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888)). Do not let it alone trigger REINDEX. |
| `tree_level` | A bloated index can carry an extra level; the planner charges B-tree descent per level, so a level drop after rebuild is a real plan-cost gain ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)). |
| `NaN` edge | `avg_leaf_density` and `leaf_fragmentation` are `NaN` for an index with no counted leaf capacity/pages (e.g., empty index); filter `leaf_pages > 0` ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356), [pgstattuple.out#empty-btree-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)). |

Combined Stage 2 + scoring query. It runs `pgstatindex` once per surviving
index, so the generous `statement_timeout` is deliberate; tighten the size
floor or add `LIMIT` to pace it:

```sql
SET statement_timeout = '30min';
SET lock_timeout = '2s';

SELECT /* wiki_index_bloat_pgstatindex_v12 */
       n.nspname                          AS schema_name,
       ci.relname                         AS index_name,
       s.idx_scan,
       ps.index_size,
       ps.tree_level,
       ps.leaf_pages,
       ps.deleted_pages,
       ps.empty_pages,
       ps.avg_leaf_density,
       ps.leaf_fragmentation,
       (ps.index_size
          * GREATEST(90.0 - ps.avg_leaf_density, 0) / 90.0)::bigint
                                          AS est_wasted_bytes,
       round(((ps.index_size
          * GREATEST(90.0 - ps.avg_leaf_density, 0) / 90.0)
          * ln(1 + s.idx_scan))::numeric) AS reindex_priority
FROM pg_index x
JOIN pg_class ci            ON ci.oid = x.indexrelid
JOIN pg_namespace n         ON n.oid = ci.relnamespace
JOIN pg_am am               ON am.oid = ci.relam AND am.amname = 'btree'
JOIN pg_stat_user_indexes s ON s.indexrelid = ci.oid
CROSS JOIN LATERAL pgstatindex(ci.oid::regclass) ps
WHERE x.indisvalid
  AND ci.relpersistence <> 't'
  AND pg_relation_size(ci.oid) >= 64 * 1024 * 1024
  AND ps.leaf_pages > 0
ORDER BY reindex_priority DESC;
```

Replace the `90.0` constant with the index's configured fillfactor when one
is set (`pg_class.reloptions`); the default target is
`BTREE_DEFAULT_FILLFACTOR = 90`
([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171)).

### Stage 3 - Priority Queue and Execution

The score `est_wasted_bytes * ln(1 + idx_scan)` encodes the proposal: fix the
biggest waste on the most-used indexes first. `est_wasted_bytes` approximates
the bytes a rebuild to fillfactor would return; `ln(1 + idx_scan)` damps raw
scan counts so one hot index does not drown every other signal. The weights
are operational choices, not PostgreSQL-derived constants. Apply these
modifiers before executing:

- Boost an index whose `tree_level` exceeds what its live-entry count needs,
  and broad range/ordered/bitmap scan workloads; both effects are visible to
  the v12 planner through `index->pages` and `tree_height`, as detailed in
  [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
  ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417),
  [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815)).
- Boost when `idx_blks_read` stays high — the bloat is costing uncached disk
  reads, not just cache footprint
  ([monitoring.sgml#idx_blks](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3024-L3073)).
- Demote insert-mostly growing indexes whose Stage 2 signal is mainly
  `deleted_pages`: FSM recycling will reuse those pages for future splits
  ([nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).
- Demote fragmentation-only candidates unless their workload is cold-storage
  range scans; the planner never reads `leaf_fragmentation`
  ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)).
- A zero-scan, non-unique, non-constraint index is a DROP-evaluation
  candidate, not a REINDEX candidate
  ([pg_index.h#indisunique](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L40)).

Execution in PostgreSQL 12:

- Plain `REINDEX INDEX` locks the parent table in `ShareLock` (writes blocked,
  reads allowed) and the index itself in `AccessExclusiveLock` (scans using
  that index blocked), then rebuilds into a new relfilenode via the sorted
  B-tree build
  ([indexcmds.c#ReindexIndex-lock](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2344-L2360),
  [index.c#reindex_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470),
  [index.c#reindex-rebuild](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530),
  [ref/reindex.sgml#locking](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L231-L243)).
- `REINDEX INDEX CONCURRENTLY` (new surface in v12) works under
  `ShareUpdateExclusiveLock`, costs roughly two table scans plus wait phases,
  and is the production default per the docs
  ([indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2714-L2737),
  [ref/reindex.sgml#concurrent-cost](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L283-L297),
  [ref/reindex.sgml#concurrent-steps](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L299-L361),
  [maintenance.sgml#reindex-locks](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L890-L896)).
- Concurrent rebuild exclusions: system catalogs error out
  (`cannot reindex system catalogs concurrently`), exclusion-constraint
  indexes are skipped or error, and it cannot run in a transaction block
  ([indexcmds.c#catalog-error](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2804-L2807),
  [ref/reindex.sgml#system-and-exclusion](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L392-L414)).
- A failed concurrent run leaves an invalid `*_ccnew` index that still takes
  update overhead; detect via `pg_index.indisvalid`, drop it, retry
  ([ref/reindex.sgml#failed-concurrent](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)).
- Monitor the rebuild in `pg_stat_progress_create_index`, which reports
  REINDEX as well as CREATE INDEX
  ([monitoring.sgml#create-index-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3488-L3497),
  [index.c#reindex-progress](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470),
  [system_views.sql:994](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L994)).

After the rebuild: leaf density returns to the fillfactor target and pages
come out physically sequential
([nbtsort.c#leaf-fill](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L713-L734),
[nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12)).
`REINDEX` also applies any storage-parameter change, so chronically rebloating
indexes can be rebuilt with a lower fillfactor in the same step
([ref/reindex.sgml#storage-param](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L59-L64)).
Record the new `pg_relation_size / reltuples` baseline immediately; Stage 1's
drift signal measures against it. `index_update_stats` refreshes the index's
`pg_class` row as part of the rebuild
([index.c#index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680)).

### Measuring the Improvement After a REINDEX

Capture a baseline before the rebuild, re-capture afterward, and compare on
three layers: physical shape, per-index runtime counters, and query-level
timing. A v12-specific behavior makes the counter layer work across both
REINDEX forms: per-index cumulative counters survive the rebuild, so
before/after deltas stay valid.

- Plain `REINDEX` rebuilds storage for the same index relation — it opens the
  existing index and assigns a new relfilenode — so the index keeps its OID
  and its counters simply continue
  ([index.c#reindex_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470),
  [index.c#reindex-rebuild](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530)).
- `REINDEX CONCURRENTLY` builds a separate new index relation and swaps it in,
  but `index_concurrently_swap` copies the old index's collector entry —
  `numscans`, `tuples_returned`, `tuples_fetched`, `blocks_fetched`,
  `blocks_hit` — into the new relation's pending counts, which reach the
  collector at the next `pgstat_report_stat()` call. A readout taken in the
  instant after the swap can therefore briefly show missing counters until
  that flush happens
  ([index.c#index_concurrently_create_copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1233-L1241),
  [index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705)).

**Layer 1 — physical shape (immediate, run once).** Compare
`pg_relation_size()` before and after; the difference is the reclaimed disk
space ([dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)).
One post-rebuild `pgstatindex` run verifies the shape: `avg_leaf_density` at
the fillfactor target, `deleted_pages`/`empty_pages` at zero,
`leaf_fragmentation` near zero from the sequential sorted build, and possibly
a lower `tree_level`
([nbtsort.c#leaf-fill](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L713-L734),
[nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12),
[pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)).
This run costs a full read of the now-smaller index, so do it once and store
the row as the new Stage 1 baseline. The rebuild also refreshed the index's
`pg_class` row, so re-running plain `EXPLAIN` (no `ANALYZE`) on affected
queries shows the planner-side gain at once: estimated costs fall through the
smaller `index->pages` and, when a level dropped, the lower `tree_height`
descent charge
([index.c#index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680),
[plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417),
[selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)).

**Layer 2 — per-index counters (steady state).** Snapshot the counter row
before the rebuild and again over a comparable workload window afterward:

```sql
SET statement_timeout = '30s';
SET lock_timeout = '2s';

SELECT /* wiki_index_reindex_baseline_v12 */
       now()                          AS captured_at,
       s.schemaname,
       s.indexrelname,
       pg_relation_size(s.indexrelid) AS index_bytes,
       s.idx_scan,
       s.idx_tup_read,
       io.idx_blks_read,
       io.idx_blks_hit
FROM pg_stat_user_indexes s
JOIN pg_statio_user_indexes io USING (indexrelid)
WHERE s.indexrelid = 'my_schema.my_index'::regclass;
```

Compare window deltas, not raw cumulative values:

```text
space_saving    = 1 - index_bytes_after / index_bytes_before
blocks_per_scan = delta(idx_blks_read + idx_blks_hit) / delta(idx_scan)
miss_per_scan   = delta(idx_blks_read) / delta(idx_scan)
tuples_per_scan = delta(idx_tup_read) / delta(idx_scan)
```

For an unchanged workload, `tuples_per_scan` should stay flat — the same
queries still return the same entries — while `blocks_per_scan` falls because
denser leaves pack more entries per page. That pairing separates a real
density win from a workload shift. The counters carry the documented
semantics: `idx_scan` counts scan starts, `idx_tup_read` counts index entries
returned, and `idx_blks_read`/`idx_blks_hit` count block reads and buffer
hits
([monitoring.sgml#index-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2884-L2941),
[monitoring.sgml#idx_blks](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3024-L3073)).
Remember the collector lag — backends flush when idle and the collector
republishes at most every 500 ms — when timing the snapshots
([monitoring.sgml#stats-lag](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L240)).

**Layer 3 — query level.** For the queries that motivated the rebuild,
compare `EXPLAIN (ANALYZE, BUFFERS)` output before and after: shared
hit/read counts on the index-scan nodes shrink with the page count, though
v12 does not split buffer counters into index versus heap pages — see
[EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](explain-analyze-buffers-output.md)
([explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978)).
For workload-wide measurement, snapshot `pg_stat_statements` (v12 ships
extension version 1.7) per `queryid` and compare `calls`, `total_time`,
`mean_time`, and `shared_blks_read`/`shared_blks_hit` deltas across equal
windows; version 1.7's `pg_stat_statements_reset(userid, dbid, queryid)` can
clear just the affected statements to start a clean after-window
([pg_stat_statements.control](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.control#L1-L5),
[pg_stat_statements--1.4.sql#view](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.4.sql#L12-L43),
[pg_stat_statements--1.6--1.7.sql#selective-reset](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.6--1.7.sql#L13-L22)).
See [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](pg-stat-statements.md)
for setup. Enabling `track_io_timing` adds `blk_read_time`/`blk_write_time`
to `pg_stat_statements` and I/O timings to `EXPLAIN (BUFFERS)`; it is
`PGC_SUSET`, so superusers can `SET` it with session scope, or set it in
`postgresql.conf` and reload
([guc.c#track_io_timing](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1402-L1409)).

Measurement caveats: the rebuilt index starts cold, so the first
post-rebuild scans show elevated `idx_blks_read` and I/O time — judge the
steady-state window, not the first minutes — and all cumulative counters
measure since the last stats reset, so always difference two snapshots over
matching workload periods
([monitoring.sgml#stats-lag](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L240)).

### Why leaf_fragmentation Is Not in the Priority Score

This is a deliberate design decision, not an omission. Stage 2 still
collects `leaf_fragmentation`, and Stage 3 keeps it as a workload-conditional
modifier — but the ranking formula uses only density-derived wasted bytes,
and fragmentation alone never triggers a REINDEX. Five reasons, each grounded
in the pinned v12 source:

**1. The planner cannot reward fixing it.** The v12 cost model sees an index
through `index->pages` and `tree_height`; neither input changes when the same
pages are rearranged into sequential order
([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417)).
Stronger than that: `genericcostestimate()` prices the estimated index page
fetches at the tablespace's `random_page_cost` — the model already assumes
every index page fetch is non-sequential. A fragmented index is not charged
extra, and a freshly defragmented one earns no sequential-access discount;
physical adjacency is absent from the model in both directions
([selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815),
[config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765)).
A density-driven rebuild, by contrast, shrinks `index->pages` (and sometimes
`tree_height`), which the planner does see
([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116));
the full mechanism is in
[Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md).

**2. It has no byte value, so it cannot enter a wasted-bytes score without an
arbitrary conversion.** A density shortfall converts directly into pages and
bytes — that is what `est_wasted_bytes` computes. Fragmentation describes the
physical order of the same pages: an index can be 100% fragmented with
exactly the same `index_size`, `leaf_pages`, and `avg_leaf_density` as an
unfragmented twin, because the metric counts backward sibling links, not
space
([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307),
[pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)).
Mixing a dimensionless order percentage into a bytes-times-usage score would
need a made-up exchange rate; the score stays honest by excluding it.

**3. The metric is too coarse to predict elapsed time.** The counter
increments once per live leaf page whose `btpo_next` names a lower block
number — a per-page yes/no
([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307)).
It encodes no jump distance, no run length, no operating-system read-ahead
behavior, and no cache residency. Two indexes both reporting 50% can behave
completely differently at runtime, so a score term built on it would rank
noise.

**4. Its runtime cost is narrow and workload-conditional, and the docs rate
it small.** The executor pays for fragmentation only when a scan walks many
leaves: `_bt_steppage()` hands the right-link block number to
`_bt_readnextpage()`, which reads exactly the named block via `_bt_getbuf()`
— wherever it sits on disk
([nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1630),
[nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1771),
[nbtpage.c#_bt_getbuf](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).
That only turns into wall-clock time on cold, seek-sensitive storage during
wide range or ordered scans; point lookups touch about one leaf, and pages
already in shared buffers cost a hit regardless of their disk position. The
v12 manual's own claim for fresh physical adjacency is correspondingly
modest: a newly built index is "slightly faster to access"
([maintenance.sgml#adjacency](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888)).
The docs' definition of REINDEX-worthy bloat is space-based — "many empty or
nearly-empty pages" — with adjacency mentioned only as a secondary
speed note
([ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)).
See
[B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
for the full I/O comparison.

**5. Density-triggered rebuilds repair fragmentation for free, and
fragmentation-only repairs do not last.** The sorted rebuild that restores
fillfactor density also writes the leaf level out sequentially, so every
REINDEX this heuristic schedules resets `leaf_fragmentation` toward zero as a
byproduct
([nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12),
[nbtsort.c#leaf-fill](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L713-L734)).
The two symptoms also share one cause: a page split acquires its new right
page with `_bt_getbuf(P_NEW)`, which asks the free space map for a recycled
page before extending the file, so a logical neighbor lands at an arbitrary
older block while the split halves sit half-full
([nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1403-L1440),
[nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).
On a churning index a fragmentation-only rebuild therefore decays again
through the same split-and-reuse process, while the density component of the
same decay is what the score already tracks in bytes.

When to override the default: elevate fragmentation to a boost (as Stage 3
allows) when the workload is dominated by wide range or ordered scans, the
data is cold on seek-sensitive storage, density sits near fillfactor yet
scan latency regressed, and Layer 3 measurements (`blk_read_time`, buffer
reads on index-scan nodes) point at index-side I/O. Measure the result with
wall-clock and I/O-timing comparisons, not plan costs — per reason 1, the
planner will not register the fix.

### Scope Limits

The whole heuristic is B-tree-specific because `pgstatindex` is
([pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238)).
For non-B-tree indexes the v12 manual offers only the coarse rule: bloat there
is not well researched, so monitor physical size periodically
([maintenance.sgml#non-btree](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880));
Stage 1 of this page (minus the density confirmation) is the available
tooling.

### Tests and Coverage

- The v12 `pgstattuple` regression suite exercises `pgstatindex` entry points,
  the empty-index `NaN` output, and wrong-AM/relkind errors, but no populated
  B-tree density or fragmentation assertions
  ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113),
  [pgstattuple.out#empty-btree-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).
- The core `stats` regression test asserts that `seq_scan`/`idx_scan`-style
  counters advance through the collector
  ([stats.sql#counter-checks](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L19-L56),
  [stats.sql#counter-asserts](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L159-L166)).
- `create_index` covers `REINDEX INDEX/TABLE CONCURRENTLY` success, warning,
  and exclusion-constraint error paths
  ([create_index.sql#reindex-concurrently](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L793-L832)).
- No regression test ties `pgstatindex` output to a REINDEX decision; the
  thresholds and score in this page have no in-tree test anchor.

## Context Reviewed

- Required wiki navigation: [versions](../../versions.md),
  [wiki index](../../index.md), [v12/index](../index.md), and recent
  [log](../../log.md).
- `pgstattuple` extension surface and worker: `pgstatindex.c`,
  `pgstattuple--1.4--1.5.sql`, `pgstattuple.control`, `pgstattuple.sgml`,
  regression `pgstattuple.sql` / `pgstattuple.out`.
- Cumulative statistics path: `system_views.sql` view definitions,
  `pgstat.h` counting macros, `_bt_first` call site in `nbtsearch.c`,
  `_bt_doinsert`/`_bt_check_unique` in `nbtinsert.c` (non-counting path),
  `monitoring.sgml` field semantics and lag note, `track_counts` in `guc.c`.
- `pg_class` freshness: `vac_update_relstats` (`vacuum.c`), index branch of
  ANALYZE (`analyze.c`), `index_update_stats` (`index.c`),
  `_bt_vacuum_needs_cleanup` / `btvacuumcleanup` skip path (`nbtree.c`),
  `lazy_cleanup_index` (`vacuumlazy.c`),
  `vacuum_cleanup_index_scale_factor` GUC/reloption/doc.
- Size probes: `calculate_relation_size` / `pg_relation_size` (`dbsize.c`),
  one-argument wrapper in `pg_proc.dat`,
  `RelationGetNumberOfBlocks` (`bufmgr.h`).
- B-tree bloat semantics: nbtree `README` page-deletion section,
  `btvacuumpage` FSM recording and `_bt_getbuf` reuse, `nbtsort.c` build
  density and sequential layout, `BTREE_DEFAULT_FILLFACTOR`.
- REINDEX paths: `ReindexIndex` and `ReindexRelationConcurrently`
  (`indexcmds.c`), `reindex_index` (`index.c`), `ref/reindex.sgml`,
  `maintenance.sgml` routine-reindex, progress view wiring.
- Estimator components: `itup.h`, `itemptr.h`, `itemid.h`,
  `pg_statistic.h#stawidth`.
- Post-REINDEX measurement: `index_concurrently_create_copy` and the
  statistics copy in `index_concurrently_swap` (`index.c`),
  `pg_stat_statements` control and extension scripts, `track_io_timing`
  (`guc.c`), and `show_buffer_usage` (`explain.c`).
- Fragmentation-exclusion rationale: the fragmentation counter in
  `pgstatindex.c`, the tablespace page-cost fetch inside
  `genericcostestimate` (`selfuncs.c`), page-cost constants (`config.sgml`),
  the split right-page allocation (`nbtinsert.c`), and the leaf-chain walk
  (`_bt_steppage` / `_bt_readnextpage` in `nbtsearch.c`).

## Evidence Map

| Claim | Evidence |
|---|---|
| Stage 1 inputs cost no index page reads; `pg_relation_size` is per-segment `stat()` | [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336) |
| `pgstatindex` reads every non-metapage block under `AccessShareLock` with a 256 kB `BAS_BULKREAD` ring | [pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212), [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [pgstatindex.c#bulkread](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L218-L223), [freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L547-L576) |
| `pgstatindex` is B-tree-only and rejects other-session temp relations | [pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238) |
| `idx_scan` increments at B-tree scan start (`_bt_first`), including bitmap scans; insert-time uniqueness checks bypass it | [pgstat.h#pgstat_count_index_scan](../../../raw/postgres-12/src/include/pgstat.h#L1374-L1378), [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L745-L768), [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335), [nbtinsert.c#_bt_doinsert](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L213-L260) |
| HOT updates create no new index entries, so `n_tup_upd - n_tup_hot_upd` plus `n_tup_del` proxies index churn | [monitoring.sgml#n_tup_hot_upd](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2766-L2771), [README.HOT:6](../../../raw/postgres-12/src/backend/access/heap/README.HOT#L6) |
| Index `relpages`/`reltuples` are estimates refreshed by VACUUM/ANALYZE/DDL and can stale-skip via the cleanup scale factor | [catalogs.sgml#relpages](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1782), [nbtree.c#btvacuumcleanup-skip](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L942), [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1771-L1830), [analyze.c#index-relstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629), [config.sgml#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7852-L7891) |
| Stats views lag activity (per-process flush, 500 ms republish) and need `track_counts` | [monitoring.sgml#stats-lag](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L240), [guc.c#track_counts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1400) |
| VACUUM deletes only completely empty B-tree pages and never merges partly-full pages; reclaimed pages recycle through the FSM | [nbtree-README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L200-L212), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1177), [nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820) |
| A sorted rebuild fills leaves to fillfactor (default 90) and lays pages sequentially | [nbtsort.c#leaf-fill](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L713-L734), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171), [nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12), [maintenance.sgml#adjacency](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888) |
| Plain REINDEX: `ShareLock` table + `AccessExclusiveLock` index, new relfilenode; CONCURRENTLY: `ShareUpdateExclusiveLock`, two scans, multi-phase | [index.c#reindex_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470), [indexcmds.c#ReindexIndex-lock](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2344-L2360), [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2714-L2737), [ref/reindex.sgml#concurrent-steps](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L299-L361) |
| Concurrent REINDEX excludes system catalogs and exclusion-constraint indexes; failures leave invalid `*_ccnew` indexes | [indexcmds.c#catalog-error](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2804-L2807), [ref/reindex.sgml#system-and-exclusion](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L392-L414), [ref/reindex.sgml#failed-concurrent](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390) |
| REINDEX progress is visible in `pg_stat_progress_create_index` | [index.c#reindex-progress](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470), [monitoring.sgml#create-index-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3488-L3497) |
| Planner-side payoff of REINDEX flows through `index->pages` and `tree_height`, not `pgstatindex` metrics | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116) |
| Per-index counters survive both REINDEX forms: plain REINDEX keeps the same index relation (new relfilenode only); CONCURRENTLY builds a new index relation and `index_concurrently_swap` copies `numscans`, tuple, and block counters into its pending stats, flushed at the next `pgstat_report_stat()` | [index.c#reindex-rebuild](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530), [index.c#index_concurrently_create_copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1233-L1241), [index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705) |
| `pg_stat_statements` 1.7 exposes per-query `calls`, timing, and shared-buffer columns plus selective reset for clean before/after windows | [pg_stat_statements.control](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.control#L1-L5), [pg_stat_statements--1.4.sql#view](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.4.sql#L12-L43), [pg_stat_statements--1.6--1.7.sql#selective-reset](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.6--1.7.sql#L13-L22) |
| `track_io_timing` (`PGC_SUSET`) enables block I/O timing in `pg_stat_statements` and `EXPLAIN (BUFFERS)`; v12 buffer counters are node-level, not split index-vs-heap | [guc.c#track_io_timing](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1402-L1409), [pg_stat_statements--1.4.sql#view](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.4.sql#L12-L43), [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978) |
| `genericcostestimate()` prices estimated index page fetches at the tablespace `random_page_cost`, with no physical-adjacency input, so fragmentation can neither raise nor lower estimated cost | [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765), [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417) |
| `leaf_fragmentation` increments once per live leaf page whose right link points to a lower block number; it carries no jump-distance, run-length, or cache information and is independent of `index_size` and density | [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356) |
| Page splits acquire the new right page via `_bt_getbuf(P_NEW)` (FSM first), so churn co-produces low density and fragmentation; the sorted rebuild resets both together | [nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1403-L1440), [nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820), [nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12) |

## Open Questions

- PostgreSQL 12 defines no `avg_leaf_density` or `leaf_fragmentation`
  threshold for REINDEX; the 70% / 50-60% bands, the 64 MB size floor, and the
  `wasted_bytes * ln(1 + idx_scan)` score are operational proposals layered on
  cited mechanics, consistent with the bands in
  [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
  ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L897)).
- The optional expected-leaf-pages estimator is approximate by construction:
  `stawidth` covers stored column data only, expression indexes lack
  `pg_statistic` rows, and v12 stores one leaf tuple per indexed heap row
  version with no deduplication, so real entry counts track row versions, not
  distinct keys. It has not been validated against live data here
  ([pg_statistic.h#stawidth](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L39-L49)).
- `idx_scan` is cumulative since the last stats reset and can also be bumped
  by planner range probes against stale histogram endpoints, so
  zero-vs-nonzero is reliable but small magnitudes are noisy
  ([monitoring.sgml#optimizer-access](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2904-L2929)).
- The v12 `empty_pages` column counts half-dead pages, not zero-item pages;
  see the classification quirk documented in
  [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md).
- The `index_concurrently_swap` statistics copy reads the old index's entry
  from the statistics collector (`pgstat_fetch_stat_tabentry`) at swap time.
  Counter increments still pending in other backends at that moment would be
  reported against the old, dropped index OID and so appear lost to the new
  index — an inference from the copy source, not a behavior the source or
  docs state explicitly
  ([index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705)).
- There is no v12-derivable threshold at which a fragmentation-only rebuild
  pays for itself. The break-even depends on per-device seek economics and
  cache state, which the v12 cost framework models only through the global or
  per-tablespace `seq_page_cost` / `random_page_cost` constants, never per
  index
  ([config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765)).

## Related Pages

- [v12/index](../index.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](pgstatindex-sample-variant-proposal.md)
- [versions](../../versions.md)

## Source References

- [pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212) - `AccessShareLock` open and shared worker dispatch.
- [pgstatindex.c#bulkread](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L218-L223) - `BAS_BULKREAD` strategy for the scan.
- [pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238) - B-tree-only and other-temp checks.
- [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315) - full physical block walk.
- [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356) - density/fragmentation formulas and `NaN` cases.
- [pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92) - SQL surface, output columns, grants.
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) - extension default version 1.5.
- [pgstattuple.sgml#access](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L24) and [pgstattuple.sgml#non-snapshot](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L278) - privilege model and page-by-page caveat.
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113) and [pgstattuple.out#empty-btree-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82) - regression coverage and empty-index `NaN` output.
- [system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L682), [system_views.sql#pg_statio_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L684-L708), [system_views.sql:994](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L994) - view definitions used by Stage 1 and progress monitoring.
- [monitoring.sgml#stats-lag](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L240), [monitoring.sgml#table-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2741-L2792), [monitoring.sgml#index-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2884-L2941), [monitoring.sgml#idx_blks](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3024-L3073), [monitoring.sgml#create-index-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3488-L3497) - documented counter semantics and progress view.
- [pgstat.h#pgstat_count_index_scan](../../../raw/postgres-12/src/include/pgstat.h#L1374-L1378) - counter macro.
- [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L745-L768) - the B-tree `idx_scan` increment site.
- [nbtinsert.c#_bt_doinsert](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L213-L260) and [nbtinsert.c#_bt_check_unique](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L320-L343) - insert/uniqueness path that bypasses `_bt_first`.
- [nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335) - bitmap scans start through `_bt_first`.
- [README.HOT:6](../../../raw/postgres-12/src/backend/access/heap/README.HOT#L6) and [monitoring.sgml#n_tup_hot_upd](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2766-L2771) - HOT updates avoid index entries.
- [catalogs.sgml#relpages-reltuples](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1782) - estimate semantics of `pg_class` stats.
- [vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1117-L1191) - in-place `pg_class` stats update.
- [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1771-L1830) - index stats update skipped when cleanup returns no stats.
- [nbtree.c#_bt_vacuum_needs_cleanup](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L783-L846) and [nbtree.c#btvacuumcleanup-skip](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L942) - cleanup skip logic.
- [guc.c#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3424-L3432), [reloptions.c#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1417-L1418), [config.sgml#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7852-L7891) - skip threshold GUC/reloption.
- [analyze.c#index-relstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629) - ANALYZE refresh of index `pg_class` stats.
- [index.c#index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2680) - stats refresh after CREATE INDEX/REINDEX.
- [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336), [pg_proc.dat#pg_relation_size-1arg](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6888) - size probe path.
- [bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199) - block-count macro used by ANALYZE.
- [guc.c#track_counts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1400), [guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2386), [guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2389-L2397), [guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2880-L2888) - GUC contexts for the settings this page touches.
- [nbtree-README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L200-L212) - only empty pages deleted; no merging.
- [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1177) and [nbtpage.c#_bt_getbuf-new-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820) - FSM recording and reuse of deleted pages.
- [nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12) and [nbtsort.c#leaf-fill](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L713-L734) - rebuild density and layout.
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171) - default leaf fill target.
- [indexcmds.c#ReindexIndex-lock](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2344-L2360), [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2714-L2737), [indexcmds.c#catalog-error](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2804-L2807) - REINDEX lock selection and concurrent path.
- [index.c#reindex_index](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470) and [index.c#reindex-rebuild](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530) - heap/index locks, progress reporting, new relfilenode build.
- [ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57), [ref/reindex.sgml#storage-param](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L59-L64), [ref/reindex.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L153-L171), [ref/reindex.sgml#locking](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L231-L243), [ref/reindex.sgml#concurrent-cost](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L283-L297), [ref/reindex.sgml#concurrent-steps](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L299-L361), [ref/reindex.sgml#failed-concurrent](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390), [ref/reindex.sgml#system-and-exclusion](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L392-L414) - REINDEX behavior and caveats.
- [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L897) - bloat shape, non-B-tree note, adjacency, lock summary.
- [pg_class.h#fields](../../../raw/postgres-12/src/include/catalog/pg_class.h#L49-L81) and [pg_class.h#relkind-index](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163) - catalog fields used in the queries.
- [pg_index.h#flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L40) - `indisunique`, `indisvalid`.
- [pg_am.dat#btree](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L20) - B-tree access method row.
- [itup.h#IndexTupleData](../../../raw/postgres-12/src/include/access/itup.h#L25-L53), [itemptr.h#ItemPointerData](../../../raw/postgres-12/src/include/storage/itemptr.h#L22-L47), [itemid.h#ItemIdData](../../../raw/postgres-12/src/include/storage/itemid.h#L17-L32), [pg_statistic.h#stawidth](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L39-L49) - estimator components.
- [freelist.c#GetAccessStrategy](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L547-L576) - bulk-read ring size.
- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5815), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116) - planner visibility of page count and height.
- [stats.sql#counter-checks](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L19-L56), [stats.sql#counter-asserts](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L159-L166), [create_index.sql#reindex-concurrently](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L793-L832) - test coverage for counters and concurrent REINDEX.
- [index.c#index_concurrently_create_copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1233-L1241) and [index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705) - new index relation creation and the cumulative-counter copy at swap.
- [pg_stat_statements.control](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.control#L1-L5), [pg_stat_statements--1.4.sql#view](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.4.sql#L12-L43), [pg_stat_statements--1.6--1.7.sql#selective-reset](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.6--1.7.sql#L13-L22) - query-level measurement columns and selective reset.
- [guc.c#track_io_timing](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1402-L1409) - I/O timing GUC context.
- [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2867-L2978) - `EXPLAIN (ANALYZE, BUFFERS)` output path used for before/after comparison.
- [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307) - the backward-right-link test behind `leaf_fragmentation`.
- [nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1630) and [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1771) - forward leaf-chain walk that reads the block named by the right link.
- [nbtinsert.c#split-new-right-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1403-L1440) - split path that places the new right page wherever `_bt_getbuf(P_NEW)` finds space.
- [config.sgml#seq-random-page-cost](../../../raw/postgres-12/doc/src/sgml/config.sgml#L4696-L4765) - global/tablespace page-cost constants; no per-index adjacency input.
