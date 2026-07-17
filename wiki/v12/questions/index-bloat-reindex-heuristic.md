---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-17T15:18:58Z
---

# Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Decision rule](#decision-rule)
  - [Prerequisites and safety](#prerequisites-and-safety)
  - [Stage 1: Build an inexpensive shortlist](#stage-1-build-an-inexpensive-shortlist)
  - [Stage 2: Confirm with `pgstatindex`](#stage-2-confirm-with-pgstatindex)
  - [Stage 3: Rank and execute](#stage-3-rank-and-execute)
  - [Measure the improvement](#measure-the-improvement)
  - [Why `leaf_fragmentation` is only a modifier](#why-leaf_fragmentation-is-only-a-modifier)
  - [Scope limits](#scope-limits)
  - [Tests and reproduced checks](#tests-and-reproduced-checks)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12: based on PostgreSQL's usually available statistics and
execution of pgstatindex, propose a heuristic to find bloated indexes and
prioritize them for reindexing.

## Answer

Use a three-stage funnel:

1. Shortlist physical, valid B-tree indexes from relation size, table churn,
   index-use counters, and index-I/O counters. This reads catalogs and file
   metadata, but not the candidate indexes' data pages: `pg_relation_size()`
   takes `AccessShareLock` and sums relation-segment sizes with `stat()`
   ([dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336),
   [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)).
2. Run `pgstatindex()` only on that shortlist. It reads block 0 for metadata,
   captures the main-fork length once, and then physically reads every block
   from 1 through that captured end. This is a full physical observation, not
   a sample, but concurrent changes mean it is not a whole-index snapshot
   ([pgstatindex.c#metapage-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315),
   [pgstattuple.sgml#page-by-page-caveat](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279)).
3. Estimate reclaimable pages relative to each index's own fillfactor, rank
   material waste by a measured scan rate, apply workload modifiers, and then
   choose plain or concurrent `REINDEX`. PostgreSQL 12 supports `REINDEX INDEX
   CONCURRENTLY`; the manual says it is useful in production because writes
   continue, at the cost of two table scans, more total work, and transaction
   waits
   ([ref/reindex.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L153-L171),
   [ref/reindex.sgml#concurrent-cost](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L271-L297)).

Bloat here means the v12 manual's space problem: many empty or nearly empty
B-tree pages that a fresh index copy can omit
([ref/reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L55)).
VACUUM removes index tuples and can delete a completely empty B-tree page, but
it does not merge partly full pages; a page with a few surviving keys remains
allocated
([nbtree-README#page-deletion](../../../raw/postgres-12/src/backend/access/nbtree/README#L200-L212),
[maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874)).

### Decision rule

Use these as operational starting rules, not PostgreSQL-defined thresholds:

- Put invalid indexes in a separate correctness queue. `pg_index.indisvalid`
  is the planner-use flag, and a directly named invalid index can be rebuilt by
  `REINDEX INDEX CONCURRENTLY`; table-level concurrent REINDEX skips invalid
  indexes instead
  ([pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L44),
  [indexcmds.c#invalid-index-selection](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2812-L2839),
  [indexcmds.c#named-invalid-index](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2893-L2915)).
- Shortlist valid physical B-trees above a local size floor. The examples use
  64 MB only to bound diagnostic cost. Increase that floor on large clusters.
- Prefer to measure after a completed VACUUM cycle. `avg_leaf_density` measures
  physical occupancy and counts the stored bytes of retained `LP_DEAD` tuples;
  it does not determine tuple liveness
  ([pgstatindex.c#leaf-occupancy](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307),
  [itemid.h#ItemIdMarkDead](../../../raw/postgres-12/src/include/storage/itemid.h#L160-L182),
  [nbtpage.c#vacuum-delete-items](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L965-L1013)).
- Treat a density gap of about 10 percentage points below the configured
  fillfactor as a review signal. Treat a gap of 20 points, or reclaimable space
  large enough to matter operationally, as stronger evidence. These bands are
  proposals; v12 defines no trigger threshold.
- Rank with paired counter snapshots. A useful starting score is:

  ```text
  scan_rate_per_day = delta(idx_scan) / elapsed_days
  priority = estimated_reclaimable_bytes
             * (1 + ln(1 + scan_rate_per_day))
  ```

  The leading `1` preserves the space-saving priority of a zero-scan index.
  This matters because uniqueness checks do not increment B-tree `idx_scan`:
  they use `_bt_search` and `_bt_check_unique`, whereas `_bt_first` increments
  the scan counter
  ([nbtinsert.c#_bt_doinsert](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L213-L260),
  [nbtinsert.c#_bt_check_unique](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L320-L343),
  [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L745-L768)).

### Prerequisites and safety

- Install the `pgstattuple` extension in the database. Its default extension
  version is 1.5. Version 1.5 revokes `pgstatindex(regclass)` from `PUBLIC` and
  grants it to `pg_stat_scan_tables`; superusers bypass ordinary function ACL
  checks
  ([pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5),
  [pgstattuple--1.4--1.5.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
  [pgstattuple.sgml#access](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L24)).
- The cumulative views require `track_counts`. It defaults to on and has
  `PGC_SUSET` context. A superuser can change it for a session or transaction,
  but useful workload-wide evidence requires it to be enabled in the workload
  backends whose activity is being measured
  ([guc.c#track_counts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1400)).
- Cumulative statistics lag. A backend reports before going idle and no more
  often than the collector interval; the collector writes a new report file at
  most every 500 ms. Wait for reports and compare snapshots rather than reading
  immediately after a workload action
  ([monitoring.sgml#stats-lag](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L240)).
- The diagnostic SQL below uses session-scoped `statement_timeout` and
  `lock_timeout`; both are `PGC_USERSET`, so they require neither restart nor
  reload when set for the session
  ([guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386),
  [guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)).

### Stage 1: Build an inexpensive shortlist

The scope filter should keep physical B-tree indexes (`relkind = 'i'`), not
partitioned parent indexes (`relkind = 'I'`). `pgstatindex` accepts only a
physical `RELKIND_INDEX` with `BTREE_AM_OID`; a physical B-tree index on a table
partition is still eligible
([pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L238),
[pg_class.h#relation-kinds](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167)).
The example also skips all temporary indexes. The function could inspect a
temporary index owned by the current session, but rejects another session's
temporary relation because its local buffers are invisible
([pgstatindex.c#other-session-temp](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L230-L238)).

Use these signals:

| Signal | Interpretation |
|---|---|
| `pg_relation_size(index)` | Current main-fork bytes from segment-file metadata. It does not read index data blocks ([dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)). |
| `n_tup_upd - n_tup_hot_upd`, plus `n_tup_del` | Table-level dead-entry pressure. HOT updates need no separate index update; non-HOT updates and deletes leave old index entries for cleanup ([monitoring.sgml#n_tup_hot_upd](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2766-L2771), [README.HOT:6](../../../raw/postgres-12/src/backend/access/heap/README.HOT#L6)). This is not index-specific. |
| `n_tup_ins` | Table insert volume, not proof of bloat. Inserts that satisfy an index predicate add index tuples and can eventually split B-tree pages; use this only as growth context, especially for partial indexes ([system_views.sql#pg_stat_all_tables](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [nbtinsert.c#split-decision](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L966-L1025)). |
| `n_dead_tup`, `last_vacuum`, `last_autovacuum` | Table-level context for whether index cleanup is likely to have run; `n_dead_tup` is an estimate, not an index dead-tuple count ([monitoring.sgml#table-stats](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2741-L2792)). |
| `idx_scan`, `idx_tup_read` | Cumulative index scan starts and entries returned. Bitmap scans count through `_bt_first`; planner probes can also affect small counts ([nbtree.c#btgetbitmap](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L287-L335), [monitoring.sgml#index-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2884-L2929)). Use deltas over equal windows. |
| `idx_blks_read`, `idx_blks_hit` | All shared-buffer misses and hits attributed to the index, not query-scan-only I/O. `ReadBufferExtended` increments these for every relation buffer request, so builds, writes, VACUUM, and `pgstatindex` itself can contribute ([bufmgr.c#ReadBufferExtended-stats](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L669), [system_views.sql#pg_statio_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L684-L708)). Compare pre-diagnostic deltas, not lifetime ratios. |
| Bytes per estimated index tuple | Trend signal only. `pg_class.reltuples` is estimated and can be stale; ANALYZE estimates a partial index's tuple fraction from sampled rows that satisfy its predicate ([catalogs.sgml#relpages-reltuples](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1782), [analyze.c#partial-index-tuple-fraction](../../../raw/postgres-12/src/backend/commands/analyze.c#L771-L843)). |
| Constraint and identity flags | Prevent a zero-scan index from being misclassified as disposable. `pg_index` records unique, primary, exclusion, clustered, and replica-identity roles, while `pg_constraint.conindid` identifies an index supporting a constraint ([pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L44), [pg_constraint.h#conindid](../../../raw/postgres-12/src/include/catalog/pg_constraint.h#L72-L80)). |

Index `relpages` and `reltuples` are estimates. VACUUM, ANALYZE, and index DDL
can refresh them
([catalogs.sgml#relpages-reltuples](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1782),
[index.c#index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2989)).
B-tree VACUUM can skip its cleanup callback and therefore skip the index
`pg_class` update when `_bt_vacuum_needs_cleanup` says cleanup is unnecessary
([nbtree.c#cleanup-decision](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L783-L846),
[nbtree.c#cleanup-skip](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L896-L942),
[vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1771-L1830)).
The `vacuum_cleanup_index_scale_factor` default is 0.1; it is `PGC_USERSET`
(session or transaction scope) and also has a per-index reloption
([guc.c#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3424-L3432),
[reloptions.c#vacuum_cleanup_index_scale_factor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1417-L1418)).
A standalone ANALYZE refreshes index block counts and estimates index tuples,
so use `pg_relation_size()` for current bytes and treat catalog tuple counts as
estimates
([analyze.c#index-relstats](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629)).

Run this query first, before any `pgstatindex` calls contaminate the index-I/O
counter window:

```sql
SET /* wiki_index_bloat_shortlist_statement_timeout_v12 */ statement_timeout = '30s';
SET /* wiki_index_bloat_shortlist_lock_timeout_v12 */ lock_timeout = '2s';

SELECT /* wiki_index_bloat_shortlist_v12 */
       n.nspname                                  AS schema_name,
       ct.relname                                 AS table_name,
       ci.relname                                 AS index_name,
       pg_relation_size(ci.oid)                   AS index_bytes,
       ci.reltuples                               AS index_reltuples,
       CASE WHEN ci.reltuples > 0
            THEN pg_relation_size(ci.oid) / ci.reltuples
       END                                        AS bytes_per_index_tuple,
       x.indisunique,
       x.indisprimary,
       x.indisexclusion,
       x.indisclustered,
       x.indisreplident,
       EXISTS (SELECT 1
               FROM pg_constraint co
               WHERE co.conindid = ci.oid)        AS supports_constraint,
       s.idx_scan,
       s.idx_tup_read,
       io.idx_blks_read,
       io.idx_blks_hit,
       t.n_tup_ins,
       t.n_tup_upd - t.n_tup_hot_upd              AS non_hot_updates,
       t.n_tup_del,
       t.n_dead_tup,
       t.last_vacuum,
       t.last_autovacuum
FROM pg_index x
JOIN pg_class ci               ON ci.oid = x.indexrelid
JOIN pg_class ct               ON ct.oid = x.indrelid
JOIN pg_namespace n            ON n.oid = ci.relnamespace
JOIN pg_am am                  ON am.oid = ci.relam
                               AND am.amname = 'btree'
JOIN pg_stat_user_indexes s    ON s.indexrelid = ci.oid
JOIN pg_statio_user_indexes io ON io.indexrelid = ci.oid
JOIN pg_stat_user_tables t     ON t.relid = ct.oid
WHERE x.indisvalid
  AND ci.relkind = 'i'
  AND ci.relpersistence <> 't'
  AND pg_relation_size(ci.oid) >= 64 * 1024 * 1024
ORDER BY (t.n_tup_upd - t.n_tup_hot_upd) + t.n_tup_del DESC,
         index_bytes DESC;
```

The query does not estimate bloat. It narrows the set that justifies a full
physical scan.

### Stage 2: Confirm with `pgstatindex`

Schedule this stage deliberately:

- Both v1.5 wrappers open the index with `AccessShareLock`. That lock conflicts
  with `AccessExclusiveLock`, but ordinary DML and lazy VACUUM can overlap it;
  each non-metapage receives only a per-buffer shared content lock while it is
  inspected
  ([pgstatindex.c#v1.5-wrappers](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L213),
  [pgstatindex.c#page-lock](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
  [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105)).
- The scan uses a `BAS_BULKREAD` ring nominally sized at `256 kB / BLCKSZ`,
  capped at `NBuffers / 8`. The ring limits the scan's replacement working set;
  it does not mean every block is a device read or guarantee no cache impact.
  A shared-buffer miss reaches `smgrread()` and `FileRead()`, so PostgreSQL's
  block counter does not distinguish an operating-system-cache hit from device
  service
  ([pgstatindex.c#bulkread](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L223),
  [freelist.c#BAS_BULKREAD](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L587),
  [bufmgr.c#ReadBufferExtended](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L669),
  [md.c#mdread](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L573-L604)).
- The call increments index block-fetch/hit counters while reading. Capture the
  Stage 1 I/O window first, and take any post-REINDEX runtime baseline only
  after the post-REINDEX diagnostic call has finished
  ([bufmgr.c#ReadBufferExtended-stats](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L660-L669)).
- It is a physical diagnostic, not an integrity checker. The worker does not
  call `_bt_getmeta` or `_bt_checkpage`, and its pages do not form one
  transactionally consistent snapshot
  ([pgstatindex.c#raw-page-access](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315),
  [nbtpage.c#_bt_getmeta](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148),
  [nbtpage.c#_bt_checkpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).

Interpret the result precisely:

| Output | Interpretation |
|---|---|
| `avg_leaf_density` | Summed physical occupancy of live leaf pages. Compare it with the index's configured fillfactor, not always 90. Sorted B-tree build uses fillfactor as a soft page-full threshold; minimum-item and last-page effects mean a fresh result is near, not exactly equal to, that number ([nbtsort.c#leaf-fillfactor](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L730), [nbtsort.c#soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L880-L900)). |
| `deleted_pages` | Every page marked `BTP_DELETED`, including pages whose deletion XID is not old enough for reuse. VACUUM records only recyclable pages in the free space map; later page allocation can reuse those pages ([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.c#recyclable-pages](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1187), [nbtpage.c#new-page-reuse](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820), [nbtpage.c#recyclability](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963)). A growing index may therefore absorb some of this space without REINDEX. |
| `empty_pages` | In this v12 implementation, the post-deleted `P_IGNORE` branch counts half-dead pages, not pages selected by a zero-item test ([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.h#page-state-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L183-L196)). |
| `leaf_fragmentation` | Percentage of live leaf pages whose non-null right link points to a lower physical block. It is a backward-link percentage, not a byte or run-length measure ([pgstatindex.c#fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#fragmentation-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)). |
| `tree_level` | Level of the true root read from the metapage. The B-tree planner cost adds a descent charge based on the measured tree height ([pgstatindex.c#metapage](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253), [selfuncs.c#btree-height-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6104-L6116)). |
| `NaN` | `avg_leaf_density` is `NaN` when no live-leaf capacity was accumulated; `leaf_fragmentation` is `NaN` when `leaf_pages` is zero. The query filters `leaf_pages > 0` ([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)). |

Estimate pages rather than multiplying the whole index by a density gap. The
whole index includes its metapage, internal pages, deleted pages, and half-dead
pages, while density describes live leaf pages only
([pgstatindex.c#index-size-and-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)).
Use this approximation:

```text
target_live_leaf_pages = ceil(
    leaf_pages * avg_leaf_density / configured_fillfactor
)

estimated_reclaimable_pages =
    deleted_pages + empty_pages
    + max(leaf_pages - target_live_leaf_pages, 0)

estimated_reclaimable_bytes =
    estimated_reclaimable_pages * block_size
```

`pg_options_to_table(pg_class.reloptions)` exposes the configured fillfactor;
absence means the B-tree default of 90. `block_size` is the read-only internal
GUC backed by `BLCKSZ`
([func.sgml#pg_options_to_table](../../../raw/postgres-12/doc/src/sgml/func.sgml#L19067-L19072),
[reloptions.c#btree-fillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186),
[guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888)).
The formula deliberately omits possible internal-page savings. It remains an
estimate because density is rounded to two decimals, fillfactor is a soft
limit, high keys and line pointers occupy space, and concurrent changes can
alter the observed page set
([pgstatindex.c#density-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356),
[nbtsort.c#soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L880-L900)).

```sql
SET /* wiki_index_bloat_pgstatindex_statement_timeout_v12 */ statement_timeout = '30min';
SET /* wiki_index_bloat_pgstatindex_lock_timeout_v12 */ lock_timeout = '2s';

SELECT /* wiki_index_bloat_pgstatindex_v12 */
       n.nspname                                  AS schema_name,
       ci.relname                                 AS index_name,
       ff.fillfactor,
       s.idx_scan,
       ps.index_size,
       ps.tree_level,
       ps.leaf_pages,
       ps.deleted_pages,
       ps.empty_pages,
       ps.avg_leaf_density,
       ps.leaf_fragmentation,
       current_setting('block_size')::bigint
         * (ps.deleted_pages
            + ps.empty_pages
            + GREATEST(
                ps.leaf_pages
                  - CEIL(ps.leaf_pages * ps.avg_leaf_density
                         / ff.fillfactor)::bigint,
                0::bigint))                       AS est_reclaimable_bytes
FROM pg_index x
JOIN pg_class ci            ON ci.oid = x.indexrelid
JOIN pg_namespace n         ON n.oid = ci.relnamespace
JOIN pg_am am               ON am.oid = ci.relam
                            AND am.amname = 'btree'
JOIN pg_stat_user_indexes s ON s.indexrelid = ci.oid
CROSS JOIN LATERAL
     (SELECT COALESCE(
          (SELECT option_value::double precision
           FROM pg_options_to_table(ci.reloptions)
           WHERE option_name = 'fillfactor'),
          90.0) AS fillfactor) ff
CROSS JOIN LATERAL pgstatindex(ci.oid::regclass) ps
WHERE x.indisvalid
  AND ci.relkind = 'i'
  AND ci.relpersistence <> 't'
  AND pg_relation_size(ci.oid) >= 64 * 1024 * 1024
  AND ps.leaf_pages > 0
ORDER BY est_reclaimable_bytes DESC;
```

This query orders by physical opportunity only. Replace the cumulative
`idx_scan` value with a paired-window scan rate before making the final queue.

### Stage 3: Rank and execute

Apply these modifiers to the size-times-scan-rate score:

- Boost broad range, ordered, and bitmap workloads. They can visit many leaf
  pages, so density can reduce page fetches; point lookups usually still visit
  one leaf unless the rebuild also drops a tree level. The planner sees current
  index pages and B-tree height through `IndexOptInfo.pages` and
  `tree_height`
  ([nbtsearch.c#_bt_search](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L71-L139),
  [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417),
  [selfuncs.c#genericcostestimate-pages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5835)).
- Boost indexes with sustained pre-diagnostic `idx_blks_read` deltas during
  read-dominated windows. Do not interpret `idx_blks_read / (read + hit)` as a
  cache hit ratio: the counters cover all index buffer activity and do not
  identify the operation that caused it
  ([system_views.sql#pg_statio_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L684-L708),
  [bufmgr.c#ReadBufferExtended-stats](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L660-L669)).
- Demote a growing index whose estimate consists mainly of `deleted_pages` if
  those pages are becoming recyclable; future splits can obtain free pages
  from the free space map before extending the relation
  ([nbtree.c#recyclable-pages](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1177),
  [nbtpage.c#new-page-reuse](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820)).
- Keep a zero-scan unique, primary, exclusion, replica-identity, or
  constraint-supporting index in the REINDEX analysis. Evaluate an apparently
  unused ordinary index separately before considering `DROP INDEX`; cumulative
  counters can have been reset, and optimizer probes can make small values
  noisy
  ([pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L44),
  [monitoring.sgml#optimizer-index-access](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2904-L2929)).
- Use `leaf_fragmentation` only as a boost for cold, wide scans on
  seek-sensitive storage. Do not let it trigger a rebuild by itself.

For a production table, the concurrent form is often preferable. The example
uses session-scoped timeouts; choose the statement timeout from the measured
build duration and maintenance window. `REINDEX CONCURRENTLY` cannot run inside
a transaction block
([utility.c#REINDEX-CONCURRENTLY](../../../raw/postgres-12/src/backend/tcop/utility.c#L773-L806),
[ref/reindex.sgml#transaction-block](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L392-L400)).

```sql
SET /* wiki_reindex_btree_statement_timeout_v12 */ statement_timeout = '2h';
SET /* wiki_reindex_btree_lock_timeout_v12 */ lock_timeout = '5s';

REINDEX /* wiki_reindex_btree_v12 */ INDEX CONCURRENTLY my_schema.my_index;
```

Choose the command with these boundaries:

- Plain `REINDEX INDEX` takes `ShareLock` on the parent table, blocking writes,
  and `AccessExclusiveLock` on the index, blocking scans that use it. It assigns
  the existing index relation a new relfilenode and runs the AM build in one
  transaction
  ([index.c#reindex_index-locks](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470),
  [index.c#reindex-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3531)).
- Concurrent REINDEX uses session-level `ShareUpdateExclusiveLock` on the old
  index, new index, and heap across six phases. It performs a build scan, a
  validation scan, three writer/old-snapshot waits, and two additional waits
  for readers before making the old index dead and dropping it
  ([indexcmds.c#concurrent-phases](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955),
  [indexcmds.c#build-and-validate](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3080-L3199),
  [indexcmds.c#reader-waits](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3264-L3332)).
- A directly named system-catalog index cannot be rebuilt concurrently.
  Exclusion-constraint indexes error when named directly and are skipped with a
  warning during table-level concurrent REINDEX. Temporary relations fall back
  to the cheaper non-concurrent path. PostgreSQL 12 does not reindex a
  partitioned parent table or partitioned parent index; rebuild each physical
  partition index separately
  ([indexcmds.c#catalog-and-exclusion](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2804-L2839),
  [indexcmds.c#partitioned-parent](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2917-L2928),
  [ref/reindex.sgml#partitioned-relations](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L258-L261),
  [ref/reindex.sgml#system-and-exclusion](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L402-L413),
  [ref/reindex.sgml#temporary-relations](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L153-L169)).
- A failure before the swap can leave an invalid `*_ccnew`; a failure after the
  atomic swap can leave the old invalid index as `*_ccold`. The new index takes
  the original name and becomes valid in the same transaction in which the old
  index is renamed and made invalid, so a healthy original name is not turned
  into an invalid no-index state by a failed concurrent rebuild
  ([index.c#index_concurrently_swap](../../../raw/postgres-12/src/backend/catalog/index.c#L1442-L1538),
  [ref/reindex.sgml#failed-concurrent](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390)).
  Inspect `indisvalid` and the suffix before dropping a leftover; a named
  invalid original index can instead be rebuilt concurrently.
- The caller must own the index or its table; superusers can reindex any index
  ([ref/reindex.sgml#ownership](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L245-L255)).
  Budget storage for both copies: phase 1 creates a separate new index, while
  phase 6 drops the old one only after the build, validation, swap, and waits
  have committed
  ([indexcmds.c#create-copy-phase](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2957-L3036),
  [indexcmds.c#drop-old-phase](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3296-L3332)).
- Monitor both REINDEX forms in `pg_stat_progress_create_index`; the command
  field distinguishes ordinary and concurrent REINDEX
  ([index.c#reindex-progress](../../../raw/postgres-12/src/backend/catalog/index.c#L3456-L3464),
  [indexcmds.c#concurrent-progress](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2980-L2991),
  [monitoring.sgml#create-index-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3488-L3497)).

A sorted B-tree build writes pages sequentially and uses the index's current
fillfactor reloption. Change fillfactor with `ALTER INDEX ... SET` before the
rebuild only when the insert pattern justifies reserving more page space; the
REINDEX then makes that already-set storage parameter take full effect
([nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12),
[nbtsort.c#leaf-fillfactor](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L730),
[index.c#copy-reloptions](../../../raw/postgres-12/src/backend/catalog/index.c#L1287-L1293),
[index.c#create-copy-with-reloptions](../../../raw/postgres-12/src/backend/catalog/index.c#L1355-L1380),
[ref/reindex.sgml#storage-parameter](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L59-L64)).

### Measure the improvement

Measure independent windows before and after the rebuild. Do not rely on one
cross-REINDEX cumulative delta:

- Plain REINDEX keeps the index OID and collector entry while replacing only
  its relfilenode
  ([index.c#reindex-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3531)).
- Concurrent REINDEX creates a new index OID. At swap time it copies the old
  collector entry's scan, tuple, and block counters into the new relation's
  pending counters, which the rebuilding backend sends on its next statistics
  report
  ([index.c#index_concurrently_create_copy](../../../raw/postgres-12/src/backend/catalog/index.c#L1233-L1241),
  [index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705)).
  That source copies the collector's published entry; it does not establish
  that increments still pending in other backends are transferred. Independent
  windows avoid depending on that boundary.

Run this snapshot twice before and twice after. Calculate each window only when
`delta(idx_scan) > 0`:

```sql
SET /* wiki_index_reindex_baseline_statement_timeout_v12 */ statement_timeout = '30s';
SET /* wiki_index_reindex_baseline_lock_timeout_v12 */ lock_timeout = '2s';

SELECT /* wiki_index_reindex_baseline_v12 */
       clock_timestamp()                AS captured_at,
       s.indexrelid,
       s.schemaname,
       s.indexrelname,
       pg_relation_size(s.indexrelid)   AS index_bytes,
       s.idx_scan,
       s.idx_tup_read,
       io.idx_blks_read,
       io.idx_blks_hit
FROM pg_stat_user_indexes s
JOIN pg_statio_user_indexes io USING (indexrelid)
WHERE s.indexrelid = 'my_schema.my_index'::regclass;
```

```text
space_saving = 1.0
               - index_bytes_after::numeric
                 / nullif(index_bytes_before, 0)

buffer_accesses_per_scan =
    delta(idx_blks_read + idx_blks_hit)::numeric
    / nullif(delta(idx_scan), 0)

misses_per_scan = delta(idx_blks_read)::numeric
                  / nullif(delta(idx_scan), 0)

tuples_per_scan = delta(idx_tup_read)::numeric
                  / nullif(delta(idx_scan), 0)
```

The access ratios are useful only for comparable, read-dominated windows because
index writes, VACUUM, builds, and diagnostics also increment block counters;
read plus hit counts buffer accesses, not distinct block identities. For the
same broad-scan workload, lower `buffer_accesses_per_scan` with similar
`tuples_per_scan` supports a density improvement; point probes need not show a
leaf-page reduction unless the tree level falls
([monitoring.sgml#index-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2884-L2941),
[monitoring.sgml#index-block-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3024-L3082),
[bufmgr.c#ReadBufferExtended-stats](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L660-L669)).

Use three complementary checks:

1. Compare `pg_relation_size()` and one post-rebuild `pgstatindex()` row. A
   quiet plain rebuild should remove deleted and half-dead pages, restore live
   leaves near fillfactor, lay them out sequentially, and may lower
   `tree_level`. Concurrent writes can already split or dirty the replacement
   before concurrent REINDEX finishes, so those outcomes are not exact
   postconditions for the concurrent form
   ([nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12),
   [index.c#index_concurrently_build](../../../raw/postgres-12/src/backend/catalog/index.c#L1390-L1439)).
2. Compare `EXPLAIN (ANALYZE, BUFFERS)` for representative reads. In v12,
   buffer fields are node totals and an ordinary `Index Scan` node's shared
   blocks are not separated into index and heap blocks. `ANALYZE` executes the
   statement and adds profiling overhead, so treat writes and volatile queries
   accordingly
   ([nodeIndexscan.c#IndexNext](../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L75-L145),
   [instrument.c#node-buffer-delta](../../../raw/postgres-12/src/backend/executor/instrument.c#L61-L99),
   [explain.c#show_buffer_usage](../../../raw/postgres-12/src/backend/commands/explain.c#L2863-L2984),
   [ref/explain.sgml#ANALYZE-side-effects](../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L78-L105),
   [ref/explain.sgml#profiling-overhead](../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L293-L304)).
3. Compare equal `pg_stat_statements` windows for the affected `queryid` values.
   Version 1.7 exposes calls, timing, and shared-block columns and has selective
   reset, but paired snapshots avoid disrupting other monitoring and remain
   statement-level rather than index-specific
   ([pg_stat_statements.control](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.control#L1-L5),
   [pg_stat_statements--1.4.sql#view](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.4.sql#L12-L45),
   [pg_stat_statements--1.6--1.7.sql#selective-reset](../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.6--1.7.sql#L13-L22)).

`track_io_timing` adds block read/write time to these tools. It defaults off and
has `PGC_SUSET` context, so a superuser can enable it for a session or
transaction without restart or reload; measure its timing overhead on the
system
([guc.c#track_io_timing](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1401-L1409),
[config.sgml#track_io_timing](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6854-L6872)).

### Why `leaf_fragmentation` is only a modifier

Excluding fragmentation from the base score is deliberate:

1. The planner does not read `leaf_fragmentation`. It sees current index pages
   and B-tree height. `genericcostestimate()` charges the tablespace's
   `random_page_cost` per estimated index page, with no input for index-page
   physical adjacency
   ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417),
   [selfuncs.c#genericcostestimate-pages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5835)).
2. The metric has no byte value. It counts one fragment when a live leaf's
   right link points backward, then divides by all live leaves, including the
   rightmost leaf. An index can therefore have fragmentation near 100% while
   retaining the same size and density as a differently ordered copy
   ([pgstatindex.c#fragmentation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307),
   [pgstatindex.c#fragmentation-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).
3. It records neither jump distance, fragmented run length, cache residency,
   operating-system read-ahead, nor device latency. Equal percentages need not
   imply equal elapsed time; those properties are absent from the formula
   ([pgstatindex.c#scan-and-fragment-test](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
4. Runtime impact is concentrated in scans that walk many leaves. Forward
   B-tree scans follow the stored right-link block number; cached pages avoid a
   storage read regardless of physical order. The v12 manual characterizes a
   freshly adjacent B-tree only as "slightly faster"
   ([nbtsearch.c#_bt_steppage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1618-L1630),
   [nbtsearch.c#_bt_readnextpage](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1727-L1771),
   [maintenance.sgml#adjacency](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888)).
5. A density-motivated sorted rebuild also writes logical neighbors in
   sequential block order. Later page splits can reintroduce nonadjacent
   neighbors because `_bt_getbuf(P_NEW)` consults the free space map before
   extending. Split occupancy is not always 50/50: non-rightmost pages normally
   balance free space, while a rightmost page tries to leave its left half at
   fillfactor
   ([nbtsort.c#sequential-build](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1-L12),
   [nbtpage.c#new-page-reuse](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L820),
   [nbtsplitloc.c#split-balance](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L89-L105)).

Override the default only when measured wide-scan latency and read I/O regress
while density remains near fillfactor, the data is cold, and the storage is
seek-sensitive. Measure that case with elapsed time and I/O timing; plan cost
will not reflect rearranging the same number of pages.

### Scope limits

- The confirmation formula is B-tree-specific because `pgstatindex` reads
  private B-tree page structures and rejects every other access method
  ([pgstatindex.c#includes-and-object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L28-L73),
  [pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L238)).
  For non-B-tree indexes, the v12 manual recommends monitoring physical size
  because their bloat behavior was not well researched
  ([maintenance.sgml#non-btree](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880)).
- Partial, expression, and `INCLUDE` B-tree indexes are physical B-trees and can
  be scanned, but tuple-count and width shortcuts require care. In particular,
  v12 ANALYZE collects separate `pg_statistic` rows for analyzable
  expression-index attributes; ordinary column keys do not get separate
  index-relation statistics rows
  ([analyze.c#index-expression-attributes](../../../raw/postgres-12/src/backend/commands/analyze.c#L413-L468),
  [analyze.c#index-expression-stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L693-L855)).
- `pgstatindex` is not an integrity proof. Use integrity tooling and recovery
  procedures separately when corruption, sibling-link inconsistency, key
  order, or tuple-to-heap correspondence is the concern
  ([pgstatindex.c#raw-page-access](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315)).
- The contrib worker includes private `access/nbtree.h` and compares against
  `BTREE_AM_OID`. That OID arrives through generated `pg_am_d.h`, produced from
  catalog headers and `pg_am.dat` by the catalog build
  ([pgstatindex.c#includes](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L28-L73),
  [pg_am.h#generated-header](../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29),
  [catalog/Makefile#generated-headers](../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L100)).

### Tests and reproduced checks

- The `pgstattuple` regression test covers the `pgstatindex` entry points, an
  empty B-tree with `NaN` ratios, wrong access methods and relation kinds, and a
  physical index on a partition. It has no populated-index density,
  fragmentation, half-dead, deleted-page, concurrent-growth, or malformed-page
  assertion
  ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119),
  [pgstattuple.out#empty-btree](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82),
  [pgstattuple/Makefile#test-target](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13)).
- Core `stats.sql` checks that scan and block counters advance through the
  collector, while `create_index.sql` covers concurrent REINDEX success,
  warnings, and the exclusion-constraint error. Neither test connects a
  `pgstatindex` result to a rebuild threshold or priority score
  ([stats.sql#counter-checks](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L19-L56),
  [stats.sql#counter-assertions](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L159-L172),
  [create_index.sql#reindex-concurrently](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L793-L832)).
- The filed shortlist, fillfactor-aware diagnostic, snapshot, and tagged
  timeout statements were executed against a server built from the pinned
  12.2 checkout. The check also exercised default and nondefault fillfactors,
  expression-index statistics, a post-VACUUM sparse index, and
  `REINDEX INDEX CONCURRENTLY`.

## Context Reviewed

- Required navigation: [versions](../../versions.md),
  [wiki index](../../index.md), [v12/index](../index.md), and recent
  [log](../../log.md).
- Exact evidence pin: `raw/postgres-12/` at
  `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- `pgstatindex` SQL wrappers, ACLs, `BTIndexStat`, metapage read, block loop,
  classification, formulas, result/error paths, concurrency limits, private
  B-tree includes, generated AM OID, and regression target.
- Cumulative statistics views, collector lag, B-tree scan-count sites,
  uniqueness-check path, relation buffer counting, VACUUM cleanup, ANALYZE
  index statistics, and `pg_class` refresh paths.
- B-tree sorted build, fillfactor soft limit, leaf scan, page deletion,
  recyclability/FSM reuse, split placement and occupancy, planner page/height
  costing, and tests.
- Plain and concurrent REINDEX dispatch, locks, six phases, five waits,
  old/new index state transition, atomic swap, statistics copy, failures,
  restrictions, progress reporting, docs, and regression coverage.
- Production SQL was parsed and run on an isolated exact-pin PostgreSQL 12
  server under `.wiki-runtime/`; the temporary server was stopped after the
  checks.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Stage 1 reads relation file metadata but not candidate index data pages | [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336), [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308) |
| `pgstatindex` reads the metapage and every captured non-metapage block, but not as one snapshot | [pgstatindex.c#metapage-and-scan](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315), [pgstattuple.sgml#page-by-page-caveat](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279) |
| `avg_leaf_density` is physical live-leaf occupancy, including retained dead-item storage | [pgstatindex.c#leaf-occupancy](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307), [itemid.h#ItemIdMarkDead](../../../raw/postgres-12/src/include/storage/itemid.h#L160-L182) |
| `empty_pages` means half-dead; `deleted_pages` includes not-yet-recyclable deleted pages | [pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.c#recyclable-pages](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1187), [nbtpage.c#recyclability](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963) |
| Fillfactor is per index, defaults to 90, and is a soft sorted-build threshold | [reloptions.c#btree-fillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186), [nbtsort.c#leaf-fillfactor](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L730), [nbtsort.c#soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L880-L900) |
| `idx_scan` counts B-tree scan starts but not insert-time uniqueness checks | [nbtsearch.c#_bt_first](../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L745-L768), [nbtinsert.c#_bt_doinsert](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L213-L260) |
| Index block counters cover every `ReadBufferExtended` request, including diagnostics and maintenance | [bufmgr.c#ReadBufferExtended-stats](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L660-L669), [system_views.sql#pg_statio_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L684-L708) |
| The planner sees index page count and B-tree height, not `pgstatindex` fragmentation | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417), [selfuncs.c#genericcostestimate-pages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5835) |
| Plain REINDEX replaces the relfilenode under blocking locks; concurrent REINDEX uses a new index and six phases | [index.c#reindex_index-locks](../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3470), [index.c#reindex-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3531), [indexcmds.c#concurrent-phases](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2955) |
| Concurrent swap atomically gives the valid replacement the original name and copies published statistics | [index.c#index_concurrently_swap](../../../raw/postgres-12/src/backend/catalog/index.c#L1442-L1538), [index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705) |
| Failures can leave invalid `_ccnew` or `_ccold` indexes | [ref/reindex.sgml#failed-concurrent](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L363-L390) |
| Direct tests cover mechanics but not the proposed thresholds or score | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119), [create_index.sql#reindex-concurrently](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L793-L832) |

## Open Questions

- PostgreSQL 12 defines no density gap, reclaimable-byte threshold, size floor,
  observation window, or score. The 64 MB floor, 10/20-point review bands, and
  size-times-log-scan-rate formula must be calibrated against local storage,
  maintenance windows, and query latency
  ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L852-L897)).
- The reclaimable-page formula is not an exact prediction. It uses a rounded
  physical occupancy ratio and a soft fillfactor target, treats all half-dead
  and deleted pages as absent from the rebuilt file, ignores possible
  internal-page savings, and cannot predict concurrent writes during the
  rebuild
  ([pgstatindex.c#index-size-and-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356),
  [nbtsort.c#soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L880-L900)).
- The concurrent-swap statistics copy reads the collector's published old-index
  entry. Source does not state that increments still pending in other backends
  are transferred to the replacement OID. This is why the measurement method
  compares independent windows rather than assuming exact cross-swap
  continuity
  ([index.c#index_concurrently_swap-stats](../../../raw/postgres-12/src/backend/catalog/index.c#L1682-L1705)).
- There is no source-derived break-even point for a fragmentation-only rebuild.
  It depends on scan width, cache state, filesystem behavior, and storage
  latency that the v12 per-index cost input does not represent
  ([selfuncs.c#genericcostestimate-pages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5835)).

## Source References

- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365) - object checks, scan, classification, formulas, and result construction.
- [pgstattuple--1.4--1.5.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92) - v1.5 signature and role grant.
- [pgstattuple.sgml#page-by-page-caveat](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279) - index-size relationship and non-snapshot warning.
- [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L336) - file-metadata size path and relation lock.
- [system_views.sql#statistics-views](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L708) - table, index, and index-I/O views used by the queries.
- [bufmgr.c#ReadBufferExtended-stats](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L669) - block fetch/hit accounting.
- [analyze.c#index-expression-attributes](../../../raw/postgres-12/src/backend/commands/analyze.c#L413-L468) and [analyze.c#index-expression-stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L693-L855) - expression-index statistics and index tuple estimates.
- [nbtree.c#recyclable-pages](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1187) and [nbtpage.c#recyclability](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963) - deleted, half-dead, and reusable-page distinctions.
- [nbtsort.c#leaf-fillfactor](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L724-L730), [nbtsort.c#soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L880-L900), and [nbtsplitloc.c#split-balance](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L89-L105) - build and split occupancy.
- [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L388-L417) and [selfuncs.c#genericcostestimate-pages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5765-L5835) - planner page/height inputs and page cost.
- [index.c#index_concurrently_swap](../../../raw/postgres-12/src/backend/catalog/index.c#L1442-L1716) - atomic swap, dependencies, flags, and statistics copy.
- [indexcmds.c#ReindexRelationConcurrently](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2714-L3345) - candidate selection, six phases, locks, waits, and cleanup.
- [ref/reindex.sgml#concurrent-reindex](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L263-L415) - concurrent cost, phases, failures, and restrictions.
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119), [stats.sql#counter-checks](../../../raw/postgres-12/src/test/regress/sql/stats.sql#L19-L56), and [create_index.sql#reindex-concurrently](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L793-L832) - direct test surfaces.

## Navigation

- [v12/index](../index.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
- [versions](../../versions.md)
