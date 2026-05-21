# PostgreSQL 18

## Source Pin

- Branch: `REL_18_STABLE`
- Commit: `6cb307251c5c6261286c1566496920976640108e`
- Status: `primary`
- Source path: `raw/postgres-18/`
- Added: 2026-04-30

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`.

## Questions

- [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](questions/avg-leaf-density-during-vacuum.md) - Reviewed design and evidence-backed cons for computing `pgstatindex`-style `avg_leaf_density` inside B-tree VACUUM, with metapage/statistics storage options and explicit caveats for skipped scans and page deletion.
- [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](questions/custom-cumulative-statistics.md) - How extension-defined `PgStat_KindInfo` entries register through `shared_preload_libraries`, store variable or fixed custom stats, flush and snapshot counters, and persist across clean shutdowns.
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](questions/explain-analyze-buffers-output.md) - Detailed explanation of shared, local, temp, planning, serialization, parallel-worker, and I/O timing fields in `EXPLAIN (ANALYZE, BUFFERS)` output.
- [Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)](questions/extension-hooks-vacuum-autovacuum.md) - Enumerates every extension point on the VACUUM/ANALYZE and autovacuum code paths in v18: in-process hook variables, table/index AM and FDW callbacks, and adjacent extension surfaces, with the manual-vs-autovacuum coverage of each.
