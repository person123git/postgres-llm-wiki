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

- [[v18/questions/avg-leaf-density-during-vacuum|Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)]] - Reviewed design and evidence-backed cons for computing `pgstatindex`-style `avg_leaf_density` inside B-tree VACUUM, with metapage/statistics storage options and explicit caveats for skipped scans and page deletion.
- [[v18/questions/custom-cumulative-statistics|How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)]] - How extension-defined `PgStat_KindInfo` entries register through `shared_preload_libraries`, store variable or fixed custom stats, flush and snapshot counters, and persist across clean shutdowns.
- [[v18/questions/extension-hooks-vacuum-autovacuum|Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)]] - Enumerates every extension point on the VACUUM/ANALYZE and autovacuum code paths in v18: in-process hook variables, table/index AM and FDW callbacks, and adjacent extension surfaces, with the manual-vs-autovacuum coverage of each.
