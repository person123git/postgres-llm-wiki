# PostgreSQL 12.2

## Source Pin

- Branch: `REL_12_STABLE`
- Commit: `45b88269a353ad93744772791feb6d01bc7e1e42`
- Status: `legacy`
- Source path: `raw/postgres-12/`
- Added: 2026-05-02

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-12/`.

## Pages

- [Foreign-Key Join Optimization for Two-Table Joins (unverified)](questions/fk-join-optimization-two-tables.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](questions/explain-analyze-buffers-output.md)
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](questions/pg-stat-statements.md)
- [psql Environment Variables and Timeout Settings in PostgreSQL 12 (unverified)](questions/psql-environment-variables-and-timeouts.md) - Lists psql-specific frontend variables, libpq connection/session environment variables inherited by `psql`, and the session impact of `statement_timeout` and `lock_timeout` via `SET`, `SET LOCAL`, and `PGOPTIONS`.
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](questions/how-pgstatindex-calculates-information.md) - Explains the exact v12 `pgstatindex` path: extension SQL surface, access checks, metapage fields, full non-metapage block scan, page classification, leaf-density and fragmentation formulas, and regression coverage gaps.
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](questions/leaf-density-60-vs-90-query-impact.md) - Analyzes planner page-count scaling, executor leaf-page walking via `_bt_steppage` / `_bt_readnextpage` / `_bt_getbuf`, buffer manager effects, test coverage gaps, and the difference between point lookups and wide range scans for 60% vs 90% `avg_leaf_density`.
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](questions/leaf-density-vs-fragmentation-index-scan-io.md) - Compares density-driven leaf-page count growth with fragmentation-driven physical-order penalties, including density and fragmentation level estimates, planner visibility, executor leaf walking, and cache/storage sensitivity.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](questions/pgstatindex-sample-variant-proposal.md) - Includes a contrib `pageinspect` diagnostic SQL prototype and a v12 `bt_metap` unsigned-`oldest_xact` overflow workaround.
