# Wiki Index

This is the global catalog for the PostgreSQL engine wiki.

## Entry Points

- [versions](versions.md) - PostgreSQL version index and source pin manifest.
- [overview](overview.md) - Cross-version architecture overview.
- [log](log.md) - Chronological activity log.


## Version-Specific Pages

### PostgreSQL 18

- [v18/index](v18/index.md) - Primary version landing page. Source checkout pinned to `REL_18_STABLE` commit `6cb307251c5c6261286c1566496920976640108e`.
- [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](v18/questions/avg-leaf-density-during-vacuum.md) - Reviewed design and evidence-backed cons for computing `pgstatindex`-style `avg_leaf_density` during B-tree VACUUM, including metapage/statistics storage, scan-skip caveats, and empty-page deletion accuracy gaps.
- [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](v18/questions/custom-cumulative-statistics.md) - Explains custom cumulative statistics registration, variable and fixed custom stat storage, flushing, snapshots, reset/drop behavior, and clean-shutdown persistence in PostgreSQL 18.
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](v18/questions/explain-analyze-buffers-output.md) - Detailed field-by-field explanation of `EXPLAIN (ANALYZE, BUFFERS)` shared, local, temp, planning, serialization, parallel-worker, and I/O timing output in PostgreSQL 18.
- [Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)](v18/questions/extension-hooks-vacuum-autovacuum.md) - Catalogs every extension point on the VACUUM/ANALYZE and autovacuum paths: in-process hook variables, table and index AM and FDW callbacks, and adjacent surfaces (background workers, custom cumulative statistics), with manual-vs-autovacuum coverage.
- [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](v18/questions/pgstatindex-approx-sampling.md) - Explains why `pgstattuple_approx`'s heap visibility-map/FSM shortcut does not transfer to B-tree `pgstatindex`, scopes a separate approximate diagnostic, and catalogs current `pgstatindex` regression coverage.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 18 (unverified)](v18/questions/pgstatindex-sample-variant-proposal.md) - Designs a `pgstatindex_approx` that random-samples physical index blocks instead of reading all of them; covers the exact-vs-estimated field split, the ratio-vs-count estimator, extension wiring, a `pageinspect` SQL prototype, and pros/cons.
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 18 (unverified)](v18/questions/pg-stat-statements.md) - Explains the shared-preload extension's query-ID, normalization, hook, shared-hash, query-text file, readout, reset/save, permissions, and configuration behavior in PostgreSQL 18.
- [Limiting Query Text Size in pg_stat_statements in PostgreSQL 18 (unverified)](v18/questions/pg-stat-statements-query-text-size.md) - There is no v18 GUC to cap individual query-text length; `track_activity_query_size` does not apply, texts store full-length in the external file, and the only levers are `pg_stat_statements.max`, the wholesale-discard garbage collector, and read-time `showtext`/`left()`.
- [How track_activity_query_size Is Used in PostgreSQL 18 (unverified)](v18/questions/track-activity-query-size.md) - Explains how the postmaster-context GUC sizes backend activity-string shared memory, caps `pg_stat_activity.query`, interacts with `track_activities`, feeds deadlock/crash activity reporting, and does not truncate `pg_stat_statements.query`.
- [Usage of NUM_BUFFER_PARTITIONS in PostgreSQL 18 (unverified)](v18/questions/num-buffer-partitions.md) - Explains the shared buffer mapping hash partition count, `BufferMapping` LWLocks, normal lookup/allocation and invalidation paths, relation extension, wait events, test-coverage gap, and the full source commit history (16->128) in PostgreSQL 18.



### PostgreSQL 17.10

- [v17/index](v17/index.md) - Active version landing page. Source checkout pinned to `REL_17_STABLE` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- [PostgreSQL 17 Contrib Extensions (unverified)](v17/questions/contrib-extensions.md) - Lists all 53 control-file-backed contrib extensions in PostgreSQL 17, with cited explanations covering install scope, data types, index/search helpers, diagnostics, FDWs, procedural-language transforms, and SPI trigger examples.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)](v17/questions/pgstatindex-sample-variant-proposal.md) - Designs a `pgstatindex_approx` that random-samples physical B-tree blocks; covers exact-vs-estimated fields, pros/cons, extension wiring, and a `pageinspect` SQL prototype using PostgreSQL 17 source behavior.



### PostgreSQL 12.2

- [v12/index](v12/index.md) - Legacy version landing page. Source checkout pinned to `REL_12_STABLE` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- [Foreign-Key Join Optimization for Two-Table Joins (unverified)](v12/questions/fk-join-optimization-two-tables.md) - How the v12 planner uses foreign-key constraints when joining two tables.
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](v12/questions/explain-analyze-buffers-output.md) - Detailed field-by-field explanation of `EXPLAIN (ANALYZE, BUFFERS)` shared, local, temp, and I/O timing output in PostgreSQL 12.
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](v12/questions/pg-stat-statements.md) - Explains the v12 extension's in-extension query jumbling, hooks, shared-hash and external query-text storage, readout, permissions, persistence, and its four GUCs (`max`, `track`, `track_utility`, `save`) plus `shared_preload_libraries` and `track_io_timing`.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) - Designs a `pgstatindex_approx` that random-samples physical B-tree blocks in PostgreSQL 12; covers exact-vs-estimated fields, pros/cons, extension wiring, and a contrib `pageinspect` SQL prototype.



## Maintenance Tooling

- `scripts/recent_log` - recent wiki activity.
- `scripts/wiki_lint` - wiki health checks.

## Maintenance Notes

- Update this page whenever a wiki page is created or substantially changed.
- Keep version-specific entries tagged with their PostgreSQL major version.
- Prefer links to version landing pages, such as `vNN/index`, once versions exist.
