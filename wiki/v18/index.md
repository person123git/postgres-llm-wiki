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
- [GUC Default-Value Changes Since PostgreSQL 12 (unverified)](questions/guc-default-changes-since-v12.md) - Summarizes settings present in both PostgreSQL 12 and 18 whose built-in defaults changed: eight semantic default changes plus the v18 `log_connections` type/default-spelling change that keeps disabled-by-default behavior.
- [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](questions/pgstatindex-approx-sampling.md) - Why `pgstattuple_approx`'s heap-only visibility-map/FSM shortcut cannot preserve `pgstatindex` semantics for B-tree pages, what a separate approximate index function could report, and what current tests cover.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 18 (unverified)](questions/pgstatindex-sample-variant-proposal.md) - Concrete design for a `pgstatindex_approx` that random-samples physical index blocks: exact columns, direct ratio estimates, scaled page counts, a `pageinspect` SQL prototype, pros/cons, and tests to add.
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 18 (unverified)](questions/pg-stat-statements.md) - How `pg_stat_statements` hooks parse analysis, planning, execution, utility commands, shared memory, query-text files, reset/save paths, permissions, and the direct plus adjacent settings that affect its output.
- [Limiting Query Text Size in pg_stat_statements in PostgreSQL 18 (unverified)](questions/pg-stat-statements-query-text-size.md) - Why v18 has no GUC to cap individual query-text length, that `track_activity_query_size` does not apply, full-length external-file storage, the wholesale-discard garbage collection edge path, and the `pg_stat_statements.max`/`showtext`/`left()` workarounds.
- [How track_activity_query_size Is Used in PostgreSQL 18 (unverified)](questions/track-activity-query-size.md) - How PostgreSQL 18 uses the postmaster-context `track_activity_query_size` GUC to reserve per-backend activity text storage, truncate `pg_stat_activity.query`, clip multibyte text, report activity in deadlock/crash paths, and leave `pg_stat_statements.query` storage separate.
- [Usage of NUM_BUFFER_PARTITIONS in PostgreSQL 18 (unverified)](questions/num-buffer-partitions.md) - How PostgreSQL 18 uses the fixed `NUM_BUFFER_PARTITIONS` count for shared buffer mapping hash partitions, `BufferMapping` LWLocks, read/allocation, extension, and invalidation paths, plus the macro's source commit history (16->128).
