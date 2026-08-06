# PostgreSQL 18

## Source Pin

- Branch: `REL_18_STABLE`
- Commit: `6cb307251c5c6261286c1566496920976640108e`
- Status: `primary`
- Source path: `raw/postgres-18/`
- Added: 2026-04-30

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`.

- [PostgreSQL 18 Codebase Navigation Guide (unverified)](codebase-navigation-guide.md) - Mandatory root-level question-style map for navigating the pinned v18 source tree: layout, SQL statement flow, utility dispatch, generated/catalog artifacts, key structs, contrib boundaries, tests, and docs.

## Questions

### Indexing

- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 18 (unverified)](questions/indexing/create-index-concurrently.md) - Comprehensive walkthrough of the v18 `CREATE INDEX CONCURRENTLY` path: utility dispatch and restrictions, four internal transactions, two heap scans, three waits, table/session `ShareUpdateExclusiveLock`, `indislive`/`indisready`/`indisvalid` progression, failure outcomes, tests, and changes from PostgreSQL 17 (`pg_index` TOAST snapshot handling, parallel GIN build, virtual-generated-column rejection, and temporal `WITHOUT OVERLAPS` shared plumbing).
- [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](questions/indexing/avg-leaf-density-during-vacuum.md) - Reviewed design and evidence-backed cons for computing `pgstatindex`-style `avg_leaf_density` inside B-tree VACUUM, with metapage/statistics storage options and explicit caveats for skipped scans and page deletion.

### Storage and Vacuum

- [Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)](questions/storage-and-vacuum/extension-hooks-vacuum-autovacuum.md) - Enumerates every extension point on the VACUUM/ANALYZE and autovacuum code paths in v18: in-process hook variables, table/index AM and FDW callbacks, and adjacent extension surfaces, with the manual-vs-autovacuum coverage of each.
- [Usage of NUM_BUFFER_PARTITIONS in PostgreSQL 18 (unverified)](questions/storage-and-vacuum/num-buffer-partitions.md) - How PostgreSQL 18 uses the fixed `NUM_BUFFER_PARTITIONS` count for shared buffer mapping hash partitions, `BufferMapping` LWLocks, read/allocation, extension, and invalidation paths, plus the macro's source commit history (16->128).

### Replication and WAL

- [How Bi-Directional Logical Replication Works in PostgreSQL 18, and New Logical Replication Features Since PostgreSQL 17 (unverified)](questions/replication-and-wal/bidirectional-logical-replication.md) - How two v18 nodes replicate to each other without loops via mutual `origin = none` subscriptions (apply-worker origin tagging into WAL, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, limitations), plus the new v18 conflict detection (`update_origin_differs`/`delete_origin_differs` and friends, logged and counted in `pg_stat_subscription_stats`); with a verified section on all logical replication features new since v17: conflict logging/stats, generated-column replication (`publish_generated_columns`), `streaming = parallel` default, alterable `two_phase`, the `max_active_replication_origins` and `idle_replication_slot_timeout` GUCs, `pg_createsubscriber` `--all`/`--clean`/`--enable-two-phase`, `pg_recvlogical` improvements, and contrib `pg_logicalinspect`.

### Observability

- [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](questions/observability/custom-cumulative-statistics.md) - How extension-defined `PgStat_KindInfo` entries register through `shared_preload_libraries`, store variable or fixed custom stats, flush and snapshot counters, and persist across clean shutdowns.
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](questions/observability/explain-analyze-buffers-output.md) - Detailed explanation of shared, local, temp, planning, serialization, parallel-worker, and I/O timing fields in `EXPLAIN (ANALYZE, BUFFERS)` output.
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 18 (unverified)](questions/observability/pg-stat-statements.md) - How `pg_stat_statements` hooks parse analysis, planning, execution, utility commands, shared memory, query-text files, reset/save paths, permissions, and the direct plus adjacent settings that affect its output.
- [Limiting Query Text Size in pg_stat_statements in PostgreSQL 18 (unverified)](questions/observability/pg-stat-statements-query-text-size.md) - Why v18 has no GUC to cap individual query-text length, that `track_activity_query_size` does not apply, full-length external-file storage, the wholesale-discard garbage collection edge path, and the `pg_stat_statements.max`/`showtext`/`left()` workarounds.
- [How track_activity_query_size Is Used in PostgreSQL 18 (unverified)](questions/observability/track-activity-query-size.md) - How PostgreSQL 18 uses the postmaster-context `track_activity_query_size` GUC to reserve per-backend activity text storage, truncate `pg_stat_activity.query`, clip multibyte text, report activity in deadlock/crash paths, and leave `pg_stat_statements.query` storage separate.

### Server Administration

- [GUC Default-Value Changes Since PostgreSQL 12 (unverified)](questions/server-administration/guc-default-changes-since-v12.md) - Summarizes settings present in both PostgreSQL 12 and 18 whose built-in defaults changed: eight semantic default changes plus the v18 `log_connections` type/default-spelling change that keeps disabled-by-default behavior.
- [Row-Level Security (RLS) in PostgreSQL 18: Implementation, Performance, Settings, and Fixes Since PostgreSQL 14 (unverified)](questions/server-administration/row-level-security-rls.md) - Fully reviewed rewrite/relcache/planner/executor walkthrough with command-layered policies, WCO order, recursion and RI boundaries, selectivity/cost, locks, partitions/views/COPY/replication, settings, and all five PostgreSQL 14-derived performance follow-ups. The scalar-subquery review covers lazy per-occurrence/per-outer-plan-execution InitPlans, residual filters, index runtime keys, lossy rechecks and rescans, selectivity tradeoffs, and volatility/folding hazards. Exact-pin tests reproduce those boundaries plus stale same-role cached results after membership-edge `inherit_option`, `BYPASSRLS`, superuser, and current-database-owner changes until `DISCARD PLANS`; the release-based history contains 21 scoped changes not already present in PostgreSQL 14.0.
