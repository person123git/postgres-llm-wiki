# PostgreSQL 19

## Source Pin

- Branch: `master` (`19beta1`)
- Commit: `298bdd379552148f6043b4595374a7a6fbdd13c3`
- Status: `active`
- Source path: `raw/postgres-19/`
- Added: 2026-05-30
- Repinned: 2026-06-03

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`. This is the beta branch state on `master` (`19beta1`); there is no `REL_19_STABLE` branch yet, so the pin tracks an exact `master` commit. Filed coverage includes comprehensive walkthroughs of the new `pg_plan_advice` contrib module and the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command, each with scoped source history checked through `298bdd37`.

## Questions

- [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](questions/pg-plan-advice.md) - Comprehensive explanation of the new `pg_plan_advice` contrib module: the core `pgs_mask` strategy-mask and five planner hooks it relies on, per-index disabling for index-specific advice, the advice language (tags, relation identifiers), plan-to-advice generation, advice enforcement, feedback/EXPLAIN output, GUCs, prepared-statement and plan-cache interaction, the round-trip test harness, and the full scoped source history: core planner-enabling commits, 22 direct module commits, and test/doc/build support commits.
- [How the REPACK Command Works in PostgreSQL 19, and Its 40 Feature-Scope Commits (unverified)](questions/repack-command.md) - Comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (which absorbs `VACUUM FULL` and `CLUSTER`): the blocking rewrite path, the concurrent online rewrite via logical decoding (decoding `bgworker`, `pgrepack` output plugin, temporary replication slot, change spill/replay, lock upgrade and swap), the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, I/O impact (no cost-delay; the BULKREAD/BULKWRITE ring buffers), tests, and 40 feature-scope commits.
