# PostgreSQL Versions

This page indexes the PostgreSQL versions covered by the wiki.

## Supported Versions

| Version | Status | Wiki Home | Branch | Pinned Commit | Coverage |
|---|---|---|---|---|---|
| 19 | active | [v19/index](v19/index.md) | `REL_19_STABLE` | `99e47536bbf1a165f5dc8d504f928821ebc8df6a` | Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`; filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module (core `pgs_mask` strategy mask and planner hooks, per-index disabling, advice language including underscore-separated occurrence numbers, generation, enforcement, feedback/EXPLAIN, GUCs, tests) and its scoped source history (20 core planner foundation/enabling/fix commits, 26 direct module commits, and test/doc/build support commits), plus a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (blocking new-heap rewrite/swap, lock-free multi-table candidate discovery followed by locked per-table open/recheck, partitioned-target discovery hardening, concurrent online rewrite via logical decoding with a decoding `bgworker`, the `pgrepack` output plugin, a temporary replication slot, race-safe on-demand decoding activation, change spill/replay, lock upgrade and swap, the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, generic `MAINTAIN` / `pg_maintain` documentation, and tests) with 50 feature-scope source-history commits; and a comprehensive, commit-backed walkthrough of three v19 autovacuum/VACUUM features (parallel autovacuum via the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, autovacuum table scoring via `AutoVacuumScores`/the five `*_score_weight` GUCs/`pg_stat_autovacuum_scores` plus the post-beta1 MXID-score division-by-zero fix `1f2297b5487`, and read-only query scans setting pages all-visible in the visibility map through on-access pruning, including the post-beta1 `e9eaeb04248` fix that records free space when a page becomes all-visible so its FSM entry does not go stale, plus the `b01c31eef9c` VM-clear WAL/incremental-backup correction, `9171f77db23` TAP test, and `3180ce3d7a8` wrong-buffer error-path hardening); all three v19 question histories are pinned to the `REL_19_STABLE` commit `99e47536`. |
| 18 | primary | [v18/index](v18/index.md) | `REL_18_STABLE` | `6cb307251c5c6261286c1566496920976640108e` | Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`; filed coverage includes B-tree VACUUM density design, `pgstatindex` approximation limits and tests, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, custom cumulative statistics, extension hooks for VACUUM/autovacuum, `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `track_activity_query_size` activity text storage, `NUM_BUFFER_PARTITIONS` buffer mapping usage, GUC default-value changes since v12 through v18 (`effective_io_concurrency` and `log_connections` included), a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, limitations, and the new v18 origin-differs conflict logging) with a source-verified section on all logical replication features new since v17 (conflict detection/logging/statistics, generated-column replication, `streaming = parallel` default, alterable `two_phase`, `max_active_replication_origins`, `idle_replication_slot_timeout`, `pg_createsubscriber` and `pg_recvlogical` options, contrib `pg_logicalinspect`), and a fully reviewed Row-Level Security walkthrough covering rewrite-time command-layered security quals and WCOs, relcache/generated-catalog plumbing, policy recursion and permissions, WCO execution order, RI and whole-table boundaries, planner selectivity/cost/index/pruning/parallel effects, DDL locks, partitions/views/COPY/logical replication, direct and adjacent settings, and five PostgreSQL 14-derived performance follow-ups covering plan caching, MultiXact, aggregation, whether RLS filters must be repeated in the query, and scalar-InitPlan versus index-runtime-key behavior; exact-pin tests reproduce stale prepared results under the same effective-role OID after membership-edge `inherit_option`, `BYPASSRLS`, superuser, and current-database-owner changes until `DISCARD PLANS`; release-based history evidence covers 21 scoped RLS fixes/improvements not already present in PostgreSQL 14.0, with the 14.0-shipped pg_dump bulk-query improvement excluded, four previously omitted changes added, and the exact cross-branch first-release matrix omitted, and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, table/session `ShareUpdateExclusiveLock`, `indislive`/`indisready`/`indisvalid` progression, invalid-index failure outcomes, regression/isolation coverage, and v18 changes from v17: `pg_index` TOAST snapshot handling, parallel GIN build, virtual-generated-column rejection, and temporal `WITHOUT OVERLAPS` shared plumbing). |
| 17 | active | [v17/index](v17/index.md) | `REL_17_STABLE` | `54eeefaedbee0385529f3edf321bb99e49232aaa` | Behavioral claims cite the matching pinned checkout under `raw/postgres-17/`; filed coverage includes the complete contrib extension inventory, explanations for all 53 control-file-backed contrib extensions, partial-index pros/cons with ordinary expression indexes as the baseline, partial expression indexes, planner predicate implication, expression-key matching, operational restrictions, `ANALYZE` row-count and expression-statistics behavior, planner costing, regression coverage, and a what-changed-from-v12 section covering the post-v12 documentation warning, concurrent-build safe-wait exclusion for partial/expression indexes, MERGE target rechecks, expression statistics, `NULLS NOT DISTINCT`, partial-unique proof restriction, and partial-hash planning fix, a fully reviewed sampling `pgstatindex` proposal using v17 `BlockSampler`, a strict 10% effective-fraction floor below 100 MiB, a one-read `pageinspect` prototype, and an exact-pin seven-fixture/100-seed comparison including the finite-sample effect on a small partial index, and a reviewed summary of the seven GUC default-value changes since v12 (old/new value, introducing major version, apply scope, exclusions, and test-coverage notes, grounded in `guc_tables.c`/`config.sgml` and the checkout's own commit history), and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, the `indislive`/`indisready`/`indisvalid` progression, a dedicated steps-and-locks section on `ShareUpdateExclusiveLock` + the `WaitForLockers`/`WaitForOlderSnapshots` waits, and a "what changed from v12" section centered on the PG14 `PROC_IN_SAFE_IC` snapshot-wait optimization, the reverted VACUUM-ignores-CIC attempt, and the `PGXACT`->`PGPROC` `statusFlags` move, anchored to the checkout's own commit history), two performance follow-ups covering `maintenance_work_mem` and the broader GUC matrix (build/parallel controls, AM and read-stream I/O boundaries, storage, WAL/checkpoint/commit costs, waits/timeouts, apply scopes, and commonly confused non-controls), including the separate AM-build and serial-validation budgets, B-tree/BRIN planned-versus-launched participants and mandatory worker tapes, hash/GiST/GIN/SP-GiST/BRIN/contrib-Bloom boundaries, conditional GIN pending-list cleanup, scoped observability, and the workload-specific plateau, a comprehensive walkthrough of the `REINDEX INDEX CONCURRENTLY` implementation with a source-backed GUC-performance follow-up covering build and validation memory, B-tree/BRIN worker request and availability, AM-specific GiST/GIN boundaries, v17 read streams, temporary and explicit output storage, WAL/checkpoint/commit costs across nine internal commits for one index, all five timeout-sensitive waits, exact apply scopes, and commonly confused non-controls, while the implementation covers (`ReindexRelationConcurrently`'s six phases and five waits, the atomic phase-4 `index_concurrently_swap` new-valid/old-invalid flip via the transactional `CatalogTupleUpdate`, the `_ccnew`/`_ccold` naming with a per-phase failure table showing a healthy `index_name` is never left invalid, a dedicated "can a failure leave an invalid index with the original index name?" section confirming a RIC failure never converts a bloated-but-valid `index_name` into no usable index (the phase-4 swap renames and flips validity in one transaction; only an already-invalid `index_name` stays invalid, the `concur_reindex_ind5` repair case), and a since-v12 section: partitioned REINDEX support via `ReindexPartitions` leaf recursion, `REINDEX (TABLESPACE ...)`, transactional `index_set_state_flags`, `PROC_IN_SAFE_IC` for RIC, the progress-view `WAIT_5` fix, the v15 shared-memory stats rework, and the v17 `MAINTAIN` privilege, anchored to the checkout's own commit history), and a fully reviewed analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes (the built-in path cannot, but event triggers and `ProcessUtility_hook` can issue independent DDL; core coverage includes parser/utility dispatch, parent/child lock serialization with DROP, rollback, exact `CompareIndexInfo` matching—including reusable partial/expression indexes and exclusion-index non-matches—catalog/dependency re-parenting, physical and multilevel creation, generated/catalog/AM/contrib boundaries, tests and gaps, and meaningful changes since v12; exact-pin execution also exposed an open multilevel validity edge where recursive creation absorbs an invalid leaf while the pre-existing top parent stays valid), and a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging into WAL, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, and limitations) with its commit history grouped by minor version (16-cycle core commits `366283961a`/`8756930190`/`0324651573`, 17.0 `54ccfd6586`, 17.5 `0ae1245e04`, 17.10 CVE-2026-6638 `f0f59b658e`, plus foundations and adjacent origin-infrastructure commits), and a table-partitioning optimizations page covering v17 planning/execution optimizations per partitioning type (legacy inheritance: plan-time constraint exclusion only; declarative: plan-time + run-time pruning with per-HASH/LIST/RANGE bound matching, partitionwise join with merged non-identical bounds, partitionwise aggregate FULL/PARTIAL, and executor tuple routing/row movement/COPY batching, plus the four `PGC_USERSET` GUCs) with an "optimizations since v12" section attributing v13 advanced partitionwise join (`c8434d64ce0`), v14 run-time pruning for `UPDATE`/`DELETE` (`86dc90056df`) + `DETACH PARTITION CONCURRENTLY` (`71f4c8c6f74`), v15 `live_parts`/run-time-prune refactor/`MERGE` (`475dbd0b718`/`297daa9d435`/`7103ebb7aae`), v16 cached tuple routing (`3592e0ff98b`), and v17 `boolcol IS [NOT] UNKNOWN` pruning (`07c36c1333e`) + concurrent-detach robustness (`11f1218ce81`) to the checkout's own Stamp-bracketed commit history, plus a follow-up section on per-optimization locking reduction and random-I/O sensitivity (only plan-time pruning removes read locks; run-time pruning and constraint exclusion keep the locks `AcquireExecutorLocks`/`find_all_inheritors` already took; scan-eliminating optimizations gain most on slow random storage because avoided per-partition fetches are priced at `random_page_cost`; and, unlike v12, v17 partitionwise aggregate gains by avoiding a hash-agg spill whose writes `cost_agg` prices at `random_page_cost`, while hash-join spills stay `seq_page_cost`). |
| 14 | active | [v14/index](v14/index.md) | `REL_14_STABLE` | `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156` | Behavioral claims cite the matching pinned checkout under `raw/postgres-14/`; source version 14.23, pinned to the `REL_14_STABLE` tip `5c00f4e2e3b` (`REL_14_23-3-g5c00f4e2e3b`, three commits past the 14.23 release stamp). Filed coverage so far is the mandatory codebase navigation guide: source-tree layout from the top-level and backend makefiles, the normal SQL statement path through `tcop`/rewrite/planner/executor (`exec_simple_query` -> `pg_parse_query` -> `pg_analyze_and_rewrite`/`QueryRewrite` -> `pg_plan_queries`/`planner` -> `PortalRun`/executor entry points), utility-command dispatch via `ProcessUtility`, generated catalog/header artifacts (backend `generated-headers` target, the backend catalog `generated-header-symlinks`/`genbki.pl` path, and the `pg_index.h` catalog header), key parser/planner/executor/relcache/AM data structures, and contrib and test/doc surfaces. Filed question coverage adds a Row-Level Security walkthrough: RLS as a rewrite-time feature (`pg_policy` storage, `relrowsecurity`/`relforcerowsecurity`, relcache `rd_rsdesc`, `check_enable_rls` bypass logic, `get_row_security_policies` security-barrier `USING` quals and `WITH CHECK` options, permissive-OR/restrictive-AND with default-deny, the planner `security_level`/leakproof ordering, executor `ExecWithCheckOptions`, plan-cache `dependsOnRLS` re-planning, partition/inheritance and view-owner handling, generated catalog/header implications, and COPY/CTAS/whole-table/RI/logical-replication/error-detail boundaries), its source-evident scalability/performance issues (including a dedicated RLS-and-the-plan-cache section: RLS-aware plan caching keyed on role + `row_security` (including same-role membership, `INHERIT`, `BYPASSRLS`, `SUPERUSER`, intermediate-role, and database-owner invalidation blind spots plus the `DISCARD PLANS` mitigation), the role-independent relcache `rd_rsdesc` policy cache, the scenarios where caching does not help — role/`row_security` mismatch at reuse, the never-applicable simple-validity fast path, un-cached simple-protocol and one-shot SPI queries, the conditional first-five custom-plan opportunities, DDL invalidation — and the no-cache re-analyze/rewrite/re-plan cost; a query-vs-policy planning subsection showing RLS policy conditions need not be restated in the query — `USING` quals are transferred into `baserestrictinfo` at ordered security levels (the first sublist is level 0; restrictive quals precede the permissive OR in mixed sets), safely promotable clauses can match indexes, partition pruning sees the full restriction list, and query-text copies land at the maximum security level; a sub-SELECT-wrapping subsection showing that wrapping an uncorrelated function call in a scalar `(SELECT ...)` (e.g. `(SELECT auth.uid())`) can make each generated InitPlan occurrence lazy, at most once per execution, and cached in a `PARAM_EXEC` parameter, avoiding repeated function-expression work when the unwrapped expression would remain a per-tuple filter, with caveats (only uncorrelated sub-SELECTs qualify, exact index runtime keys can avoid per-row calls but lossy paths recheck, and rewrite copies and separate occurrences are not generally memoized together); an RLS-and-aggregation mitigation section showing that `Agg` nodes consume RLS-filtered subplans and mitigations focus on indexable policy predicates, leakproof/indexable filters, partition pruning/partitionwise aggregation (`enable_partitionwise_aggregate` is session-scoped), stable plan-cache environments, or deliberately trusted/pre-aggregated reporting paths; plus the FK-validation/two MultiXact paths (policy-authored `FOR SHARE` locking subqueries and RLS-induced FK validation fallback) where RLS disables the bulk RI path and per-row `FOR KEY SHARE` checks can create/read MultiXacts on hot referenced rows), and the RLS control/inspection surface (`row_security`, general `plan_cache_mode` and aggregation-adjacent `enable_partitionwise_aggregate` scopes, table flags, `BYPASSRLS`, policy options, extension hooks, `row_security_active`, catalog/psql views, and pg_dump/pg_restore controls), with explicit core/hook test coverage. Filed question coverage also adds a MultiXact walkthrough: how MultiXact backs shared row locks (a `MultiXactId` in `xmax` for a set of `(xid, MultiXactStatus)` members, the offsets/members SLRUs under `pg_multixact/`, `MultiXactStateData` + per-backend oldest arrays, the `MultiXactIdCreate`/`Expand`/`CreateFromMembers` -> `GetNewMultiXactId`/`RecordNewMultiXact` write path, and the `GetMultiXactIdMembers` read path), the backend-local 256-entry per-transaction LRU cache whose tail eviction only frees memory rather than spilling an entry (`mXactCacheEnt`/`MAX_CACHE_ENTRIES`/`MXactContext`/`mXactCachePut`), and the separate miss path through the shared 8-offset/16-member SLRU pools to a possible filesystem `pg_pread`/`SLRURead` that may still hit the OS page cache; the distinction between system-wide MXID/member allocation and one backend's cache pressure; foreign keys (RI `FOR KEY SHARE` -> `heap_lock_tuple` -> `compute_new_xmax_infomask`, including null/partition/committed-updater edges), compatible explicit locks, UPDATE/DELETE/update chains, executor/trigger/logical-apply callers, and exact VACUUM replacement branches; three degradation paths (hot-row/full-member-array creation work, exclusive SLRU control/per-buffer/file-I/O waits after cache misses, and ID/member growth driving aggressive vacuum plus separate hard stops); observability (`pg_stat_slru`, exact generated wait names, `mxid_age`, targeted member decoding) with attribution limits; the four vacuum GUC scopes (three `PGC_USERSET` settings and restart-only `autovacuum_multixact_freeze_max_age`), per-table autovacuum MultiXact reloptions, build/generated/WAL/tool/contrib boundaries, verified diagnostic SQL, passing full isolation coverage, and explicit cache/SLRU performance-test absence. Filed question coverage also adds a fully source-reviewed functions/procedures-in-a-WHERE-clause performance walkthrough: procedures (`prokind='p'`) are rejected in expressions and run through utility `CALL`, whose arguments still use standalone expression evaluation; set-returning, aggregate, and window functions are also rejected directly in `WHERE`; scalar residual quals scale with candidate scan tuples or join pairs, subject to short-circuiting, `STRICT` null skips, SQL inlining, constant folding, pseudoconstant gating and rescans, and index runtime-key placement; the volatility effects (`VOLATILE` per row where needed, `STABLE` single-eval as an index-scan runtime key or no-`Var` one-time qual, `IMMUTABLE` constant-folded with constant args), the cost model (`add_function_cost` = `procost * cpu_operator_cost`, with omitted `CREATE FUNCTION COST` defaults of 1 for C/internal functions and 100 otherwise) and bare-Boolean-function fallback selectivity (`function_selectivity` 0.3333333), the narrower no-statistics `boolvarsel` 0.5 fallback, and expression-index/extended-expression statistics, plain-index search-key limits, expression-index/generated-column matching, post-planning immutability checks, non-volatile runtime keys, and support-generated index conditions, the separate leakproof security-promotion/index-pushdown rule and same-node `< 10 * cpu_operator_cost` ordering cutoff, plus `PARALLEL UNSAFE`/`RESTRICTED` plan effects; plus the mitigations (reduce candidate tuples/pairs, expression indexes/generated columns or estimate-only expression statistics, uncorrelated scalar sub-SELECT -> lazy once-per-execution InitPlan, truthful volatility/`STRICT`/`COST`, SQL-function inlining, planner `SUPPORT`, truthful `LEAKPROOF`/`PARALLEL SAFE`, and measured JIT/function tracking), with `ROWS` explicitly excluded for direct WHERE functions; generated `pg_proc_d.h`, fmgr/contrib-`sepgsql`, extension-bitcode, GUC-scope, and regression/absence boundaries are included; the RLS page is agent-verified by `GPT-5` at `2026-07-20T18:13:13Z`, the functions/procedures page at `2026-07-13T19:41:59Z`, and the MultiXact page at `2026-07-13T20:08:07Z`; only the codebase navigation guide remains `not yet`. |
| 12 | legacy | [v12/index](v12/index.md) | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` | Behavioral claims cite the matching pinned checkout under `raw/postgres-12/`; the reviewed mandatory codebase navigation guide covers build ownership, backend/standalone entry, simple and extended protocol including Sync, parse/analyze/RIR rewrite/planner/portal/full executor flow, utility and error boundaries, generated catalog/parser/fmgr/LWLock artifacts, core hooks, packaged extensions, custom AM registration/dispatch/tests, key data structures, and test/doc surfaces; filed coverage includes two `CREATE INDEX CONCURRENTLY` performance follow-ups: a `maintenance_work_mem` analysis (AM-dependent first-build and common validation-sort uses, B-tree planned-versus-launched participant shares and required tape runs, conditional GIN cleanup, BRIN's empty enumeration, GiST's buffered temporary-file path, scoped observability, and the workload-specific plateau) plus a broader CIC GUC matrix (B-tree worker request and pool limits, AM-specific settings, v12 heap-scan and storage boundaries, WAL/checkpoint/commit costs, waits/timeouts, exact apply scopes, absent later-version controls, and commonly confused non-controls), foreign-key join selectivity, partial-index pros/cons with ordinary expression indexes as the baseline, partial expression indexes, planner predicate implication, expression-key matching, operational restrictions, `ANALYZE` row-count and expression-statistics behavior, planner costing, and regression coverage, a fully reviewed planner-statistics-source analysis (`pg_stat_all_tables`, `pg_stats`, and `pg_stats_ext` excluded as direct feedback; `pg_class` plus physical size, `pg_statistic`, built extended data, ordinary/partial index sizing, inherited/expression statistics including partial-index correlation, defaults/security/errors, indirect auto-analyze and planner endpoint counter increments, estimate/cost metadata, AM/FDW/hook/contrib/generated-cache boundaries, and direct tests included), `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, a production database-health checklist covering observability prerequisites and apply scopes, activity/connections/locks, cumulative database and table counters, vacuum and wraparound, checkpoints, WAL archiving, replication, checksums, query and relation diagnostics, and database-log issue interpretation, `psql` environment variables and session timeout behavior, reviewed full non-snapshot `pgstatindex` behavior (extension install/API and privileges, true-root metadata, one-time main-fork scan, half-dead/deleted/live classification, size/density/fragmentation formulas and two-decimal formatting, physical `LP_DEAD` occupancy, concurrency and integrity-check limits, generated-header boundary, and test gaps), planner penalties for bloated B-tree indexes through physical pages and tree height including the storage-manager path and I/O/CPU boundary for `RelationGetNumberOfBlocks()`, a proposed `avg_leaf_density` / `leaf_fragmentation` triage heuristic for REINDEX candidates, B-tree leaf density (60% vs 90%) impact on planner page-count costs and executor leaf-page walking, a density-versus-fragmentation comparison for index scan I/O with level estimates and cache/storage sensitivity, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, including the v12 `bt_metap` unsigned-`oldest_xact` overflow workaround, and a three-stage REINDEX triage heuristic that shortlists candidates from `pg_class`/`pg_stat_*`/`pg_relation_size()` signals, confirms density with gated `pgstatindex` runs, prioritizes by configured-fillfactor reclaimable pages times paired-window scan rate with `REINDEX (CONCURRENTLY)` execution notes, measures post-REINDEX improvement via size/shape deltas, per-scan counter rates (per-index plain REINDEX keeps its OID while concurrent REINDEX copies the published collector entry, with pending-count continuity treated as an open boundary, including the v12 `index_concurrently_swap` stats copy), and `EXPLAIN (ANALYZE, BUFFERS)` / `pg_stat_statements` before/after windows, and documents the cited rationale for excluding `leaf_fragmentation` from the REINDEX priority score, and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation in `DefineIndex` (four internal transactions, two heap scans, three deliberate transaction-set waits, the `indislive`/`indisready`/`indisvalid` progression set non-transactionally by `index_set_state_flags`; the reviewed CIC page now also covers parser/utility/generated-catalog artifacts, the core-vs-index-AM/contrib-Bloom boundary, four deliberate transaction barriers versus additional event-trigger/predicate/AM/unique-validation/parallel-worker waits, temp and `IF NOT EXISTS` early-exit ordering, a post-`SET_VALID` `ddl_command_end` failure that leaves a valid index, and crash durability scoped to reviewed in-tree AMs rather than arbitrary third-party callbacks; it also retains the preconditions/restrictions and full failure-state coverage, and a dedicated steps-and-locks section covering transaction- and session-level `ShareUpdateExclusiveLock`, the `WaitForLockers(ShareLock)` writer waits, `WaitForOlderSnapshots`, and the `LockConflicts` rationale for why DML proceeds while another CIC/`VACUUM`/`ANALYZE`/DDL is blocked, plus a reviewed blocker-by-blocker enumeration of the four deliberate core transaction barriers separately from additional event-trigger/predicate/AM/unique-validation/parallel-worker waits; the four barriers cover conflicting-lock holders at acquisition with the autovacuum-cancel and prepared-transaction nuances, open writers at the two lock waits, and same-database old-snapshot holders at the snapshot wait, with the `pg_stat_progress_create_index` wait phases and worked examples for a running `pg_dump` and an hour-long open / idle-in-transaction session, plus a failure-scenarios section mapping each CIC phase to the leftover on the table — no index, an invalid not-ready index, an invalid ready index that still takes writes and enforces uniqueness, or a valid index after a post-`SET_VALID` command-end error — with each leftover's planner/write/uniqueness/HOT cost, session-lock release on ordinary ERROR/cancel paths, a resolved crash/immediate-shutdown recovery outcome (xidless `heap_inplace_update` flag flips commit asynchronously and are redone by `heap_xlog_inplace`, so recovery lands on one of the four documented leftovers without losing completed in-tree build data while retaining a later durable `SET_VALID`; this crash-durability result does not repair the separate prepared-transaction gap or prove arbitrary third-party AM callbacks), `DROP`/`REINDEX` recovery), a companion comprehensive walkthrough of the v12 `REINDEX INDEX CONCURRENTLY` implementation with a source-backed GUC-performance follow-up covering AM-dependent build/validation memory, B-tree-only worker controls, v12 bulk-read/synchronized-scan boundaries, temporary and fixed output storage, B-tree immediate sync plus WAL/checkpoint/commit costs, all five timeout-sensitive waits, exact apply scopes, absent later-version controls, and commonly confused non-controls in `ReindexRelationConcurrently` (its six phases — create the `_ccnew` copy, build, validate, swap, set-dead, drop — and five wait points, namely the three shared CIC waits plus two extra `AccessExclusiveLock` "wait for readers" waits before marking the old index dead and before dropping it; the atomic `index_concurrently_swap` that marks the new index valid and the old invalid, renames `_ccnew`→original and original→`_ccold`, and moves constraints/triggers/comment/dependencies and per-index cumulative stats; `ShareUpdateExclusiveLock` held at transaction and session level on the old index, new index, and heap; the no-transaction-block/temp-fallback/system-catalog/partitioned/exclusion/invalid restrictions and toast rebuild; the two-index state-flag progression; a per-phase failure table leaving an invalid `_ccnew` before the swap or `_ccold` after, with regression evidence; multi-index batching; and the source-vs-view progress-phase discrepancy where phase 6 reports `WAIT_4` so `waiting for readers before dropping` is never emitted), and an analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes from the attached table (it cannot: `AttachPartitionEnsureIndexes` is strictly match-or-create via `IndexSetParentIndex`/`DefineIndex`, incompatible/extra indexes are kept, no `performDeletion`/`RemoveInheritance` runs in the attach path and the pre-index `CreateInheritance` merges CHECK constraints only; v12's looser matching — no `indisvalid` skip, existence-only constraint check — still never drops; covering the three look-alikes, the `ALTER INDEX ... ATTACH PARTITION` re-parent/error path, and the docs' plain-`CREATE INDEX` pre-build workflow), and a comprehensive enumeration of all outcomes that leave an invalid index (`indisvalid=false`) derived from the full set of `indisvalid=false` writers in the checkout: a failed/cancelled/crashed `CREATE INDEX CONCURRENTLY`, a failed `REINDEX CONCURRENTLY` (`_ccnew`/`_ccold`), a failed/interrupted `DROP INDEX CONCURRENTLY` (retryable clear-valid step), an incomplete partitioned parent (`CREATE INDEX ON ONLY` / attached invalid child, validated by `validatePartitionedIndex` on `ATTACH PARTITION`), and `pg_upgrade` from <= 9.6 invalidating hash indexes in the new cluster, with the persistent state table, the single-transaction-DDL contrast, the planner/executor/VACUUM cost and the commands that reject invalid indexes, and per-outcome repair, and the partitioning optimizations and configuration during query planning and execution (partition pruning plan-time before child expansion in `expand_partitioned_rtentry` and run-time initial + per-scan pruning on `Append`/`MergeAppend`, with per-strategy HASH/LIST/RANGE bound-matching and tuple routing; constraint exclusion `relation_excluded_by_constraints` modes for legacy inheritance; partitionwise join `build_joinrel_partition_info`/`have_partkey_equi_join` and partitionwise aggregate `group_by_has_partkey`; executor tuple routing, cross-partition UPDATE row movement, and per-partition COPY multi-insert; the four session-scoped GUCs with defaults; and the v12 limitation that execution-time pruning covers only `Append`/`MergeAppend`, not `ModifyTable`; plus a follow-up locking/random-I/O analysis per optimization: only plan-time pruning removes read locks (it prunes before `expand_partitioned_rtentry` locks partitions, while `find_all_inheritors` locks all inheritance children before constraint exclusion runs and `AcquireExecutorLocks` locks every plan-time-surviving partition despite run-time pruning; write-path routing takes `RowExclusiveLock` lazily per touched partition), and the scan-eliminating optimizations gain most on slow random storage because avoided per-partition index heap fetches are priced at `random_page_cost` = 4× `seq_page_cost`, whereas partitionwise join/aggregate cut CPU/memory/locality (v12 costs hash-join spill as sequential and keeps hash aggregation in memory)). |

## Coverage Notes

- 2026-07-29: completed a full corrective review of [How wal_sender_timeout Is
  Used and What It Impacts in PostgreSQL 12
  (unverified)](v12/questions/wal-sender-timeout.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). Re-resolved and
  label-checked all 364 citations. Fixed a wrong `logicalfuncs.c`
  `CreateDecodingContext` label (that symbol lives in `logical.c`) and its
  too-narrow ranges, two `LogicalRepSyncTableStart` ranges that started
  mid-function, a `process_syncing_tables` label that pointed at the
  `process_syncing_tables_for_apply` comment, a `StartLogStreamer` range that
  started inside `LogStreamerMain`, a `pg_replication_slot_advance` label on the
  physical helper, two single-line label forms on multi-line ranges, and a
  `high-availability.sgml` citation attached to a `requested WAL segment ...
  has already been removed` error text that appears only in `walsender.c`.
  Added the `pg_stat_replication` superuser/`pg_read_all_stats` detail gate, the
  `immediately_reserve` default that leaves a new physical slot unreserved, and
  the `work_mem` identity of the closest protocol test. One isolated server
  built from the exact pin reproduced the reload-driven persistent-slot timeout
  in the same logged millisecond as the reload, the client seeing only
  `server closed the connection unexpectedly`, the temporary-slot drop, a
  20-second per-connection value outranking a 1-second configuration-file
  value, and the statistics privilege split; the created role and slot were
  dropped and the server was stopped. Agent verification was recorded as
  `claude-opus-5-max 2026-07-29T15:30:31Z`; human verification remains false.
- 2026-07-29: completed a full corrective review of [A Heuristic to Detect
  B-Tree Index Bloat in PostgreSQL 12
  (unverified)](v12/questions/index-bloat-heuristic.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). Repaired four
  malformed `## Evidence Map` rows, a `nbtsplitloc.c`-labelled citation that
  pointed at `reloptions.c`, a `pg_stat_user_indexes` column claim cited to the
  schema filter instead of the `pg_stat_all_indexes` definition, and an
  `idx_scan` column that never reached the query output. Corrected the
  1.4-to-1.5 overload claim, removed the unsupported `pg_options_to_table`
  access requirement, and narrowed the planner-overestimate claim to
  `empty_pages`/`deleted_pages`. Added the `pg_stat_scan_tables` privilege
  requirement, exact GUC apply scopes, lock-conflict and `BAS_BULKREAD`
  boundaries, the `relkind` filter that excludes `pgstatindex`-rejecting
  partitioned parents, the contrib/`genbki.pl` build boundary, the regression
  coverage gap, and a mandatory `leaf_pages = 0` guard for the `NaN` case that
  otherwise reports `is_bloated = t`. Documented two systematic false positives
  with exact-pin measurements: random-key indexes settle at
  `avg_leaf_density = 64.91` with no deletions, recover to `90.05` under a
  `REINDEX` whose reclaimed bytes matched the prediction within 1.1 %, and drift
  back to `67.65`; and `deleted_pages` enter the free space map only at a second
  VACUUM, after which 430 of 685 were reused with no file growth. Every fenced
  statement, plus fixture, privilege, and `NaN` tests, ran against an isolated
  server built from the exact pin; the server was stopped afterwards. Agent
  verification was recorded as `claude-opus-5-max 2026-07-29T14:57:04Z`; human
  verification remains false.
- 2026-07-27: repinned v19 `REL_19_STABLE` from
  `3aa54433b0cdce48facb610a5b720208cc760654` to
  `99e47536bbf1a165f5dc8d504f928821ebc8df6a` (52 new commits; a clean
  forward move). Reviewed every commit and changed path. Updated the three
  related questions: `dfde93581dc` raises the direct `pg_plan_advice` history
  to 26 commits and fixes underscore-separated occurrence numbers;
  `69fe2514fdd`, `10389a7e15e`, and `99e47536bbf` raise the REPACK feature
  history to 50 commits and cover partitioned discovery, LSN debug formatting,
  and race-safe on-demand logical-decoding activation; and
  `3180ce3d7a8` restores the unconditional wrong-buffer error in
  `visibilitymap_clear()`. Updated the mandatory navigation guide, refreshed
  shifted source anchors, and left all `verified_by_agent` fields at `not yet`
  because this was a changed-range repin rather than a full-page re-audit.
- 2026-07-22: source-reviewed the scalar-function subquery follow-up in
  [Row-Level Security (RLS) in PostgreSQL 14: Implementation, Scalability and
  Performance, and Settings
  (unverified)](v14/questions/row-level-security-rls.md) against unchanged pin
  `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156`. Clarified that each surviving
  uncorrelated scalar sublink is lazy zero-or-once per occurrence and execution
  of its containing outer plan, not a query-wide function cache. Added RLS
  rewrite-copy, prepared-execution, residual-filter, runtime-key/rescan,
  lossy-recheck, equality-selectivity, partial-index, volatility,
  immutable-folding, expression-placement, privilege/leakproof, and external
  helper boundaries. Exact-pin direct-query and RLS-policy tests reproduced
  call-count, prepared-execution, volatile-semantic, and skewed-estimate
  differences. This scoped follow-up review leaves `verified_by_agent: not yet`;
  human `verified` remains `false`.
- 2026-07-22: source-reviewed the scalar-function subquery follow-up in
  [Row-Level Security (RLS) in PostgreSQL 18: Implementation, Performance,
  Settings, and Fixes Since PostgreSQL 14
  (unverified)](v18/questions/row-level-security-rls.md) against unchanged pin
  `6cb307251c5c6261286c1566496920976640108e`. Clarified that a surviving
  uncorrelated scalar sublink is a lazy zero-or-one InitPlan per occurrence and
  execution of its containing outer plan, not a query-wide function cache.
  Added rewrite-copy,
  correlation, prepared-execution, residual-filter,
  index-runtime-key/lossy-recheck/rescan,
  equality-selectivity, partial-index, parallel-safety, function-volatility,
  immutable-folding, and external-auth-helper boundaries. Exact-pin focused
  tests reproduced the call-count, prepared-plan, semantic, and estimate
  distinctions. This was a scoped follow-up review, so `verified_by_agent`
  remains `not yet` and human `verified` remains `false`.
- 2026-07-22: re-audited [Row-Level Security (RLS) in PostgreSQL 18:
  Implementation, Performance, Settings, and Fixes Since PostgreSQL 14
  (unverified)](v18/questions/row-level-security-rls.md) against unchanged pin
  `6cb307251c5c6261286c1566496920976640108e`. Replaced the v15 development
  stamp with the PostgreSQL 14.0 release as the history boundary. Removed
  `bd3611db5a`, whose v14 backpatch shipped in 14.0; added the omitted
  `f062cddafe`, `5ba4cc3090`, `cffca3665d`, and `0dca5d68d7`; and corrected
  the summaries and evidence for the retained rows. The resulting broad
  taxonomy contains 21 changes. Also corrected the current-v18
  logical-replication role boundary: DML and table sync honor `run_as_owner`
  before ACL/RLS checks, while replicated `TRUNCATE` checks as subscription
  owner before its trigger-only owner switches. This was a scoped history
  review, so `verified_by_agent` remains `not yet` and human `verified`
  remains `false`.
- 2026-07-22: added the five PostgreSQL 14 RLS performance follow-up questions
  to the PostgreSQL 18 page using user-approved PostgreSQL 18-adapted wording.
  The existing source-reviewed answer sections already covered plan caching,
  MultiXact, aggregation, query-versus-policy predicates, and scalar-subquery
  InitPlans; no behavioral claims, citations, source pin, or verification fields
  changed.
- 2026-07-22: completed a full corrective review of [Row-Level Security (RLS)
  in PostgreSQL 18: Implementation, Performance, Settings, and Fixes Since
  PostgreSQL 12 (unverified)](v18/questions/row-level-security-rls.md) against
  unchanged pin `6cb307251c5c6261286c1566496920976640108e`. Removed five
  carry-over follow-up prompts that were not established as verbatim v18 user
  questions while retaining their source-backed topics inside the answer.
  Corrected command-specific policy layering, recursion and RI boundaries,
  WCO order, error-detail conditions, planner selectivity/cost and cache-layer
  distinctions, membership-edge `inherit_option` semantics, MultiXact
  citations, and
  scalar-InitPlan versus index-runtime-key behavior. Exact-pin prepared-plan
  tests reproduced stale same-role results after membership-edge
  `inherit_option` changes,
  `BYPASSRLS`, superuser, and current-database-owner changes until
  `DISCARD PLANS`; the full 231-test core regression suite passed. Corrected
  the scoped history from 25 to 24 entries by removing a test-only commit and
  removed the cross-branch minor-release matrix because its required refs and
  tag set are absent from the current pinned checkout.
- 2026-07-21: completed a full corrective review of [PostgreSQL 12 Database
  Health Checklist (unverified)](v12/questions/database-health-checklist.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Reworked monitoring access and statistics-snapshot
  semantics, client capacity and blocker/prepared-lock SQL, independent
  vacuum/analyze and XID/MultiXact checks, checkpoint/background-writer
  interpretation, archive/replication/slot metrics, full-capture
  `pg_stat_statements` deltas, checksum and base-backup accounting, relation
  diagnostics, and the operational log matrix. Added exact GUC apply scopes,
  generated/build boundaries, direct tests and test gaps, an evidence map, and
  explicit open questions. An out-of-tree build of the exact pin, with its
  matching `pg_stat_statements` and `pgstattuple` modules, executed every final
  fenced SQL statement successfully against an isolated server; diagnostic
  relation placeholders were mapped to local heap and B-tree fixtures. The
  server was stopped after testing. Independent final audits found no remaining
  page defect. Agent verification was recorded as
  `GPT-5 2026-07-21T13:34:54Z`; human verification remains false.
- 2026-07-20: completed a new full claim-to-source review of [Row-Level Security
  (RLS) in PostgreSQL 14: Implementation, Scalability and Performance, and
  Settings (unverified)](v14/questions/row-level-security-rls.md) and all five
  follow-ups against unchanged pin
  `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156` (PostgreSQL 14.23). Corrected the
  relcache policy-order description, `ON CONFLICT DO UPDATE` SELECT-policy
  layering, base-restriction versus residual-scan behavior, default-deny DML,
  partial-index implication, plan-time pruning citations, direct-versus-
  subquery aggregate legality, scalar-InitPlan per-occurrence/cardinality and
  lossy-recheck boundaries, table/role permissions, and initial-sync versus
  ongoing/apply logical-replication behavior. Added policy selectivity/cost and
  parallel-hazard effects, exact lock conflicts, CTAS source rewrite, and the
  hook/RI error-detail boundaries. Expanded the plan-cache flaw from same-role
  membership changes to `INHERIT`, `BYPASSRLS`, `SUPERUSER`, intermediate-role,
  and `pg_database_owner` inputs. An exact-pin isolated run reproduced stale
  prepared results for membership, `INHERIT`, `BYPASSRLS`, and database-owner
  changes in both directions; `DISCARD PLANS` restored the new state each time,
  including the security-sensitive `NOBYPASSRLS` case. The server was stopped.
  Agent verification was recorded as `GPT-5 2026-07-20T18:13:13Z`; human
  verification remains false.
- 2026-07-17: completed a full claim-to-source review of [Can ALTER TABLE ...
  ATTACH PARTITION Drop Indexes in PostgreSQL 17
  (unverified)](v17/questions/attach-partition-index-drops.md) against unchanged
  pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10). Scoped the
  answer precisely: core table ATTACH is match-or-create and never drops an
  existing index, while user event triggers or a `ProcessUtility_hook` can issue
  independent DDL before or after the built-in path. Added parser/utility
  dispatch, parent/child lock serialization with DROP, transactional rollback,
  catalog/dependency state, the physical and recursive creation paths,
  generated parser/catalog headers, object-access and custom-AM boundaries,
  shipped-contrib behavior, and explicit test gaps. Corrected the old claim
  that partial and expression indexes are inherently incompatible: mapped
  matching definitions are reusable. Added exclusion-index non-matching and
  separated the direct invalid-index skip from recursive `DefineIndex`, which
  can absorb an invalid leaf. Expanded the since-v12 history with parser,
  event-trigger metadata, recursive matching/validity, exclusion-constraint,
  search-path, and v17-stable propagation fixes. Exact-pin smoke tests confirmed
  matching partial/expression reuse, exclusion non-reuse, rollback, custom
  start/end-trigger drops, and an open multilevel validity edge: an invalid new
  intermediate child can sit below a still-valid pre-existing top index. The
  exact-pin core regression suite passed. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-17T19:53:21Z`; human `verified` remains
  `false`.
- 2026-07-17: completed a full claim-to-source review of [Proposing a Sampling
  pgstatindex Variant for PostgreSQL 17
  (unverified)](v17/questions/pgstatindex-sample-variant-proposal.md) against
  unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10),
  and carried over both user-approved corrected follow-ups from the
  corresponding v12 question. Reworked the design around v17's
  `pg_prng_state`/`uint32` `BlockSampler`, v17's invalid-index rejection,
  continuous relation and per-page lock boundaries, stronger sampled-page
  validation, and Make plus Meson extension wiring. Added a strict 10%
  effective-fraction floor below 100 MiB, direct sample-count outputs, and a
  one-read `pageinspect` 1.12 prototype using 64-bit block arguments and the
  `d`/`D` deleted-page mapping. Exact-pin execution over six sub-50-MiB
  bloat/partial/small/empty fixtures plus a 107.13 MiB control confirmed every
  target and strict threshold edge and matched all shared standard-
  `pgstatindex` fields at 100%. One hundred seeds at requested 1% and 5%
  quantified the floor results; the 3.88 MiB partial index moved from 5/25 to
  50 pages, with leaf MAPE changing 0.80% -> 0.81% and 0.95% -> 0.81%, while
  the 0.66 MiB control lost its two leaf-free 1% draws. The `pgstattuple`
  regression target and all eight `pageinspect` targets passed. Set
  `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-17T19:25:39Z`; human `verified` remains
  `false`.
- 2026-07-17: extended [Proposing a Sampling pgstatindex Variant for PostgreSQL
  12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) with a
  second user-approved corrected follow-up: apply a 10% effective sample floor
  when captured main-fork size is strictly below 100 MiB, rerun the comparison,
  and explain the small-partial-index effect. Updated the C-policy formula,
  `pageinspect` prototype, pros/cons, boundary tests, and open questions. Reused
  the six original fixtures and added a 107.13 MiB healthy control. Exact-pin
  execution confirmed the strict target policy, synthetic byte edges immediately
  below/at/above 100 MiB, and full-sample equivalence across all seven indexes.
  One hundred identical seeds at requested 1% and 5% showed the 3.88 MiB partial
  index moving from 5/25 pages to 50 pages for either request, with leaf-count
  MAPE changing from 1.18%/1.08% to 0.86%; density mean error stayed 0.02 points.
  `verified_by_agent` remains `not yet` because this was another scoped follow-up.
- 2026-07-17: extended [Proposing a Sampling pgstatindex Variant for PostgreSQL
  12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) with the
  user-approved corrected follow-up requesting comparative tests, against the
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`. Added an executed,
  reproducible `pageinspect`-prototype comparison over six indexes, all below
  50 MB: uniformly sparse, contiguous-range-deleted, split/churn, sparse
  partial, small healthy, and empty. A 100% sample matched every shared
  standard-`pgstatindex` field in all six cases. One hundred deterministic
  `setseed()` draws at 1% and 5% quantify leaf/deleted count MAPE, density and
  fragmentation error, and leaf-free draws; the results expose particularly
  high variance for deleted-page counts, fragmentation, and a one-page sample
  of the 0.66 MiB index. Reset `verified_by_agent` to `not yet` because this was
  a scoped follow-up rather than a new full-page audit.
- 2026-07-17: completed a full claim-to-source review of [Proposing a Sampling
  pgstatindex Variant for PostgreSQL 12
  (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
  Replaced the misleading exact-versus-estimated split with separately timed
  direct observations versus estimates and added direct sample counts. Reworked
  the C proposal around v12's native Algorithm-S `BlockSampler`, including its
  without-replacement ascending output, `int` sample-size ceiling, potentially
  O(number-of-blocks) skip loop, and cancellation boundary. Added the complete
  relation/page-lock, concurrent DML/VACUUM, one-time length, bulk-read ring,
  metapage and sampled-page validation, privilege, extension-upgrade,
  fmgr/generated-header, and test envelopes. Revised the `pageinspect` SQL to
  remove the hidden sub-100-MiB 10% floor, evaluate relation size once, read each
  sampled ordinary page once, expose sample sizes, and document its lock,
  malformed-page, signed-`int4`, helper-lifecycle, and invalid-input limits.
  Exact-pin execution confirmed full-sample equivalence on a populated B-tree
  and the metapage-only edge; the one `pgstattuple` and all five `pageinspect`
  regression targets passed. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-17T18:12:12Z`; human `verified` remains
  `false`.
- 2026-07-17: extended [How `wal_sender_timeout` Is Used and What It Impacts in
  PostgreSQL 12 (unverified)](v12/questions/wal-sender-timeout.md) with the
  user-approved corrected follow-up, “How does `wal_sender_timeout` affect
  `pg_replication_slots`, and how is WAL retained during error or failure
  scenarios?”, against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Added exact shared-view row and
  column transitions; persistent, temporary, and unreserved-slot boundaries;
  timeout, EOF/protocol/error, orderly `CopyDone`, and crash/restart outcomes;
  `restart_lsn` versus logical confirmation and row horizons; checkpointed
  state and conservative crash rollback; and the checkpoint/restartpoint,
  `wal_keep_segments`, and archive limits on eventual WAL removal. An isolated
  exact-pin libpq smoke test observed a temporary physical slot as active with
  reserved WAL, then absent after its silent stream reached a per-connection
  20-second sender timeout. Reset `verified_by_agent` to `not yet` because this
  was a scoped follow-up rather than a new full-page audit.
- 2026-07-17: completed a full claim-to-source review of [How
  `wal_sender_timeout` Is Used and What It Impacts in PostgreSQL 12
  (unverified)](v12/questions/wal-sender-timeout.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`). Corrected the
  blanket half-time description by adding the earlier non-requesting logical
  WAL-wait heartbeat and shutdown-request branches. Added exact activation and
  exclusion boundaries (`START_REPLICATION` only; not the main base backup,
  logical slot snapshot build, initial table copy, or SQL decoding functions),
  process-local versus exposed reply timestamps, server-stall accounting,
  physical client tools, table-sync catch-up, and receiver/retry controls.
  Expanded operational effects to live reload, `max_wal_senders`, reconnects,
  monitoring limits, synchronous waiters that timeout does not release,
  persistent-versus-temporary slot cleanup, inactive-slot WAL retention, build
  and extension boundaries, and direct-test absence. An isolated exact-pin
  smoke test confirmed active-sender reload, timeout logging, sender-row
  removal, and persistent-slot retention. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-17T16:13:56Z`; human `verified` remains
  `false`.
- 2026-07-17: completed a full claim-to-source review of [Query Planner
  Statistics Sources in PostgreSQL 12
  (unverified)](v12/questions/query-planner-statistics-sources.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`. Confirmed that
  `pg_stat_all_tables` is cumulative monitoring rather than direct planner
  input, while its underlying modification counter affects planning indirectly
  by triggering auto-analyze. Added the reverse edge where a planner B-tree
  endpoint probe increments an exposed index-scan counter without reading that
  counter as input. Corrected ordinary versus partial index sizing
  and separated CHECK, NOT NULL, and partition-constraint origins. Added the
  complete caller/data-structure path; inherited and expression-index
  statistics, including partial-index correlation costing; built/unbuilt and
  partial-ANALYZE extended-data behavior; defaults, stale/security/error paths
  and the v12 MCV error-label defect; operator/function/type/index/FK metadata;
  table/index AM, FDW, hook, shipped-contrib, generated-header, syscache, and
  relcache boundaries; and direct tests plus explicit negative-test absence.
  The exact-pin core regression suite passed. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-17T15:45:58Z`; human `verified` remains
  `false`.
- 2026-07-17: completed a full claim-to-source review of Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified) against unchanged
  pin `45b88269a353ad93744772791feb6d01bc7e1e42`. Replaced the former
  whole-index, hard-coded-90 density formula with a configured-fillfactor page
  estimator that separates live-leaf shortfall, half-dead pages, and deleted
  pages. Corrected the false claim that expression indexes lack
  `pg_statistic` rows, downgraded `pgstatindex` from an exact snapshot to a full
  non-snapshot physical observation, and added its buffer-ring, integrity, and
  I/O-counter-pollution boundaries. Reworked priority around paired-window scan
  rates and independent pre/post-REINDEX windows; narrowed concurrent counter
  continuity to the published collector entry copied at swap; and corrected
  split occupancy, invalid-index routing, `_ccnew`/`_ccold` failures, and
  production SQL. Exact-pin checks covered default and nondefault fillfactors,
  expression statistics, sparse post-VACUUM shape, diagnostic counter effects,
  and concurrent REINDEX. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-17T15:18:58Z`; human `verified` remains
  `false`.
- 2026-07-17: extended [How CREATE INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) with
  the user-approved corrected follow-up, “What GUCs have a performance impact
  on it?”, against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Added a source-backed matrix for
  AM-dependent build and validation memory, B-tree-only worker request and pool
  limits, GiST/GIN boundaries, v12 bulk-read rings and synchronized scans,
  index and temporary storage, backend writeback, B-tree immediate sync,
  WAL/checkpoint/four-transaction commit costs, three timeout-sensitive waits,
  exact restart/reload/session scopes, absent later-version controls, and
  commonly confused non-controls. `verified_by_agent` remains `not yet`
  because this was a scoped expansion, not a full-page re-audit.
- 2026-07-17: extended [How REINDEX INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 12 (unverified)](v12/questions/reindex-index-concurrently.md) with
  the user-approved corrected follow-up, “What GUCs have a performance impact
  on it?”, against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Added a source-backed matrix for
  AM-dependent build and validation memory, B-tree-only worker request and pool
  limits, GiST/GIN boundaries, v12 bulk-read rings and synchronized scans,
  temporary storage and the fixed preserve-old-index output tablespace,
  B-tree's immediate data sync, WAL/checkpoint costs, xidless commit semantics,
  nine internal commits for one index, five timeout-sensitive waits, exact
  restart/reload/session scopes, absent later-version controls, and commonly
  confused non-controls. `verified_by_agent` remains `not yet` because this was
  a scoped expansion, not a full-page re-audit.
- 2026-07-17: extended [How REINDEX INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 17 (unverified)](v17/questions/reindex-index-concurrently.md) with
  the user-approved corrected follow-up, “What GUCs have a performance impact
  on it?”, against unchanged pin
  `54eeefaedbee0385529f3edf321bb99e49232aaa`. Added a source-backed matrix for
  AM-dependent build and validation memory, B-tree/BRIN worker request and pool
  limits, GiST/GIN boundaries, v17 sequential read streams, synchronized scans,
  temporary storage and RIC's preserve-old-or-explicit output tablespace rule,
  permanent-build WAL/checkpoint costs, nine internal commits for one index,
  five timeout-sensitive waits, exact restart/reload/session scopes, and
  commonly confused non-controls. Retained the source-visible
  `READ_STREAM_MAINTENANCE` comment versus `READ_STREAM_SEQUENTIAL`
  implementation mismatch under `## Open Questions`. Reset
  `verified_by_agent` to `not yet` because this was a scoped expansion, not a
  full-page re-audit.
- 2026-07-17: repinned v19 `REL_19_STABLE` from
  `8055e3375aa1c2237181e06be26b05b964d18ed5` to
  `3aa54433b0cdce48facb610a5b720208cc760654` (31 new commits; a clean
  forward move). The branch is now stamped `19beta2`. Reviewed every commit,
  changed path, and v19 source citation. Added REPACK documentation commit
  `8a84ddd8c63`, which puts `REPACK` in the generic `MAINTAIN` and predefined
  `pg_maintain` descriptions; the REPACK feature history is now 47 commits.
  Expanded the visibility-map coverage for `56bf5fa5d67` /
  `b01c31eef9c` / `9171f77db23`: heap operations now register modified VM
  pages in WAL, redo preserves the clears, and a new TAP test checks combined
  incremental-backup restore. The on-access VM-setting algorithm, parallel
  autovacuum, and scoring are unchanged. No `pg_plan_advice` module, doc, test,
  or direct core-infrastructure commit landed. Updated all four v19 page pins
  and refreshed 12 content-identical shifted source ranges; all remaining
  existing ranges stayed at the same lines. `verified_by_agent` remains
  `not yet` because this was a changed-range repin, not a full-page re-audit.
- 2026-07-16: extended [How CREATE INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 17 (unverified)](v17/questions/create-index-concurrently.md) with
  the user-approved corrected follow-up, “What GUCs have a performance impact
  on it?”, against unchanged pin
  `54eeefaedbee0385529f3edf321bb99e49232aaa`. Added a source-backed matrix for
  direct build and validation memory, parallel request and worker-pool limits,
  GiST and GIN boundaries, v17 sequential read-stream I/O, synchronized scans,
  index and temporary storage, WAL compression/checkpoints/four-transaction
  commit costs, waits and timeout semantics, apply scopes, and commonly
  confused settings that do not control CIC. The source-visible mismatch
  between the `READ_STREAM_MAINTENANCE` comment and the heap AM's actual
  `READ_STREAM_SEQUENTIAL` flag is retained under `## Open Questions`.
  `verified_by_agent` remains `not yet` because this was a scoped expansion,
  not a full-page re-audit.
- 2026-07-13: completed a full claim-to-source review of [How MultiXact Works
  in PostgreSQL 14, and How Foreign Keys and Other Operations Degrade
  Performance When the Local Cache Spills to Secondary Storage
  (unverified)](v14/questions/multixact-foreign-keys-cache-spill.md) against
  unchanged pin `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156`. Corrected the
  premise: the 256-entry backend-local transaction cache evicts with `pfree`
  and never spills an entry; only a subsequent shared-SLRU miss requests a
  filesystem read, which may still hit the OS page cache. Separated one
  backend's cache pressure from cluster-wide MXID/member allocation and added
  exact cache-count, obsolete-read, committed-updater, immutable expansion,
  conflict/wait, WAL/recovery/2PC, horizon/truncation, table-AM, and extension
  boundaries. Corrected the user-visible wait names to the generated forms
  without `Lock`, added the separate MXID-counter hard stop, and narrowed
  freezing to all actual replacement outcomes including a singleton surviving
  locker. Added FK null/partition/restrict edges, update-chain and internal
  tuple-lock callers, `vacuum_multixact_failsafe_age`, exact GUC/reloption
  scopes, build/generated artifacts, attribution limits, and direct tests.
  The exact-pin full isolation suite and all filed diagnostic SQL passed; the
  temporary server was stopped. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-13T20:08:07Z`; human `verified` remains
  `false`.
- 2026-07-13: completed a full claim-to-source review of [Performance
  Implications of Functions and Procedures in a WHERE Clause in PostgreSQL 14,
  and How to Minimize the Overhead
  (unverified)](v14/questions/functions-procedures-in-where-clause.md) against
  unchanged pin `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156`. Corrected the
  `CALL` boundary: it has no optimizer plan, but evaluates argument expressions
  separately before direct fmgr invocation, and those standalone expressions
  do not enter JIT. Added direct `WHERE` rejection of SRFs/aggregates/window
  functions and removed `ROWS` as a scalar-WHERE lever. Reworked call
  multiplicity around residual scan tuples, join pairs, strict-null skips,
  pseudoconstant rescans, runtime keys, and SQL/JIT inlining. Corrected the
  plain-index claim, resolved expression-index and extended-expression
  statistics, and separated leakproof security promotion from the strict
  `< 10 * cpu_operator_cost` runtime-order cutoff. Added JIT,
  `track_functions`, exact GUC scopes, generated `pg_proc_d.h`, fmgr/contrib
  `sepgsql`, extension-bitcode, and direct-test/absence boundaries. Exact-pin
  spot checks reproduced one-time gating, row-dependent versus strict-null and
  scalar-InitPlan call counts, and expression-index search; the full core
  regression check passed. Set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-13T19:41:59Z`; human `verified` remains
  `false`; `wiki_lint`: 0 errors, 0 warnings.
- 2026-07-13: completed a full claim-to-source review of [Row-Level Security
  (RLS) in PostgreSQL 18: Implementation, Performance, Settings, and Fixes
  Since PostgreSQL 12
  (unverified)](v18/questions/row-level-security-rls.md) against unchanged pin
  `6cb307251c5c6261286c1566496920976640108e`. Corrected MERGE INSERT to use
  ordinary `WCO_RLS_INSERT_CHECK` and logical replication to check its selected
  action role (table owner by default, subscription owner with
  `run_as_owner = true`). Added generated `pg_policy_d.h`/relcache/contrib
  boundaries, index/pruning/parallel and policy-cache performance details,
  complete direct settings/tool surfaces, and the source-visible distinction
  between per-tuple filters and optimizable quals. Reproduced both membership
  revoke and grant under an unchanged effective role: cached RLS results stayed
  stale until session-local `DISCARD PLANS`. Expanded the explicitly scoped fix
  history from 20 to 25 by adding seven omitted fixes and removing two
  independent additions; verified all hashes, subjects, ancestry, five
  post-fork v18/master pairs, and first-release tags/dates. The exact-pin build
  and core regression check passed; set `verified_by_agent` to
  `GPT-5-6-Sol-Max-Thinking 2026-07-13T18:58:20Z`; human `verified` remains
  `false`; `wiki_lint`: 0 errors, 0 warnings.
- 2026-07-13: repinned v19 `REL_19_STABLE` from
  `01c544e1afb99bc2a76803870010b7cd2907f3b5` to
  `8055e3375aa1c2237181e06be26b05b964d18ed5` (12 new commits; still stamped
  `19beta1`). Reviewed every commit, changed path, and v19 source citation.
  Added REPACK commit `133eba078f7`, which removes ineffective discovery-time
  table locks and hardens the locked per-table open/recheck path; the REPACK
  feature history is now 46 commits. Added the post-beta1 on-access VM fix
  `e9eaeb04248`, which records a newly all-visible page's free space so a later
  VM-based VACUUM skip does not leave stale FSM state. No `pg_plan_advice`
  module, doc, test, or core-planner commit landed in the range. Updated all
  four v19 page pins, refreshed shifted `repack.c`, `pruneheap.c`, and
  `rewriteHandler.c` anchors, and corrected two pre-existing end-line anchors.
  `verified_by_agent` remains `not yet` because this was a changed-range repin,
  not a full claim-by-claim re-verification.
- 2026-07-13: extended [How CREATE INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 17 (unverified)](v17/questions/create-index-concurrently.md) with
  the user-approved corrected `maintenance_work_mem` follow-up against unchanged
  pin `54eeefaedbee0385529f3edf321bb99e49232aaa`. The source-backed section
  separates transaction 2's AM-specific first build from transaction 3's serial
  encoded-TID validation sort; traces B-tree and BRIN planned-versus-launched
  parallel participants, 32 MB automatic worker shares, and mandatory worker
  tapes; maps first-build and plateau behavior for every core AM and contrib
  Bloom; and covers GIN pending-list cleanup, GUC scope, observability, extension
  and build boundaries, and direct test gaps. There is no universal byte cutoff;
  the practical plateau is workload-specific. Reset `verified_by_agent` to
  `not yet` because this was a scoped expansion, not a full-page re-audit;
  `wiki_lint`: 0 errors, 0 warnings.
- 2026-07-13: source-reviewed the `maintenance_work_mem` follow-up in [How
  CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Corrected the parallel B-tree
  accounting to distinguish planned from launched workers and to count the
  leader's worker-role output run; qualified the setting as the intended
  primary-sort budget rather than an exact total-command cap; made GIN cleanup
  conditional on a nonempty pending list; documented BRIN's empty validation
  enumeration, GiST's own temporary-file path, and the GIN soft threshold; and
  narrowed temp statistics to file-volume evidence plus the regression-test
  absence claim to direct CIC coverage. The scoped review retains
  `verified_by_agent: not yet` because the full page was not re-verified;
  `wiki_lint`: 0 errors, 0 warnings.
- 2026-07-13: re-audited the `maintenance_work_mem` follow-up in [How CREATE
  INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Preserved the no-universal-cutoff
  conclusion, but replaced the misleading “two allocations” framing with an
  AM-dependent first-build phase plus the common serial validation-TID sort.
  Clarified the MVCC-to-B-tree alive-tuple boundary, generalized the 32 MB
  parallel participant thresholds, added the session/transaction scope and
  runtime-availability boundary for `max_parallel_maintenance_workers`, and
  distinguished true hash CIC (`NBuffers`) from its temporary non-concurrent
  fallback (`NLocBuffer`). Also sharpened the GiST `auto` switch and GIN forced
  cleanup boundaries, refreshed all three summaries, and retained
  `verified_by_agent: not yet` because this remained a scoped section review;
  `wiki_lint`: 0 errors, 0 warnings.
- 2026-07-13: completed a scoped claim-to-source verification of the
  `maintenance_work_mem` follow-up section in [How CREATE INDEX CONCURRENTLY Is
  Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Re-checked that section's claims
  against `raw/postgres-12/`: the B-tree serial/parallel budgets and the 32 MB
  per-participant worker cap (0/1/2 workers at below-64/64/96 MB), the full-budget
  serial encoded-TID validation sort, the hash `sort_threshold` selection, the
  per-AM first-build matrix, and the `trace_sort`/`log_temp_files`/progress
  observability plus test-coverage citations all resolve to the cited lines.
  Fixed two precision defects: the misleading `index_concurrently_build`
  "returns before setting indisready" wording (it runs `index_build`, then sets
  `INDEX_CREATE_SET_READY`, then returns) and the `autovacuum_work_mem = -1`
  fallback citation (added the `config.sgml` `autovacuum_work_mem` definition).
  The global and v12 landing-page summaries remain accurate. `verified_by_agent`
  stays `not yet` because this re-audited only one section, not the whole page;
  `wiki_lint`: 0 errors, 0 warnings.
- 2026-07-10: extended [How CREATE INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) with
  the user-approved corrected follow-up on `maintenance_work_mem`. The new
  source-backed section separates the transaction-2 AM build from the
  transaction-3 validation TID sort, explains B-tree's serial and parallel
  budgets (including automatic 32 MB participant thresholds and one required
  tape run per worker), and maps first-build behavior for B-tree, hash, GiST,
  GIN, SP-GiST, BRIN, and contrib Bloom. It explains why no universal byte
  cutoff exists, defines the per-AM plateau conditions, distinguishes fixed CIC
  scan/wait/write costs from memory-sensitive work, and covers `trace_sort`,
  `log_temp_files`, progress phases, and aggregate temp counters. The only
  direct v12 regression case forces non-concurrent hash sorting; CIC has no
  memory/plateau test. Same-checkout history for parallel B-tree build, encoded
  validation TIDs, and hash pre-sorting was ancestry-checked. Refreshed
  `wiki/index.md`, `wiki/v12/index.md`, and this coverage row;
  `verified_by_agent` is `not yet` because this was a scoped expansion rather
  than a full-page claim re-audit.
- 2026-07-10: completed a full claim-to-source review of [How `pgstatindex`
  Calculates B-Tree Index Statistics in PostgreSQL 12
  (unverified)](v12/questions/how-pgstatindex-calculates-information.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`. Preserved the
  core formulas but made their exact boundaries explicit: `index_size` is the
  one-time captured main-fork block count times `BLCKSZ`; density is a ratio of
  summed physical leaf occupancy using `PageGetFreeSpace` and includes retained
  `LP_DEAD` storage; fragmentation is the percentage of live leaves whose
  right-link points to a lower physical block; both percentages are formatted
  to two decimals. Confirmed and expanded the page's existing `empty_pages`
  half-dead-state explanation, and distinguished true-root metadata from the
  fast root. Added fresh-install 1.4-to-1.5 update wiring, function privileges,
  invalid-index acceptance, generated `pg_am_d.h`/`BTREE_AM_OID` implications,
  deleted-page recyclability, one-time length/extension behavior, per-page and
  relation locks, the unlocked/unvalidated metapage read, missing ordinary-page
  B-tree sanity checks, and explicit regression absences. An exact-pin build
  passed the module's one declared `installcheck`; a populated spot check before
  and after `VACUUM` matched the physical-occupancy interpretation. Refreshed
  `wiki/index.md` and `wiki/v12/index.md`; set `verified_by_agent:
  GPT-5-6-Sol-Max-Thinking 2026-07-10T16:00:43Z` after the final claim pass.
  Human `verified` remains `false`.
- 2026-07-10: completed a full claim-to-source review of the [PostgreSQL 12
  Codebase Navigation Guide (unverified)](v12/codebase-navigation-guide.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`.
  Corrected the RIR expansion (`Retrieve-Instead-Retrieve`, not
  "retrieve-instead-rewrite"), added the omitted `PGC_SU_BACKEND` GUC context,
  completed the executor lifecycle through `ExecutorEnd`, and repaired ranges
  that stopped before Parse-source storage, best-path selection, aggregate plan
  dispatch, and complete AM callback definitions. Expanded backend/standalone
  callers, simple-query snapshots, Parse/Bind/Execute/Sync boundaries, portal
  strategies and failure cleanup, outer protocol/transaction error recovery,
  utility caller/event-trigger routing, plan-cache and portal structs, custom AM
  registration/relcache/wrapper/test boundaries, and generated catalog/parser/
  fmgr/LWLock build and reverse-include implications. Refreshed `wiki/index.md`
  and `wiki/v12/index.md`; set `verified_by_agent:
  GPT-5-6-Sol-Max-Thinking 2026-07-10T15:06:04Z` after the final claim pass.
  Human `verified` remains `false`.
- 2026-07-10: completed a full claim-to-source review of [How CREATE INDEX
  CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Corrected three material scope
  errors: the four explicit core synchronization barriers are not every backend
  wait (DDL event triggers, predicates/expressions, AM callbacks, B-tree unique
  validation, and parallel-worker coordination can also wait); an error in a
  `ddl_command_end` trigger after non-transactional `SET_VALID` can report CIC
  failure while leaving a valid index; and crash recovery cannot itself lose
  completed in-tree build data while retaining `SET_VALID`, but that does not
  rule out the separate prepared-transaction valid-but-incomplete defect or
  prove third-party AM durability. This qualification supersedes the shorthand
  "never a valid-but-incomplete index" in the historical 2026-06-19 coverage
  note below. Added parser/utility/generated-catalog coverage, temp and
  `IF NOT EXISTS` ordering, the core/index-AM boundary, contrib Bloom, direct
  REINDEX citations, test-presence/absence detail, and scoped source history.
  Exact-pin temporary builds reproduced both the prepared-write missing-entry
  defect and the late event-trigger valid-index outcome. Refreshed
  `wiki/index.md` and `wiki/v12/index.md`; set
  `verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-10T14:38:16Z` after the
  final claim pass. Human `verified` remains `false`.
- 2026-07-10: completed a full claim-to-source review of [Row-Level Security
  (RLS) in PostgreSQL 14: Implementation, Scalability and Performance, and
  Settings (unverified)](v14/questions/row-level-security-rls.md) against the
  unchanged `REL_14_STABLE` pin `5c00f4e2e3b`. Corrected the blanket per-row
  wording (execution frequency depends on filter, InitPlan, SQL-function
  inlining, and index-runtime-key shape), restrictive-policy sorting phase
  (rewrite, not relcache build), mixed-policy security-level order, and
  partition fan-out wording. Expanded the MultiXact answer from the FK-only
  path to both source-supported paths: policy-authored `FOR SHARE` lookup
  locking and the RLS-induced initial-FK-validation fallback. Added generated
  catalog/header implications, view-owner identity, whole-table/RI/publisher
  replication boundaries, policy-DDL `AccessExclusiveLock`, inspection and
  pg_dump/pg_restore surfaces, GUC scope boundaries, and explicit core/hook
  test coverage. Documented and reproduced the unchanged-effective-user
  role-membership invalidation blind spot in cached policy selection, plus the
  `DISCARD PLANS` mitigation; the in-tree prepared-plan test does not cover
  membership changes. Refreshed `wiki/index.md`, `wiki/v14/index.md`, and the v14
  coverage cell here.
- 2026-07-09: repinned v19 to the new `REL_19_STABLE` branch. Upstream `master`
  had advanced to `20devel` (branch commit `a281a3e6dbb` "Stamp HEAD as 20devel."),
  and the `REL_19_STABLE` branch was created, so the former "post-`REL_19_BETA1`
  `master`" pin was no longer the v19 line. Repinned `raw/postgres-19/` from
  `cdae794af31b3e9cfc323fc654292d86fa746f77` to the `REL_19_STABLE` tip
  `01c544e1afb99bc2a76803870010b7cd2907f3b5` (2026-07-09, still stamped `19beta1`;
  a clean forward move — the old pin is an ancestor, 105 new v19 commits).
  Switched the Branch column from `` `master` (post-19beta1) `` to `REL_19_STABLE`.
  Reviewed the `cdae794a..01c544e1` changed range against the three v19 question
  pages and the codebase navigation guide, refreshed all shifted source line
  anchors (verified via a difflib old->new line map plus content-equality checks),
  and added the new feature-scope commits: REPACK grew 42 -> 45
  (`fb284f2f9bd` stored generated columns, `5e450df50dc` change-replay range-table
  init, tree-wide `da8889ccd7e` `PG_MODULE_MAGIC_EXT`), and `pg_plan_advice` module
  commits grew 22 -> 25 (`89f5f860cc5` FOREIGN_JOIN single-relation fix,
  `ea203d371de` pgindent follow-up, `da8889ccd7e`). Also updated the autovacuum
  page's release-note "typo" caveat because commit `1a7fa06dbcd` fixed the
  `autovacuum_vacuum_insert_score_weight` prefix typo in `release-19.sgml` within
  the range. Refreshed `wiki/index.md`, `wiki/v19/index.md`, the v19 row here, and
  each page's `pinned_commit`. `verified_by_agent` is `not yet` on all v19 pages
  (reset from the prior agent-timestamp on the `pg_plan_advice` page): a repin and
  changed-range review, not a claim-by-claim re-verification.
- 2026-07-09: extended [Table Partitioning Optimizations and Configuration During
  Query Planning and Execution in PostgreSQL 17
  (unverified)](v17/questions/partitioning-planning-execution-optimizations.md)
  with a new `## Locking reduction and random-I/O sensitivity per optimization`
  section for the follow-up ("Add an extra analysis on how each optimization will
  reduce locking, and how each will have big gains if random I/O has higher
  latency"; supplied grammatically clean and recorded verbatim under
  `## Question`). Same unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`,
  drafted from a claim-to-source map built directly from `raw/postgres-17/`.
  Locking: expansion/locking runs before costing (`planmain.c` `query_planner`:
  `add_other_rels_to_query` before `make_one_rel`); plan-time pruning is the only
  optimization that reduces read locks because `expand_partitioned_rtentry` prunes
  to `live_parts` before `try_table_open` locks the survivors, whereas inheritance
  locks every member up front via `find_all_inheritors` and constraint exclusion
  (`set_append_rel_size` -> `relation_excluded_by_constraints` ->
  `set_dummy_rel_pathlist`) runs only afterward; run-time pruning keeps all locks
  because `AcquireExecutorLocks` locks every `RTE_RELATION` in the finished plan's
  rtable; write-path routing (`ExecInitPartitionInfo`) takes `RowExclusiveLock`
  lazily per touched leaf, and cross-partition `UPDATE` writes two. Random I/O:
  `random_page_cost` = 4.0 vs `seq_page_cost` = 1.0 (`cost.h`/`costsize.c`), the
  docs advise raising for magnetic/less-cached storage, index-scan heap fetches
  are charged at `random_page_cost` (`cost_index`) and seq scans at
  `seq_page_cost` (`cost_seqscan`), so the scan-eliminating optimizations win most
  on high-latency random storage. Key v17-vs-v12 difference: v17 hash aggregation
  spills to disk (`nodeAgg.c` spill mode; limit `work_mem × hash_mem_multiplier`
  via `get_hash_memory_limit`) and `cost_agg` prices spill writes at
  `random_page_cost`, so partitionwise aggregate gains under slow random I/O by
  avoiding the spill; partitionwise-join batch spills stay `seq_page_cost`
  (`initial_cost_hashjoin`). Updated `## Contents`, `## Question` (follow-up),
  Evidence Map, Context Reviewed, Open Questions, and Source References; refreshed
  `wiki/index.md`, `wiki/v17/index.md`, and the v17 coverage cell.
  `verified_by_agent` stays `not yet`: scoped follow-up, not a full
  claim-by-claim re-verification.
- 2026-07-09: filed [Table Partitioning Optimizations and Configuration During
  Query Planning and Execution in PostgreSQL 17
  (unverified)](v17/questions/partitioning-planning-execution-optimizations.md)
  against pin `54eeefaedbee0385529f3edf321bb99e49232aaa`. Applied prompt hygiene:
  the asker approved a grammar-corrected restatement (recorded under
  `## Question`). Drafted from a claim-to-source map built directly from
  `raw/postgres-17/`: the inheritance-vs-declarative split; plan-time constraint
  exclusion (`relation_excluded_by_constraints`); declarative plan-time pruning
  (`expand_partitioned_rtentry` prunes via `prune_append_rel_partitions` before
  locking only `live_parts` with `try_table_open`) and run-time pruning on
  `Append`/`MergeAppend` (`make_partition_pruneinfo`, `ExecInitPartitionPruning`
  / unified `ExecFindMatchingSubPlans`); per-HASH/LIST/RANGE bound matching
  (`perform_pruning_base_step` + `get_matching_*_bounds`); partitionwise join
  with merged non-identical bounds (`build_joinrel_partition_info` /
  `have_partkey_equi_join` / `compute_partition_bounds` -> `partition_bounds_merge`);
  partitionwise aggregate FULL/PARTIAL (`create_ordinary_grouping_paths` /
  `group_by_has_partkey`); executor tuple routing (`ExecFindPartition` /
  `get_partition_for_tuple` with last-found cache), cross-partition UPDATE row
  movement (`ExecCrossPartitionUpdate`), and per-partition COPY batching
  (`CIM_MULTI_CONDITIONAL`); and the four `PGC_USERSET` GUCs with defaults. Added
  an "optimizations since v12" section attributing (via the checkout's own
  `Stamp HEAD as NNdevel` brackets) v13 advanced partitionwise join
  (`c8434d64ce0`), v14 run-time pruning for `UPDATE`/`DELETE`/`ModifyTable`
  (`86dc90056df`; v12 doc caveat removed by `1692d0c3a3f`) and
  `DETACH PARTITION CONCURRENTLY` (`71f4c8c6f74`), v15 `live_parts` bitmap
  (`475dbd0b718`) / run-time-prune refactor (`297daa9d435`) / `MERGE`
  (`7103ebb7aae`), v16 cached tuple routing (`3592e0ff98b`), and v17
  `boolcol IS [NOT] UNKNOWN` pruning (`07c36c1333e`) + concurrent-detach
  robustness (`11f1218ce81`). Updated `wiki/index.md`, `wiki/v17/index.md`, and
  the v17 coverage cell. `verified_by_agent` stays `not yet`: fresh draft, not a
  claim-by-claim re-verification.
- 2026-07-09: extended [Table Partitioning Optimizations and Configuration During
  Query Planning and Execution in PostgreSQL 12
  (unverified)](v12/questions/partitioning-planning-execution-optimizations.md)
  with a new `## Locking reduction and random-I/O sensitivity per optimization`
  section for the approved, grammar-corrected follow-up ("how each optimization
  will reduce locking, and how each will have big gains if random I/O has higher
  latency"; original wording recorded under `## Question`). Same unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`, drafted from a claim-to-source map
  built directly from `raw/postgres-12/`. Locking: expansion/locking runs before
  costing (`planmain.c` `add_other_rels_to_query` -> `make_one_rel`); plan-time
  pruning is the only optimization that reduces read locks because
  `expand_partitioned_rtentry` calls `prune_append_rel_partitions` before
  `table_open(childOID, lockmode)` on the surviving `live_parts`, whereas
  inheritance locks every member up front via `find_all_inheritors` and
  constraint exclusion (`relation_excluded_by_constraints` in
  `set_append_rel_size`) runs only afterward; run-time pruning keeps all locks
  because `AcquireExecutorLocks` locks every `RTE_RELATION` in the finished plan's
  rtable; write-path routing (`ExecInitPartitionInfo`) takes `RowExclusiveLock`
  lazily per touched partition, and cross-partition `UPDATE` locks two. Random
  I/O: `random_page_cost` = 4.0 vs `seq_page_cost` = 1.0 (`cost.h`/`costsize.c`),
  the docs note the 4.0 default assumes ~90% cache and should be raised for slow
  random storage, index-scan heap fetches are charged at `random_page_cost`
  (`cost_index`) and seq scans at `seq_page_cost` (`cost_seqscan`), so the
  scan-eliminating optimizations (pruning, constraint exclusion) win most on
  high-latency random storage; partitionwise join/aggregate are CPU/memory/
  locality wins, not random-I/O ones (v12 costs hash-join `BufFile` spill as
  sequential in `initial_cost_hashjoin`, and hash aggregation `build_hash_table`
  is in-memory). Updated `## Contents`, `## Question` (follow-up), Evidence Map,
  Context Reviewed, Open Questions, and Source References; refreshed
  `wiki/index.md`, `wiki/v12/index.md`, and the v12 coverage cell.
  `verified_by_agent` stays `not yet`: scoped follow-up, not a full
  claim-by-claim re-verification.
- 2026-07-09: filed [Table Partitioning Optimizations and Configuration During
  Query Planning and Execution in PostgreSQL 12
  (unverified)](v12/questions/partitioning-planning-execution-optimizations.md)
  against the unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`. Applied
  prompt hygiene: the asker approved a grammar-corrected restatement (recorded
  under `## Question`). Drafted from a claim-to-source map built directly from
  `raw/postgres-12/`: partition pruning (plan-time `prune_append_rel_partitions`
  invoked from `expand_partitioned_rtentry` before child expansion, so pruned
  partitions never become child rels; run-time initial + per-scan pruning via
  `ExecFindInitialMatchingSubPlans`/`ExecFindMatchingSubPlans` on
  `Append`/`MergeAppend`, gated by `enable_partition_pruning` in
  `prune_append_rel_partitions` and `createplan.c`) with per-strategy HASH/LIST/
  RANGE differences in `get_matching_hash_bounds`/`get_matching_list_bounds`/
  `get_matching_range_bounds` and `get_partition_for_tuple`; constraint exclusion
  (`relation_excluded_by_constraints` switch on `constraint_exclusion` modes,
  plan-time only, mainly legacy inheritance); partitionwise join
  (`build_joinrel_partition_info` gate + `have_partkey_equi_join` +
  `partition_bounds_equal`) and partitionwise aggregate (`create_grouping_paths`
  patype, `group_by_has_partkey` FULL-vs-PARTIAL); executor tuple routing,
  cross-partition UPDATE row movement (DELETE+INSERT), and per-partition COPY
  multi-insert; the four session-scoped GUCs with defaults; and the v12
  limitation that execution-time pruning covers only `Append`/`MergeAppend`, not
  `ModifyTable`. Updated `wiki/index.md`, `wiki/v12/index.md`, and the v12
  coverage cell. `verified_by_agent` stays `not yet`: fresh draft, not a
  claim-by-claim re-verification.
- 2026-07-07: review-fixed the same v14 RLS sub-SELECT follow-up to remove an
  uncited external-helper assumption. The PostgreSQL 14 source still proves the
  uncorrelated scalar sub-SELECT -> InitPlan -> `PARAM_EXEC` once-per-execution
  behavior, including inside RLS policy expressions, but the exact definitions
  and volatility declarations of external helpers such as `auth.uid()` and
  `auth.jwt()` are now listed as an Open Question to verify in the installed
  schema. Tightened the answer lead to note the main performance win is when the
  unwrapped call would otherwise be a per-row filter; index-qualifiable
  comparisons may already evaluate a `STABLE` function once per scan as a
  runtime key. `verified_by_agent` stays `not yet`: scoped review-fix, not a
  full claim-by-claim re-verification.
- 2026-07-07: review-fixed the `### Wrapping a function in a subquery to
  evaluate it once` section of [Row-Level Security (RLS) in PostgreSQL 14
  (unverified)](v14/questions/row-level-security-rls.md) against the unchanged
  pin `5c00f4e2e3b`. Re-verified every source citation in the section (the
  `subselect.c` uncorrelated-`EXPR_SUBLINK`-to-InitPlan / `PARAM_EXEC` path and
  correlated-`SubPlan` contrast, `ExecEvalParamExec`/`ExecSetParamPlan`
  once-only param caching, `ExecScan`/`EEOP_FUNCEXPR` per-tuple evaluation, the
  `clauses.c` `STABLE`-not-folded rule, `ExecReScanIndexScan` runtime keys,
  `policy.c` `hassublinks`, `rewriteHandler.c` policy-sublink handling, and
  `pg_proc.dat` `current_setting` `provolatile => 's'`); all line ranges are
  accurate. Fixed one citation-completeness gap: the claim that `WITH CHECK`
  options also flow through `preprocess_expression`/`SS_process_sublinks` cited
  only the `securityQuals` loop, so added the `WITH CHECK` preprocessing loop
  citation (`planner.c` L818-L828) in the body, Evidence Map, and Context
  Reviewed. `verified_by_agent` stays `not yet`: scoped review-fix, not a full
  claim-by-claim re-verification.
- 2026-07-07: review-fixed [Performance Implications of Functions and
  Procedures in a WHERE Clause in PostgreSQL 14, and How to Minimize the
  Overhead (unverified)](v14/questions/functions-procedures-in-where-clause.md)
  against the unchanged pin `5c00f4e2e3b`. Corrected the overbroad claim that a
  WHERE-clause function is always a per-row cost: row-dependent or volatile
  quals still run through the `ExecScan`/`ExecQual`/`EEOP_FUNCEXPR` per-row path,
  but a no-`Var`, non-volatile qual is marked pseudoconstant, costed as startup
  work, and executed once as a gating `Result` / `One-Time Filter`. Tightened
  the `STABLE` caveat, the scalar sub-SELECT mitigation caveat, the `COST`
  default wording (`CREATE FUNCTION` defaults to 1 for C/internal and 100
  otherwise), and the expression-index/generated-column immutability citations.
  Updated `wiki/index.md`, `wiki/v14/index.md`, and the v14 coverage cell.
  `verified_by_agent` stays `not yet`: scoped review-fix, not a full
  claim-by-claim re-verification.
- 2026-07-07: filed [Performance Implications of Functions and Procedures in a
  WHERE Clause in PostgreSQL 14, and How to Minimize the Overhead
  (unverified)](v14/questions/functions-procedures-in-where-clause.md) against
  the unchanged pin `5c00f4e2e3b`. Applied prompt hygiene: the asker approved a
  grammar-corrected restatement (recorded under `## Question`) and chose to
  cover procedures explicitly. Drafted from a claim-to-source map built directly
  from `raw/postgres-14/`: procedures (`prokind='p'`) are rejected in every
  expression context by `ParseFuncOrColumn` and run only via the `CALL` utility
  statement (`gram.y`/`transformCallStmt`/`ExecuteCallStmt`, with a
  `create_procedure.out` test); row-dependent or volatile WHERE-clause functions
  are per-row costs (`ExecScan` qual loop -> `ExecQual` -> `EEOP_FUNCEXPR` fmgr
  dispatch), while no-`Var`, non-volatile quals can run once as pseudoconstant
  gating `Result` filters; volatility effects (`VOLATILE` per row where needed,
  `STABLE` single-evaluation as an index-scan comparison value via
  `ExecReScanIndexScan`/`ExecIndexEvalRuntimeKeys` or as a no-`Var` one-time qual,
  `IMMUTABLE` constant-folded by `eval_const_expressions`/`evaluate_function`);
  the cost model (`add_function_cost` = `procost * cpu_operator_cost`, omitted
  `CREATE FUNCTION COST` default 1 for C/internal functions and 100 otherwise)
  and fixed default selectivity (`function_selectivity` 0.3333333, `boolvarsel`
  0.5); expression-index matching (`match_index_to_operand`) and the
  non-volatile-operand runtime-key rule (`match_opclause_to_indexcol`);
  leakproof/security-level qual ordering (`order_qual_clauses`,
  `RestrictInfo.security_level`, `qual_is_pushdown_safe`); parallel-safety
  (`PROPARALLEL_*`, `planner.c` parallel-mode gate, `parallel.sgml`); and the
  mitigations (volatility labeling, expression indexes/generated columns,
  uncorrelated scalar sub-SELECT -> once-per-execution InitPlan via
  `build_subplan`/`ExecEvalParamExec`, SQL-function inlining `inline_function`,
  `COST`/`ROWS`, planner `SUPPORT` functions `supportnodes.h`, `LEAKPROOF`/
  `PARALLEL SAFE`). Updated `wiki/index.md`, `wiki/v14/index.md`, and the v14
  coverage cell. `verified_by_agent` stays `not yet`: fresh draft, not a
  claim-by-claim re-verification.
- 2026-07-07: extended [Row-Level Security (RLS) in PostgreSQL 14
  (unverified)](v14/questions/row-level-security-rls.md) with a `### Wrapping a
  function in a subquery to evaluate it once` section for the approved,
  grammar-corrected follow-up on whether wrapping a JWT helper like `auth.uid()`
  in `(SELECT auth.uid())` caches the call. Documented from `raw/postgres-14/`:
  an uncorrelated scalar sub-SELECT becomes an InitPlan whose `PARAM_EXEC` result
  is evaluated once per execution and reused (`build_subplan` EXPR_SUBLINK path,
  `ExecSetParamPlan`/`ExecEvalParamExec`); the sublink is expanded even inside a
  policy because security quals and `WITH CHECK` options run through
  `SS_process_sublinks`; a bare `STABLE` function is not constant-folded and is
  re-invoked per row as a scan filter (`ExecScan`/`EEOP_FUNCEXPR`); the
  index-runtime-key nuance (a `STABLE` function on an index-qualifiable
  comparison is already evaluated once per scan); and the correlated-sub-SELECT
  and once-per-execution caveats. Noted that `auth.uid()`/`auth.jwt()` are
  external functions absent from the pin, and originally compared the common
  claim-reader pattern with core `current_setting`; a later same-day review note
  above narrows that to a verification caveat. Updated `wiki/index.md`,
  `wiki/v14/index.md`, and the v14 coverage cell; `verified_by_agent` stays `not
  yet` because this was a scoped follow-up, not a full claim-by-claim
  re-verification.
- 2026-07-07: review-fixed [Row-Level Security (RLS) in PostgreSQL 14
  (unverified)](v14/questions/row-level-security-rls.md) for the approved,
  grammar-corrected aggregation follow-up. Added an `### RLS and aggregation`
  section explaining that aggregate nodes consume the RLS-filtered input
  subplan, so mitigation focuses on cheaper/indexable policy predicates,
  leakproof/indexable filters, partition pruning and session-scoped
  `enable_partitionwise_aggregate`, stable role/`row_security` plan-cache
  environments, or deliberately trusted/pre-aggregated reporting paths. Also
  documented that `row_security = off` errors for subject users rather than
  bypassing RLS, and that aggregate calls are rejected inside policy
  expressions. Updated `wiki/index.md` and `wiki/v14/index.md`;
  `verified_by_agent` stays `not yet` because this was a scoped follow-up, not
  a full claim-by-claim re-verification.
- 2026-07-07: review-fixed [Row-Level Security (RLS) in PostgreSQL 14
  (unverified)](v14/questions/row-level-security-rls.md) against the unchanged
  pin `5c00f4e2e3b`. Corrected the RLS plan-cache section to avoid treating SQL
  `DECLARE CURSOR` / `WITH HOLD` cursors as a saved-plan-cache reuse surface
  (ordinary cursor planning stores copied plan trees in a portal; held cursors
  survive by materializing rows), added `plan_cache_mode` as an RLS-relevant
  `PGC_USERSET` setting controlling generic-vs-custom prepared-statement
  planning, fixed the distinct `SET ROLE` vs `SET SESSION AUTHORIZATION` call
  paths, and cited pg_dump's default `row_security = off` behavior. Updated
  `wiki/index.md` and `wiki/v14/index.md`; `verified_by_agent` stays `not yet`
  because this was a scoped review-fix, not a full claim-by-claim
  re-verification.
- 2026-07-06: filed the PostgreSQL 14 MultiXact question page, [How MultiXact
  Works in PostgreSQL 14, and How Foreign Keys and Other Operations Degrade
  Performance When the Local Cache Spills to Secondary Storage
  (unverified)](v14/questions/multixact-foreign-keys-cache-spill.md), against the
  unchanged pin `5c00f4e2e3b`. Applied prompt hygiene: the asker approved a
  lightly grammar-corrected restatement, recorded under `## Question`. Drafted
  from a claim-to-source map built directly from `raw/postgres-14/`: the MultiXact
  manager (`multixact.c`) — two SLRUs, `MultiXactStateData`, the create/read
  paths, and the 256-entry per-transaction local cache with LRU tail eviction;
  the SLRU layer (`slru.c`) — exclusive-locked reads, buffer-miss physical
  `pg_pread`/`SLRURead`, victim selection and I/O waits; the row-lock producers
  (`heapam.c` `compute_new_xmax_infomask`/`heap_lock_tuple`, `heapam_handler.c`,
  `ri_triggers.c` `FOR KEY SHARE`); member-space guards and
  `MultiXactMemberFreezeThreshold`; the vacuum GUC contexts (`guc.c`);
  observability (`pg_stat_slru`, wait-event names, `mxid_age`,
  `pg_get_multixact_members`); the `fk-contention` isolation test; and the
  "Multixacts and Wraparound" docs. Updated `wiki/index.md`, `wiki/v14/index.md`,
  and the v14 coverage cell above. `verified_by_agent` stays `not yet`: fresh
  draft, not a claim-by-claim re-verification.
- 2026-06-30: filed the first PostgreSQL 14 question page, [Row-Level Security
  (RLS) in PostgreSQL 14 (unverified)](v14/questions/row-level-security-rls.md),
  against the unchanged pin `5c00f4e2e3b`. Drafted from a claim-to-source map
  built directly from the `raw/postgres-14/` checkout: the rewrite path
  (`get_row_security_policies`, `fireRIRrules`), the bypass decision
  (`check_enable_rls`, `has_bypassrls_privilege`, `InNoForceRLSOperation`),
  `pg_policy`/`pg_class`/`pg_authid` catalogs, the relcache build
  (`RelationBuildRowSecurity`), the planner `security_level`/`order_qual_clauses`
  leakproof mechanism and inheritance handling, executor `ExecWithCheckOptions`,
  plan-cache `dependsOnRLS` re-planning, the COPY/CTAS/RI/error-leak boundaries,
  and all settings (`row_security` GUC, the `ALTER TABLE` table flags, the
  `BYPASSRLS` role attribute, `CREATE`/`ALTER POLICY` options, and the two
  extension hooks). Confirmed v14 has no `prepsecurity.c` (RLS quals flow through
  the `qual_security_level` machinery). Updated `wiki/index.md`,
  `wiki/v14/index.md`, and the v14 coverage cell above. `verified_by_agent`
  stays `not yet`: this is a fresh draft, not a claim-by-claim re-verification.
- 2026-06-30: added PostgreSQL 14 as a supported (`active`) version. Created the
  `raw/postgres-14/` source checkout and pinned it to the `REL_14_STABLE` tip
  `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156` (`REL_14_23-3-g5c00f4e2e3b`, three
  commits past the 14.23 release stamp; source version 14.23), mirroring how
  v17/v18 pin to their `_STABLE` tips. The checkout was built offline by local
  clone from the existing `raw/postgres-12/` repo (which already carried the
  `REL_14_STABLE` objects and all REL_14 tags), then its `origin` was set to
  `https://github.com/postgres/postgres.git`; working tree is clean at the pin.
  Filed the mandatory [PostgreSQL 14 Codebase Navigation Guide
  (unverified)](v14/codebase-navigation-guide.md) and the
  [v14/index](v14/index.md) landing page, with every guide citation verified
  against the v14 checkout (line numbers differ from v17, and v14's subsystem
  layout genuinely differs — no backend `archive`/`backup` dirs, no storage
  `aio` dir, a single `pg_analyze_and_rewrite`, and catalog headers generated by
  the backend catalog makefile rather than `src/include/catalog/`).
  `verified_by_agent` stays `not yet`: this was a version-add scaffold, not a
  claim-by-claim re-verification.
- 2026-06-29: reviewed [PostgreSQL 18 Row-Level Security
  (unverified)](v18/questions/row-level-security-rls.md) against the unchanged
  pin `6cb307251c`. Re-verified that all 120 unique source citations bracket
  their named symbols, spot-checked the most claim-sensitive source
  (`row_security` GUC, `check_enable_rls()`, the RLS `COPY`/pg_dump paths, the
  CVE-2025-8713 evidence), and independently rebuilt the fixes and
  minor-release-first-appearance provenance from local history — all 20 fix
  hashes, the five CVEs, the four not-back-patched fixes, every back-patch
  reach, and every per-branch first-release tag still hold. Two value
  corrections: the `bd3611db5a` row's 13.5/12.9 minor date `2021-11-11` ->
  `2021-11-08` (the `REL_12_9`/`REL_13_5` stamp date, matching the page's stated
  method), and the `CREATE POLICY`-statement count `97` -> `100` in the
  performance section. `verified_by_agent` stays `not yet`: scoped review-fix,
  not a full claim-by-claim re-verification.
- 2026-06-29: moved the `raw/postgres-19/` source checkout to the declared pin
  `cdae794af31b3e9cfc323fc654292d86fa746f77`. The 2026-06-26 repin updated every
  `wiki/v19/**` page's `pinned_commit:` and line anchors to `cdae794a` but left
  the standalone source checkout at the prior pin `9a60f295` (which does not
  contain `cdae794a`), so page citations could not be verified against the
  evidence base. Fetched upstream `master`+tags and checked out `cdae794a`
  (26 commits ahead of `9a60f295`; tip "Take into account default_tablespace
  during MERGE/SPLIT PARTITION(S)"). Then reviewed the commit sections of all
  three v19 question pages against the pin and verified every listed commit for
  hash, subject, author, and ancestry: `pg_plan_advice` (50 commits: 20 core, 22
  module, 8 support; exact 22-commit module completeness), `REPACK` (42
  feature-scope commits; completeness re-checked by file-path and subject grep),
  and autovacuum/VACUUM (24 commits plus `f83d709760d`/`4abf411e232`/
  `a7f59b252a8`; post-beta1 and repin-range claims confirmed). One correction:
  `pg_plan_advice` `47c110f7` had used its commit date `2026-03-26`; switched to
  the author date `2026-03-20` to match the author-date convention the other two
  pages use, with a "Dates are author dates." note added and the `b3a95566`
  repin-range wording tightened. `verified_by_agent` remains `not yet` on all
  v19 pages: this was a commit-section review, not a full claim-by-claim
  re-verification.
- 2026-06-26: repinned PostgreSQL 19 from `9a60f295` to
  `cdae794af31b3e9cfc323fc654292d86fa746f77` after fetching upstream `master`
  and tags (26 new commits; new tip `cdae794a`, 2026-06-26, "Take into account
  default_tablespace during MERGE/SPLIT PARTITION(S)"). Reviewed all v19
  question pages and the codebase navigation guide against the changed-file
  range. No `pg_plan_advice`, REPACK-specific, or
  autovacuum/scoring/parallel-vacuum/visibility feature files changed in
  `9a60f295..cdae794a`; relevant commit references were added as review notes:
  `b3a95566` only touched the out-of-scope `pg_stash_advice` module,
  `4abf411e232` changed `pg_stat_io` autovacuum-launcher accounting outside the
  filed autovacuum feature scope, and `a7f59b252a8` shifted
  `guc_parameters.dat` line anchors. Refreshed GUC citations on the REPACK and
  autovacuum pages and `execnodes.h` line anchors in the codebase guide.
  `verified_by_agent` remains `not yet` on v19 pages because this was a repin
  and changed-file review, not a full claim-by-claim re-verification.
- 2026-06-22: repinned PostgreSQL 19 from `ff8bec8c` to
  `9a60f295bcb186a729d04e76377b7f122b2a1dd9` after fetching upstream `master`
  (16 new commits; new tip `9a60f295`, 2026-06-22, "Strip removed-relation
  references from PlaceHolderVars at join removal"). Reviewed all v19 question
  pages and the codebase navigation guide against the changed-file range. Only
  one changed file is cited by any v19 page — `autovacuum.c` on the autovacuum
  scoring page — and only the autovacuum MXID-score division-by-zero fix
  `1f2297b5487` (post-`REL_19_BETA1`, touching `autovacuum.c` and
  `maintenance.sgml`) is relevant to filed coverage. Updated [PostgreSQL 19
  Autovacuum and VACUUM (unverified)](v19/questions/autovacuum-parallel-scoring-visibility.md):
  documented the `Max(1, multixact_freeze_max_age)` divisor guard and the
  member-space scaling, added `1f2297b5487` to the scoring commit history,
  corrected the now-stale "no autovacuum feature files changed since beta1"
  note, and shifted the four post-line-3048 `autovacuum.c` citations
  (`L3064-L3078`->`L3067-L3081`, `L3195-L3238`->`L3198-L3243`,
  `L3239-L3240`->`L3244-L3245`, `L3287-L3315`->`L3292-L3320`). Verified no
  `pg_plan_advice`, `REPACK`, or visibility-map/pruning feature files changed in
  the range, so those pages took only the pin bump. `verified_by_agent` remains
  `not yet` on all v19 pages: this was a repin and changed-file review, not a
  full claim-by-claim re-verification.
- 2026-06-19: resolved the remaining PostgreSQL 18 Row-Level Security open
  question — minor-release first-appearance labels. Against the unchanged pin
  `6cb307251c` (a `REL_18_STABLE` commit; that branch forked from master at
  `9c5b9a280c` on 2025-06-29), traced each of the 20 `### Fixes Since PostgreSQL
  12` fixes by finding every same-subject commit across all refs
  (`git log --all --fixed-strings --grep`) and mapping each distinct per-branch
  cherry-pick to its earliest containing release tag
  (`git tag --contains | grep '^REL_NN_M$' | sort -V | head -1`), with tag stamp
  dates from `git log -1 --format=%cs`. Added a `### Minor-Release First
  Appearance` section and table giving each fix's master development-cycle major
  plus its first minor release on every tracked v12+ branch, the back-patch
  reach, and coordinated release dates. Key findings: 16 of 20 fixes were
  back-patched (so they first shipped in coordinated back-branch minors that
  precede the master `.0`), 4 were not back-patched (`a2ab9c06ea`->15.0,
  `6572bd55b0`->17.0, `0dca5d68d7`->18.0, `cd3c45125d`->18.0), and two listed
  hashes (`64f77c6a65` CVE-2025-8713, `749f4ce4d9` doc fix) are REL_18_STABLE
  back-patch commits — not master commits — whose master commits
  (`22424953cde`, `7dc4fa91413`) are on the unreleased v19 line; corrected the
  Fixes-table intro accordingly. Removed the resolved open-question bullet and
  updated Contents, Context Reviewed, Evidence Map, and the index/landing
  summaries. `verified_by_agent` stays `not yet`: this was a targeted
  open-question investigation, not a full-page re-verification.
- 2026-06-19: re-investigated and resolved the commit-provenance open question on
  the PostgreSQL 18 Row-Level Security page now that `raw/postgres-18/` is
  unshallowed (the follow-up the unshallow note below flagged). Against the
  unchanged pin `6cb307251c` (`REL_18_3-113-g6cb307251c5`), verified with `git`
  that all 20 commits in the page's `### Fixes Since PostgreSQL 12` table exist,
  that their subjects match the summaries, and that each is an ancestor of the
  pin and a descendant of the v13 development stamp `615cebc94b5` (2019-07-01);
  confirmed the five RLS CVEs (CVE-2023-2455, -2023-39418, -2024-4317,
  -2024-10976, -2025-8713); and corrected the `d907bd0543` row, which had
  described "altering BYPASSRLS-bearing roles" but actually fixed an over-broad
  superuser requirement on *any* property change of a BYPASSRLS role. A
  completeness scan of `615cebc94b5..6cb307251c` (core RLS files plus an
  RLS/policy/WCO/CVE keyword search) found no significant standalone RLS fix
  missing; the extra commits are companion/follow-up fixes to already-listed
  areas, docs/wording/cosmetic changes, or the v15 security-invoker-views
  feature already covered. Relabeled the table's `Unverified fix description`
  column to `Fix description`, rewrote the table intro and the Open Questions /
  Context Reviewed / Evidence Map provenance items, and refreshed the
  shallow-caveat wording in the v18 row above and the `index.md` / `v18/index.md`
  summaries. Minor-release first-appearance labels remain open (most fixes were
  back-patched to all supported branches, so the first shipping minor release
  needs per-branch cherry-pick tracing). `verified_by_agent` stays `not yet`:
  this was a targeted open-question investigation, not a full-page
  re-verification.
- 2026-06-19: unshallowed the `raw/postgres-18/` source checkout at the user's
  request (`git fetch --unshallow --tags origin`), converting it from a depth-1,
  single-commit clone (1 commit, 0 tags) to full history. The pin is unchanged:
  HEAD stays at `6cb307251c5c6261286c1566496920976640108e`, which `git describe`
  now resolves as `REL_18_3-113-g6cb307251c5` on `origin/REL_18_STABLE`; the
  working tree is clean and the checkout now holds 62,317 commits and 685 tags
  (parity with the v17/v12 checkouts). No `pinned_commit:` value changed on any
  of the 15 `wiki/v18/**` pages and no source-citation line numbers shift, so
  this is not a repin. This lifts the shallow-checkout limitation previously
  cited as the reason v18 commit provenance was left open (the RLS page Open
  Questions and the v18 row/summaries below): local `git log`/tags can now verify
  it. That provenance has not been re-investigated yet, so those "shallow" notes
  remain to be refreshed when it is. `raw/postgres-12/`, `raw/postgres-17/`, and
  `raw/postgres-19/` were left untouched.
- 2026-06-19: investigated the first `## Open Questions` item on the PostgreSQL
  18 Row-Level Security page against pinned commit
  `6cb307251c5c6261286c1566496920976640108e` — whether the checkout has a
  benchmark that quantifies RLS overhead by policy/role/partition count or
  predicate shape. Confirmed it does not: the tree's benchmark/perf tooling
  (`pgbench`, `contrib/intarray/bench`, the ecpg and JSON-parser perf tests) has
  no RLS scenario, the RLS regression and hook tests use only
  `EXPLAIN (COSTS OFF)` plan-shape checks (no timing primitives), and the docs
  give only qualitative guidance. Expanded `### Scalability and Performance
  Issues`, added Context Reviewed and Evidence Map entries, and removed the
  resolved open-question bullet. `verified_by_agent` stays `not yet` because
  this was a targeted open-question investigation, not a full-page
  re-verification.
- 2026-06-19: reviewed the `### Can walsenders or replication-slot xmin holders
  appear in the Wait 3 set?` section of the PostgreSQL 12 `CREATE INDEX
  CONCURRENTLY` page against pinned commit
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Re-verified the three independent
  reasons walsenders/slots stay out of `WaitForOlderSnapshots`: (1) slot xmin is
  a global `procArray->replication_slot_xmin`/`replication_slot_catalog_xmin`
  that `GetCurrentVirtualXIDs` never reads (only `GetOldestXmin` and
  `GetSnapshotData`'s `RecentGlobalXmin` do); (2) a physical walsender connects
  to no database and `GetCurrentVirtualXIDs` lacks `GetOldestXmin`'s "always
  include WalSender" clause; (3) the valid-VXID gate plus the `lxid`/`xmin` clear
  in `ProcArrayEndTransaction`. Confirmed the only in-set walsender is a logical
  one mid-`REPEATABLE READ` snapshot export (`SnapBuildInitialSnapshot`), an
  ordinary in-database holder. Verified xmin-setter exhaustiveness by grepping
  all `(MyPgXact|pgxact)->xmin =` sites. Conclusion holds. Fixed one precision
  error shared by the body and an Evidence Map row: `xmin` is not "set in
  `StartTransaction`" (only `lxid` is; `xmin` is set at snapshot time), so
  reworded to state that an ordinary backend only advertises xmin from within a
  transaction and `lxid`/`xmin` are cleared together at transaction end.
  `verified_by_agent` stays `not yet` because this was a scoped section review,
  not a full-page re-verification.
- 2026-06-19: reviewed the `### Is skipping prepared transactions in the writer
  waits safe?` section of the PostgreSQL 12 `CREATE INDEX CONCURRENTLY` page
  against pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42`. Re-verified
  every claim and citation: the `lmgr.c` "prepared xacts certainly aren't going
  to do anything anymore" comment; the `validate_index` "terminate"/"inserted by
  their originating transaction" invariant; the Wait 1 HOT-safety comment;
  `MarkAsPreparingGuts` setting the dummy proc's `xid` to the real XID but
  `xmin`/`backendId` invalid; the `GetLockConflicts` invalid-VXID drops and the
  `VirtualTransactionIdIsValid` `backendId` check; the Wait 3 `xmin`/VXID filter;
  the `SnapshotAny` `INSERT_IN_PROGRESS` branch vs. the MVCC `else` branch;
  `FinishPreparedTransaction` doing no index work; the not-ready-index insert
  skip; and the non-concurrent `ShareLock`-vs-`RowExclusiveLock` contrast. Also
  confirmed the negative claims (no "prepared" mention in `create_index.sgml`, no
  prepared/two-phase handling in `indexcmds.c`). The "Not on its own [safe]"
  conclusion holds. Fixed one precision error: the writer-wait bullet implied the
  prepared xact's invalid VXID is dropped in "both the fast-path and
  primary-table scans," but its `RowExclusiveLock` is transferred to the primary
  lock table at prepare, so it is encountered and dropped only in the
  primary-table scan; reworded to say both `GetLockConflicts` scan phases drop
  invalid VXIDs in general while the prepared xact hits only the primary-table
  one. `verified_by_agent` stays `not yet` because this was a scoped section
  review, not a full-page re-verification.
- 2026-06-19: reviewed the `## Open Questions` section of the PostgreSQL 12
  `CREATE INDEX CONCURRENTLY` page against pinned commit
  `45b88269a353ad93744772791feb6d01bc7e1e42`. Re-verified every crash /
  immediate-shutdown citation behind the "None for source behavior" conclusion
  (xidless `heap_inplace_update` flag flips, the async-commit branch,
  `heap_xlog_inplace` physical redo, `XLogFlush`-through-position monotonicity,
  and the `DB_SHUTDOWNED` recovery trigger) — the conclusion holds. Closed one
  precision gap: the "recovered valid index is always complete" argument cited
  build durability only for B-tree (`smgrimmedsync`); verified the other five
  core AMs (hash/GiST/SP-GiST/GIN/BRIN) make build pages durable before
  SET_VALID through shared-buffer WAL (`log_newpage_range` / buffered build
  path), and that unlogged tables reset heap and indexes together on crash.
  Rewrote the completeness paragraph to be AM-general, split the Evidence Map
  completeness row and added an unlogged-reset row, added Source References and
  a Context Reviewed entry, and tightened the `## Open Questions` text while
  keeping the "None" marker. `verified_by_agent` stays `not yet` because this
  was a scoped review, not a full-page re-verification.
- 2026-06-19: expanded PostgreSQL 12 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42` to resolve
  the crash / immediate-shutdown recovery open question. Source-level
  conclusion: a crash or `immediate` shutdown leaves the same four states as an
  ERROR/cancel and never a valid-but-incomplete index. The state-flag flips run
  in XID-less transactions and are written by `heap_inplace_update`
  (`XLOG_HEAP_INPLACE`, page LSN set, no `XLogFlush`), so `RecordTransactionCommit`
  commits them asynchronously (`XLogSetAsyncXactLSN`, not `XLogFlush`) regardless
  of `synchronous_commit`; a flip therefore becomes durable only when the WAL
  writer, a checkpoint, or a later synchronous commit flushes WAL past its LSN.
  On recovery `heap_xlog_inplace` redoes the flip physically and unconditionally,
  so a flip can survive a phase that wrote no commit record, or be lost although
  `CommitTransactionCommand` returned — but never inconsistently: `XLogFlush`
  flushes WAL sequentially, so durable SET_VALID implies durable SET_READY (state
  monotone, never `(indisready=f, indisvalid=t)`), and a recovered valid index is
  always complete because the B-tree build `smgrimmedsync`s its file before the
  build transaction may commit and the validate scan's WAL precedes SET_VALID
  under the `FlushBuffer` WAL-before-data rule. Rewrote the
  `#### Server crash or immediate shutdown` subsection, added a recovered-state
  table, six Evidence Map rows, a Context Reviewed entry, eight Source References,
  and replaced the resolved `## Open Questions` bullet with the "none" marker.
  `verified_by_agent` stays `not yet` because this was a targeted expansion, not
  a full-page re-verification.
- 2026-06-19: expanded PostgreSQL 12 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42` to resolve
  the walsender / replication-slot Wait 3 open question. Source-level
  conclusion: walsenders and replication-slot xmin holders do **not** appear in
  CIC's `WaitForOlderSnapshots` (Wait 3) set through replication's
  xmin-holdback machinery. A slot's reserved xmin/catalog_xmin lives in the
  global `procArray->replication_slot_xmin`/`replication_slot_catalog_xmin`
  (set by `ProcArraySetReplicationSlotXmin`), which `GetCurrentVirtualXIDs`
  never reads — only `GetOldestXmin` and `GetSnapshotData`'s
  `RecentGlobalXmin` do. A physical walsender that advertises its own
  `MyPgXact->xmin` via `hot_standby_feedback` (no slot) is filtered out by
  database — `InitPostgres` returns early for `am_walsender && !am_db_walsender`
  leaving `databaseId = InvalidOid`, and `GetCurrentVirtualXIDs`'s same-db test
  lacks `GetOldestXmin`'s `|| proc->databaseId == 0 /* always include
  WalSender */` clause — and by the valid-VXID gate. `lxid` and `xmin` are set
  in `StartTransaction` and cleared together in `ProcArrayEndTransaction`, so a
  non-transaction backend has neither. The only walsender that can be in the
  set is a logical walsender running `SnapBuildInitialSnapshot` inside a
  `REPEATABLE READ` transaction (slot creation/export), which is an ordinary
  in-database snapshot holder the page already covers. Added a new
  `### Can walsenders or replication-slot xmin holders appear in the Wait 3
  set?` section, Contents entry, a cross-referenced Point 4 bullet, six
  Evidence Map rows, a Context Reviewed entry, twelve Source References, and
  removed the resolved `## Open Questions` bullet. `verified_by_agent` stays
  `not yet` because this was a targeted expansion, not a full-page
  re-verification.
- 2026-06-18: expanded PostgreSQL 12 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42` to resolve
  the prepared-transaction writer-wait open question. Source-level assessment:
  skipping prepared transactions in the two `WaitForLockers(ShareLock)` writer
  waits is not sufficient for index correctness. The `validate_index` /
  `DefineIndex` invariant requires that index-modifying transactions either
  terminate before the scans or insert their own tuples; a prepared transaction
  does neither. Its dummy proc carries the real (in-progress) `xid` but an
  invalid `xmin`/VXID, so the MVCC build and validate scans never see its
  writes, all three waits skip it, and `COMMIT PREPARED` does no index
  maintenance — a write committed after the build is left unindexed. Added a new
  `### Is skipping prepared transactions in the writer waits safe?` section,
  Contents entry, Evidence Map rows, Context Reviewed and Source References
  entries, and removed the resolved `## Open Questions` bullet.
  `verified_by_agent` stays `not yet` because this was a targeted expansion, not
  a full-page re-verification.
- 2026-06-18: expanded PostgreSQL 12 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42` to resolve
  the first-build-scan tuple-visibility open question. The page now traces, line
  by line into the heap table AM, that a concurrent first scan
  (`index_concurrently_build` -> `index_build` -> `ambuild` ->
  `table_index_build_scan` -> `heapam_index_build_range_scan`) leaves
  `OldestXmin` invalid because of `!ii_Concurrent`, scans with a fresh MVCC
  snapshot, relies on `heapgetpage` -> `HeapTupleSatisfiesMVCC` for visibility,
  and therefore never indexes `RECENTLY_DEAD` tuples nor sets
  `ii_BrokenHotChain` (unlike the non-concurrent `SnapshotAny` build).
  `verified_by_agent` stays `not yet` because this was a targeted expansion, not
  a full-page re-verification.
- 2026-06-18: expanded PostgreSQL 12 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42` to resolve
  the inter-builder open question. The page now distinguishes same-table CIC
  serialization by self-conflicting `ShareUpdateExclusiveLock`, different-table
  CIC/RIC interaction through the database-wide `WaitForOlderSnapshots` /
  `GetCurrentVirtualXIDs` old-snapshot wait, and the final no-`xmin` boundary
  that prevents mutual CIC snapshot-wait deadlocks. `verified_by_agent` was
  reset to `not yet` because this was a targeted update, not a full-page
  re-verification.
- 2026-06-18: filed PostgreSQL 12 `wal_sender_timeout` coverage against
  pinned commit `45b88269a353ad93744772791feb6d01bc7e1e42`. The page covers
  GUC scope/default/disable behavior, sender reply tracking, half-time
  keepalives and full-time disconnect, physical and logical streaming paths,
  publisher-side impact on logical subscriptions and `pg_recvlogical`, the
  subscriber-side `wal_receiver_*` distinction, monitoring and synchronous
  replication effects, replication-slot WAL retention, and the direct test
  coverage gap.
- 2026-06-18: filed PostgreSQL 18 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `6cb307251c5c6261286c1566496920976640108e`. The
  page covers utility dispatch, restrictions, four internal transactions, two
  heap scans, three waits, table/session `ShareUpdateExclusiveLock`,
  `indislive`/`indisready`/`indisvalid` progression, invalid-index failure
  outcomes, regression/isolation coverage, and v18 changes from v17: `pg_index`
  TOAST snapshot handling, parallel GIN build, virtual-generated-column
  rejection, and temporal `WITHOUT OVERLAPS` shared plumbing.
- 2026-06-18: reviewed PostgreSQL 17 `CREATE INDEX CONCURRENTLY` coverage
  against pinned commit `54eeefaedbee0385529f3edf321bb99e49232aaa`. Fixed the
  v12-era `index_set_state_flags` wording: v17 uses transactional
  `CatalogTupleUpdate` for CIC state-flag flips, not non-transactional
  `heap_inplace_update`; added the required Contents block and direct
  `WaitForLockersMultiple` evidence; advanced agent verification.
- 2026-06-18: repinned PostgreSQL 19 from `e18b0cb7` to `ff8bec8c`
  after fetching 59 new upstream `master` commits. Re-reviewed all v19
  question pages against the changed-file range. `pg_plan_advice` module/doc/test
  citations and the autovacuum/parallel-vacuum/pruning feature files did not
  change in this range. The REPACK page gained the explicit `e2a8cabc`
  concurrent-repack correctness commit, now tracking 42 feature-scope commits,
  and its `repack.c` / `parsenodes.h` citation ranges were refreshed.
- 2026-06-17: expanded PostgreSQL 17 partial-index pros/cons coverage with a
  `What changed from PostgreSQL 12` section. It records the unchanged core
  storage/build/planner/ANALYZE shape and the v17-side deltas: the post-v12
  documentation warning against using many partial indexes as partitioning, the
  PG14 concurrent-index `PROC_IN_SAFE_IC` safe-wait optimization and its
  exclusion for expression/partial indexes, PG15 MERGE target-relation rechecks,
  PG14 expression statistics as an expression-index alternative, PG15 `NULLS NOT
  DISTINCT`, PG16 partial-unique-proof restriction, and the pinned v17 stable
  partial-hash-index planning fix.
- 2026-06-17: filed PostgreSQL 17 partial-index pros/cons coverage. The page
  connects partial-index storage, ordinary expression-index behavior,
  expression-key storage, predicate validation, build/DML predicate filtering,
  planner predicate implication, expression-key matching, operational
  restrictions, `ANALYZE` row-count and expression-statistics behavior,
  planner costing, and regression plans to pinned v17 source.
- 2026-06-17: filed PostgreSQL 12 partial-index pros/cons coverage. The page
  connects partial-index storage, ordinary expression-index behavior,
  expression-key storage, predicate validation, build/DML predicate filtering,
  planner predicate implication, expression-key matching, operational
  restrictions, `ANALYZE` row-count and
  expression-statistics behavior, planner costing, and regression plans to
  pinned v12 source.
- 2026-06-17: filed PostgreSQL 12 NULL-handling-by-index-type coverage for
  B-tree, hash, GiST, SP-GiST, GIN, BRIN, and contrib Bloom. The page uses
  pinned v12 source to connect executor `isnull[]` handling, `amsearchnulls`,
  access-method insertion/search behavior, and regression-test coverage.
- 2026-06-14: tightened the PostgreSQL 12 codebase navigation guide after
  review. Added the extended-query protocol Parse/Bind/Execute path through
  `exec_parse_message()`, `exec_bind_message()`, and `exec_execute_message()`;
  added the core `src/backend/commands/extension.c` boundary for contrib
  extension control files, `CREATE EXTENSION`, and extension script execution;
  and cited the direct `standard_ProcessUtility()` `EXPLAIN`/`SET` dispatch.
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
