# PostgreSQL 19

## Source Pin

- Branch/tag: `master` (post-`REL_19_BETA1`)
- Commit: `cdae794af31b3e9cfc323fc654292d86fa746f77`
- Status: `active`
- Source path: `raw/postgres-19/`
- Added: 2026-05-30
- Repinned: 2026-06-26

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`. PostgreSQL 19 has no `REL_19_STABLE` branch yet, so the pin tracks the post-`REL_19_BETA1` `master` commit `cdae794a` (the v19 development line). Filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module, a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command with 42 feature-scope commits, and a commit-backed walkthrough of parallel autovacuum, autovacuum table scoring (now including the post-beta1 MXID-score division-by-zero fix `1f2297b5487`), and read-only scans setting all-visible VM bits. The 2026-06-26 repin from `9a60f295` to `cdae794a` found no new feature-scope commits for those three question topics; question pages record the relevant changed-range commit references and refreshed line anchors.

- [PostgreSQL 19 Codebase Navigation Guide (unverified)](codebase-navigation-guide.md) - Mandatory root-level question-style map for navigating the pinned v19 source tree: layout, SQL statement flow, utility dispatch, generated/catalog artifacts, key structs, contrib boundaries, tests, and docs.

## Questions

- [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](questions/pg-plan-advice.md) - Comprehensive explanation of the new `pg_plan_advice` contrib module: the core `pgs_mask` strategy-mask and five planner hooks it relies on, per-index disabling for index-specific advice, the advice language (tags, relation identifiers), plan-to-advice generation, advice enforcement, feedback/EXPLAIN output, GUCs, prepared-statement and plan-cache interaction, the round-trip test harness, and the full scoped source history: 20 core planner foundation/enabling/fix commits, 22 direct module commits, and test/doc/build support commits.
- [How the REPACK Command Works in PostgreSQL 19, and Its 42 Feature-Scope Commits (unverified)](questions/repack-command.md) - Comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (which absorbs `VACUUM FULL` and `CLUSTER`): the blocking rewrite path, the concurrent online rewrite via logical decoding (decoding `bgworker`, `pgrepack` output plugin, temporary replication slot, change spill/replay, lock upgrade and swap), the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, I/O impact (no cost-delay; the BULKREAD/BULKWRITE ring buffers), tests, and 42 feature-scope commits.
- [PostgreSQL 19 Autovacuum and VACUUM: Parallel Workers, Table Scoring, and Setting Pages All-Visible During Reads (unverified)](questions/autovacuum-parallel-scoring-visibility.md) - Comprehensive, commit-backed explanation of three independent v19 features: parallel autovacuum (the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, the autovacuum-vs-maintenance worker cap in `parallel_vacuum_compute_workers`, and `PVSharedCostParams` cost-delay propagation), autovacuum table scoring (`AutoVacuumScores`, the five `*_score_weight` GUCs and all-zero escape hatch, descending sort, and `pg_stat_autovacuum_scores`), and read-only query scans setting all-visible in the visibility map via on-access pruning (`ScanRelIsReadOnly` -> `SO_HINT_REL_READ_ONLY` -> `heap_page_prune_opt` -> `HEAP_PAGE_PRUNE_SET_VM`), with the VACUUM VM-skip benefit and the read-query no-extra-dirtying safety valve; includes per-feature source-commit history.
