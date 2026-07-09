# PostgreSQL 19

## Source Pin

- Branch/tag: `REL_19_STABLE`
- Commit: `01c544e1afb99bc2a76803870010b7cd2907f3b5`
- Status: `active`
- Source path: `raw/postgres-19/`
- Added: 2026-05-30
- Repinned: 2026-07-09

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`. PostgreSQL 19 now has a `REL_19_STABLE` branch (upstream `master` has moved on to `20devel`), so the pin tracks the `REL_19_STABLE` tip `01c544e1` (still stamped `19beta1`). Filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module, a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command with 45 feature-scope commits, and a commit-backed walkthrough of parallel autovacuum, autovacuum table scoring (including the post-beta1 MXID-score division-by-zero fix `1f2297b5487`), and read-only scans setting all-visible VM bits. The 2026-07-09 repin from the former `master` pin `cdae794a` to `REL_19_STABLE` `01c544e1` added three REPACK feature-scope commits (stored-generated-columns fix `fb284f2f9bd`, change-replay range-table init `5e450df50dc`, and the tree-wide `PG_MODULE_MAGIC_EXT` cleanup `da8889ccd7e`) and three `pg_plan_advice` module commits (FOREIGN_JOIN single-relation fix `89f5f860cc5`, a pgindent follow-up `ea203d371de`, and `da8889ccd7e`); it also picked up the release-note score-weight prefix typo fix `1a7fa06dbcd`. Shifted line anchors were refreshed across the range.

- [PostgreSQL 19 Codebase Navigation Guide (unverified)](codebase-navigation-guide.md) - Mandatory root-level question-style map for navigating the pinned v19 source tree: layout, SQL statement flow, utility dispatch, generated/catalog artifacts, key structs, contrib boundaries, tests, and docs.

## Questions

- [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](questions/pg-plan-advice.md) - Comprehensive explanation of the new `pg_plan_advice` contrib module: the core `pgs_mask` strategy-mask and five planner hooks it relies on, per-index disabling for index-specific advice, the advice language (tags, relation identifiers), plan-to-advice generation, advice enforcement, feedback/EXPLAIN output, GUCs, prepared-statement and plan-cache interaction, the round-trip test harness, and the full scoped source history: 20 core planner foundation/enabling/fix commits, 25 direct module commits, and test/doc/build support commits.
- [How the REPACK Command Works in PostgreSQL 19, and Its 45 Feature-Scope Commits (unverified)](questions/repack-command.md) - Comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (which absorbs `VACUUM FULL` and `CLUSTER`): the blocking rewrite path, the concurrent online rewrite via logical decoding (decoding `bgworker`, `pgrepack` output plugin, temporary replication slot, change spill/replay, lock upgrade and swap), the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, I/O impact (no cost-delay; the BULKREAD/BULKWRITE ring buffers), tests, and 45 feature-scope commits.
- [PostgreSQL 19 Autovacuum and VACUUM: Parallel Workers, Table Scoring, and Setting Pages All-Visible During Reads (unverified)](questions/autovacuum-parallel-scoring-visibility.md) - Comprehensive, commit-backed explanation of three independent v19 features: parallel autovacuum (the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, the autovacuum-vs-maintenance worker cap in `parallel_vacuum_compute_workers`, and `PVSharedCostParams` cost-delay propagation), autovacuum table scoring (`AutoVacuumScores`, the five `*_score_weight` GUCs and all-zero escape hatch, descending sort, and `pg_stat_autovacuum_scores`), and read-only query scans setting all-visible in the visibility map via on-access pruning (`ScanRelIsReadOnly` -> `SO_HINT_REL_READ_ONLY` -> `heap_page_prune_opt` -> `HEAP_PAGE_PRUNE_SET_VM`), with the VACUUM VM-skip benefit and the read-query no-extra-dirtying safety valve; includes per-feature source-commit history.
