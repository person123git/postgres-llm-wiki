# PostgreSQL 19

## Source Pin

- Branch/tag: `REL_19_STABLE`
- Commit: `8055e3375aa1c2237181e06be26b05b964d18ed5`
- Status: `active`
- Source path: `raw/postgres-19/`
- Added: 2026-05-30
- Repinned: 2026-07-13

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`. PostgreSQL 19 has a `REL_19_STABLE` branch (upstream `master` has moved on to `20devel`), and the pin tracks its `8055e337` tip (still stamped `19beta1`). Filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module, a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command with 46 feature-scope commits, and a commit-backed walkthrough of parallel autovacuum, autovacuum table scoring (including the post-beta1 MXID-score division-by-zero fix `1f2297b5487`), and read-only scans setting all-visible VM bits. The 2026-07-13 repin reviewed 12 commits after `01c544e1`: `133eba078f7` removed ineffective REPACK discovery locks and hardened the final per-table open/recheck path; `e9eaeb04248` made on-access pruning update the FSM when it newly sets a page all-visible, preventing stale free-space data after VACUUM skips that page. No `pg_plan_advice` commit landed in the range. Shifted source anchors were refreshed.

- [PostgreSQL 19 Codebase Navigation Guide (unverified)](codebase-navigation-guide.md) - Mandatory root-level question-style map for navigating the pinned v19 source tree: layout, SQL statement flow, utility dispatch, generated/catalog artifacts, key structs, contrib boundaries, tests, and docs.

## Questions

- [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](questions/pg-plan-advice.md) - Comprehensive explanation of the new `pg_plan_advice` contrib module: the core `pgs_mask` strategy-mask and five planner hooks it relies on, per-index disabling for index-specific advice, the advice language (tags, relation identifiers), plan-to-advice generation, advice enforcement, feedback/EXPLAIN output, GUCs, prepared-statement and plan-cache interaction, the round-trip test harness, and the full scoped source history: 20 core planner foundation/enabling/fix commits, 25 direct module commits, and test/doc/build support commits.
- [How the REPACK Command Works in PostgreSQL 19, and Its 46 Feature-Scope Commits (unverified)](questions/repack-command.md) - Comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (which absorbs `VACUUM FULL` and `CLUSTER`): the blocking rewrite path, lock-free multi-table candidate discovery followed by locked per-table open/recheck, the concurrent online rewrite via logical decoding (decoding `bgworker`, `pgrepack` output plugin, temporary replication slot, change spill/replay, lock upgrade and swap), the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, I/O impact (no cost-delay; the BULKREAD/BULKWRITE ring buffers), tests, and 46 feature-scope commits.
- [PostgreSQL 19 Autovacuum and VACUUM: Parallel Workers, Table Scoring, and Setting Pages All-Visible During Reads (unverified)](questions/autovacuum-parallel-scoring-visibility.md) - Comprehensive, commit-backed explanation of three independent v19 features: parallel autovacuum (the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, the autovacuum-vs-maintenance worker cap in `parallel_vacuum_compute_workers`, and `PVSharedCostParams` cost-delay propagation), autovacuum table scoring (`AutoVacuumScores`, the five `*_score_weight` GUCs and all-zero escape hatch, descending sort, and `pg_stat_autovacuum_scores`), and read-only query scans setting all-visible in the visibility map via on-access pruning (`ScanRelIsReadOnly` -> `SO_HINT_REL_READ_ONLY` -> `heap_page_prune_opt` -> `HEAP_PAGE_PRUNE_SET_VM`), with the VACUUM VM-skip benefit, the read-query no-extra-dirtying safety valve, and the post-beta1 `e9eaeb04248` fix that records free space when a page becomes all-visible so its FSM entry does not go stale; includes per-feature source-commit history.
