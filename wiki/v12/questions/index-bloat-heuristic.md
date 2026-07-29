---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: claude-opus-5-max 2026-07-29T14:57:04Z
---

# A Heuristic to Detect B-Tree Index Bloat in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Statistics that are available](#statistics-that-are-available)
  - [Who is allowed to run it](#who-is-allowed-to-run-it)
  - [The heuristic](#the-heuristic)
  - [SQL example](#sql-example)
  - [How the thresholds work](#how-the-thresholds-work)
  - [Why 30%](#why-30)
  - [Two systematic false positives](#two-systematic-false-positives)
  - [Exact-pin measurements](#exact-pin-measurements)
  - [Limits and caveats](#limits-and-caveats)
  - [Build, generated-file, and test boundary](#build-generated-file-and-test-boundary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

based on PostgreSQL's usually available statistics and execution of pgstatindex, propose a heuristic to tell if an index is bloated or not.

## Answer

A practical B-tree bloat heuristic compares the current physical B-tree state reported by `pgstatindex` with the index's configured `fillfactor`, and separately counts the pages that hold no live leaf data at all. Flag the index when the estimated recoverable bytes exceed 30 % of `index_size` and the index is larger than 1 MiB. Two conditions must be checked before acting on a flag: the index must not be a random-key index whose density is structural rather than recoverable, and its `deleted_pages` must not already be queued for reuse in the free space map. Both are covered in [Two systematic false positives](#two-systematic-false-positives).

### Statistics that are available

`pgstatindex` returns one row per B-tree index with the fields `version`, `tree_level`, `index_size`, `root_block_no`, `internal_pages`, `leaf_pages`, `empty_pages`, `deleted_pages`, `avg_leaf_density`, and `leaf_fragmentation`. The `regclass` overload is the documented interface; the `text` overload exists only for backward compatibility and the source says it will be deprecated ([pgstattuple--1.4.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L74), [pgstatindex.c#interface-note](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L54), [pgstattuple.sgml#pgstatindex-text](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L283-L296)). In v12 the extension's `default_version` is `1.5`, so a fresh `CREATE EXTENSION pgstattuple` installs the `_v1_5` C entry points defined by the 1.4-to-1.5 update script ([pgstattuple.control:3](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L3), [pgstattuple--1.4--1.5.sql#v1_5-pgstatindexbyid](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).

`pgstatindex` opens the index with `AccessShareLock`, then reads every main-fork block after the metapage through a `BAS_BULKREAD` strategy ring, taking a share lock on one buffer at a time ([pgstatindex.c#pgstatindexbyid_v1_5](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L203-L213), [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)). The `BAS_BULKREAD` ring is 256 KiB, so a full-index scan does not evict the rest of shared buffers ([freelist.c#BAS_BULKREAD](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L559-L561)). Counters accumulate in a `BTIndexStat` struct ([pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L80-L95)).

Page classification is a strict if/else chain, tested in this order: `P_ISDELETED` pages become `deleted_pages`, remaining `P_IGNORE` pages become `empty_pages` (the source comment calls this the "half dead" state), remaining `P_ISLEAF` pages become `leaf_pages`, and everything else becomes `internal_pages` ([pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)). Only live leaf pages contribute to density: each adds `pd_special - SizeOfPageHeaderData` to `max_avail` and `PageGetFreeSpace(page)` to `free_space` ([pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L300)). The reported values are then

- `index_size = (1 + leaf_pages + internal_pages + deleted_pages + empty_pages) * BLCKSZ`, the extra page being the metapage ([pgstatindex.c#index-size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L336-L341), [pgstattuple.sgml#index_size-metapage](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L273));
- `avg_leaf_density = 100 - free_space / max_avail * 100`, or `NaN` when no live leaf contributed capacity ([pgstatindex.c#density-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351));
- `leaf_fragmentation = fragments / leaf_pages * 100`, or `NaN` when `leaf_pages` is zero, where a fragment is a live leaf whose `btpo_next` is not `P_NONE` and points to a lower block number ([pgstatindex.c#fragment-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#fragmentation-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).

`PageGetFreeSpace` is `pd_upper - pd_lower - sizeof(ItemIdData)`, so it counts dead line pointers and their still-present tuples as occupied ([bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L580-L597)). Index entries for deleted heap rows therefore stay invisible to `avg_leaf_density` until something physically removes them: an index scan only sets `LP_DEAD` hints in `_bt_killitems`, `_bt_vacuum_one_page` compacts one page at insert time, and `btvacuumpage` calls `_bt_delitems_vacuum` -> `PageIndexMultiDelete` during VACUUM ([nbtutils.c#_bt_killitems-mark-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1786-L1797), [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2244-L2281), [nbtpage.c#_bt_delitems_vacuum](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L985-L999)).

Two catalog inputs complete the picture:

- `pg_stat_all_indexes` supplies `idx_scan`, `idx_tup_read`, and `idx_tup_fetch` from `pg_stat_get_numscans(I.oid)` and friends, and restricts itself to `C.relkind IN ('r', 't', 'm')`; `pg_stat_user_indexes` is that view filtered to non-system schemas ([system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672), [system_views.sql#pg_stat_user_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L679-L682)). The `relkind` filter is what keeps partitioned-parent indexes out of the candidate list, which matters because `pgstatindex` rejects them; see [Limits and caveats](#limits-and-caveats). These counters are a per-transaction snapshot of collector data, not live values ([monitoring.sgml#stats-snapshot](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L242-L260)).
- `pg_class.relpages` is a cheap, possibly stale size pre-filter, and `pg_class.reloptions` holds `fillfactor` ([pg_class.h#relpages-reltuples](../../../raw/postgres-12/src/include/catalog/pg_class.h#L58-L63), [pg_class.h#reloptions](../../../raw/postgres-12/src/include/catalog/pg_class.h#L133-L134)). `pg_options_to_table` turns that array into `(option_name, option_value)` rows ([pg_proc.dat#pg_options_to_table](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3603-L3608)).

`block_size` is the preset GUC that reports `BLCKSZ`. Its context is `PGC_INTERNAL` with `min = max = BLCKSZ`, so it is compile-time fixed and cannot be changed by restart, reload, or session ([guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888)).

### Who is allowed to run it

The 1.4-to-1.5 update revokes `EXECUTE` on both `pgstatindex` overloads from `PUBLIC` and grants it to the `pg_stat_scan_tables` role ([pgstattuple--1.4--1.5.sql#pgstatindex-regclass-acl](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L91-L92), [pgstattuple.sgml#access](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L15-L24)). A monitoring role must therefore be a superuser, be granted `pg_stat_scan_tables`, or hold a direct `GRANT EXECUTE`. `pg_stat_user_indexes` and `pg_options_to_table` need no extra grant. On the pinned checkout, a plain `LOGIN` role got `ERROR:  permission denied for function pgstatindex` and succeeded after `GRANT pg_stat_scan_tables`.

### The heuristic

1. Read the configured leaf `fillfactor` from `reloptions`. The B-tree default is 90 and the minimum is 10 ([nbtree.h#fillfactor-constants](../../../raw/postgres-12/src/include/access/nbtree.h#L168-L171), [reloptions.c#btree-fillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186)). If the option is absent, use 90.
2. Run `pgstatindex(index_oid::regclass)` for `leaf_pages`, `empty_pages`, `deleted_pages`, `avg_leaf_density`, `index_size`, and `leaf_fragmentation`.
3. Estimate how many leaf pages the live data would occupy if repacked to the target fillfactor. Guard the `leaf_pages = 0` case, because that is exactly when `avg_leaf_density` is `NaN`:

```text
ideal_leaf_pages       = leaf_pages * (avg_leaf_density / fillfactor)
underfilled_leaf_pages = max(0, leaf_pages - ideal_leaf_pages)
underfilled_leaf_bytes = underfilled_leaf_pages * block_size
```

4. Count the pages that hold no live leaf data. These are recoverable by a rebuild, but see [Two systematic false positives](#two-systematic-false-positives) for why they are not necessarily lost:

```text
dead_page_bytes = (empty_pages + deleted_pages) * block_size
```

5. Combine:

```text
wasted_bytes = underfilled_leaf_bytes + dead_page_bytes
bloat_ratio  = wasted_bytes / index_size
```

6. Treat the index as a rebuild candidate when `bloat_ratio > 0.30` and `index_size > 1 MiB`. Report `leaf_fragmentation > 30 %` as a separate signal, not as part of the ratio. Use `idx_scan` to rank the work, and read `avg_leaf_density` against the workload before acting.

The 1 MiB floor, the 0.30 ratio, and the 30 % fragmentation flag are operational guardrails. None of them exists in the core code.

### SQL example

Both settings below are `PGC_USERSET`, so they apply at session or transaction scope with no reload or restart ([guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)).

```sql
SET /* wiki_btree_bloat_guard */ statement_timeout = '60s';
SET /* wiki_btree_bloat_guard */ lock_timeout = '5s';

WITH cand AS (
    SELECT s.schemaname,
           s.relname,
           s.indexrelname,
           s.indexrelid,
           s.idx_scan,
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
      AND i.relpages > 128
),
raw_stat AS (
    SELECT c.*,
           current_setting('block_size')::int AS block_size,
           st.index_size,
           st.leaf_pages,
           st.empty_pages,
           st.deleted_pages,
           st.avg_leaf_density,
           st.leaf_fragmentation
    FROM cand c
    CROSS JOIN LATERAL pgstatindex(c.indexrelid::regclass) st
),
calc AS (
    SELECT r.*,
           CASE WHEN r.leaf_pages = 0 THEN 0::bigint
                ELSE GREATEST(0, r.leaf_pages * r.block_size
                                 * (1 - r.avg_leaf_density / r.fillfactor))::bigint
           END AS underfilled_leaf_bytes,
           ((r.empty_pages + r.deleted_pages) * r.block_size)::bigint AS dead_page_bytes
    FROM raw_stat r
)
SELECT /* wiki_btree_bloat_candidates */
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    fillfactor,
    index_size,
    leaf_pages,
    empty_pages,
    deleted_pages,
    avg_leaf_density,
    leaf_fragmentation,
    underfilled_leaf_bytes,
    dead_page_bytes,
    underfilled_leaf_bytes + dead_page_bytes AS wasted_bytes,
    round((underfilled_leaf_bytes + dead_page_bytes)::numeric
          / NULLIF(index_size, 0), 3) AS bloat_ratio,
    (leaf_pages > 0 AND leaf_fragmentation > 30.0) AS high_fragmentation,
    (index_size > 1048576
     AND (underfilled_leaf_bytes + dead_page_bytes)::numeric
         / NULLIF(index_size, 0) > 0.30) AS is_bloated
FROM calc
ORDER BY wasted_bytes DESC;
```

Notes on the query, all confirmed by running it on the pinned checkout:

- The `i.relpages > 128` predicate keeps the full-fork `pgstatindex` scan away from indexes that cannot clear the 1 MiB floor anyway (128 blocks x 8 KiB = 1 MiB). `relpages` may be stale or zero before the first `ANALYZE`, so it is only a pre-filter.
- The `CASE WHEN r.leaf_pages = 0` guard is required, not cosmetic. `pgstatindex` returns `NaN` for both ratios when `leaf_pages = 0`, and PostgreSQL orders `NaN` above every other float value, so `NaN > 0.30` is true ([float.h#float8_gt](../../../raw/postgres-12/src/include/utils/float.h#L337-L341), [pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)). Without the guard, a metapage-only index reports `bloat_ratio = NaN` with `is_bloated = t`.
- `NaN` cannot be detected with `x <> x` here: PostgreSQL defines `NaN = NaN` as true and `NaN <> NaN` as false ([float.h#float8_eq](../../../raw/postgres-12/src/include/utils/float.h#L289-L293), [float.h#float8_ne](../../../raw/postgres-12/src/include/utils/float.h#L301-L305)). Guarding on `leaf_pages` is both simpler and exactly equivalent, because `max_avail` is nonzero if and only if at least one live leaf was counted.
- `lock_timeout` applies to each lock wait inside the statement. `pgstatindex` takes only `AccessShareLock`, which conflicts solely with `AccessExclusiveLock`, so the query waits only behind operations such as a non-concurrent `REINDEX` ([lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L69), [maintenance.sgml#reindex-locks](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L890-L896)).
- One statement over every candidate index is convenient but fragile at scale: a single lock wait or the `statement_timeout` aborts the whole scan. For large databases, drive the same arithmetic from a client loop, one index per statement.

### How the thresholds work

`avg_leaf_density` is a percentage of usable leaf-page space, so it is directly comparable to `fillfactor`. A freshly built index lands just above its configured fillfactor. `_bt_pagestate` sets the build's per-page free-space target to `RelationGetTargetPageFreeSpace(index, BTREE_DEFAULT_FILLFACTOR)`, which honors the `fillfactor` reloption and expands to `BLCKSZ * (100 - fillfactor) / 100` ([nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L709-L730), [rel.h#RelationGetTargetPageFreeSpace](../../../raw/postgres-12/src/include/utils/rel.h#L293-L309)). That is a soft limit: `_bt_buildadd` starts a new page when the incoming tuple does not fit at all, or when free space has already fallen below the target *and* the page holds more than one data item, so the finished page usually sits slightly fuller than the nominal target ([nbtsort.c#_bt_buildadd-soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L888-L900)). Measured on the pinned checkout, a 300,000-row sorted build with `fillfactor = 90` reports `avg_leaf_density = 90.05` and with `fillfactor = 50` reports `49.82`.

If `avg_leaf_density` has drifted far below the configured fillfactor, the live entries are spread over more leaf pages than a rebuild would need, and step 3 converts that gap into recoverable pages. `empty_pages` and `deleted_pages` never contribute to `avg_leaf_density`, so they are accounted separately in step 4. `leaf_fragmentation` is a physical-ordering measure, not a space measure, which is why it stays a side signal; the v12 manual is the basis for treating it as an access-speed concern rather than a size concern ([maintenance.sgml#freshly-constructed](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888)).

`idx_scan` is deliberately outside the ratio. An index with large `wasted_bytes` and heavy `idx_scan` is a better rebuild target than an equally bloated index nothing reads. A zero `idx_scan` does not mean the index is useless: it may still enforce a unique or foreign-key constraint, and the counter is only as old as the last statistics reset.

### Why 30%

PostgreSQL defines neither a `bloat_ratio` nor a 30 % threshold. Three source-backed observations support 30 % as a starting default.

1. The v12 manual describes a bloated index operationally as one that "contains many empty or nearly-empty pages" and points at `REINDEX` to write a new index without the dead pages ([reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57)). `routine-reindex` names the concrete pattern: completely empty B-tree pages are reclaimed, but a page whose keys are all-but-a-few deleted stays allocated, so usage patterns that delete most keys in each range waste space and warrant periodic reindexing ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874)). A 30 % ratio means roughly a third of the bytes are recoverable, which matches that description.
2. The planner already prices physical index size. `get_relation_info` sets `IndexOptInfo.pages` from the index's live block count for a non-partial index, so bloat is priced without waiting for `ANALYZE` ([plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L398)). `genericcostestimate` then estimates pages touched pro rata as `ceil(numIndexTuples * index->pages / index->tuples)` ([selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5777-L5780)), and `btcostestimate` adds `(index->tree_height + 1) * 50.0 * cpu_operator_cost` explicitly so that bloated indexes do not look as cheap as unbloated ones on single-leaf lookups ([selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6105-L6116)). For underfilled leaves the pro-rata estimate tracks reality, because the entries really are spread across those pages; for `empty_pages` and `deleted_pages` it overestimates, because those pages hold no entries yet still inflate `index->pages`. A 30 % ratio is where that distortion becomes material.
3. A `bloat_ratio` of 0.30 means the index occupies about 1.43 times the size it needs (`1 / (1 - 0.30)`). At the 1 MiB floor that is only ~430 KiB, which is why the floor exists; on a 1 GiB index it is ~300 MiB.

Because the threshold is not in the code, sites with cheap storage can raise it and sites under buffer pressure can lower it.

### Two systematic false positives

**Random-key indexes.** The v12 leaf fillfactor is applied during index build and when splitting a *rightmost* page. Non-rightmost leaf splits use `fillfactormult = 0.50`, an equal division, and an all-duplicates page splits at `BTREE_SINGLEVAL_FILLFACTOR`, 96 % ([nbtree.h#fillfactor-comment](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171), [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330)). An index fed random keys therefore settles well below its fillfactor with no deletions at all, so the heuristic's "waste" is structural. The arithmetic is still correct as a size prediction, but a `REINDEX` only buys time. On the pinned checkout, an index populated by 300,000 random-key inserts reported `avg_leaf_density = 63.63` and `leaf_fragmentation = 50` while never having had a row deleted; `REINDEX` took it to `90.05` with zero fragmentation and shrank the file from 9,560,064 to 6,758,400 bytes, 2,801,664 bytes against a predicted 2,789,097 (a 0.45 % under-estimate); 150,000 further random inserts pushed density back to `67.65` and fragmentation to `49.97`. Compare `avg_leaf_density` against the value the same index reached after its last rebuild, not against `fillfactor`, when keys arrive out of order.

**Deleted pages already queued for reuse.** `deleted_pages` are not permanently lost. `_bt_page_recyclable` accepts a deleted page once `btpo.xact` precedes `RecentGlobalXmin`, `btvacuumpage` hands already-recyclable pages to `RecordFreeIndexPage`, `btvacuumscan` then calls `IndexFreeSpaceMapVacuum`, and `_bt_getbuf(P_NEW)` asks `GetFreeIndexPage` before extending the relation ([nbtpage.c#_bt_page_recyclable](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L935-L963), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1165-L1172), [nbtree.c#btvacuumscan-fsm](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1088-L1095), [nbtpage.c#_bt_getbuf-free-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L798-L812)). The unlink step stamps `btpo.xact` with `ReadNewTransactionId()` as it flips `BTP_HALF_DEAD` to `BTP_DELETED`, so the VACUUM that deletes a page can never also record it; a later VACUUM does ([nbtpage.c#_bt_unlink_halfdead_page-xact](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L2002-L2006)). The measured two-stage behaviour is in [Exact-pin measurements](#exact-pin-measurements). Where `dead_page_bytes` dominates `wasted_bytes` and the index is still taking inserts, prefer another VACUUM and a re-measure over an immediate `REINDEX`.

### Exact-pin measurements

Every row below is output from the query above, run against the exact pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2) built out of tree, on 300,000-row `(int, text)` fixtures with `fillfactor` at its default of 90 unless stated.

| Fixture | `leaf_pages` | `deleted_pages` | `avg_leaf_density` | `leaf_fragmentation` | `bloat_ratio` | `is_bloated` |
|---|---|---|---|---|---|---|
| Sorted build, default fillfactor | 820 | 0 | 90.05 | 0 | 0.000 | f |
| Sorted build, `fillfactor = 50` | 1486 | 0 | 49.82 | 0 | 0.004 | f |
| Random-key inserts, nothing ever deleted | 1162 | 0 | 63.63 | 50 | 0.292 | f |
| Sorted build, every key except each 10th deleted, after VACUUM | 820 | 0 | 9.27 | 0 | 0.892 | t |
| Sorted build, contiguous 83 % key range deleted, after VACUUM | 137 | 685 | 89.83 | 0 | 0.831 | t |

Three behaviours the table does not show, each reproduced on the same server:

- **VACUUM makes the waste visible, it does not create it.** After deleting every key except each tenth, the index still reported `avg_leaf_density = 90.05` with `deleted_pages = 0`; `VACUUM` dropped density to `9.27` with `leaf_pages` unchanged at 820 and still no page deleted. This is the `routine-reindex` "all but a few keys on a page have been deleted" pattern, and it means a bloat scan run before vacuum catches up reports a healthy index.
- **A deleted page is reused only after a later VACUUM.** On a fixture with `autovacuum_enabled = false`, the deleting `VACUUM` left 137 live leaves, 685 deleted pages, and a 6,758,400-byte file. Inserting 40,000 rows with no further VACUUM took the live leaves to 286, left `deleted_pages` at 685, and grew the file to 7,979,008 bytes: not one deleted page was reused. A second `VACUUM` changed no counter but recorded the now-aged pages, and the next 120,000 inserts consumed 448 of them (`deleted_pages` 685 -> 237) with the file size unchanged. On an equivalent fixture with autovacuum left enabled, `pg_stat_user_tables` recorded one autovacuum between the two measurements and reuse appeared without a second manual `VACUUM`, so in production the later pass usually arrives on its own.
- **The `NaN` guard fires on a real relation.** `pgstatindex` on a metapage-only index (`CREATE TABLE metaonly (id int PRIMARY KEY)`) returns `leaf_pages = 0` with `NaN` for both ratios. The unguarded expression yields `bloat_ratio = NaN`, `is_bloated = t`, and `high_fragmentation = t`; the guarded expression yields `0`, `f`, and `f`.

### Limits and caveats

- `pgstatindex` is B-tree only, and its `IS_INDEX` test requires `relkind = 'i'`, so a partitioned parent index (`relkind = 'I'`) is rejected with `relation "..." is not a btree index` ([pgstatindex.c#relkind-am-macros](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L71), [pgstatindex.c#AM-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228), [pgstattuple.out#partitioned-index-error](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L165-L166)). The query never reaches that error because `pg_stat_all_indexes` only reports indexes whose table `relkind` is `r`, `t`, or `m`, and a partitioned table is `p`; leaf partitions and their indexes are included normally. It also rejects another session's temporary relations ([pgstatindex.c#other-temp-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L230-L238)).
- The result is not an instantaneous snapshot. Blocks are read and share-locked one at a time while the index keeps changing ([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [pgstattuple.sgml#not-instantaneous](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L275-L280)).
- The formula ignores internal pages on both sides. `internal_pages` are never counted as waste, and the savings a rebuild makes on upper levels are not estimated, so `bloat_ratio` is conservative. Non-leaf pages use a fixed 70 % fillfactor ([nbtree.h#fillfactor-constants](../../../raw/postgres-12/src/include/access/nbtree.h#L168-L171), [nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L709-L730)).
- An all-zero page is classified as `internal_pages`, not as waste, because its `btpo_flags` are zero and the classification chain falls through. v12 states such pages can exist, for example when a backend extends the relation and crashes before logging the new page ([nbtpage.c#_bt_page_recyclable](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L945-L952), [pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).
- `fillfactor` and `avg_leaf_density` use slightly different denominators: the build target is free space against `BLCKSZ`, while density divides free space by the smaller `pd_special - SizeOfPageHeaderData`. On its own that would put a `fillfactor = 90` rebuild marginally *below* 90; the soft page-full test then overshoots the target, and the measured result is `90.05`. Both effects are far below any sensible threshold, but they are why a rebuild does not land on exactly `90.00`.
- `idx_scan` is a per-transaction snapshot of collector data and is lost on `pg_stat_reset` ([monitoring.sgml#stats-snapshot](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L242-L260)).
- Remediation is not free. Plain `REINDEX` takes `ACCESS EXCLUSIVE`; v12's `REINDEX INDEX CONCURRENTLY` needs only `SHARE UPDATE EXCLUSIVE` ([maintenance.sgml#reindex-locks](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L890-L896)). See [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md).
- The 30 % ratio, the 1 MiB floor, and the 30 % fragmentation flag are operational choices, not PostgreSQL defaults.

### Build, generated-file, and test boundary

`pgstattuple` is an out-of-core contrib module: its `MODULE_big`, the three `.c` files, the extension SQL scripts installed as `DATA`, and a single `REGRESS = pgstattuple` suite all come from its own makefile ([pgstattuple/Makefile](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L20)). Nothing in the heuristic depends on a generated header beyond the ordinary catalog build: `pg_options_to_table`'s OID 2289 lives in `pg_proc.dat`, which `genbki.pl` turns into the `*_d.h` catalog headers during `generated-header-symlinks` ([catalog/Makefile#generated-headers](../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L90)). `pg_stat_all_indexes` and `pg_stat_user_indexes` are created by `initdb` from `system_views.sql`, not by `genbki.pl` ([system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672)).

Test coverage for the inputs is thin and worth knowing before trusting a threshold. The `pgstattuple` regression suite exercises all four `pgstatindex` input types, the empty-index output including both `NaN` ratios, and the rejection paths for GIN, hash, partitioned, view, and foreign-table arguments; the file's own comment says platform-independent cases are hard, so it never populates a B-tree and asserts a non-`NaN` `avg_leaf_density` or a nonzero `leaf_fragmentation` ([pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119), [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236)). No in-tree test covers the heuristic itself; the numbers in [Exact-pin measurements](#exact-pin-measurements) are this page's substitute.

## Context Reviewed

- `contrib/pgstattuple/pgstatindex.c` - `pgstatindex` entry points, `BTIndexStat`, relation checks, page classification, output formulas.
- `contrib/pgstattuple/pgstattuple--1.4.sql`, `pgstattuple--1.4--1.5.sql`, `pgstattuple.control`, `Makefile` - overloads, `1.5` default version, `pg_stat_scan_tables` grants, build and regression wiring.
- `contrib/pgstattuple/sql/pgstattuple.sql`, `contrib/pgstattuple/expected/pgstattuple.out` - regression coverage and the checked-in `NaN` output.
- `doc/src/sgml/pgstattuple.sgml` - documented columns, metapage accounting, non-instantaneous warning, access restriction.
- `src/backend/storage/page/bufpage.c` - `PageGetFreeSpace`.
- `src/include/access/nbtree.h` - fillfactor constants and the comment on where each applies.
- `src/backend/access/nbtree/nbtsort.c`, `nbtsplitloc.c` - build fill target, the soft page-full test, and split-point fillfactor selection.
- `src/backend/access/nbtree/nbtpage.c`, `nbtree.c`, `nbtinsert.c`, `nbtutils.c` - page recyclability, FSM record and reuse, `LP_DEAD` marking, page compaction.
- `src/backend/access/common/reloptions.c`, `src/include/utils/rel.h` - `fillfactor` registration and `RelationGetTargetPageFreeSpace`.
- `src/backend/catalog/system_views.sql`, `src/include/catalog/pg_class.h`, `src/include/catalog/pg_proc.dat`, `src/backend/catalog/Makefile` - statistics views, catalog columns, `pg_options_to_table`, generated-header path.
- `src/backend/utils/adt/selfuncs.c`, `src/backend/optimizer/util/plancat.c` - how index size and tree height reach the cost model.
- `src/backend/utils/misc/guc.c`, `src/include/utils/float.h`, `src/backend/storage/lmgr/lock.c`, `src/backend/storage/buffer/freelist.c` - `block_size`/timeout contexts, `NaN` ordering and equality, lock conflicts, `BAS_BULKREAD` ring size.
- `doc/src/sgml/ref/reindex.sgml`, `doc/src/sgml/maintenance.sgml`, `doc/src/sgml/monitoring.sgml` - bloat definition, reindex guidance and locks, statistics-snapshot semantics.

## Evidence Map

| Claim | Source |
|---|---|
| `pgstatindex(regclass)` returns the ten documented columns; the `text` overload is kept for backward compatibility | [pgstattuple--1.4.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L74), [pgstatindex.c#interface-note](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L54) |
| The v12 extension default version is `1.5`, whose script defines the `_v1_5` entry points | [pgstattuple.control:3](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L3), [pgstattuple--1.4--1.5.sql#v1_5-pgstatindexbyid](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92) |
| `EXECUTE` on `pgstatindex` is revoked from `PUBLIC` and granted to `pg_stat_scan_tables` | [pgstattuple--1.4--1.5.sql#pgstatindex-regclass-acl](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L91-L92), [pgstattuple.sgml#access](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L15-L24) |
| The function opens the index with `AccessShareLock` and scans every non-metapage block through a `BAS_BULKREAD` ring | [pgstatindex.c#pgstatindexbyid_v1_5](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L203-L213), [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [freelist.c#BAS_BULKREAD](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L559-L561) |
| `deleted_pages` are `P_ISDELETED`; `empty_pages` are the remaining `P_IGNORE` (half-dead) pages; anything not deleted, ignored, or leaf counts as internal | [pgstatindex.c#page-classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310) |
| Only live leaves add `pd_special - SizeOfPageHeaderData` to `max_avail` and `PageGetFreeSpace` to `free_space` | [pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L300) |
| `index_size` counts the metapage plus all classified pages times `BLCKSZ` | [pgstatindex.c#index-size](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L336-L341), [pgstattuple.sgml#index_size-metapage](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L273) |
| `avg_leaf_density = 100 - free_space / max_avail * 100`, else `NaN` | [pgstatindex.c#density-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351) |
| `leaf_fragmentation = fragments / leaf_pages * 100`, else `NaN`; a fragment is a live leaf whose `btpo_next` is not `P_NONE` and is lower than its own block | [pgstatindex.c#fragment-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L302-L307), [pgstatindex.c#fragmentation-formula](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356) |
| `PageGetFreeSpace` is `pd_upper - pd_lower - sizeof(ItemIdData)`, so dead entries still read as occupied | [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L580-L599) |
| Dead index entries are removed by `_bt_vacuum_one_page` at insert time or `_bt_delitems_vacuum` during VACUUM, after `_bt_killitems` sets `LP_DEAD` | [nbtutils.c#_bt_killitems-mark-dead](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1786-L1797), [nbtinsert.c#_bt_vacuum_one_page](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2244-L2281), [nbtpage.c#_bt_delitems_vacuum](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L985-L999) |
| B-tree default `fillfactor` is 90, minimum 10, non-leaf 70, single-value 96 | [nbtree.h#fillfactor-constants](../../../raw/postgres-12/src/include/access/nbtree.h#L168-L171), [reloptions.c#btree-fillfactor](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186) |
| The build fill target honors the reloption and expands to `BLCKSZ * (100 - fillfactor) / 100`; the page-full test is a soft limit | [nbtsort.c#_bt_pagestate](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L709-L730), [rel.h#RelationGetTargetPageFreeSpace](../../../raw/postgres-12/src/include/utils/rel.h#L293-L309), [nbtsort.c#_bt_buildadd-soft-limit](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L888-L900) |
| Fillfactor applies to builds and rightmost splits; other leaf splits divide equally and all-duplicate splits use 96 % | [nbtree.h#fillfactor-comment](../../../raw/postgres-12/src/include/access/nbtree.h#L159-L171), [nbtsplitloc.c#fillfactor-selection](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L277-L330) |
| Deleted pages become recyclable, are recorded in the FSM by a later VACUUM, and are handed back out by `_bt_getbuf(P_NEW)`; the deleting VACUUM cannot record them because the unlink stamps `btpo.xact` with `ReadNewTransactionId()` | [nbtpage.c#_bt_page_recyclable](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L935-L963), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1165-L1172), [nbtree.c#btvacuumscan-fsm](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1088-L1095), [nbtpage.c#_bt_getbuf-free-page](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L798-L812), [nbtpage.c#_bt_unlink_halfdead_page-xact](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L2002-L2006) |
| All-zero B-tree pages can exist in v12 | [nbtpage.c#_bt_page_recyclable](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L945-L952) |
| `pg_stat_all_indexes` sources `idx_scan` from `pg_stat_get_numscans(I.oid)` and restricts to `relkind IN ('r','t','m')`; `pg_stat_user_indexes` filters it by schema | [system_views.sql#pg_stat_all_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672), [system_views.sql#pg_stat_user_indexes](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L679-L682) |
| Collector statistics are frozen per transaction | [monitoring.sgml#stats-snapshot](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L242-L260) |
| `pg_class` carries `relpages` and `reloptions`; `pg_options_to_table` expands the option array | [pg_class.h#relpages-reltuples](../../../raw/postgres-12/src/include/catalog/pg_class.h#L58-L63), [pg_class.h#reloptions](../../../raw/postgres-12/src/include/catalog/pg_class.h#L133-L134), [pg_proc.dat#pg_options_to_table](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L3603-L3608) |
| `block_size` is a `PGC_INTERNAL` preset fixed to `BLCKSZ`; `statement_timeout` and `lock_timeout` are `PGC_USERSET` | [guc.c#block_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888), [guc.c#statement_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396) |
| `NaN` sorts above every other float value, and `NaN = NaN` is true, so the guard must test `leaf_pages`, not self-inequality | [float.h#float8_gt](../../../raw/postgres-12/src/include/utils/float.h#L337-L341), [float.h#float8_eq](../../../raw/postgres-12/src/include/utils/float.h#L289-L293), [float.h#float8_ne](../../../raw/postgres-12/src/include/utils/float.h#L301-L305) |
| `pgstatindex` requires `relkind = 'i'` and a B-tree AM, and rejects other sessions' temp relations | [pgstatindex.c#relkind-am-macros](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L71), [pgstatindex.c#AM-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228), [pgstatindex.c#other-temp-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L230-L238) |
| `AccessShareLock` conflicts only with `AccessExclusiveLock`; plain `REINDEX` takes `ACCESS EXCLUSIVE` while `CONCURRENTLY` takes `SHARE UPDATE EXCLUSIVE` | [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L69), [maintenance.sgml#reindex-locks](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L890-L896) |
| The planner prices a non-partial index from its live block count, pro-rates pages touched, and adds a height-based bloat charge | [plancat.c#get_relation_info](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L398), [selfuncs.c#genericcostestimate-numIndexPages](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L5777-L5780), [selfuncs.c#btcostestimate-bloat-charge](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6105-L6116) |
| The manual defines bloat as many empty or nearly-empty pages and names the all-but-a-few-keys-deleted pattern | [reindex.sgml#bloat](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L57), [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L866-L874) |
| A freshly built B-tree is faster to access because logically adjacent pages are usually physically adjacent | [maintenance.sgml#freshly-constructed](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L882-L888) |
| Regression coverage stops at entry points, empty-index `NaN`, and rejection paths | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119), [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236), [pgstattuple.out#empty-index](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L52), [pgstattuple.out#partitioned-index-error](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L165-L166) |
| Contrib build wiring and the catalog generated-header path | [pgstattuple/Makefile](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L20), [catalog/Makefile#generated-headers](../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L90) |

## Open Questions

- The 0.30 ratio, the 1 MiB floor, and the 30 % fragmentation flag remain operator calibration, not source-derived constants. Nothing in the pinned checkout ties a specific ratio to a measured regression.
- The right reference density for a random-key index is the density that index reached after its own last rebuild. This page shows that such an index settles below `fillfactor` on the pinned checkout, but it does not derive a closed-form expected steady-state density from source.
- No source-backed rule separates "deleted pages that will be reused by ongoing inserts" from "deleted pages that will sit idle". The two-VACUUM behaviour is reproduced above, but the decision still needs a second measurement over time rather than a single reading.
- The heuristic does not estimate internal-page savings from a rebuild, so it understates recoverable space. Comparing `tree_level` before and after a trial `REINDEX INDEX CONCURRENTLY` would quantify it, at the cost of a second pass and extra locking.
- `pgstatindex` reads every block. Sequencing a full-database sweep so it neither times out nor competes with production I/O is left to the operator; `pg_class.relpages` and `pg_statio_user_indexes` read patterns are the cheap pre-screens.
- All-zero B-tree pages are counted as `internal_pages` rather than as waste. How often that happens in practice, and by how much it understates `bloat_ratio`, is not determinable from source alone.

## Source References

- [pgstatindex.c](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c)
- [pgstattuple--1.4.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql)
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql)
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control)
- [pgstattuple/Makefile](../../../raw/postgres-12/contrib/pgstattuple/Makefile)
- [pgstattuple.sql](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql)
- [pgstattuple.out](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out)
- [pgstattuple.sgml](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml)
- [bufpage.c](../../../raw/postgres-12/src/backend/storage/page/bufpage.c)
- [nbtree.h](../../../raw/postgres-12/src/include/access/nbtree.h)
- [nbtsort.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c)
- [nbtsplitloc.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c)
- [nbtpage.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c)
- [nbtree.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c)
- [nbtinsert.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c)
- [nbtutils.c](../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c)
- [reloptions.c](../../../raw/postgres-12/src/backend/access/common/reloptions.c)
- [rel.h](../../../raw/postgres-12/src/include/utils/rel.h)
- [system_views.sql](../../../raw/postgres-12/src/backend/catalog/system_views.sql)
- [catalog/Makefile](../../../raw/postgres-12/src/backend/catalog/Makefile)
- [pg_class.h](../../../raw/postgres-12/src/include/catalog/pg_class.h)
- [pg_proc.dat](../../../raw/postgres-12/src/include/catalog/pg_proc.dat)
- [plancat.c](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c)
- [selfuncs.c](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c)
- [guc.c](../../../raw/postgres-12/src/backend/utils/misc/guc.c)
- [float.h](../../../raw/postgres-12/src/include/utils/float.h)
- [lock.c](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c)
- [freelist.c](../../../raw/postgres-12/src/backend/storage/buffer/freelist.c)
- [reindex.sgml](../../../raw/postgres-12/doc/src/sgml/ref/reindex.sgml)
- [maintenance.sgml](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml)
- [monitoring.sgml](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml)

## Navigation

- [PostgreSQL 12.2 landing page](../index.md)
- [PostgreSQL 12 Codebase Navigation Guide](../codebase-navigation-guide.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](pgstatindex-sample-variant-proposal.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
- [versions](../../versions.md)
