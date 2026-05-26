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
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](questions/how-pgstatindex-calculates-information.md) - Explains the exact v12 `pgstatindex` path: extension SQL surface, access checks, metapage fields, full non-metapage block scan, page classification, leaf-density and fragmentation formulas, and regression coverage gaps.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](questions/pgstatindex-sample-variant-proposal.md) - Includes a contrib `pageinspect` diagnostic SQL prototype and a v12 `bt_metap` unsigned-`oldest_xact` overflow workaround.
