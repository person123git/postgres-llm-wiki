# Wiki Log

Append one entry after every scaffold change, version lifecycle event, ingest, trace, lint pass, or filed answer.

## [2026-06-09] answer v12 | query planner statistics sources

- Filed [Query Planner Statistics Sources in PostgreSQL 12 (unverified)](v12/questions/query-planner-statistics-sources.md).
- Explained that `pg_stat_all_tables` is a monitoring view over cumulative `pg_stat_get_*()` counters, not a core planner input.
- Traced planner inputs through `pg_class`, `pg_statistic`, `pg_statistic_ext`, `pg_statistic_ext_data`, index and foreign-key metadata, CHECK/NOT NULL/partition constraints, and planner statistics hooks.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-04] repin v19 | move PostgreSQL 19 back to beta 1

- Fetched the official `REL_19_BETA1` tag for `raw/postgres-19/` and moved the source checkout from post-beta `master` commit `298bdd379552148f6043b4595374a7a6fbdd13c3` back to tag commit `4b0bf0788b066a4ca1d4f959566678e44ec93422`.
- Updated `wiki/versions.md`, `wiki/v19/index.md`, `wiki/index.md`, and both v19 question-page `pinned_commit:` fields to match the beta 1 checkout.
- Removed stale v19 page notes that described the 2026-06-03 post-beta master repin.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-04] review-fix v12 | bloated-index planner cross-AM scope and citation cleanup

- Reviewed [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md) against the pinned `45b88269` checkout; all sampled planner, storage-manager, `pgstatindex`, B-tree write-path, and executor claims matched source.
- Added the `amcostestimate` entry point (`bthandler` -> `btcostestimate`, copied in `get_relation_info`, invoked by `cost_index`) and scoped the mechanisms: the page-count penalty is shared by B-tree, hash, GiST, and SP-GiST via `genericcostestimate()`, while GIN and BRIN use separate cost models and the tree-height charge is B-tree-only (`tree_height = -1` elsewhere).
- Noted that deleted B-tree pages can be recycled by later splits through `_bt_getbuf(P_NEW)` / `GetFreeIndexPage()`, so page-count inflation is not always permanent.
- Normalized drifting citation ranges (`genericcostestimate` L5750 -> L5765, `cost_index` L541 -> L470) and split the section-2 descent citation into the comparison charge and the bloat charge; added the new sources and an Evidence Map row.

## [2026-06-04] expand v12 | reindex candidate heuristic for bloated indexes

- Added a proposed `avg_leaf_density` / `leaf_fragmentation` triage section to [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md).
- Split the heuristic into ordinary non-partial and partial-index subsections, reflecting the different `get_relation_info()` / `estimate_rel_size()` tuple-estimation paths.
- Framed the density and fragmentation bands as operational heuristics, not PostgreSQL-defined thresholds, and tied them to index size, workload shape, planner-visible page count, executor leaf-page traversal, and REINDEX locking tradeoffs.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries.

## [2026-06-03] review-fix v12 | bloated-index planner GUC scope and cost-formula guard

- Reviewed [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md) against the pinned `45b88269` checkout; all sampled planner, storage-manager, `pgstatindex`, and executor claims matched source.
- Added GUC apply-scope per the GUC-change rule: `seq_page_cost`, `random_page_cost`, and `effective_cache_size` are `PGC_USERSET` (`user`-context), session/transaction scope, no restart or reload, cited to `guc.c`.
- Made the `genericcostestimate()` page-count snippet faithful to source by showing the `index->pages > 1 && index->tuples > 1` guard and its `numIndexPages = 1.0` floor.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-03] review-fix v12 | bloated-index planner page

- Fixed [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md) after review.
- Clarified that the direct `index->pages / index->tuples` page-count penalty applies cleanly to ordinary non-partial indexes, while partial indexes estimate tuples from catalog tuple density scaled by current pages.
- Corrected the B-tree height explanation: planner costing uses `_bt_getrootheight()` / `btm_fastlevel`, while `pgstatindex.tree_level` reports `btm_level`.
- Tightened `pgstatindex()` page-category wording so `deleted_pages` are marked-deleted pages and `empty_pages` are the remaining `P_IGNORE` / half-dead case.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-03] expand v12 | RelationGetNumberOfBlocks cost in bloated-index planner page

- Expanded [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md) with a source-backed explanation of `info->pages = RelationGetNumberOfBlocks(indexRelation)`.
- Traced the call path through `RelationGetNumberOfBlocksInFork`, `RelationOpenSmgr`, `smgrnblocks`, `mdnblocks`, `_mdnblocks`, and `FileSize()` / `lseek(SEEK_END)`, clarifying that planner page-count acquisition performs storage-manager file-size probes rather than B-tree page reads.
- Added I/O and CPU boundaries: zero PostgreSQL relation-page I/O for the block-count read, possible filesystem metadata latency, `O(1)` CPU for the common last-segment check, and `O(number_of_segment_files_probed)` for multi-segment discovery.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries.

## [2026-06-03] repin v19 | check new master commits through 298bdd37

- Fetched PostgreSQL upstream `master` for `raw/postgres-19/`; upstream advanced ten commits from `4b0bf0788b066a4ca1d4f959566678e44ec93422` to `298bdd379552148f6043b4595374a7a6fbdd13c3`.
- Re-pinned the v19 source checkout and updated `wiki/versions.md`, `wiki/v19/index.md`, `wiki/index.md`, and both v19 question-page `pinned_commit:` fields.
- Checked the new commits against the v19 question scopes: no new `contrib/pg_plan_advice/` direct commit exists, and the REPACK page remains at 40 feature-scope commits. The closest new shared-infrastructure change (`f2081a78`, `ReplicationSlotRelease()` for `RS_EPHEMERAL` slots) does not change REPACK's `RS_TEMPORARY` slot/drop path.
- Added Context Reviewed notes to both v19 question pages; `verified_by_agent` remains `not yet` because this was a repin and scoped commit-history check, not a full claim-by-claim re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-02] answer v12 | planner penalties for bloated indexes

- Filed [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md).
- Explained that v12 has indirect planner penalties for B-tree bloat through `IndexOptInfo.pages`, B-tree `tree_height`, `genericcostestimate`, the explicit `btcostestimate` bloat-charge comment, and cache/page-cost modeling, but no direct planner input for `avg_leaf_density` or `leaf_fragmentation`.
- Added examples for low-density live leaf pages, empty/deleted pages, taller B-trees, and physically fragmented leaf chains, separating planner-visible page-count effects from executor/storage locality effects.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.

## [2026-06-02] answer v12 | psql environment variables and timeouts

- Filed [psql Environment Variables and Timeout Settings in PostgreSQL 12 (unverified)](v12/questions/psql-environment-variables-and-timeouts.md).
- Listed psql-specific frontend variables (`COLUMNS`, editor, pager, history, rc-file, shell, temp-dir, and diagnostic-color variables), libpq connection/session environment variables inherited by `psql`, and the v12 source precedence path from explicit connection values through service files, environment fallbacks, and compiled defaults.
- Explained how to set `statement_timeout` and `lock_timeout` for one `psql` session with `SET`, `SET LOCAL`, or `PGOPTIONS`, including `PGC_USERSET` scope, default disabled values, lock-wait-only behavior, combined-timeout precedence, and same-checkout isolation-test coverage.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.

## [2026-06-02] review v12 | density vs fragmentation index scan I/O

- Reviewed [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](v12/questions/leaf-density-vs-fragmentation-index-scan-io.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Re-checked every behavioral citation (`pgstatindex` density/fragmentation formulas, `PageGetFreeSpace`, B-tree fillfactors and split-location logic, the `_bt_first`/`_bt_next`/`_bt_steppage`/`_bt_readnextpage`/`_bt_walk_left`/`_bt_endpoint` scan path, planner `get_relation_info`/`estimate_rel_size`/`genericcostestimate`/`btcostestimate` costing, `seq_page_cost`/`random_page_cost` docs, `EXPLAIN BUFFERS` instrumentation, split/page-reuse and VACUUM recycle paths, `btgetbitmap`, index-only VM check, maintenance/reindex docs, and the `btree_index`/`pgstattuple` tests) and re-derived all density, fragmentation, and combined multiplier tables; no substantive factual errors.
- Tightened citation ranges for `costsize.c#index_pages_fetched`, `_bt_getbuf`, `BufferUsage`, and `btree_index.sql`; added a direct `estimate_rel_size` citation for partial-index page estimates.
- Clarified that `R / S = 4` is the v12 default nonsequential-sensitive model, not a fully cold-storage model, and made the fragmentation step estimate use `F * (L - 1)` transitions.
- Advanced `verified_by_agent` to `gpt-5 2026-06-02T17:37:56Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-02] answer v12 | density vs fragmentation index scan I/O

- Filed [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](v12/questions/leaf-density-vs-fragmentation-index-scan-io.md).
- Compared density-driven leaf-page count growth with `leaf_fragmentation` physical-order reversals using pinned `raw/postgres-12/` evidence from `pgstatindex`, B-tree scan paths, split/page-reuse paths, planner costing, cost-constant docs, and `EXPLAIN BUFFERS` instrumentation.
- Added density-level, fragmentation-level, and combined I/O multiplier estimates, explicitly separating PostgreSQL-visible page-count effects from storage/cache-sensitive physical-order latency.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.

## [2026-06-02] review v12 | btree leaf density 60% vs 90% query impact

- Reviewed [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](v12/questions/leaf-density-60-vs-90-query-impact.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Found no substantive factual issues in the `pgstatindex` density math, B-tree fillfactor/split behavior, planner page-count costing, executor leaf-page walking, buffer-manager notes, `EXPLAIN BUFFERS` boundary, or test-coverage claims.
- Tightened two citation ranges (`selfuncs.c` bloat-charge comment and `costsize.c#index_pages_fetched`) and split a non-contiguous `costsize.c` source reference into separate Markdown links.
- Advanced `verified_by_agent` to `GitHub-Copilot 2026-06-02T15:27:25Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-02] repin v19 | restore raw/postgres-19 and advance to 19beta1

- Restored the missing `raw/postgres-19/` checkout by cloning the official PostgreSQL upstream repository, then detached it at `origin/master` commit `4b0bf0788b066a4ca1d4f959566678e44ec93422` (`4b0bf078`, 2026-06-01, "Stamp 19beta1.").
- Upstream had advanced five commits past the previous pin `21298c2c`: `39343218` debug parse xreflabel docs, `4ff61509` release-note markup, `78ec4b69` release-note links, `ef6a95c7` translation updates, and `4b0bf078` the `19beta1` stamp.
- Updated the v19 pin in `wiki/versions.md`, `wiki/v19/index.md`, `wiki/index.md`, and both v19 question-page `pinned_commit:` fields.
- Intersected changed upstream paths with v19 raw-source citations; no cited v19 file changed, so existing line-number citations remain valid. The new commits are outside the documented `pg_plan_advice` and REPACK feature scopes.
- `verified_by_agent` remains `not yet` on both v19 question pages because this was a raw-data sync and metadata repin, not a full claim-by-claim re-verification.

## [2026-05-31] repin v19 | re-pin raw/postgres-19 db5ed032 → 21298c2c; pg_plan_advice + REPACK commit coverage re-verified

- Fetched `master` (`19devel`); upstream had advanced two commits past `db5ed032` to `21298c2c` (2026-05-30): an `array_append` doc clarification and an io_uring AIO race fix. Neither touches `contrib/pg_plan_advice/` or any REPACK file, so no new feature commits exist.
- Re-pinned the checkout to `21298c2c`. No file cited by either v19 question page changed between the two commits, so all line-number citations remain valid.
- Updated the `pinned_commit:` and Context Reviewed lines of both v19 question pages, plus `wiki/versions.md`, `wiki/v19/index.md`, and `wiki/index.md`.
- Re-verified commit coverage against the new pin: pg-plan-advice.md documents all 22 direct module commits (newest `b1901e28`, 2026-05-29) plus the 14 core-enabling and 8 test/doc/build support commits; repack-command.md's 40 feature-scope commits (newest `repack.c` commit `ac58465e`, 2026-05-30) are complete.
- `verified_by_agent` left as `not yet` on both pages — commit lists re-checked, full claim-by-claim re-verification still pending.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-30] answer v18 | GUC default-value changes since v12

- Filed [GUC Default-Value Changes Since PostgreSQL 12 (unverified)](v18/questions/guc-default-changes-since-v12.md).
- Compared PostgreSQL 18 `guc_tables.c` with the pinned PostgreSQL 12 GUC table, then grounded current defaults, apply scope, docs, release-note context, and tests in pinned `raw/postgres-18/`.
- Covered the seven v13-v15 default changes, the v18 `effective_io_concurrency` default increase to 16, and the v18 `log_connections` type/default-spelling change whose empty-string default preserves disabled-by-default behavior.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-30] review-fix v19 | repack command question

- Fixed review findings in [How the REPACK Command Works in PostgreSQL 19, and Its 40 Feature-Scope Commits (unverified)](v19/questions/repack-command.md).
- Corrected the parser/routing description: `REPACK` and `CLUSTER` parse as `RepackStmt`, while `VACUUM FULL` parses as `VacuumStmt` and reaches the same rewrite engine from `vacuum.c`.
- Tightened the WAL-retention wording, added same-checkout GUC citations for `maintenance_work_mem` and `enable_indexscan`, and removed uncited OS-level throttling advice.
- Renamed the commit-history section and page title to the 40 feature-scope commits, clarified the source-history scope, and removed the commit-list completeness open question.
- Updated `wiki/v19/index.md`, `wiki/index.md`, and `wiki/versions.md` coverage summaries.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-30] review-fix v19 | pg_plan_advice question

- Fixed review findings in [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](v19/questions/pg-plan-advice.md).
- Corrected the control-mechanism description: `pg_plan_advice` primarily clears `pgs_mask` bits but also uses `IndexOptInfo.disabled` for index-specific advice; full joins remain an exception where merge/hash paths can still be considered when disabled.
- Corrected the load-method wording (`LOAD`, `session_preload_libraries`, `shared_preload_libraries`) and the prepared-statement/generated-advice trigger wording (advisor hooks supply advice but do not themselves request advice generation).
- Expanded source history beyond the 22 `contrib/pg_plan_advice/` commits to include core planner-enabling/fix commits and test/doc/build support commits.
- Updated `wiki/v19/index.md`, `wiki/index.md`, and `wiki/versions.md` coverage summaries.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-30] review v17 | GUC default-value changes since v12

- Reviewed [GUC Default-Value Changes Since PostgreSQL 12 (unverified)](v17/questions/guc-default-changes-since-v12.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- Re-checked the seven changed built-in defaults and apply scopes in `guc_tables.c`/`config.sgml`/`postgresql.conf.sample`, and re-checked introducing commits plus `AC_INIT` version strings (`b1abfec8` v13; `c7eab0e9`, `e19594c5`, `bbcc4eb2` v14; `f7bda63a`, `64da07c4` v15).
- Added citations for added/removed-setting scope exclusions, `wal_compression`'s enum-with-off-default case, range-only edits, and same-checkout tests that cover individual settings rather than the full cross-version inventory.
- Advanced `verified_by_agent` to `GPT-5-Codex 2026-05-30T11:59:13Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-30] answer v17 | summary of GUC default-value changes since v12

- Filed [GUC Default-Value Changes Since PostgreSQL 12 (unverified)](v17/questions/guc-default-changes-since-v12.md).
- Asked the prompt-hygiene question first (the verbatim `## Question` would have read "GUC defaults changes"); user chose to polish it to "In PostgreSQL 17, summarize all GUC default-value changes since version 12."
- Enumerated changes by extracting every `name -> boot_val` pair from v17 `guc_tables.c` and diffing against v12 `guc.c` (v12 read for cross-check only; page cites `raw/postgres-17/` exclusively). After filtering range-only edits, macro renames, type changes, and preprocessor-line parse noise, exactly seven settings present in both versions show a changed built-in default; the `postgresql.conf.sample` diff agrees.
- The seven: `ssl_min_protocol_version` TLSv1->TLSv1.2 (v13), `password_encryption` md5->scram-sha-256 (v14), `vacuum_cost_page_miss` 10->2 (v14), `checkpoint_completion_target` 0.5->0.9 (v14), `shared_buffers` boot value 8MB->128MB (v15), `log_checkpoints` off->on (v15), `log_autovacuum_min_duration` -1->10min (v15).
- Pinned each introducing major version inside the single checkout: `git blame` -> introducing commit, then the in-tree `AC_INIT([PostgreSQL], [NNdevel], ...)` string at that commit. Apply scope read from each entry's `PGC_*` flag (postmaster->restart, sighup->reload, userset->session).
- Caught and excluded two false positives: `lc_messages` and `krb_server_keyfile` only changed their sample comment (boot values `""`/`PG_KRB_SRVTAB`=`""` unchanged in both v12 and v17); also noted `wal_compression` bool->enum keeps `off`. Documented these under a "Not Default-Value Changes" section.
- Cited only `raw/postgres-17/` (commit `54eeefae`) in Markdown form: `guc_tables.c` boot values + `PGC_*` flags, `config.sgml` default statements, and `postgresql.conf.sample` lines. Filed `verified_by_agent: not yet`; title carries `(unverified)`.
- Updated `wiki/v17/index.md`, `wiki/index.md`, and `wiki/versions.md`.

## [2026-05-30] expand v19 | add prepared-statement / plan-cache section to pg_plan_advice page

- Added a "Prepared Statements and Plan Caching" section to [pg-plan-advice.md](v19/questions/pg-plan-advice.md) answering how the feature interacts with prepared statements.
- Key verified-from-source findings: the module has no plan-cache integration and registers `pg_plan_advice.advice` with GUC flag `0`, so changing it does not invalidate cached plans; advice is read only in `pgpa_planner_setup` at plan time and any generated advice/feedback is stashed in the `PlannedStmt`. Enforcement is therefore frozen into a cached plan — current advice re-applies only when the statement actually re-plans, governed by core `choose_custom_plan` (unparameterized → generic once; parameterized → custom for first 5 then cost-based; `plan_cache_mode` force_custom/force_generic; DDL/relcache invalidation). Generated advice/feedback are absent from `EXPLAIN (PLAN_ADVICE) EXECUTE` unless `always_store_advice_details = on` was set when the cached plan was built, as the `prepared` regression test (`pt1`/`pt3` vs `pt2`/`pt4`) shows.
- Extended the page Evidence Map (3 rows), Source References (`plancache.c`, `prepared.sql`), and Context Reviewed; updated coverage lines in `wiki/v19/index.md` and `wiki/index.md`.
- `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-30] expand v19 | add I/O impact and throttling section to REPACK page

- Added an "I/O Impact and Throttling" section to [repack-command.md](v19/questions/repack-command.md) answering whether REPACK I/O can be throttled.
- Key verified findings from `raw/postgres-19/`: the rewrite path has no `vacuum_delay_point()`, so `vacuum_cost_delay`/`vacuum_cost_limit` do not apply (`cluster_rel`, `heapam_relation_copy_for_cluster`); large sequential-scan reads use an automatic `BAS_BULKREAD` ring buffer (`initscan`, gated on `NBuffers/4` and `SO_ALLOW_STRAT` which `table_beginscan` always sets); concurrent inserts use a `BAS_BULKWRITE` ring (`GetBulkInsertState`); the `USING INDEX` index-scan path uses no ring buffer; `maintenance_work_mem` (`PGC_USERSET`) bounds sort/index-build memory but does not throttle.
- Recommended levers: prefer physical-order/scan-and-sort, one table/partition at a time, moderate `maintenance_work_mem`, and OS-level cgroup v2 `io.max`/`io.weight` or `ionice` for genuine rate-limiting.
- Updated the Evidence Map and Source References on the page; extended the coverage lines in `wiki/v19/index.md` and `wiki/index.md`. `scripts/wiki_lint` passes (0 errors, 0 warnings).

## [2026-05-30] answer v19 | how the REPACK command works and all its commits

- Filed [How the REPACK Command Works in PostgreSQL 19, and All Its Commits (unverified)](v19/questions/repack-command.md).
- Scope check first: the third-party `pg_repack` extension is absent from `raw/postgres-19/`; the user asked about it, but the citable feature is the new in-core `REPACK` command. Asked the user, who chose to document in-core `REPACK` and to correct the two prompt typos ("compreensive", "explaination") in the verbatim `## Question`.
- Traced the feature end to end from the pinned checkout: `REPACK` absorbs `VACUUM FULL` + `CLUSTER` (`RepackStmt`/`RepackCommand`, `gram.y`, `utility.c`, `vacuum.c` routing); the blocking rewrite (`ExecRepack` -> `cluster_rel` -> `rebuild_relation` -> `make_new_heap`/`copy_table_data`/`finish_heap_swap`); and the `CONCURRENTLY` online path (`check_concurrent_repack_requirements`, decoding `bgworker` `RepackWorkerMain`, the `pgrepack` output plugin spill format, temporary `repack_<pid>` slot, `DecodingWorkerShared` DSM coordination, double catch-up + lock upgrade + swap, `apply_concurrent_*` replay through the identity index with `TABLE_*_NO_LOGICAL`, error/FATAL teardown, and WAL recycling).
- Documented the `PGC_POSTMASTER` `max_repack_replication_slots` GUC and its dedicated slot pool in `slot.c`, `pg_stat_progress_repack`/`pg_stat_progress_cluster`, the `REPACK_WORKER_EXPORT` wait event, and the `test_decoding` + injection-point test coverage.
- Listed and explained all ~40 commits (2026-03-10 `ac58465e` "Introduce the REPACK command" through 2026-05-30 `45b02984` WAL recycling), grouped into foundational, CONCURRENTLY infrastructure (incl. the `0d3dba38`/`01a80f06` revert pair), correctness fixes, error/doc/style, and tests.
- Cited only `raw/postgres-19/` (commit `db5ed03217b9c238703df8b4b286115d6e940488`) in Markdown form; filed `verified_by_agent: not yet`, title carries `(unverified)`.
- Updated `wiki/v19/index.md`, `wiki/index.md`, and `wiki/versions.md`.

## [2026-05-30] answer v19 | how pg_plan_advice works and all its commits

- Filed [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits (unverified)](v19/questions/pg-plan-advice.md), the first v19 question page.
- Asked the prompt-hygiene question first; user chose to correct the two typos ("compreensive", "explaination") in the verbatim `## Question`.
- Traced the new `contrib/pg_plan_advice` module end to end: the v19 core additions it depends on (`PGS_*`/`pgs_mask` strategy mask seeded from `enable_*` GUCs in `planner.c`, consumed in `joinpath.c`; the five new planner hooks `planner_setup`/`planner_shutdown`/`build_simple_rel`/`joinrel_setup`/`join_path_setup`; the `extendplan` per-object state API), the advice mini-language (20 tags, ordered/unordered targets, `alias#occ/schema.part@plan` relation identifiers), plan-to-advice generation (`pgpa_planner_shutdown` -> `pgpa_plan_walker` -> `pgpa_output_advice`), enforcement (each hook only clears `pgs_mask` bits), feedback flags + `EXPLAIN (PLAN_ADVICE)` output, the five `PGC_USERSET` GUCs, and the round-trip `test_plan_advice` harness plus contrib regression suite.
- Documented all 22 commits touching `contrib/pg_plan_advice/` (2026-03-12 `5883ff30` foundational through 2026-05-29 `b1901e28`), noting which three are tree-wide cleanups.
- Cited only `raw/postgres-19/` (commit `db5ed03217b9c238703df8b4b286115d6e940488`) in Markdown form; filed `verified_by_agent: not yet`, title carries `(unverified)`.
- Updated `wiki/v19/index.md`, `wiki/index.md`, and `wiki/versions.md`.

## [2026-05-30] repin v19 | advance pin to current upstream master tip

- Fetched `origin/master` (user-requested source fetch); the prior pin `5ab239c9` was 5 commits behind upstream.
- Repinned `raw/postgres-19/` to the new `master` tip `db5ed03217b9c238703df8b4b286115d6e940488` (`19devel`, 2026-05-29 21:51 -0400, "Avoid leaking system path from pg_available_extensions") by detaching HEAD.
- Skipped commits: `89d243d5` (OpenSSL 4 build), `08127c64` (OAuth doc), `45b02984` (REPACK CONCURRENTLY WAL recycling), `7dc5bbcf` (COPY TO FORMAT JSON encoding), `db5ed032` (pg_available_extensions path leak).
- Updated the commit in `wiki/versions.md`, `wiki/v19/index.md`, and `wiki/index.md`. No filed v19 pages cite the old commit, so no citations needed updating.

## [2026-05-30] version-add v19 | added PostgreSQL 19 as active version

- Pinned the existing `raw/postgres-19/` checkout (official `git.postgresql.org` `master`, `19devel`) to exact commit `5ab239c9a908ba5d8614d23fcdc425859a2fed3c` by detaching HEAD, matching the detached-pin convention of `raw/postgres-18/`.
- Confirmed the commit matches `origin/master` and is a genuine upstream commit. No `REL_19_STABLE` branch exists yet, so the pin tracks an exact `master` commit.
- Created `wiki/v19/index.md` landing page with status `active`; PostgreSQL 18 remains `primary`.
- Registered the version in `wiki/versions.md` and `wiki/index.md`.

## [2026-05-28] review v12 | sampling pgstatindex variant proposal

- Verified every behavioral claim and source citation in [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed: `pgstatindex_impl` scan loop, page classification, leaf accumulation, density/fragmentation formulas, and NaN guards (`pgstatindex.c`); `index_size` exactness via `pg_relation_size` main fork; `pgstattuple_approx` precedent (`output_type`, `scanned_percent`, non-random-sample note in `pgstatapprox.c`); extension wiring (`pgstattuple.control` 1.5, v1.5 SQL grant model in `pgstatindex.c`/`pgstattuple--1.4--1.5.sql`); pageinspect building blocks — `bt_page_stats` `d/e/l/r/i` type codes summing to pgstatindex's deleted/empty/leaf/internal, `free_size = PageGetFreeSpace`, `max_avail = pd_special - SizeOfPageHeaderData` (=24), `page_header.special`, block-0 error, superuser scope; the `bt_metap` unsigned-`oldest_xact` `int4` overflow and the 6-column compat wrapper that stops `BuildTupleFromCStrings` (loops over `tupdesc->natts`) before the unsigned column; `PGC_USERSET` timeout contexts; and all cited SQL built-ins and doc syntax.
- Corrected one citation precision error: the `create_function.sgml#configuration-parameter` link (`SET search_path FROM CURRENT`) pointed at `#L484-L493`, which is the planner `SUPPORT` clause; retargeted both occurrences to `#L494-L505`, the `configuration_parameter` description.
- `verified_by_agent` left at `not yet`: this is a `type: question` page and the proposed C function / SQL prototype is a design proposal, not behavior defined by the pinned checkout.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-28] review v12 | btree leaf density 60% vs 90% query impact re-verification

- Re-checked every behavioral claim in [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](v12/questions/leaf-density-60-vs-90-query-impact.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed: `avg_leaf_density` formula and `PageGetFreeSpace` one-line-pointer reservation (`pgstatindex.c`, `bufpage.c`); `BTREE_DEFAULT_FILLFACTOR` 90 / `BTREE_NONLEAF_FILLFACTOR` 70 and the split-fillfactor comment (`nbtree.h`); planner sizing via `get_relation_info` (non-partial `RelationGetNumberOfBlocks`, partial `estimate_rel_size`) and `tree_height` from `_bt_getrootheight` (`plancat.c`); `genericcostestimate` `numIndexPages = ceil(numIndexTuples * index->pages / index->tuples)` and the `(tree_height + 1) * 50.0 * cpu_operator_cost` bloat charge (`selfuncs.c`); executor walking `btgettuple`/`btgetbitmap` -> `_bt_first`/`_bt_next` -> `_bt_steppage`/`_bt_readnextpage` (forward `_bt_getbuf` on `btpo_next`, backward `_bt_walk_left`), `P_FIRSTDATAKEY` start, endpoint scan (`nbtree.c`, `nbtsearch.c`); `NUM_BUFFER_PARTITIONS = 128` and the `BufferAlloc` hit path (`lwlock.h`, `bufmgr.c`); index-only-scan VM check (`indexam.c`, `nodeIndexonlyscan.c`); empty-`pgstatindex` and `fillfactor=10`/page-deletion test coverage gaps; `EXPLAIN` buffer-usage labels (`explain.c`); and the `maintenance.sgml`/`ref/reindex.sgml` REINDEX boundaries.
- Corrected one symbol-name error: the B-tree AM handler is `bthandler`, not `btreehandler`; fixed both the prose and the citation link label (line range L128-L148 was already correct).
- Advanced `verified_by_agent` to `claude-opus-4-8 2026-05-28T19:03:01Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-27] review-fix v12 | btree leaf density 60% vs 90% query impact

- Reviewed and corrected [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](v12/questions/leaf-density-60-vs-90-query-impact.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Corrected planner sizing: ordinary non-partial indexes use `RelationGetNumberOfBlocks(indexRelation)` for `index->pages`, while partial indexes use `estimate_rel_size`; `avg_leaf_density` is not a planner input.
- Corrected executor leaf walking: `_bt_moveright` / `_bt_relandgetbuf` are not the per-leaf range-scan path; `_bt_next` steps through `_bt_steppage` / `_bt_readnextpage`, with forward scans reading `btpo_next` pages via `_bt_getbuf` and backward scans using `_bt_walk_left`.
- Removed the unsupported universal "below 50%" density threshold, added the same-checkout `REINDEX` documentation boundary, and added explicit v12 test coverage notes for `pgstatindex` and `btree_index`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries.
- Advanced `verified_by_agent` to `GPT-5-Codex 2026-05-27T13:20:15Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-27] review v12 | pgstatindex calculation behavior verification

- Re-checked every behavioral claim, formula, edge case (empty-index NaN handling, `empty_pages` as half-dead only, `PageGetFreeSpace` + `pd_special - SizeOfPageHeaderData` density math, backward `btpo_next` fragmentation test), permission model (v1.5 SQL `GRANT` to `pg_stat_scan_tables`), scan mechanics, and regression coverage gap in [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](v12/questions/how-pgstatindex-calculates-information.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed exact alignment with `pgstatindex_impl` (metapage read of `btm_version`/`btm_level`/`btm_root`, `RelationGetNumberOfBlocks` + `BAS_BULKREAD` loop from blkno 1, `CHECK_FOR_INTERRUPTS`, `BUFFER_LOCK_SHARE` per page, `P_ISDELETED` / `P_IGNORE` / `P_ISLEAF` classification from `BTPageOpaqueData`, `BTIndexStat` accumulation, result tuple construction with `NaN` strings, and `AccessShareLock` close).
- No factual corrections and no citation updates needed; the page's Markdown source citations (page-relative `../../../raw/postgres-12/...` with `#L` ranges) and Evidence Map remain precise. The review used the grammatically corrected form of the prompt per user direction.
- Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-26] lint | wiki lint

- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-26] cleanup | corrected filed question prompts

- Corrected typos, grammar, capitalization, and punctuation in the `## Question` sections of all 17 filed question pages across v12, v17, and v18.
- Kept changes scoped to prompt text; answer bodies, citations, metadata, indexes, and version coverage summaries were unchanged.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-26] source-fetch v17 | restore raw checkout

- Cloned PostgreSQL `REL_17_STABLE` into `raw/postgres-17/` and checked out the existing v17 pin `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- Confirmed `raw/postgres-17/` now contains the pinned source files used by the v17 wiki citations.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-25] fix v12 | pageinspect bt_metap integer overflow

- Reviewed `ERROR: value "4145147631" is out of range for type integer` from the diagnostic `pgstatindex_approx_pageinspect` wrapper in PostgreSQL 12.
- Updated [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) to explain that v12 pageinspect 1.7 declares `bt_metap.oldest_xact` as `int4` while the C function formats the underlying unsigned `TransactionId`.
- Added `pgstatindex_pageinspect_bt_metap_compat(text)`, a six-column wrapper around the same `bt_metap` C symbol, and changed the SQL prototype to use it so the wrapper does not convert `oldest_xact`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries.

## [2026-05-25] fix v12 | pageinspect helper search path

- Updated [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) to address `ERROR: function bt_page_stats(text, integer) does not exist` in the diagnostic `pgstatindex_approx_pageinspect` wrapper.
- Clarified that v12 does define `bt_page_stats(text, int4)` and that the error points to missing `pageinspect` setup or a `search_path` that does not include the extension schema.
- Added an extension-schema discovery query and `SET search_path FROM CURRENT` to the wrapper definition so helper lookups resolve at execution time.

## [2026-05-25] fix v12 | pageinspect pgstatindex sampling SQL return type

- Fixed [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) so the diagnostic `pgstatindex_approx_pageinspect` SQL wrapper returns `scanned_percent` as `float8` instead of an inferred `numeric` expression.
- Reset `verified_by_agent` to `not yet` because the page changed after the previous agent verification timestamp.

## [2026-05-25] verify v12 | sampling pgstatindex variant proposal

- Re-checked every claim and citation in [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed the page-type mapping (`'d'/'e'/'l'/'i','r'`) matches `pgstatindex_impl` classification order against pageinspect `GetBTPageStatistics`, the `index_size`/`pg_relation_size` exactness reasoning, `max_avail = special - 24` (`SizeOfPageHeaderData` = `offsetof(PageHeaderData, pd_linp)` = 24), `statement_timeout`/`lock_timeout` as `PGC_USERSET` session scope, pageinspect `default_version = '1.7'` `bt_metap` shape, and the `pgstatindex` extension SQL / control-file `default_version = '1.5'`.
- Fixed two citation imprecisions: `nbtree.h#BTPageOpaqueData` range widened to `L55-L68` to cover `btpo_next`; relabeled the NaN empty-index test citation to `pgstatindex-tests` (`L18-L45`).
- Set `verified_by_agent: claude-opus-4-7 2026-05-25T17:36:44Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.

## [2026-05-25] answer v12 | sampling pgstatindex variant proposal

- Filed [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](v12/questions/pgstatindex-sample-variant-proposal.md).
- Designed a `pgstatindex_approx` that random-samples physical B-tree blocks from pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, preserving the current `pgstatindex_impl` guards, metapage read, page classification, and leaf free-space/fragmentation accumulation.
- Noted the v12-specific SQL prototype detail: `bt_page_stats.free_size` uses the same `PageGetFreeSpace(page)` routine as `pgstatindex_impl`, but v12 `bt_page_stats(text, int4)` does not expose `max_avail`, so the prototype also uses `page_header(get_raw_page(...))` for the page `special` offset.
- Added pros/cons and a diagnostic SQL-language wrapper using contrib `pageinspect` helpers (`bt_metap`, `bt_page_stats`, `get_raw_page`, and `page_header`) plus session-scoped `statement_timeout` and `lock_timeout` examples.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-25] answer v17 | sampling pgstatindex variant proposal

- Filed [Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)](v17/questions/pgstatindex-sample-variant-proposal.md).
- Designed a `pgstatindex_approx` that random-samples physical B-tree blocks from pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`, preserving the current `pgstatindex_impl` guards, per-page classification, and leaf accumulation.
- Noted the v17-specific density detail: `pgstatindex_impl` uses `PageGetFreeSpace(page)`, so a `pageinspect` SQL prototype can use `bt_page_stats.free_size` directly while reading `page_header(get_raw_page(...))` only for the page `special` offset used in `max_avail`.
- Added a diagnostic SQL-language wrapper using contrib `pageinspect` helpers (`bt_metap`, `bt_page_stats`, `get_raw_page`, and `page_header`) plus session-scoped `statement_timeout` and `lock_timeout` examples.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v17/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-26] answer v12 | pgstatindex calculation behavior

- Filed [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](v12/questions/how-pgstatindex-calculates-information.md).
- Traced the exact v12 `pgstatindex_impl` path against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`: extension SQL entry points and grants, physical B-tree checks, other-session temp rejection, metapage read, `RelationGetNumberOfBlocks` main-fork scan, `BAS_BULKREAD` page reads, share buffer locks, B-tree opaque flag classification, `PageGetFreeSpace` leaf-density math, backward-right-link fragmentation, result tuple construction, and empty-index/error regression coverage.
- Noted the v12-specific density detail that `pgstatindex` uses `PageGetFreeSpace`, which subtracts one line-pointer slot, not `PageGetExactFreeSpace`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-25] answer v17 | contrib extension inventory

- Filed [PostgreSQL 17 Contrib Extensions (unverified)](v17/questions/contrib-extensions.md).
- Enumerated all 53 PostgreSQL 17 contrib extensions from control-file-backed entries in pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- Explained each extension with same-checkout SGML or SQL evidence, covering data types, index/search helpers, storage diagnostics, FDWs, `pg_stat_statements`, transform extensions, and SPI trigger examples.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v17/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-25] research v18 | pageinspect sql prototype for pgstatindex sampling

- Expanded [Proposing a Sampling pgstatindex Variant for PostgreSQL 18 (unverified)](v18/questions/pgstatindex-sample-variant-proposal.md) with a `pageinspect`-based SQL prototype.
- Grounded the prototype in v18 contrib functions: `bt_metap(text)` for metapage fields, `bt_page_stats(text, bigint)` for B-tree page type and right-link data, and `get_raw_page(text, bigint)` plus `page_header(bytea)` for exact `upper - lower` sampled leaf free space.
- Called out the key caveat: `bt_page_stats.free_size` uses `PageGetFreeSpace`, not `PageGetExactFreeSpace`, so a closer SQL approximation needs the raw page header and may read sampled blocks twice.
- Added production diagnostic timeouts and noted that `pageinspect` is a superuser/debugging tool, not a replacement for a polished `pgstattuple` extension API.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.

## [2026-05-25] review-fix v18 | sampling pgstatindex variant proposal

- Reviewed [Proposing a Sampling pgstatindex Variant for PostgreSQL 18 (unverified)](v18/questions/pgstatindex-sample-variant-proposal.md) against the pinned `pgstatindex_impl`, `pgstattuple_approx`, extension SQL, docs, and regression-test anchors.
- Clarified the proposed sampler to random-select distinct non-metapage blocks but visit the selected blocks in ascending order, preserving the random-sample estimator while keeping the access pattern closer to the existing physical `BAS_BULKREAD` scan.
- Reworded the I/O downside from pure random I/O to sparse ordered reads with weaker sequential readahead.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-25] answer v18 | sampling pgstatindex variant proposal

- Filed [Proposing a Sampling pgstatindex Variant for PostgreSQL 18 (unverified)](v18/questions/pgstatindex-sample-variant-proposal.md).
- Designed a `pgstatindex_approx` that random-samples physical index blocks: keeps `pgstatindex_impl`'s per-page classification/accumulation but visits only a subset of `blkno = 1..nblocks-1`, verified against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Established the field split from source: `version`/`tree_level`/`root_block_no` stay exact (metapage read), `index_size` stays exact (`RelationGetNumberOfBlocks`), `avg_leaf_density`/`leaf_fragmentation` are page-local ratios estimable directly (no neighbor reads, no scaling), and only `internal/leaf/empty/deleted` page counts need `1/f` scaling.
- Modeled the result shape and extension wiring on existing precedent: `pgstattuple_approx` `scanned_percent`/`approx_` columns, `PG_FUNCTION_INFO_V1` + `_v1_5` registration, a `1.5->1.6` upgrade script with `REVOKE`/`GRANT` to `pg_stat_scan_tables`, and a `default_version` bump; marked the proposed DDL as not runnable against the pin.
- Wrote pros (real page-read reduction unlike `pgstattuple_approx`'s every-block iteration, exact-for-free columns, random-sample making linear scaling defensible per the `pgstatapprox.c` non-random-sample note) and cons (noisy rare-category counts, spatial-clustering variance, lost sequential readahead, non-snapshot, new surface area).
- Cross-linked the sibling [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](v18/questions/pgstatindex-approx-sampling.md) and the VACUUM density page; updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-25] review v18 | pgstatindex approximation limits verification

- Re-checked every behavioral claim in [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](v18/questions/pgstatindex-approx-sampling.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Confirmed: `statapprox_heap` VM-skip + `GetRecordedFreeSpace` FSM path + `vac_estimate_reltuples`, `pgstattuple_approx_internal` heap-kind/heap-AM restriction, `pgstatindex_impl` full non-metapage scan with `P_ISDELETED`/`P_IGNORE`/`P_ISLEAF`/internal classification, `avg_leaf_density = 100 - free_space/max_avail*100` with `PageGetExactFreeSpace = pd_upper - pd_lower` and `max_avail = pd_special - SizeOfPageHeaderData`, `leaf_fragmentation` via `btpo_next < blkno`, NaN guards, heap visibility-map two-bit semantics, heap FSM 256-category storage, index FSM free-vs-used (`BLCKSZ - 1`/`0`) convention, `_bt_allocbuf` recyclability recheck, `btvacuumscan` all-non-metapage scan, `btvacuumpage` `_bt_upgradelockbufcleanup` cleanup lock on every leaf, `btvacuumcleanup` skip path, full regression coverage table (empty B-tree output, wrong-AM and unsupported-relation errors, partition-index success `(4,0,8192,0,0,0,0,0,NaN,NaN)`), and source history commits `5850b20f58` / `7c91a0364f` plus absence of any `pgstatindex_approx` in tree or history.
- Corrected one citation: `pgstathashindex` is defined in `pgstattuple--1.4--1.5.sql` (the `default_version = '1.5'` upgrade), not in `pgstattuple--1.4.sql`; rewrote the test-section sentence and Source References to attribute it correctly via `pgstattuple.control` and the upgrade script.
- Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`.

## [2026-05-25] research v18 | pgstatindex current regression tests

- Expanded [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](v18/questions/pgstatindex-approx-sampling.md) with a table of the current `pgstatindex` regression coverage.
- Documented the empty B-tree output checks across text/name/regclass entry points, wrong-index-AM errors, unsupported relation-kind errors, sequence failure, and the success case for a physical B-tree index on a partition.
- Called out the remaining coverage gaps: no populated B-tree density/fragmentation tests, no internal/deleted/half-dead page tests, and no approximate index diagnostic function in the v18 extension SQL.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.

## [2026-05-25] answer v18 | pgstatindex approximation limits

- Filed [Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)](v18/questions/pgstatindex-approx-sampling.md).
- Traced `pgstatindex_impl`, `pgstattuple_approx` heap skipping, heap visibility-map semantics, heap and index FSM behavior, B-tree page flags, B-tree allocation/recycling, VACUUM scan boundaries, tests, docs, and same-checkout source history in pinned `raw/postgres-18/`.
- Clarified that `pgstattuple_approx` is not random sampling: it skips all-visible heap pages and uses FSM values, while B-tree `pgstatindex` has no equivalent side channel for live leaf free space or leaf fragmentation.
- Scoped a possible future `pgstatindex_approx` as a separate approximate API with explicit estimate fields, not a silent semantic change to `pgstatindex`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-22] answer v18 | pg_stat_statements query text size limit

- Filed [Limiting Query Text Size in pg_stat_statements in PostgreSQL 18 (unverified)](v18/questions/pg-stat-statements-query-text-size.md).
- Confirmed against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e` that no GUC caps individual query-text length: only `max`, `track`, `track_utility`, `track_planning`, and `save` are defined; `CleanQuerytext` and `qtext_store` write full-length text with only a whole-file `MaxAllocHugeSize` guard.
- Documented that `track_activity_query_size` applies to `pg_stat_activity.query`, not `pg_stat_statements.query`, and that `need_gc_qtexts`/`gc_qtexts` discard texts wholesale (set `query_len = -1`) rather than truncating, plus the docs' "reduce `pg_stat_statements.max`" recovery guidance.
- Listed workarounds: lower `pg_stat_statements.max`, read with `showtext => false`, and `left()`/`substr` at read time.
- Updated `wiki/index.md` and `wiki/v18/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-22] answer v12 | pg_stat_statements mechanics and configuration

- Filed [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](v12/questions/pg-stat-statements.md).
- Traced the v12-specific path against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`: in-extension query jumbling in `pgss_post_parse_analyze` (no `compute_query_id` GUC, no in-core jumble), the three-field `(userid, dbid, queryid)` hash key, the fixed `Counters` struct (no planning/WAL/JIT/parallel columns), executor/utility hooks, `pgss_store`, external query-text file plus `gc_qtexts` discard, readout permission filtering, reset, and clean-shutdown/crash persistence.
- Cataloged the four module GUCs (`max`, `track`, `track_utility`, `save`) with defaults and contexts, plus `shared_preload_libraries` and `track_io_timing`; mapped restart/reload/session scope from v12 GUC definitions and `catalogs.sgml` `pg_settings` context meanings.
- Noted v12 differences from later releases (23-column 1.7 view, no `track_planning`, no `pg_stat_statements_info`).
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.

## [2026-05-22] answer v18 | pg_stat_statements mechanics and configuration

- Filed [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 18 (unverified)](v18/questions/pg-stat-statements.md).
- Traced `_PG_init`, shared-preload activation, query-ID enablement, parser/planner/executor/utility hooks, shared hash keys, query-text file storage, normalization, entry deallocation, readout permissions, reset/save paths, and same-checkout test coverage in pinned `raw/postgres-18/`.
- Cataloged the direct `pg_stat_statements.*` GUCs and adjacent core settings that affect visible output: `shared_preload_libraries`, `compute_query_id`, `track_io_timing`, JIT settings, and parallel-query settings, with restart/reload/session scope called out from v18 GUC contexts.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-22] answer v18 | track_activity_query_size usage

- Filed [How track_activity_query_size Is Used in PostgreSQL 18 (unverified)](v18/questions/track-activity-query-size.md).
- Traced the `PGC_POSTMASTER` GUC definition, shared-memory sizing and initialization, `pgstat_report_activity()` write/truncation path, `pg_stat_activity` and `pg_stat_get_backend_activity()` read paths, multibyte clipping, `track_activities` disable behavior, and deadlock/crash activity readers in pinned `raw/postgres-18/`.
- Added the `pg_stat_statements` boundary: `track_activity_query_size` does not truncate `pg_stat_statements.query`, which stores representative query text in the extension's external query-text file; the shared overlap is query ID reporting, not activity-string storage.
- Noted the absence of a dedicated same-checkout test for non-default `track_activity_query_size` values under `## Open Questions`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-21] research v18 | NUM_BUFFER_PARTITIONS source commit history

- Added a `## Source Commit History` section to [Usage of NUM_BUFFER_PARTITIONS in PostgreSQL 18 (unverified)](v18/questions/num-buffer-partitions.md) documenting all 12 commits reachable from pinned `raw/postgres-18/` HEAD `6cb30725` whose diff touches `NUM_BUFFER_PARTITIONS`, oldest first, each with the verbatim commit message and an explanation tying it to the page's existing citations.
- Found via `git log -S`/`-G 'NUM_BUFFER_PARTITIONS'` in the pinned checkout: `10b9ca3d` (introduce at 16), `b25dc481` (NBuffers+partitions sizing), `f99a569a` (pgindent), `2a275e6d` (reverse-order unlock), `ea9df812` (MainLWLockArray accessors), `3acc10c9` (raise to 128), `c319991b` (BufferMapping tranche), `6e654546` (pg_buffercache stops locking), `3761fe3c` (tranche simplification), `8c0d7baf` (dshash DSHASH_NUM_PARTITIONS=128), `d03d7549` (offset macros), `3ac88fdd` (macros to static inline).
- Confirmed the value history directly: `16` in `10b9ca3d`, `128` at HEAD. Added an Evidence Map row and refreshed `verified_by_agent`.
- Updated `wiki/index.md` and `wiki/v18/index.md` summaries.

## [2026-05-21] review v18 | NUM_BUFFER_PARTITIONS buffer mapping usage verification

- Re-checked every behavioral claim and citation line range in [Usage of NUM_BUFFER_PARTITIONS in PostgreSQL 18 (unverified)](v18/questions/num-buffer-partitions.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Confirmed: `NUM_BUFFER_PARTITIONS` = 128 and its role in `BUFFER_MAPPING_LWLOCK_OFFSET`/`NUM_FIXED_LWLOCKS` offsets, `BufTableHashPartition` = `hashcode % NUM_BUFFER_PARTITIONS` with the power-of-2 invariant, `InitBufTable` `HASH_PARTITION` creation, `StrategyShmemSize`/`StrategyInitialize` `NBuffers + NUM_BUFFER_PARTITIONS` sizing, `BufTableLookup`/`Insert`/`Delete` share/exclusive contract and corruption `elog`, `BufferAlloc` hit/miss/insert-race/new-tag paths, `PrefetchSharedBuffer` shared-lock probe, `InvalidateBuffer`/`InvalidateVictimBuffer`/`GetVictimBuffer` reuse and drop paths, `ExtendBufferedRelShared` victim/insert/existing/new branches, `FindAndDropRelationBuffers` direct lookup, `InitializeLWLocks`/`LWLockShmemSize`/`LWLockReportWaitStart`/`BuiltinTrancheNames` and `BufferMapping` wait-event text, `dynahash.c` partitioned-locking comment + `HASH_PARTITION` init asserts, `localbuf.c#InitLocalBuffers` non-partitioned local hash, and `dshash.c#DSHASH_NUM_PARTITIONS` (`1 << 7`) matching note.
- Verified the two negative claims: no `buffer_partitions` GUC in `guc_tables.c`/`config.sgml`, and no regression/TAP test under `src/test`/`contrib` names the symbols. Open Question on test absence remains correctly scoped.
- No factual corrections needed. Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`.

## [2026-05-21] answer v18 | NUM_BUFFER_PARTITIONS buffer mapping usage

- Filed [Usage of NUM_BUFFER_PARTITIONS in PostgreSQL 18 (unverified)](v18/questions/num-buffer-partitions.md).
- Traced the fixed `NUM_BUFFER_PARTITIONS` definition, `MainLWLockArray` offsets, `BufferMapping` LWLock initialization, shared buffer lookup dynahash initialization, hash-to-partition selection, read allocation, relation extension, victim invalidation, relation-drop lookup, wait-event reporting, and local-buffer non-use in pinned `raw/postgres-18/`.
- Noted the absence of direct symbol-named regression or TAP coverage under `## Open Questions`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-21] review-fix | make wiki page navigation VS Code-openable

- Converted remaining active wiki page-to-page links in indexes, version manifests, overview, log entries, Related Pages sections, and v18 cross-links from Obsidian wikilinks to page-relative Markdown links so VS Code opens the targets.
- Updated `AGENTS.md` to require page-relative Markdown links for wiki page navigation.
- Updated `scripts/wiki_lint`/`scripts/wiki_tooling.py` to validate local Markdown wiki links and reject non-source Obsidian wikilinks.
- Checked 49 local Markdown wiki links for local file resolution. `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-21] review-fix | make remaining question source links VS Code-openable

- Converted the remaining managed question pages' raw PostgreSQL source citations from legacy `[[raw/postgres-NN/...]]` form to page-relative Markdown links (`../../../raw/postgres-NN/...`) so VS Code opens them from each page.
- Normalized all converted source fragments to line anchors and fixed two stale v12 citation targets while preserving wiki navigation links in Obsidian form.
- Checked 836 question-page raw Markdown links for local file resolution and in-range line fragments. `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-21] tooling v18 | make Markdown source links VS Code-openable

- Converted source citations in [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](v18/questions/explain-analyze-buffers-output.md) from repo-relative `raw/postgres-18/...` Markdown URLs to page-relative `../../../raw/postgres-18/...` URLs so VS Code opens them from the wiki page.
- Updated `scripts/wiki_tooling.py` and `scripts/wiki_lint` to normalize page-relative Markdown raw-source links before checking existence and matching-version citations.
- Updated `AGENTS.md` citation examples to prefer page-relative Markdown source links.

## [2026-05-20] review-fix v18 | explain analyze buffers source links

- Reviewed source links in [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](v18/questions/explain-analyze-buffers-output.md) after the Markdown citation convention change.
- Converted the page's raw PostgreSQL source citations from `[[raw/...#symbol|label]]` form to Markdown links with pinned `raw/postgres-18/` line ranges.
- Kept wiki navigation links in Obsidian form and left the existing content claims unchanged.

## [2026-05-20] convention | source citations switch to Markdown links

- Updated `AGENTS.md` "MANDATORY Citations" to require Markdown-form source citations: `[file.c#Symbol](raw/postgres-NN/path/file.c#L42-L58)`. Page-to-page nav links remain Obsidian wikilinks (`[[v18/index]]`).
- Existing pages on the legacy `[[raw/postgres-NN/...]]` wikilink citation form remain valid; conversion happens lazily on next edit. AGENTS.md keeps the "one citation style per page" rule.
- Updated `scripts/wiki_tooling.py` with `extract_markdown_raw_citations` and `scripts/wiki_lint` to validate Markdown `raw/...` citations for file existence and matching-version checkout. `[[raw/...]]` form remains supported for back-compat.
- `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-20] review v18 | explain analyze buffers output verification

- Re-checked every behavioral claim in [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](v18/questions/explain-analyze-buffers-output.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Confirmed: `ParseExplainOptionList` default `es->buffers = es->analyze` (no rejection without `ANALYZE`), `ExplainOnePlan` -> `INSTRUMENT_BUFFERS` -> `CreateQueryDesc`, full `BufferUsage` field set including split shared/local/temp timing, `peek_buffer_usage`/`show_buffer_usage` text-vs-non-text rules, shared/local hit/read counter sites in `PinBufferForBlock` / `ReadRecentBuffer` / `AsyncReadBuffers`, shared dirty/extend/flush sites in `MarkBufferDirty`, `MarkBufferDirtyHint`, `ExtendBufferedRelShared`, `FlushBuffer`, local sites in `MarkLocalBufferDirty`, `ExtendBufferedRelLocal`, `FlushLocalBuffer`, temp work-file counters and timings in `BufFileLoadBuffer`/`BufFileDumpBuffer`, shared/local I/O timing via `pgstat_count_io_op_time`, `InstrStartNode`/`InstrStopNode`/`BufferUsageAccumDiff` delta accounting, parallel-worker accumulation via `InstrStartParallelQuery`/`InstrEndParallelQuery`/`InstrAccumParallelQuery` and `ParallelQueryMain`/`ExecParallelFinish`, planning-buffer capture in `standard_ExplainOneQuery` and `Planning:` section printing, `SerializeMetrics` capture in `serializeAnalyzeReceive` and `ExplainPrintSerialize`, GUC contexts (`track_io_timing` `PGC_SUSET`, `statement_timeout`/`lock_timeout` `PGC_USERSET`), regression coverage (`buffers off`, `buffers`, JSON/XML/YAML, `BUFFERS` without `ANALYZE`, `SERIALIZE BUFFERS`), and same-checkout `ref/explain.sgml` BUFFERS definition plus `EXPLAIN-EXECUTE-example` `Buffers: shared hit=4`.
- No factual corrections needed. Open Question on `written` doc-vs-source phrasing remains correctly scoped.
- Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`. `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-20] answer v12 | explain analyze buffers output

- Filed [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](v12/questions/explain-analyze-buffers-output.md).
- Traced `EXPLAIN` option parsing, `INSTRUMENT_BUFFERS` setup, per-node `pgBufferUsage` delta accounting, shared/local/temp counter increment paths, `track_io_timing`, parallel-worker accumulation, trigger-report boundaries, and same-version documentation examples in pinned `raw/postgres-12/`.
- Noted the source-vs-doc wording gap for `written`: docs describe dirty-buffer eviction, while `ReadBuffer_common` also increments written counters for relation extension.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-20] answer v18 | explain analyze buffers output

- Filed [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 18 (unverified)](v18/questions/explain-analyze-buffers-output.md).
- Traced `BUFFERS` option defaults, `INSTRUMENT_BUFFERS`, plan-node `pgBufferUsage` delta accounting, shared/local/temp counter increment paths, split shared/local/temp I/O timing, planning and serialization buffer sections, parallel-worker accumulation, trigger-report boundaries, and same-version regression coverage in pinned `raw/postgres-18/`.
- Noted the source-vs-doc wording gap for `written`: docs describe dirty-buffer eviction, while shared and local relation extension paths also increment written counters.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-19] answer v18 | extension hooks for vacuum and autovacuum

- Filed [Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)](v18/questions/extension-hooks-vacuum-autovacuum.md).
- Traced the VACUUM/ANALYZE utility path (`ProcessUtility -> ExecVacuum -> vacuum()`) and the autovacuum worker path (`AutoVacWorkerMain -> do_autovacuum -> autovacuum_do_vac_analyze -> vacuum()`), and confirmed via direct grep that no `Invoke*Hook` macro, `ExecutorStart`, or `object_access_hook` invocation lives in `vacuum.c`, `vacuumlazy.c`, `analyze.c`, `vacuumparallel.c`, or `autovacuum.c` in pinned `raw/postgres-18/`.
- Enumerated in-process hook variables that actually fire on vacuum's path (`ProcessUtility_hook`, `post_parse_analyze_hook`, `emit_log_hook`, plus `shmem_request_hook` and `shmem_startup_hook` at postmaster init), the table AM (`relation_vacuum`, `scan_analyze_next_block`, `scan_analyze_next_tuple`), index AM (`ambulkdelete`, `amvacuumcleanup`, `amparallelvacuumoptions`), FDW (`AnalyzeForeignTable`) and per-type (`pg_type.typanalyze`) callbacks, and adjacent surfaces (`RegisterBackgroundWorker`, custom cumulative statistics, `pg_stat_progress_vacuum`/`pg_stat_progress_analyze`).
- Cited only `raw/postgres-18/` (commit `6cb307251c5c6261286c1566496920976640108e`); filed `verified_by_agent: not yet`, title carries `(unverified)`.
- Linked from `wiki/v18/index.md` and `wiki/index.md`; cross-links to [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](v18/questions/custom-cumulative-statistics.md) for the cumulative-stats surface.

## [2026-05-19] review v18 | custom cumulative statistics verification

- Re-checked every behavioral claim in [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](v18/questions/custom-cumulative-statistics.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Confirmed: `PgStat_Kind`/built-in/custom ID ranges and `PGSTAT_KIND_EXPERIMENTAL`, `PgStat_KindInfo` field semantics, `PgStat_HashKey` `(kind, dboid, uint64 objid)`, `pgstat_register_kind` validation (empty name, range, preload-in-progress, fixed shared_size, duplicate id, case-insensitive duplicate name), startup ordering via `StatsShmemSize`/`StatsShmemInit`, `pgstat_initialize` custom snapshot allocation and `init_backend_cb`, `pgstat_prep_pending_entry`/`pgstat_flush_pending_entries`/`pgstat_report_stat` flow, fixed-stat slot layout in `PgStat_ShmemControl.custom_data[]`, fixed reporter changecount + LWLock pattern, reset/snapshot offset compensation, `pgstat_fetch_entry` consistency modes + `shared_data_len` copy, `pgstat_build_snapshot` database filter, `pgstat_snapshot_fixed`, `stats_fetch_consistency` `PGC_USERSET`, `shared_preload_libraries` `PGC_POSTMASTER`, `pgstat_reset`/`pgstat_reset_of_kind`/`pg_stat_reset_shared`/`pg_stat_reset`/`pg_stat_have_stats`/`pgstat_have_entry` behavior, `pgstat_write_statsfile`/`pgstat_read_statsfile`/`pgstat_discard_stats` and TAP test outcomes.
- Corrected the variable-entry drop description: `pgstat_drop_entry` itself does not request reference GC; it returns the freed flag and the caller (e.g. `pgstat_drop_inj`) calls `pgstat_request_entry_refs_gc` on `false`.
- Open Question on the `pgstat_register_kind` doc vs source return-type discrepancy remains correctly scoped.
- Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`. `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-19] answer v18 | custom cumulative statistics

- Filed [How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)](v18/questions/custom-cumulative-statistics.md).
- Traced `PgStat_KindInfo`, `pgstat_register_kind`, custom kind ID ranges, shared-memory allocation, variable pending-entry flushing, fixed custom stats, snapshots, reset/drop paths, clean-shutdown persistence, crash discard, and injection-points tests in pinned `raw/postgres-18/`.
- Noted the implementation-vs-documentation discrepancy for the `pgstat_register_kind` return type under `## Open Questions`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`.
- `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-19] lint-fix v18 | avg_leaf_density verified_by_agent format

- Fixed `verified_by_agent` format in [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](v18/questions/avg-leaf-density-during-vacuum.md) to match the regex `^[a-zA-Z0-9_-]+ \d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z$` (no dots in model name, no pipe separator).
- `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-18] review v18 | avg_leaf_density during (auto)vacuum verification

- Re-checked every behavioral claim in [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](v18/questions/avg-leaf-density-during-vacuum.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Confirmed: `pgstatindex_impl` formula and NaN guard, `PageGetExactFreeSpace = pd_upper - pd_lower`, `_bt_pageinit`/`PageInit` special-space math, `btvacuumscan`/`btvacuumpage` cleanup-lock and reset points, `btbulkdelete`/`btvacuumcleanup`/`_bt_vacuum_needs_cleanup` skip logic, `_bt_set_cleanup_info` early-return + WAL write, `BTMetaPageData`/`xl_btree_metadata`/`_bt_restore_meta` deprecated `btm_last_cleanup_num_heap_tuples` handling, `bt_metap` exposure, `pgstat_report_vacuum` locked-entry + `AmAutoVacuumWorkerProcess` path, `PgStat_StatTabEntry`/`shared_stat_reset_contents`, `pg_stat_all_indexes` columns, `_bt_pagedel`/`_bt_unlink_halfdead_page` non-deletion cases, and parallel-vacuum `sizeof(IndexBulkDeleteResult)` copy with no autovacuum parallel workers.
- No factual corrections needed; design-intent and parity gaps remain correctly scoped under `## Open Questions`.
- Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`. `scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-05-19] research v18 | avg_leaf_density during vacuum cons

- Added a `## Cons Evidence` section to [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](v18/questions/avg-leaf-density-during-vacuum.md).
- Grounded the cons in pinned PostgreSQL 18 source paths for skipped index scans, metapage write/WAL costs, cumulative stats durability, exact `pgstatindex` parity, concurrent split estimate limits, and parallel VACUUM result copying.
- Updated `wiki/index.md` and `wiki/v18/index.md` summaries.

## [2026-05-17] review-fix v18 | avg_leaf_density during (auto)vacuum question

- Reviewed [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](v18/questions/avg-leaf-density-during-vacuum.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Narrowed the zero-extra-I/O claim to actual `btvacuumscan` executions and corrected `_bt_set_cleanup_info` storage cost to account for its early-return path.
- Reworked the metapage recommendation to prefer the deprecated `btm_last_cleanup_num_heap_tuples` `float8` slot over a new `float4`, while calling out WAL redo and `pageinspect` implications.
- Added explicit accuracy caveats for empty leaf pages handled by `_bt_pagedel`, plus stats reset, accessor, parallel VACUUM, and autovacuum path notes.
- Updated `wiki/v18/index.md` and `wiki/index.md` link summaries.

## [2026-05-17] lint-fix v12 | add source references

- Added the missing `## Source References` section to [Foreign-Key Join Optimization for Two-Table Joins (unverified)](v12/questions/fk-join-optimization-two-tables.md) so `scripts/wiki_lint` can validate the page.

## [2026-05-17] answer v18 | compute and store avg_leaf_density during (auto)vacuum

- Filed [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)](v18/questions/avg-leaf-density-during-vacuum.md).
- Designed accumulation in the `btvacuumpage` `P_ISLEAF` branch via new `BTVacState` sums, derived in `btvacuumscan`, mirroring `pgstatindex_impl` semantics with zero extra page I/O.
- Proposed dual storage: B-tree metapage (piggybacks the existing `_bt_set_cleanup_info` WAL-logged write) and the cumulative statistics system (`PgStat_StatTabEntry` via a `pgstat_report_vacuum`-style reporter), plus skip-scan coverage caveats.
- Cited only `raw/postgres-18/` (commit `6cb307251c5c6261286c1566496920976640108e`); filed `verified_by_agent: not yet`, title carries `(unverified)`.
- Linked from `wiki/v18/index.md` and `wiki/index.md`.

## [2026-05-13] review-fix v12 | foreign-key join optimization question

- Clarified that FK metadata directly affects join row-count estimation, while outer-join removal and semijoin reduction are uniqueness-driven.
- Corrected multicolumn FK matching to state that PostgreSQL 12 retains only fully matched FKs.
- Replaced source citations in [Foreign-Key Join Optimization for Two-Table Joins (unverified)](v12/questions/fk-join-optimization-two-tables.md) with the required `[[raw/postgres-12/...#symbol]]` form.
- Removed the resolved `eqjoinsel` open question after checking the `calc_joinrel_size_estimate` path.

## [2026-05-13] answer v12 | foreign-key join optimization for two-table joins

- Filed [Foreign-Key Join Optimization for Two-Table Joins (unverified)](v12/questions/fk-join-optimization-two-tables.md) covering `get_relation_foreign_keys`, `match_foreign_keys_to_quals`, `match_eclasses_to_foreign_key_col`, `calc_joinrel_size_estimate`, and `get_foreign_key_join_selectivity`.
- Cited only `raw/postgres-12/` (commit `45b88269a353ad93744772791feb6d01bc7e1e42`).
- Filed as `verified_by_agent: not yet`; title carries `(unverified)` per `AGENTS.md`.
- Linked from `wiki/v12/index.md` and `wiki/index.md`.

## [2026-05-13] version-add v17 | added PostgreSQL 17 as active version

- Cloned `REL_17_STABLE` into `raw/postgres-17/` pinned to commit `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10, 3 commits past tag `REL_17_10`).
- Created `wiki/v17/index.md` landing page with status `active`.
- Registered the version in `wiki/versions.md` and `wiki/index.md`.

## [2026-05-13] cleanup | reset wiki content logs

- Removed the remaining filed answer page.
- Removed stale index and coverage references for removed content.
- Cleared older activity entries from this log.

## [2026-05-13] cleanup | removed generated navigation references

- Simplified version coverage and landing pages to point only at pinned raw checkouts.
- Removed generated navigation artifact wording from wiki overview and indexes.
