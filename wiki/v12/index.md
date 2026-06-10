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
- [Query Planner Statistics Sources in PostgreSQL 12 (unverified)](questions/query-planner-statistics-sources.md) - Explains that the v12 planner does not use `pg_stat_all_tables`, `pg_stats`, or `pg_stats_ext` directly; it uses `pg_class`, `pg_statistic`, `pg_statistic_ext`, `pg_statistic_ext_data`, index metadata, FK metadata, constraints, and explicit planner statistics hooks, with regression coverage notes.
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](questions/explain-analyze-buffers-output.md)
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](questions/pg-stat-statements.md)
- [psql Environment Variables and Timeout Settings in PostgreSQL 12 (unverified)](questions/psql-environment-variables-and-timeouts.md) - Lists psql-specific frontend variables, libpq connection/session environment variables inherited by `psql`, and the session impact of `statement_timeout` and `lock_timeout` via `SET`, `SET LOCAL`, and `PGOPTIONS`.
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](questions/how-pgstatindex-calculates-information.md) - Explains the exact v12 `pgstatindex` path: extension SQL surface, access checks, metapage fields, full non-metapage block scan, page classification, leaf-density and fragmentation formulas, and regression coverage gaps.
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](questions/bloated-indexes-query-planner.md) - Explains how v12 penalizes B-tree bloat indirectly through physical index pages, B-tree height, and page-cost modeling; scopes the penalties across access methods (page-count penalty shared via `genericcostestimate()`; measured tree-height charge B-tree-only, GiST/SP-GiST descent charges use a page-count-estimated height, hash has none, GIN/BRIN use separate models); expands on `RelationGetNumberOfBlocks()` and `index->pages`; proposes an `avg_leaf_density` / `leaf_fragmentation` REINDEX triage heuristic.
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](questions/leaf-density-60-vs-90-query-impact.md) - Analyzes planner page-count scaling, executor leaf-page walking, and the difference between point lookups and wide range scans.
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](questions/leaf-density-vs-fragmentation-index-scan-io.md) - Compares density-driven leaf-page count growth with fragmentation-driven physical-order penalties.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](questions/pgstatindex-sample-variant-proposal.md) - Designs a `pgstatindex_approx` that random-samples physical B-tree blocks.
- [Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified)](questions/index-bloat-reindex-heuristic.md) - Proposes a three-stage heuristic: shortlist from `pg_class`/`pg_stat_*`/`pg_relation_size()` signals (no page I/O), confirm density with targeted `pgstatindex` runs, then rank by wasted bytes times scan frequency and execute via `REINDEX (CONCURRENTLY)`. Includes how to measure the improvement afterward: size/`pgstatindex` shape deltas, per-scan counter rates (valid across both REINDEX forms because v12 preserves per-index counters, including the `index_concurrently_swap` stats copy), and `EXPLAIN (ANALYZE, BUFFERS)` / `pg_stat_statements` comparisons; plus a cited rationale for why `leaf_fragmentation` stays out of the priority score.
- [Comprehensive plan_cache_mode Analysis in PostgreSQL 12 (unverified)](questions/plan_cache_mode_analysis.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](questions/create-index-concurrently.md) - Comprehensive walkthrough of the v12 CIC implementation in `DefineIndex`: the four internal transactions, two heap scans, and three transaction waits; the `indislive`/`indisready`/`indisvalid` state-flag progression set non-transactionally by `index_set_state_flags`; plus a dedicated section mapping every step to the table locks (transaction- and session-level `ShareUpdateExclusiveLock`, the `WaitForLockers(ShareLock)` writer waits, and `WaitForOlderSnapshots`), with the `LockConflicts` rationale for why DML proceeds while another CIC/`VACUUM`/`ANALYZE`/DDL is blocked.
