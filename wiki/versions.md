# PostgreSQL Versions

This page indexes the PostgreSQL versions covered by the wiki.

## Supported Versions

| Version | Status | Wiki Home | Branch | Pinned Commit | Coverage |
|---|---|---|---|---|---|
| 19 | active | [v19/index](v19/index.md) | `REL_19_BETA1` tag | `4b0bf0788b066a4ca1d4f959566678e44ec93422` | Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`; filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module (core `pgs_mask` strategy mask and planner hooks, per-index disabling, advice language, generation, enforcement, feedback/EXPLAIN, GUCs, tests) and its scoped source history (core planner-enabling commits, 22 direct module commits, and test/doc/build support commits), plus a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (blocking new-heap rewrite/swap, concurrent online rewrite via logical decoding with a decoding `bgworker`, the `pgrepack` output plugin, a temporary replication slot, change spill/replay, lock upgrade and swap, the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, and tests) with 40 feature-scope source-history commits; and a comprehensive, commit-backed walkthrough of three v19 autovacuum/VACUUM features (parallel autovacuum via the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, autovacuum table scoring via `AutoVacuumScores`/the five `*_score_weight` GUCs/`pg_stat_autovacuum_scores`, and read-only query scans setting pages all-visible in the visibility map through on-access pruning); all three v19 question histories are pinned to PostgreSQL 19 beta 1. |
| 18 | primary | [v18/index](v18/index.md) | `REL_18_STABLE` | `6cb307251c5c6261286c1566496920976640108e` | Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`; filed coverage includes B-tree VACUUM density design, `pgstatindex` approximation limits and tests, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, custom cumulative statistics, extension hooks for VACUUM/autovacuum, `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `track_activity_query_size` activity text storage, `NUM_BUFFER_PARTITIONS` buffer mapping usage, and GUC default-value changes since v12 through v18 (`effective_io_concurrency` and `log_connections` included). |
| 17 | active | [v17/index](v17/index.md) | `REL_17_STABLE` | `54eeefaedbee0385529f3edf321bb99e49232aaa` | Behavioral claims cite the matching pinned checkout under `raw/postgres-17/`; filed coverage includes the complete contrib extension inventory, explanations for all 53 control-file-backed contrib extensions, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, and a reviewed summary of the seven GUC default-value changes since v12 (old/new value, introducing major version, apply scope, exclusions, and test-coverage notes, grounded in `guc_tables.c`/`config.sgml` and the checkout's own commit history). |
| 12 | legacy | [v12/index](v12/index.md) | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` | Behavioral claims cite the matching pinned checkout under `raw/postgres-12/`; filed coverage includes foreign-key join selectivity, planner statistics sources (`pg_stat_all_tables`, `pg_stats`, and `pg_stats_ext` direct-use excluded; `pg_class`, `pg_statistic`, extended statistics catalogs, index/FK/constraint metadata, planner statistics hooks, and regression coverage included), `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `psql` environment variables and session timeout behavior, exact `pgstatindex` calculation behavior, planner penalties for bloated B-tree indexes through physical pages and tree height including the storage-manager path and I/O/CPU boundary for `RelationGetNumberOfBlocks()`, a proposed `avg_leaf_density` / `leaf_fragmentation` triage heuristic for REINDEX candidates, B-tree leaf density (60% vs 90%) impact on planner page-count costs and executor leaf-page walking, a density-versus-fragmentation comparison for index scan I/O with level estimates and cache/storage sensitivity, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, including the v12 `bt_metap` unsigned-`oldest_xact` overflow workaround, and a three-stage REINDEX triage heuristic that shortlists candidates from `pg_class`/`pg_stat_*`/`pg_relation_size()` signals, confirms density with gated `pgstatindex` runs, prioritizes by wasted bytes times scan frequency with `REINDEX (CONCURRENTLY)` execution notes, measures post-REINDEX improvement via size/shape deltas, per-scan counter rates (per-index counters survive both REINDEX forms, including the v12 `index_concurrently_swap` stats copy), and `EXPLAIN (ANALYZE, BUFFERS)` / `pg_stat_statements` before/after windows, and documents the cited rationale for excluding `leaf_fragmentation` from the REINDEX priority score. |

## Archived Versions

| Version | Removed On | Reason |
|---|---|---|

No PostgreSQL versions have been archived yet.

## Status Meanings

- `primary` - exactly one supported version; default target for new ingests and answers.
- `active` - kept close to the primary through active-version verification.
- `legacy` - preserved for reference, but not checked by default.
- `archived` - removed from active maintenance and kept under `wiki/_archive/`.

## Source Pin Rules

- Pins must be exact commit hashes, not floating branch names.
- Source checkouts must live under `raw/postgres-NN/`.
- Version landing pages must live under `wiki/vNN/index.md`.
- Generated runtime artifacts must live under `.wiki-runtime/`: caches under `.wiki-runtime/cache/` and logs under `.wiki-runtime/logs/`.

## Current Primary

- PostgreSQL 18
- Source path: `raw/postgres-18/`
- Branch: `REL_18_STABLE`
- Pinned commit: `6cb307251c5c6261286c1566496920976640108e`
