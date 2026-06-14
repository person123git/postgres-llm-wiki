# PostgreSQL Versions

This page indexes the PostgreSQL versions covered by the wiki.

## Supported Versions

| Version | Status | Wiki Home | Branch | Pinned Commit | Coverage |
|---|---|---|---|---|---|
| 19 | active | [v19/index](v19/index.md) | `master` (post-19beta1) | `e18b0cb7344cb4bd28468f6c0aeeb9b9241d30aa` | Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`; filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module (core `pgs_mask` strategy mask and planner hooks, per-index disabling, advice language, generation, enforcement, feedback/EXPLAIN, GUCs, tests) and its scoped source history (core planner-enabling commits, 22 direct module commits, and test/doc/build support commits), plus a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (blocking new-heap rewrite/swap, concurrent online rewrite via logical decoding with a decoding `bgworker`, the `pgrepack` output plugin, a temporary replication slot, change spill/replay, lock upgrade and swap, the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, and tests) with 41 feature-scope source-history commits; and a comprehensive, commit-backed walkthrough of three v19 autovacuum/VACUUM features (parallel autovacuum via the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, autovacuum table scoring via `AutoVacuumScores`/the five `*_score_weight` GUCs/`pg_stat_autovacuum_scores`, and read-only query scans setting pages all-visible in the visibility map through on-access pruning); all three v19 question histories are pinned to the post-`REL_19_BETA1` `master` commit `e18b0cb7`. |
| 18 | primary | [v18/index](v18/index.md) | `REL_18_STABLE` | `6cb307251c5c6261286c1566496920976640108e` | Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`; filed coverage includes B-tree VACUUM density design, `pgstatindex` approximation limits and tests, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, custom cumulative statistics, extension hooks for VACUUM/autovacuum, `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `track_activity_query_size` activity text storage, `NUM_BUFFER_PARTITIONS` buffer mapping usage, GUC default-value changes since v12 through v18 (`effective_io_concurrency` and `log_connections` included), and a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, limitations, and the new v18 origin-differs conflict logging) with a source-verified section on all logical replication features new since v17 (conflict detection/logging/statistics, generated-column replication, `streaming = parallel` default, alterable `two_phase`, `max_active_replication_origins`, `idle_replication_slot_timeout`, `pg_createsubscriber` and `pg_recvlogical` options, contrib `pg_logicalinspect`). |
| 17 | active | [v17/index](v17/index.md) | `REL_17_STABLE` | `54eeefaedbee0385529f3edf321bb99e49232aaa` | Behavioral claims cite the matching pinned checkout under `raw/postgres-17/`; filed coverage includes the complete contrib extension inventory, explanations for all 53 control-file-backed contrib extensions, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, and a reviewed summary of the seven GUC default-value changes since v12 (old/new value, introducing major version, apply scope, exclusions, and test-coverage notes, grounded in `guc_tables.c`/`config.sgml` and the checkout's own commit history), and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, the `indislive`/`indisready`/`indisvalid` progression, a dedicated steps-and-locks section on `ShareUpdateExclusiveLock` + the `WaitForLockers`/`WaitForOlderSnapshots` waits, and a "what changed from v12" section centered on the PG14 `PROC_IN_SAFE_IC` snapshot-wait optimization, the reverted VACUUM-ignores-CIC attempt, and the `PGXACT`->`PGPROC` `statusFlags` move, anchored to the checkout's own commit history), a comprehensive walkthrough of the `REINDEX INDEX CONCURRENTLY` implementation (`ReindexRelationConcurrently`'s six phases and five waits, the atomic phase-4 `index_concurrently_swap` new-valid/old-invalid flip via the transactional `CatalogTupleUpdate`, the `_ccnew`/`_ccold` naming with a per-phase failure table showing a healthy `index_name` is never left invalid, a dedicated "can a failure leave an invalid index with the original index name?" section confirming a RIC failure never converts a bloated-but-valid `index_name` into no usable index (the phase-4 swap renames and flips validity in one transaction; only an already-invalid `index_name` stays invalid, the `concur_reindex_ind5` repair case), and a since-v12 section: partitioned REINDEX support via `ReindexPartitions` leaf recursion, `REINDEX (TABLESPACE ...)`, transactional `index_set_state_flags`, `PROC_IN_SAFE_IC` for RIC, the progress-view `WAIT_5` fix, the v15 shared-memory stats rework, and the v17 `MAINTAIN` privilege, anchored to the checkout's own commit history), and an analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes (it cannot: `AttachPartitionEnsureIndexes` is strictly match-or-create via `IndexSetParentIndex`/`DefineIndex`, incompatible/extra/invalid child indexes are retained, the three look-alike "drops" are an absorbed index losing independent droppability, the manual redundant-CHECK-constraint drop, and a later cascading parent `DROP INDEX`; with a version-by-version since-v12 section covering the unchanged no-drop invariant plus per-release matching refinements: v13 attmap refactor, v14/v15 partitioned-index DROP-path fixes, v15 `NULLS NOT DISTINCT` matching, the v16 back-patched invalid-index match skip `fc55c7ff`, the v17 PK-vs-UNIQUE constraint-type match `cee8db3f` with the `CompareIndexInfo`/`validatePartitionedIndex` fixes, and the 17.x re-attach parent-validation update `becf6d2696`), and a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging into WAL, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, and limitations) with its commit history grouped by minor version (16-cycle core commits `366283961a`/`8756930190`/`0324651573`, 17.0 `54ccfd6586`, 17.5 `0ae1245e04`, 17.10 CVE-2026-6638 `f0f59b658e`, plus foundations and adjacent origin-infrastructure commits). |
| 12 | legacy | [v12/index](v12/index.md) | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` | Behavioral claims cite the matching pinned checkout under `raw/postgres-12/`; filed coverage includes foreign-key join selectivity, planner statistics sources (`pg_stat_all_tables`, `pg_stats`, and `pg_stats_ext` direct-use excluded; `pg_class`, `pg_statistic`, extended statistics catalogs, index/FK/constraint metadata, planner statistics hooks, and regression coverage included), `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `psql` environment variables and session timeout behavior, exact `pgstatindex` calculation behavior, planner penalties for bloated B-tree indexes through physical pages and tree height including the storage-manager path and I/O/CPU boundary for `RelationGetNumberOfBlocks()`, a proposed `avg_leaf_density` / `leaf_fragmentation` triage heuristic for REINDEX candidates, B-tree leaf density (60% vs 90%) impact on planner page-count costs and executor leaf-page walking, a density-versus-fragmentation comparison for index scan I/O with level estimates and cache/storage sensitivity, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, including the v12 `bt_metap` unsigned-`oldest_xact` overflow workaround, and a three-stage REINDEX triage heuristic that shortlists candidates from `pg_class`/`pg_stat_*`/`pg_relation_size()` signals, confirms density with gated `pgstatindex` runs, prioritizes by wasted bytes times scan frequency with `REINDEX (CONCURRENTLY)` execution notes, measures post-REINDEX improvement via size/shape deltas, per-scan counter rates (per-index counters survive both REINDEX forms, including the v12 `index_concurrently_swap` stats copy), and `EXPLAIN (ANALYZE, BUFFERS)` / `pg_stat_statements` before/after windows, and documents the cited rationale for excluding `leaf_fragmentation` from the REINDEX priority score, and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation in `DefineIndex` (four internal transactions, two heap scans, three transaction waits, the `indislive`/`indisready`/`indisvalid` progression set non-transactionally by `index_set_state_flags`, the preconditions/restrictions and the invalid-index failure path, and a dedicated steps-and-locks section covering transaction- and session-level `ShareUpdateExclusiveLock`, the `WaitForLockers(ShareLock)` writer waits, `WaitForOlderSnapshots`, and the `LockConflicts` rationale for why DML proceeds while another CIC/`VACUUM`/`ANALYZE`/DDL is blocked, plus a reviewed blocker-by-blocker enumeration of every operation that can block CIC at each of its four blocking points — conflicting-lock holders at acquisition with the autovacuum-cancel and prepared-transaction nuances, open writers at the two lock waits, and same-database old-snapshot holders at the snapshot wait, with the `pg_stat_progress_create_index` wait phases and worked examples for a running `pg_dump` and an hour-long open / idle-in-transaction session, plus a failure-scenarios section mapping each CIC phase to the leftover on the table — no index, an invalid not-ready index, or an invalid ready index that still takes writes and enforces uniqueness — with each leftover's planner/write/uniqueness/HOT cost, session-lock release on ordinary ERROR/cancel paths, crash/immediate-shutdown outcome scoped as an open recovery question, `DROP`/`REINDEX` recovery), a companion comprehensive walkthrough of the v12 `REINDEX INDEX CONCURRENTLY` implementation in `ReindexRelationConcurrently` (its six phases — create the `_ccnew` copy, build, validate, swap, set-dead, drop — and five wait points, namely the three shared CIC waits plus two extra `AccessExclusiveLock` "wait for readers" waits before marking the old index dead and before dropping it; the atomic `index_concurrently_swap` that marks the new index valid and the old invalid, renames `_ccnew`→original and original→`_ccold`, and moves constraints/triggers/comment/dependencies and per-index cumulative stats; `ShareUpdateExclusiveLock` held at transaction and session level on the old index, new index, and heap; the no-transaction-block/temp-fallback/system-catalog/partitioned/exclusion/invalid restrictions and toast rebuild; the two-index state-flag progression; a per-phase failure table leaving an invalid `_ccnew` before the swap or `_ccold` after, with regression evidence; multi-index batching; and the source-vs-view progress-phase discrepancy where phase 6 reports `WAIT_4` so `waiting for readers before dropping` is never emitted), and an analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes from the attached table (it cannot: `AttachPartitionEnsureIndexes` is strictly match-or-create via `IndexSetParentIndex`/`DefineIndex`, incompatible/extra indexes are kept, no `performDeletion`/`RemoveInheritance` runs in the attach path and the pre-index `CreateInheritance` merges CHECK constraints only; v12's looser matching — no `indisvalid` skip, existence-only constraint check — still never drops; covering the three look-alikes, the `ALTER INDEX ... ATTACH PARTITION` re-parent/error path, and the docs' plain-`CREATE INDEX` pre-build workflow). |

## Coverage Notes

- 2026-06-14: made the codebase navigation guide a mandatory root-level
  question-style `type: codebase-navigation-guide` page for every supported
  version. Converted the v12 guide to `wiki/v12/codebase-navigation-guide.md`
  and added matching guides for v17, v18, and v19. Each guide has `## Question`
  and inline `## Answer` sections and maps that version's pinned source layout,
  normal SQL statement path, utility-command dispatch, generated/catalog
  artifacts, key structs, contrib boundaries, tests, and docs.
- 2026-06-12: v17 `REINDEX INDEX CONCURRENTLY` gained a "Can a failure leave an
  invalid index with the original index name?" section, the v17 companion to the
  v12 RIC section. For a healthy index a RIC failure never leaves `index_name`
  invalid: the phase-4 swap renames the rebuilt copy to `index_name` and flips
  validity (`new.indisvalid = true`, `old.indisvalid = false`) atomically in one
  transaction via the transactional `CatalogTupleUpdate`, so the invalid leftover
  is always the differently-named `_ccnew` (pre-swap) or `_ccold` (post-swap) and
  a bloated-but-valid index never degrades to no usable index; the only
  `index_name`-invalid outcome is an index that was already invalid before RIC ran
  (the `concur_reindex_ind5` regression repair case). Noted the v17-specific point
  that all `index_set_state_flags` writes are transactional in v17 (unlike v12's
  in-place `heap_inplace_update`).
- 2026-06-12: filed v17 `REINDEX INDEX CONCURRENTLY` implementation page, the v17
  companion to the v12 RIC page and the v17 CIC page. Covers
  `ReindexRelationConcurrently`'s six phases / five waits, the
  `ShareUpdateExclusiveLock` footprint, the atomic phase-4 swap (new-valid /
  old-invalid flipped together with the transactional `CatalogTupleUpdate`), the
  restriction set, the per-phase failure outcomes, and a since-v12 change section
  (partitioned REINDEX support, `TABLESPACE`, transactional `index_set_state_flags`,
  `PROC_IN_SAFE_IC` for RIC, the progress `WAIT_5` fix, the v15 stats rework, and
  the v17 `MAINTAIN` privilege). Noted a sibling discrepancy: the v17 CIC page
  still calls `index_set_state_flags` non-transactional, which is v12 behavior; v17
  uses `CatalogTupleUpdate` (recorded under the RIC page's Open Questions).
- 2026-06-12: v12 `REINDEX INDEX CONCURRENTLY` gained a "Can a failure leave an
  invalid index with the original index name?" section. For a healthy index a RIC
  failure never leaves `index_name` invalid: the phase-4 swap renames the rebuilt
  copy to `index_name` and flips validity atomically in one transaction, so the
  invalid leftover is always the differently-named `_ccnew` (pre-swap) or `_ccold`
  (post-swap) and a bloated-but-valid index never degrades to no usable index; the
  only `index_name`-invalid outcome is an index that was already invalid before
  RIC ran (the regression repair case).
- 2026-06-12: v12 `REINDEX INDEX CONCURRENTLY` review tightened the `ShareLock`
  wait conflict-set wording, split partitioned-table skip from partitioned-index
  direct-target error, and added the dedicated `reindex-concurrently` isolation
  spec to the coverage summary for that question.

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
