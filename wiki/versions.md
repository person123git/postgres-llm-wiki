# PostgreSQL Versions

This page indexes the PostgreSQL versions covered by the wiki.

## Supported Versions

| Version | Status | Wiki Home | Branch | Pinned Commit | Coverage |
|---|---|---|---|---|---|
| 19 | active | [v19/index](v19/index.md) | `master` (post-19beta1) | `9a60f295bcb186a729d04e76377b7f122b2a1dd9` | Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`; filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module (core `pgs_mask` strategy mask and planner hooks, per-index disabling, advice language, generation, enforcement, feedback/EXPLAIN, GUCs, tests) and its scoped source history (core planner-enabling commits, 22 direct module commits, and test/doc/build support commits), plus a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (blocking new-heap rewrite/swap, concurrent online rewrite via logical decoding with a decoding `bgworker`, the `pgrepack` output plugin, a temporary replication slot, change spill/replay, lock upgrade and swap, the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, and tests) with 42 feature-scope source-history commits; and a comprehensive, commit-backed walkthrough of three v19 autovacuum/VACUUM features (parallel autovacuum via the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, autovacuum table scoring via `AutoVacuumScores`/the five `*_score_weight` GUCs/`pg_stat_autovacuum_scores` plus the post-beta1 MXID-score division-by-zero fix `1f2297b5487`, and read-only query scans setting pages all-visible in the visibility map through on-access pruning); all three v19 question histories are pinned to the post-`REL_19_BETA1` `master` commit `9a60f295`. |
| 18 | primary | [v18/index](v18/index.md) | `REL_18_STABLE` | `6cb307251c5c6261286c1566496920976640108e` | Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`; filed coverage includes B-tree VACUUM density design, `pgstatindex` approximation limits and tests, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, custom cumulative statistics, extension hooks for VACUUM/autovacuum, `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `track_activity_query_size` activity text storage, `NUM_BUFFER_PARTITIONS` buffer mapping usage, GUC default-value changes since v12 through v18 (`effective_io_concurrency` and `log_connections` included), a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, limitations, and the new v18 origin-differs conflict logging) with a source-verified section on all logical replication features new since v17 (conflict detection/logging/statistics, generated-column replication, `streaming = parallel` default, alterable `two_phase`, `max_active_replication_origins`, `idle_replication_slot_timeout`, `pg_createsubscriber` and `pg_recvlogical` options, contrib `pg_logicalinspect`), and a Row-Level Security walkthrough covering rewrite-time security quals and WCOs, policy storage/DDL, bypass decisions, planner/executor effects, COPY/logical replication boundaries, performance mechanisms, settings and catalog/tool surfaces, and current-source evidence for RLS-related fixes since PostgreSQL 12 with commit provenance verified against the unshallowed checkout (every listed hash exists, subjects match, and each is an ancestor of the pin and a descendant of the v13 development stamp; completeness probed by history search, and per-branch minor-release first-appearance labels now assigned by tracing each fix's distinct cherry-picks to their earliest release tags — surfacing that two listed hashes are REL_18_STABLE back-patches whose master commits are on the unreleased v19 line), and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, table/session `ShareUpdateExclusiveLock`, `indislive`/`indisready`/`indisvalid` progression, invalid-index failure outcomes, regression/isolation coverage, and v18 changes from v17: `pg_index` TOAST snapshot handling, parallel GIN build, virtual-generated-column rejection, and temporal `WITHOUT OVERLAPS` shared plumbing). |
| 17 | active | [v17/index](v17/index.md) | `REL_17_STABLE` | `54eeefaedbee0385529f3edf321bb99e49232aaa` | Behavioral claims cite the matching pinned checkout under `raw/postgres-17/`; filed coverage includes the complete contrib extension inventory, explanations for all 53 control-file-backed contrib extensions, partial-index pros/cons with ordinary expression indexes as the baseline, partial expression indexes, planner predicate implication, expression-key matching, operational restrictions, `ANALYZE` row-count and expression-statistics behavior, planner costing, regression coverage, and a what-changed-from-v12 section covering the post-v12 documentation warning, concurrent-build safe-wait exclusion for partial/expression indexes, MERGE target rechecks, expression statistics, `NULLS NOT DISTINCT`, partial-unique proof restriction, and partial-hash planning fix, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, and a reviewed summary of the seven GUC default-value changes since v12 (old/new value, introducing major version, apply scope, exclusions, and test-coverage notes, grounded in `guc_tables.c`/`config.sgml` and the checkout's own commit history), and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, the `indislive`/`indisready`/`indisvalid` progression, a dedicated steps-and-locks section on `ShareUpdateExclusiveLock` + the `WaitForLockers`/`WaitForOlderSnapshots` waits, and a "what changed from v12" section centered on the PG14 `PROC_IN_SAFE_IC` snapshot-wait optimization, the reverted VACUUM-ignores-CIC attempt, and the `PGXACT`->`PGPROC` `statusFlags` move, anchored to the checkout's own commit history), a comprehensive walkthrough of the `REINDEX INDEX CONCURRENTLY` implementation (`ReindexRelationConcurrently`'s six phases and five waits, the atomic phase-4 `index_concurrently_swap` new-valid/old-invalid flip via the transactional `CatalogTupleUpdate`, the `_ccnew`/`_ccold` naming with a per-phase failure table showing a healthy `index_name` is never left invalid, a dedicated "can a failure leave an invalid index with the original index name?" section confirming a RIC failure never converts a bloated-but-valid `index_name` into no usable index (the phase-4 swap renames and flips validity in one transaction; only an already-invalid `index_name` stays invalid, the `concur_reindex_ind5` repair case), and a since-v12 section: partitioned REINDEX support via `ReindexPartitions` leaf recursion, `REINDEX (TABLESPACE ...)`, transactional `index_set_state_flags`, `PROC_IN_SAFE_IC` for RIC, the progress-view `WAIT_5` fix, the v15 shared-memory stats rework, and the v17 `MAINTAIN` privilege, anchored to the checkout's own commit history), and an analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes (it cannot: `AttachPartitionEnsureIndexes` is strictly match-or-create via `IndexSetParentIndex`/`DefineIndex`, incompatible/extra/invalid child indexes are retained, the three look-alike "drops" are an absorbed index losing independent droppability, the manual redundant-CHECK-constraint drop, and a later cascading parent `DROP INDEX`; with a version-by-version since-v12 section covering the unchanged no-drop invariant plus per-release matching refinements: v13 attmap refactor, v14/v15 partitioned-index DROP-path fixes, v15 `NULLS NOT DISTINCT` matching, the v16 back-patched invalid-index match skip `fc55c7ff`, the v17 PK-vs-UNIQUE constraint-type match `cee8db3f` with the `CompareIndexInfo`/`validatePartitionedIndex` fixes, and the 17.x re-attach parent-validation update `becf6d2696`), and a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging into WAL, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, and limitations) with its commit history grouped by minor version (16-cycle core commits `366283961a`/`8756930190`/`0324651573`, 17.0 `54ccfd6586`, 17.5 `0ae1245e04`, 17.10 CVE-2026-6638 `f0f59b658e`, plus foundations and adjacent origin-infrastructure commits). |
| 12 | legacy | [v12/index](v12/index.md) | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` | Behavioral claims cite the matching pinned checkout under `raw/postgres-12/`; filed coverage includes foreign-key join selectivity, partial-index pros/cons with ordinary expression indexes as the baseline, partial expression indexes, planner predicate implication, expression-key matching, operational restrictions, `ANALYZE` row-count and expression-statistics behavior, planner costing, and regression coverage, planner statistics sources (`pg_stat_all_tables`, `pg_stats`, and `pg_stats_ext` direct-use excluded; `pg_class`, `pg_statistic`, extended statistics catalogs, index/FK/constraint metadata, planner statistics hooks, and regression coverage included), `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `psql` environment variables and session timeout behavior, exact `pgstatindex` calculation behavior, planner penalties for bloated B-tree indexes through physical pages and tree height including the storage-manager path and I/O/CPU boundary for `RelationGetNumberOfBlocks()`, a proposed `avg_leaf_density` / `leaf_fragmentation` triage heuristic for REINDEX candidates, B-tree leaf density (60% vs 90%) impact on planner page-count costs and executor leaf-page walking, a density-versus-fragmentation comparison for index scan I/O with level estimates and cache/storage sensitivity, a sampling `pgstatindex` variant proposal with a `pageinspect` SQL prototype, including the v12 `bt_metap` unsigned-`oldest_xact` overflow workaround, and a three-stage REINDEX triage heuristic that shortlists candidates from `pg_class`/`pg_stat_*`/`pg_relation_size()` signals, confirms density with gated `pgstatindex` runs, prioritizes by wasted bytes times scan frequency with `REINDEX (CONCURRENTLY)` execution notes, measures post-REINDEX improvement via size/shape deltas, per-scan counter rates (per-index counters survive both REINDEX forms, including the v12 `index_concurrently_swap` stats copy), and `EXPLAIN (ANALYZE, BUFFERS)` / `pg_stat_statements` before/after windows, and documents the cited rationale for excluding `leaf_fragmentation` from the REINDEX priority score, and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation in `DefineIndex` (four internal transactions, two heap scans, three transaction waits, the `indislive`/`indisready`/`indisvalid` progression set non-transactionally by `index_set_state_flags`, the preconditions/restrictions and the invalid-index failure path, and a dedicated steps-and-locks section covering transaction- and session-level `ShareUpdateExclusiveLock`, the `WaitForLockers(ShareLock)` writer waits, `WaitForOlderSnapshots`, and the `LockConflicts` rationale for why DML proceeds while another CIC/`VACUUM`/`ANALYZE`/DDL is blocked, plus a reviewed blocker-by-blocker enumeration of every operation that can block CIC at each of its four blocking points — conflicting-lock holders at acquisition with the autovacuum-cancel and prepared-transaction nuances, open writers at the two lock waits, and same-database old-snapshot holders at the snapshot wait, with the `pg_stat_progress_create_index` wait phases and worked examples for a running `pg_dump` and an hour-long open / idle-in-transaction session, plus a failure-scenarios section mapping each CIC phase to the leftover on the table — no index, an invalid not-ready index, or an invalid ready index that still takes writes and enforces uniqueness — with each leftover's planner/write/uniqueness/HOT cost, session-lock release on ordinary ERROR/cancel paths, a resolved crash/immediate-shutdown recovery outcome (xidless `heap_inplace_update` flag flips commit asynchronously and are redone by `heap_xlog_inplace`, so recovery lands on one of the four documented leftovers and never on a valid-but-incomplete index), `DROP`/`REINDEX` recovery), a companion comprehensive walkthrough of the v12 `REINDEX INDEX CONCURRENTLY` implementation in `ReindexRelationConcurrently` (its six phases — create the `_ccnew` copy, build, validate, swap, set-dead, drop — and five wait points, namely the three shared CIC waits plus two extra `AccessExclusiveLock` "wait for readers" waits before marking the old index dead and before dropping it; the atomic `index_concurrently_swap` that marks the new index valid and the old invalid, renames `_ccnew`→original and original→`_ccold`, and moves constraints/triggers/comment/dependencies and per-index cumulative stats; `ShareUpdateExclusiveLock` held at transaction and session level on the old index, new index, and heap; the no-transaction-block/temp-fallback/system-catalog/partitioned/exclusion/invalid restrictions and toast rebuild; the two-index state-flag progression; a per-phase failure table leaving an invalid `_ccnew` before the swap or `_ccold` after, with regression evidence; multi-index batching; and the source-vs-view progress-phase discrepancy where phase 6 reports `WAIT_4` so `waiting for readers before dropping` is never emitted), and an analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes from the attached table (it cannot: `AttachPartitionEnsureIndexes` is strictly match-or-create via `IndexSetParentIndex`/`DefineIndex`, incompatible/extra indexes are kept, no `performDeletion`/`RemoveInheritance` runs in the attach path and the pre-index `CreateInheritance` merges CHECK constraints only; v12's looser matching — no `indisvalid` skip, existence-only constraint check — still never drops; covering the three look-alikes, the `ALTER INDEX ... ATTACH PARTITION` re-parent/error path, and the docs' plain-`CREATE INDEX` pre-build workflow), and a comprehensive enumeration of all outcomes that leave an invalid index (`indisvalid=false`) derived from the full set of `indisvalid=false` writers in the checkout: a failed/cancelled/crashed `CREATE INDEX CONCURRENTLY`, a failed `REINDEX CONCURRENTLY` (`_ccnew`/`_ccold`), a failed/interrupted `DROP INDEX CONCURRENTLY` (retryable clear-valid step), an incomplete partitioned parent (`CREATE INDEX ON ONLY` / attached invalid child, validated by `validatePartitionedIndex` on `ATTACH PARTITION`), and `pg_upgrade` from <= 9.6 invalidating hash indexes in the new cluster, with the persistent state table, the single-transaction-DDL contrast, the planner/executor/VACUUM cost and the commands that reject invalid indexes, and per-outcome repair. |

## Coverage Notes

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
