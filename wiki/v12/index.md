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
- [Query Planner Statistics Sources in PostgreSQL 12 (unverified)](questions/query-planner-statistics-sources.md) - Explains that the v12 planner does not use `pg_stat_all_tables`; it uses `pg_class`, `pg_statistic`, `pg_statistic_ext`, `pg_statistic_ext_data`, index metadata, FK metadata, constraints, and explicit planner statistics hooks.
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](questions/explain-analyze-buffers-output.md)
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](questions/pg-stat-statements.md)
- [psql Environment Variables and Timeout Settings in PostgreSQL 12 (unverified)](questions/psql-environment-variables-and-timeouts.md) - Lists psql-specific frontend variables, libpq connection/session environment variables inherited by `psql`, and the session impact of `statement_timeout` and `lock_timeout` via `SET`, `SET LOCAL`, and `PGOPTIONS`.
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](questions/how-pgstatindex-calculates-information.md) - Explains the exact v12 `pgstatindex` path: extension SQL surface, access checks, metapage fields, full non-metapage block scan, page classification, leaf-density and fragmentation formulas, and regression coverage gaps.
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](questions/bloated-indexes-query-planner.md) - Explains how v12 penalizes B-tree bloat indirectly through physical index pages, B-tree height, and page-cost modeling; expands on `RelationGetNumberOfBlocks()` and `index->pages`.
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](questions/leaf-density-60-vs-90-query-impact.md) - Analyzes planner page-count scaling, executor leaf-page walking, and the difference between point lookups and wide range scans.
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](questions/leaf-density-vs-fragmentation-index-scan-io.md) - Compares density-driven leaf-page count growth with fragmentation-driven physical-order penalties.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](questions/pgstatindex-sample-variant-proposal.md) - Designs a `pgstatindex_approx` that random-samples physical B-tree blocks.
- [Comprehensive plan_cache_mode Analysis in PostgreSQL 12 (unverified)](questions/plan_cache_mode_analysis.md)
